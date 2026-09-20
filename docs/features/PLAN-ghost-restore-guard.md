# 되살리기 유령 차단 (ghost-restore-guard) 구현 계획

스펙: [DRAFT-ghost-restore-guard.md](DRAFT-ghost-restore-guard.md)
기준 커밋(되돌릴 지점): **`addefda`** — `ical-proxy` origin/main, 2026-09-19 17:41

---

## ⚠ 스펙 수정 — 규칙이 두 곳에 따로 구현돼 있다

스펙에는 "`restoreTodayCheckouts()` **한 곳**"이라고 적었으나, 호출 지점을 전수 확인한 결과
**같은 규칙이 두 군데에 복제돼 있다.**

| 경로 | 구현 위치 | 다루는 자료 |
|---|---|---|
| 예약앱 `GET /bookings` | `restoreTodayCheckouts()` (worker.js:690) | 예약 객체 배열 |
| 청소앱 `GET /?url=&fix=tr\|bk&room=` | `fixIcalText()` ③ (worker.js:1030~1040) | **iCal 원문 텍스트** |

한 곳만 고치면 **청소앱에는 유령이 그대로 남는다.** 두 곳 다 고친다.
다만 규칙을 또 복제하지 않기 위해, **판단 함수 하나를 새로 만들어 두 경로가 같이 쓰게 한다**
(05-known-issues #2 — 같은 규칙이 여러 곳에 흩어져 갈라지는 문제).

---

## 작업 단위 (순서대로)

### Task 0: 로컬 worker.js 를 배포본에 맞춘다 (필수 선행)
- 파일: `E:\airbnb\ical-proxy\worker.js`
- **현재 상태가 위험하다.** 로컬 작업본은 커밋 `dcd622b`(2026-08-08) 위에 past-ledger **옛 초안**이
  얹혀 있고, 배포본 대비 **539줄이 빠져 있다.** 빠진 것 중에 이번에 고칠 `restoreTodayCheckouts`,
  `untrimSegs`, `applyUntrim` 이 통째로 들어 있다.
- 또한 로컬본에는 `backfillLedger`(소급 채우기)가 있는데, 이는 2026-09-19 에 **"안 하기로 결정"** 된
  기능이다. 배포본에는 없다 → 로컬본이 버려진 초안임이 확인된다.
- 이대로 푸시하면 **지금 돌아가는 교정 로직이 전부 사라진다.**
- 조치:
  ```
  git stash push -m "past-ledger 옛 초안 (2026-09-21 보류)" worker.js
  git pull --ff-only origin main        # → addefda
  ```
- `git stash` 는 지우는 게 아니라 치워두는 것이다. `git stash list` 로 남아 있고 언제든 꺼낼 수 있다.
- 백업본도 별도 보관: `scratchpad/worker.local-draft.js`
- 예상 코드 라인: 0줄 (git 조작만)

### Task 1: 판단 함수 `shouldRestoreSeg()` 신설
- 파일: `ical-proxy/worker.js` — `restoreTodayCheckouts` 바로 위
- 변경 내용: 되살릴지 말지를 판단하는 함수 하나를 만든다. 두 경로가 이것만 쓴다.
  ```js
  // 아카이브 조각 하나를 되살릴 것인가. 두 읽기 경로(/bookings, /?url=&fix=)가 같이 쓴다.
  //   feedSegs      : 그 채널 피드의 [시작ms, 끝ms] 목록
  //   checkoutShown : 그 방에 '오늘 퇴실'이 이미 표시돼 있는가 (채널 무관)
  function shouldRestoreSeg(as, az, feedSegs, todayMs, checkoutShown) {
    if (az !== todayMs || as >= todayMs) return false;   // 기존 조건 — 오늘 퇴실 + 당일치기·꼬리 제외
    if (checkoutShown) return false;                     // (나) 이미 퇴실이 보이면 꺼낼 이유가 없다
    return !feedSegs.some(([s, e]) => as < e && s < az); // (가) 겹치면 같은 숙박의 옛 버전이다
  }
  ```
- **(가)가 기존 "날짜 완전일치" 검사를 대체한다** — 완전히 같은 조각도 겹치므로 자동 포함된다
- 예상 코드 라인: ~10줄 (주석 포함 ~18줄)

### Task 2: `restoreTodayCheckouts()` 를 판단 함수로 교체 (예약앱 경로)
- 파일: `ical-proxy/worker.js:690`
- 변경 내용: `uid` 기반 완전일치 검사를 버리고 `shouldRestoreSeg` 호출로 바꾼다.
  `checkoutShown` 을 인자로 받는다.
  ```js
  function restoreTodayCheckouts(feed, arch, todayMs, checkoutShown) {
    const segs = (feed || []).map(b => [dayMs(b.cinY,b.cinM,b.cinD), dayMs(b.coutY,b.coutM,b.coutD)]);
    const add = (arch || []).filter(a => shouldRestoreSeg(
      dayMs(a.cinY,a.cinM,a.cinD), dayMs(a.coutY,a.coutM,a.coutD), segs, todayMs, checkoutShown));
    return add.length ? [...(feed || []), ...add] : feed;
  }
  ```
- 예상 코드 라인: ~8줄 교체

### Task 3: `applyUntrim()` 에 방 단위 (나) 게이트 추가
- 파일: `ical-proxy/worker.js:701`
- 변경 내용: 방마다 "오늘 퇴실이 이미 있는가"를 네 채널에서 먼저 구하고,
  채널을 돌면서 **되살릴 때마다 갱신**한다. 갱신하지 않으면 여러 채널이 각자 하나씩 되살려
  다시 겹친다.
  ```js
  let checkoutShown = ['ab','bk','tr','lv'].some(k =>
    ((data && data[k]) || []).some(b => dayMs(b.coutY,b.coutM,b.coutD) === todayMs));
  for (const k of ['ab','bk','tr','lv']) {
    ...
    const restored = restoreTodayCheckouts(fixed, archK, todayMs, checkoutShown);
    if (restored !== fixed) { fixed = restored; checkoutShown = true; }
    ...
  }
  ```
- **채널 순서 `ab → bk → tr → lv` 는 그대로 둔다.** 에어비앤비가 유일하게 '예약'(`Reserved`)을
  보내는 채널이므로 먼저 보는 게 맞다. 나머지는 '재고 차단'이다.
- 예상 코드 라인: ~6줄 추가/수정

### Task 4: `fixIcalText()` ③ 에 (가)·(나) 적용 (청소앱 경로)
- 파일: `ical-proxy/worker.js:1030~1040`
- 변경 내용:
  1. `seen` 이 지금 `"시작_끝"` 문자열 집합이라 **겹침 판정을 못 한다.** `[시작, 끝]` 배열도 같이 모은다
  2. (나) 판단 재료로 그 방의 네 채널을 읽는다 — `synced_bookings[roomName]` (**읽기만**, KV write 없음)
  3. 되살리기 루프를 `shouldRestoreSeg` 로 교체
  ```js
  let other = null;
  try { other = JSON.parse(await env.HANA_KV.get('synced_bookings') || '{}')[roomName]; } catch (e) {}
  const checkoutShown =
    ['ab','bk','tr','lv'].some(p => ((other && other[p]) || [])
      .some(b => dayMs(b.coutY,b.coutM,b.coutD) === todayMs)) ||
    seenSegs.some(([, z]) => z === todayMs);     // 이 피드에서 이미 나온 퇴실도 센다
  ```
- **KV read 1회 추가된다** (`synced_bookings`). write 는 없다. 한도 원칙 위반 아님
- 예상 코드 라인: ~10줄 수정

### Task 5: 주석·근거 갱신
- 파일: `ical-proxy/worker.js` — `restoreTodayCheckouts` 위 주석 블록
- 변경 내용: 왜 두 조건이 생겼는지, 601호·203호 실측값, 사용자 결정(한 방에 한 팀)을 적는다.
  이 프로젝트 주석 규칙대로 **"왜"** 를 남긴다
- 예상 코드 라인: ~10줄

### Task 6: 배포 전 검증 (배포 없이 실데이터로)
- **node 가 설치돼 있지 않다.** 브라우저에서 돌린다 (CLAUDE.md 배포 전 검증 요령과 같은 방식)
- 절차:
  1. 예약앱 배포본을 브라우저로 연다
  2. `/archive` + 각 호실 채널 원본 피드(`/?url=`)를 실제로 받는다 — 이것이 워커가 보는 입력과 같다
  3. **옛 구현과 새 구현을 같은 입력으로 나란히 돌려** 출력을 대조한다
  4. 전 호실 × 4채널 조각을 덤프해 **줄어든 것만 유령인지** 확인
- 예상 코드 라인: 0줄 (검증 스크립트는 scratchpad)

### Task 7: 배포 및 배포 후 확인
- `git add worker.js && git commit && git push` → Cloudflare 자동 배포
- 배포 후 `/bookings` 를 다시 받아 **배포 전 스냅샷과 대조**
- 예약앱·청소앱 화면에서 601호·203호 육안 확인

---

## 변경 파일 목록
- [x] `ical-proxy/worker.js` — 되살리기 판단에 제외 조건 2개 추가 (두 경로 공용 함수로) — commit `4e8faaa`
- [ ] `docs/features/DRAFT-ghost-restore-guard.md` — "한 곳" → "두 곳" 정정 (`/done` 단계)
- [ ] `docs/05-known-issues.md` — 항목 추가 (`/done` 단계)

**수정하지 않는 것**
- 예약앱 `index.html` — `/bookings` 응답만 받으므로 워커 수정으로 반영된다
- 청소앱 `index.html` — `/?url=&fix=` 응답만 받으므로 워커 수정으로 반영된다
- `exportIcal` — `synced_bookings` 원본만 읽어 전 채널 무영향
- 아카이브·`synced_bookings` 저장 경로 — 읽기 전용 작업이다

---

## 롤백 방법

1. **즉시 되돌리기**
   ```
   git revert HEAD && git push        # → addefda 상태로 복귀, 자동 재배포
   ```
2. **Task 0 의 stash 복구** (past-ledger 옛 초안이 필요해지면)
   ```
   git stash list
   git stash pop
   ```
3. 데이터 변경이 없으므로 **복구할 데이터가 없다.** 코드만 되돌리면 끝난다

---

## 테스트 시나리오

**A. 이번에 고치려는 것** — 배포 전 실데이터 검증 완료 (2026-09-21)
- [x] 601호 `ab` — 아카이브 `9/18~9/21` 주입 안 됨. 진짜 `9/18~9/20` 살아남음
- [x] 203호 `ab` — 아카이브 `9/16~9/21` 주입 안 됨. 진짜 `9/19~9/21` 살아남음
- [x] 2026-09-20 재현 — `bk 9/17~9/20`·`tr 9/9~9/20` 둘 다 생략됨 (되살림 2 → 0)

**B. 망가뜨리면 안 되는 것 (#29 목적 유지)**
- [x] 퇴실이 **진짜로** 사라진 경우는 여전히 되살아난다 — 603호 `bk 9/18~9/21` 실제 유지, 합성 시험 ③ 통과
- [x] 앞날 조각 신규 **0건** — 전 호실 × 4채널 전수. 판매 가능일 불변
- [x] `untrimSegs` 미변경 — 함수 자체를 건드리지 않았고 호출 순서도 그대로
- [x] 되살린 조각 수 추이 `9/18: 1→1` (변화 없음) / `9/20: 2→0` / `9/21: 3→1` / `9/22: 1→0`

**C. 경로별** — 배포 후 확인
- [ ] `GET /bookings` — 예약앱
- [ ] `GET /?url=&fix=bk&room=603 jnj` — 청소앱 부킹 (되살림 1건 유지되어야 함)
- [ ] `GET /?url=` (fix 없음) — **원본 바이트 그대로여야 한다** (#28 판별 순서 1번 보호)
- [ ] `GET /ical/<9개 호실>` 및 `/ical/<호실>/ab` — **변화 0바이트** (배포 전 해시 `scratchpad/pre/HASHES.txt`)

> ⚠ 청소앱 경로(`fix=`)는 **오늘 실데이터로는 변화가 드러나지 않는다.**
> 오늘 유령 2건이 모두 `ab` 채널인데, 청소앱 교정은 `bk`·`tr` 에만 적용되기 때문이다.
> 대신 어제 601호 상황을 그대로 재현한 합성 시험 6건으로 판정 로직을 확인했다.

---

## 예상 주의사항

- **#29 의 목적을 좁히는 것이지 없애는 게 아니다.** 진짜로 사라진 퇴실은 계속 되살린다
- **#16 원칙 유지** — 앞날 것은 절대 안 꺼낸다. 조건을 건드리지 않는다
- **#2 규칙 분산 주의** — 그래서 판단 함수를 하나로 만든다. 나중에 고칠 때 한 곳만 보면 된다
- **`/ical/` 은 이 경로를 안 거친다** — 전 채널 영향 없음. 그래도 테스트 C 에서 바이트 대조한다
- **KV write 없음** — 읽기 경로 전용. 한도 원칙 무관 (청소앱 경로에 read 1회만 추가)
- **past-ledger ⑦ 과 겹친다** — 원장 전환 시 이 함수들은 제거 대상이다.
  지금 넣는 조건도 그때 같이 사라진다. 원장이 찰 때까지 남은 63건을 막는 임시 조치다
- **`untrimSegs` 는 범위 밖.** 조각을 추가하지 않고 시작일만 앞으로 옮기므로 이번 증상과 무관하다.
  다만 같은 채널 안에서 겹침을 만들 여지가 있어 **별건으로 확인이 필요하다**
- `render()` / `attachCellClicks()` — 앱 코드를 안 건드리므로 해당 없음

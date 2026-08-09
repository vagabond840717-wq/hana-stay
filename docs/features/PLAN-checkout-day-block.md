# 체크아웃 날 수동 블락 (checkout-day-block) 구현 계획

스펙: [DRAFT-checkout-day-block.md](DRAFT-checkout-day-block.md)

## 확정된 결정 사항 (2026-08-05 사용자 승인)
1. 청소앱도 예약앱과 **같은 반반 셀**로 맞춘다 (옵션 나)
2. 체크인이 있는 날은 **블락 불가 유지** — 필요해지면 그때 별도 논의
3. 블락 기본 기간 = 그날 하루(1박), 현행 유지
4. 반반 셀 오른쪽 글자를 `인` → **`블락`** 으로 바꾼다 (양쪽 앱 동일 적용, Task 3·6)
5. **기준 앱은 예약앱** — 겹치는 로직은 예약앱을 먼저 고치고 청소앱이 따라간다 (CLAUDE.md에 원칙 명시)
6. 이번 맞춤 범위는 **"아웃 + 블락" 케이스만**. 예약앱의 반반 셀 규칙표 전체 이식은 하지 않는다 (기존 조합 색이 바뀌므로) — 필요해지면 별도 작업

---

## 작업 단위 (순서대로)

### Task 1: 예약앱 — 블락 폼 표시 조건을 "밤 기준"으로 변경
- **파일**: `app/JnJ booking/index.html` (`openDetail()`, L1811 부근)
- **변경 내용**:
  ```js
  // 추가 (allBks 계산 직후 ~L1767 또는 blockHtml 직전)
  // 그날 '밤'을 실제로 차지하는 예약이 있는지 — 체크아웃만 있는 날은 밤이 빔
  const nightTaken = allBks.some(b => !(b.coutY===y && b.coutM===m && b.coutD===d));

  // 수정
  } else if (!allBks.length) {     // ← 기존
  } else if (!nightTaken) {        // ← 변경
  ```
- `allBks`에는 4개 플랫폼 + `abBlocks`가 모두 합쳐져 있으므로 별도 수집 불필요
- 오버부킹(체크아웃 + 다른 예약 숙박중) → `nightTaken=true` → 자동으로 숨김 유지
- **예상 코드 라인**: +2줄 / 수정 1줄

### Task 2: 청소앱 — `blockTypeFor()`에 `'start'` 반환 추가
- **파일**: `app/JnJ/index.html` L966~975
- **변경 내용**: 예약앱과 동일하게 블락 첫날은 `'start'`를 반환
  ```js
  if (cur >= s && cur <= e) return cur.getTime() === s.getTime() ? 'start' : 'block';
  ```
- **기존 영향 확인 완료**: 청소앱에서 `blockTypeFor` 결과는 `blE = (bl==='empty')` 로만 쓰인다(L1033, L1041). `'start'`도 `'block'`도 `blE=false`라 **기존 표시 변화 없음**
- **예상 코드 라인**: 수정 1줄

### Task 3: 청소앱 — `bothLbl()` 아이콘 2개 + 오른쪽 글자 교체 지원
- **파일**: `app/JnJ/index.html` L1022~1025
- **변경 내용**: 예약앱과 동일 시그니처 + 세 번째 인자(오른쪽 글자) 추가
  ```js
  function bothLbl(outIcon, inIcon, inTxt) {
    let ic = '';
    if (outIcon && inIcon && outIcon !== inIcon)
      ic = `<span style="font-size:7px;line-height:1;letter-spacing:-1px;">${outIcon}${inIcon}</span>`;
    else if (outIcon || inIcon)
      ic = `<span style="font-size:8px;line-height:1;">${outIcon || inIcon}</span>`;
    return `<span style="display:flex;...">아웃 ${ic} ${inTxt || '인'}</span>`;  // 기존 마크업 그대로
  }
  ```
- **기존 영향 확인 완료**: 현재 호출은 `bothLbl('✈')`, `bothLbl('🏨')`, `bothLbl('🌐')`, `bothLbl('🏡')`, `bothLbl()` 5종뿐. 세 번째 인자가 없으면 `'인'`이 그대로 들어가 결과 HTML은 기존과 **완전히 동일**
- **예상 코드 라인**: 수정 5줄

### Task 4: 청소앱 — `.c-mix-out-bl` CSS 추가
- **파일**: `app/JnJ/index.html` L229 (`.c-bl-block` 다음 줄)
- **변경 내용**: 다크 테마 팔레트에 맞춘 좌(체크아웃 앰버) / 우(블락 회색) 반반
  ```css
  .c-mix-out-bl { background: linear-gradient(90deg, rgba(240,179,91,0.22) 50%, rgba(61,68,82,0.45) 50%); color: var(--text2); }
  ```
- 색값은 기존 `.c-ab-blk-checkout`(체크아웃 앰버)과 `.c-bl-block`(수동 블락 회색)에서 그대로 가져옴 → 새 색 도입 없음
- **예상 코드 라인**: +1줄

### Task 5: 청소앱 — "체크아웃 + 블락 시작" 반반 셀 분기 추가
- **파일**: `app/JnJ/index.html`, `cellClsAndLbl()` 안 — `const parts = []` 폴백(L1082) **바로 앞**에 삽입
- **변경 내용**:
  ```js
  // 체크아웃 + 수동 블락 시작 → 반반 셀 (예약앱 c-mix-out-bl과 동일 규칙)
  if (bl === 'start' && abBlkE) {
    const ics = {ab:'✈', bk:'🏨', tr:'🌐', lv:'🏡'};
    const types = {ab, bk, tr, lv};
    const outs = [];
    let other = false;
    for (const k of ['ab','bk','tr','lv']) {
      if (types[k] === 'checkout') outs.push(ics[k]);
      else if (types[k] !== 'empty') other = true;
    }
    if (outs.length === 1 && !other) return {cls:'c-mix-out-bl', lbl: bothLbl(outs[0], '🔒', '블락')};
  }
  ```
- **범위를 좁게 잡은 이유**: 예약앱의 mix 블록을 통째로 이식하면 "에어비앤비 아웃 + 부킹닷컴 인" 같은 **기존 조합의 색까지 바뀐다**. 이번 요청과 무관하므로 건드리지 않고, 딱 이 케이스(플랫폼 체크아웃 1개 + 블락 시작 + 그 외 없음)만 잡는다
- 조건 미충족 시 기존처럼 `c-dual` 폴백으로 내려감 → 안전
- **예상 코드 라인**: +11줄

### Task 6: 예약앱 — 반반 셀 오른쪽 글자 `인` → `블락`
- **파일**: `app/JnJ booking/index.html` — `bothLbl()` (L1309~1314), mix 분기 (L1350~1353)
- **변경 내용**: 청소앱과 동일하게 세 번째 인자 추가 후, 수동 블락(🔒)일 때만 `'블락'` 전달
  ```js
  function bothLbl(outIcon, inIcon, inTxt) { ... ${inTxt || '인'} ... }   // 인자 추가

  // mix 분기
  const cls = ins[0]==='🔒' ? 'c-mix-out-bl' : ins[0]==='🚫' ? 'c-mix-out-abblk' : 'c-mix-both';
  return {cls, lbl: bothLbl(outs[0], ins[0], ins[0]==='🔒' ? '블락' : '')};
  ```
- **범위**: 수동 블락(🔒)만. **에어비앤비 취소잔여 블락(🚫)은 기존대로 `인` 유지** — 이번 요청 대상이 아니고 기존 표시를 바꾸지 않기 위함. 원하면 나중에 조건 하나만 추가하면 됨
- 나머지 `bothLbl` 호출(아웃+인 조합들)은 세 번째 인자가 없어 `'인'` 그대로 → 회귀 없음
- **글자 길이**: `'블락'`은 `'아웃'`과 같은 2글자라 셀 폭 안에 들어감 (실기기 확인 필요)
- **예상 코드 라인**: 수정 3줄

### Task 7: 문서 갱신 (구현 검수 후 `/done` 단계)
- `docs/05-known-issues.md` 9번에 "블락 가능 판정도 밤 기준" 한 줄 추가
- `docs/04-apps-spec.md` 예약앱 수동 블락 설명 + 청소앱 표시 반영
- `DRAFT-checkout-day-block.md` → `checkout-day-block.md` 확정
- `STATUS.md` 갱신

---

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — 블락 폼 표시 조건 (Task 1) + 오른쪽 글자 `블락` (Task 6)
- [x] `app/JnJ/index.html` — 블락 시작일 구분 + 반반 셀 표시 + 오른쪽 글자 `블락` (Task 2~5)
- [ ] `docs/05-known-issues.md` — 판정 기준 명시 (Task 7, `/done`에서)
- [ ] `docs/04-apps-spec.md` — 앱 스펙 반영 (Task 7, `/done`에서)

## 구현 중 발견 — ab-block은 cout **포함**(inclusive)
`cellTypeForBlock()`(예약앱 L1248)은 일반 예약과 달리 **`cout`까지 막힌 밤**으로 보고, 체크아웃은 `cout+1`로 계산한다.
그래서 "cout이 그날이면 밤이 빈다"는 판정을 ab-block에 그대로 적용하면 **에어비앤비 취소잔여 블락(🚫)이 걸린 날에 블락 폼이 잘못 뜬다**.
- 최초 구현에서 실제로 이 오류가 났고(202 Hana 8/2, 402 jnj 8/6 — 둘 다 1박짜리 🚫 블락), 브라우저 검증에서 잡아 수정함
- 최종 조건: `allBks.some(b => b.platform==='ab-block' || !(b.cout === 그날))`
- `docs/05-known-issues.md`에 이 동작 차이를 기록할 것 (Task 7)

**변경 없음**: 워커(`ical-proxy`), `worker.js`, KV 스키마, `manual_blocks` 데이터 구조, 요금 계산기 앱

---

## 롤백 방법
두 앱은 각자 별도 git 저장소다. 문제가 생기면 해당 저장소에서 되돌린다.

```bash
git -C "app/JnJ booking" revert HEAD
```

```bash
git -C "app/JnJ" revert HEAD
```

- 저장된 블락 데이터(`manual_blocks`)는 구조가 그대로라 **되돌려도 그대로 살아있다**. 데이터 손실 없음
- 배포는 GitHub Pages 자동 → revert push 후 자동 반영

---

## 테스트 시나리오

### 예약앱 (Task 1)
- [ ] 체크아웃만 있는 날 셀 클릭 → 🔒 수동 블락 영역이 **보인다**, 시작/종료일 기본값 = 그날
- [ ] 블락 저장 → 셀이 `아웃 ✈🔒 블락` 반반으로 바뀐다, '아웃' 글자 유지
- [ ] 아웃 + 다른 플랫폼 체크인 조합 셀 → 오른쪽 글자가 여전히 `인` (회귀 없음)
- [ ] 아웃 + 에어비앤비 취소잔여 블락(🚫) 셀 → 오른쪽 글자 `인` 유지 (의도된 범위)
- [ ] 같은 셀 재클릭 → 수정/해제 폼이 뜬다, 해제하면 원래 체크아웃 셀로 복귀
- [ ] 예약 없는 빈 날 → 기존과 동일하게 블락 폼 표시 (회귀 없음)
- [ ] 체크인 있는 날 / 아웃+인 날 / 숙박 중인 날 → 블락 폼 **안 보임** (기존과 동일)
- [ ] 오버부킹 날(⚠️) → 블락 폼 **안 보임**
- [ ] Trip.com 분할 조각의 체크아웃 날 → 블락 폼 보임, 분할 UI와 겹치지 않음
- [ ] 새로고침 후에도 블락 유지 (KV 저장 확인) / 다른 기기에서도 보임

### 청소앱 (Task 2~5)
- [ ] 예약앱에서 만든 "체크아웃 + 블락" 날이 `아웃 ✈🔒 블락` 반반 셀로 보인다
- [ ] '아웃' 글자가 남아 **청소 나가야 함이 그대로 읽힌다**
- [ ] 순수 블락 날(예약 없음) → 기존처럼 🔒 회색 셀 (회귀 없음)
- [ ] 여러 날짜 블락 시 첫날만 반반, 나머지 날은 기존 회색 블락 셀
- [ ] Airbnb 취소잔여 블락(🚫) 있는 날 → 기존 표시 그대로 (회귀 없음)
- [ ] 아웃 + 다른 플랫폼 인 같은 기존 조합 셀 → **색이 안 바뀌었다** (회귀 없음)

### 공통
- [ ] 6개월 연속 뷰 / 월간 뷰 / 주간 뷰 모두 정상
- [ ] 폰(모바일)에서 반반 셀 글씨가 깨지지 않음 — 특히 `블락` 2글자가 잘리지 않는지

---

## 예상 주의사항
- **known-issues #2 (두 앱 로직 중복)**: `blockTypeFor`가 양쪽에 따로 있다. 이번에 청소앱 쪽을 예약앱과 동일하게 맞추므로 오히려 불일치가 줄어든다
- **known-issues #9 (체크아웃 날은 숙박 아님)**: 이번 변경이 정확히 이 규칙을 블락 UI에도 적용하는 것
- **`render()` → `attachCellClicks()` 체인**: 블락 저장 핸들러(L1949)에 이미 `saveBlocks(); render(); attachCellClicks(); openDetail(...)` 가 들어 있어 **추가 작업 불필요**
- **월 0-indexed**: 새로 날짜를 만드는 코드가 없다(기존 `dstr(m+1)` 폼 그대로). 위험 없음
- **KV 저장/로드**: 기존 `saveBlocks()` 경로 그대로. write 횟수 증가 없음(사용자가 블락할 때만 1회) → 무료 한도 영향 없음

### 남겨두는 것
에어비앤비 취소잔여 블락(🚫)이 낀 반반 셀은 오른쪽이 그대로 `'인'`이다. 성격상 이것도 블락이지만 이번 요청 범위 밖이고, 손대면 기존 표시가 바뀐다. 필요하면 `ins[0]==='🚫'` 조건 하나 추가로 끝난다.

# PLAN — 경계(tr_cuts) 동기화 불일치 + 피드 갱신 무시 버그 수정

작성일: 2026-08-06
발단: 402호 8/6 트립 예약을 거절했는데 예약앱에서 사라지지 않음 → 조사 중 3건 발견

---

## 배경 — 무슨 일이 있었나

1. 8/6 402호는 레이트 체크아웃(19시 퇴실)이라 청소 불가 → 에어비앤비 달력에서 블락
2. 트립닷컴에 반영되기 전 8/6~8/8 예약 문의가 들어옴 → 거절
3. 거절 후에도 예약앱에서 트립 8/6~8/7이 사라지지 않음

조사 결과 **트립 쪽은 정상**이었다. 트립은 문의 단계에서 날짜를 잡아 iCal로 내보내고,
거절 후 푸는 데 30분 이상 걸리며 날짜별로 순차 해제된다(08:57 8/7 해제 → 09:32 8/6 해제).
KV `synced_bookings`도 트립 원본과 완전 일치했다. **앱 화면만 옛 데이터를 붙들고 있었다.**

---

## 확정된 사실 (실측)

| 확인 | 결과 |
|---|---|
| 트립 원본 iCal (`/?url=`) | 8/6 블락 없음 |
| KV `synced_bookings` 402 tr | `8/8~8/12`부터 — 8/6 없음 |
| 새 브라우저로 예약앱 로드 | 8/6 = `c-mix-out-abblk` (트립 없음) ✅ |
| 사장님 브라우저 (02:45부터 열어둠) | 8/6에 트립 1박 잔존 ❌ |
| `booking` 저장소 main | 배포본과 바이트 단위 일치 (146,879) |
| `jnjhana` 저장소 main | 배포본과 바이트 단위 일치 (94,511) |
| `E:\airbnb\app\JnJ booking\index.html` | **구버전** (155,949, `tr_splits` 시대) — 배포본 아님 |

---

## 발견된 문제 3건

### A. 청소앱이 경계를 못 읽는다 🔴 (가장 급함)

예약앱은 분할 저장 구조를 `tr_splits`(기간+경계) → `tr_cuts`(경계 날짜만)로 교체했으나,
청소앱은 여전히 `tr_splits`를 읽는다. 해당 KV 키는 이제 비어 있어(`[]`) **모든 경계가 무시된다.**

실측 (402호, 사용자가 8/10에 경계 지정):

| 날짜 | 예약앱 | 청소앱 |
|---|---|---|
| 8/8 | `c-tr-checkin` 🌐 인 | `c-tr-checkin` 🌐 인 |
| 8/9 | `c-tr-occupied` | `c-tr-occupied` |
| **8/10** | **`c-tr-both` 아웃 🌐 인** | **`c-tr-occupied`** ❌ |
| 8/11 | `c-tr-occupied` | `c-tr-occupied` |
| 8/12 | `c-tr-checkout` | `c-tr-checkout` |

**영향: 8/10 청소가 일정에 안 잡힌다.** 앞으로 경계를 지정할 때마다 동일하게 누락된다.

### B. 예약앱이 새 피드를 옛 피드로 되돌린다 🟠

`applyCuts(room)`는 자르기 전 원본을 `room['_raw_'+key]`에 보관한다(경계 편집 시 즉시 재계산용).
그런데 `mergeArchiveIntoRooms()`가 **첫머리에서 무조건 이 백업을 복원**한다.

```
triggerSync()
  ├ loadBookingsFromKV()      → room.trBookings = 새 피드 (8/6 없음)  ✅
  └ mergeArchiveIntoRooms()
      └ if (r['_raw_tr…']) r.trBookings = r['_raw_tr…']   ← 옛 피드로 되돌림 ❌
      └ … applyCutsAll() → 되돌린 배열을 다시 _raw_ 로 백업 (영구 고착)
```

`_raw_`는 메모리에만 있으므로 **페이지를 새로 열면 해소되고, ↻ 버튼으로는 영원히 해소되지 않는다.**
사장님 화면이 02:45 시점 피드에 고정돼 있던 이유.

부수 피해: 피드 변경 감지(`detectCutAlerts` / ⚠ 배지)도 옛 배열을 기준으로 돌아 신뢰할 수 없다.

이 복원 코드는 [05-known-issues.md](../05-known-issues.md) #19 수정 때 넣은 것으로,
"경계 편집 → 재계산" 경로만 고려하고 "새 피드 수신" 경로를 놓쳤다.

### C. 문서·로컬 사본이 구버전 🟡

- `CLAUDE.md`, `docs/03-data-model.md`가 `tr_splits` 기준으로 기술됨
- `E:\airbnb\app/` 의 앱 사본이 배포본과 다른 구버전 → 이번 조사에서 실제로 오진의 원인이 됨

---

## 작업 내용

### Task 1 — 청소앱 경계 읽기 교체 (`jnjhana/index.html`)

**원칙: 청소앱은 읽기 전용.** 경계의 주인은 예약앱이며, 청소앱은 절대 `saveCuts` 하지 않는다.
구버전 마이그레이션(`migrateSplitsToCuts`)도 예약앱만 수행한다.

교체 대상: L937~972 (`SPLIT_PLATFORMS` ~ `applySplitsAll`)

```js
// ── CUTS (경계 메모: "이 호실 · 이 날짜는 손님이 바뀌는 날") — 읽기 전용 ──
// 경계의 주인은 예약앱. 이 앱은 반영만 하고 절대 저장하지 않는다.
const SPLIT_PLATFORMS = ['tr'];
const SPLIT_KEYMAP = { ab:'bookings', bk:'bkBookings', tr:'trBookings', lv:'lvBookings' };
let trCuts = [];
function dnum(p){ return p.y*10000+(p.m+1)*100+p.d; }
async function loadCuts(){
  try{
    const resp = await fetch(`${PROXY_URL}/extra?key=tr_cuts`);
    if(resp.ok){ const d = await resp.json(); if(Array.isArray(d)){ trCuts = d; return; } }
  }catch(e){}
  try{ const d = JSON.parse(localStorage.getItem('hana_cuts')||'[]'); trCuts = Array.isArray(d)?d:[]; }
  catch(e){ trCuts = []; }
}
function cutsFor(roomName,plat){ return trCuts.filter(c=>c.roomName===roomName&&(c.platform||'tr')===plat); }
function makeSeg(a,z,orig){
  return {cinY:a.y,cinM:a.m,cinD:a.d,coutY:z.y,coutM:z.m,coutD:z.d,
          platform:orig.platform||'trip',summary:orig.summary||'',_split:true};
}
// 경계가 예약 한가운데 있을 때만 자른다 (입실일·퇴실일과 같으면 이미 나뉜 상태)
function applyCuts(room){
  for(const plat of SPLIT_PLATFORMS){
    const key = SPLIT_KEYMAP[plat];
    const arr = room[key];
    if(!Array.isArray(arr)) continue;
    const mine = cutsFor(room.name,plat);
    if(!mine.length) continue;
    const out = [];
    for(const b of arr){
      if(b._split){ out.push(b); continue; }
      const bs = dnum({y:b.cinY,m:b.cinM,d:b.cinD}), be = dnum({y:b.coutY,m:b.coutM,d:b.coutD});
      const inside = mine.filter(c=>dnum(c)>bs&&dnum(c)<be).sort((x,z)=>dnum(x)-dnum(z));
      if(!inside.length){ out.push(b); continue; }
      const pts = [{y:b.cinY,m:b.cinM,d:b.cinD}, ...inside.map(c=>({y:c.y,m:c.m,d:c.d})), {y:b.coutY,m:b.coutM,d:b.coutD}];
      for(let i=0;i<pts.length-1;i++) out.push(makeSeg(pts[i],pts[i+1],b));
    }
    room[key] = out;
  }
}
function applyCutsAll(){ for(const r of rooms) applyCuts(r); }
```

동반 수정:
- L920 `applySplitsAll();` → `applyCutsAll();`  (`syncAll()` 내부)
- L2059 `loadSplits()` → `loadCuts()`  (`init()` 내부)
- `splitOrigEq()` 삭제 (더 이상 참조 없음)
- `_raw_` 백업은 넣지 않는다 — 청소앱에는 경계 편집·재계산 경로가 없고, 매 `syncAll()`이 배열을 새로 채운다

**예약앱과의 차이 (의도된 것)**: 예약앱 `applyCuts`에 있는 `room['_raw_'+key]=arr` 한 줄만 없다.
나머지 판정 로직은 동일하게 유지한다 (기준 앱 = 예약앱 원칙).

### Task 2 — 예약앱 `_raw_` 폐기 (`booking/index.html`)

`loadBookingsFromKV()` 내부, `room._feed_trBookings=room.trBookings.slice();` 바로 뒤:

```js
// 서버에서 새 피드를 받았으므로 직전 사이클의 원본 백업은 폐기한다.
// (남겨두면 mergeArchiveIntoRooms 첫머리의 복원이 새 피드를 옛 피드로 되돌린다)
for(const plat of SPLIT_PLATFORMS) delete room['_raw_'+SPLIT_KEYMAP[plat]];
```

수정 후 흐름:

```
loadBookingsFromKV     → 새 피드 대입 + _raw_ 폐기
mergeArchiveIntoRooms  → 복원할 _raw_ 없음 → 새 피드에 아카이브 병합
                       → applyCutsAll() 이 _raw_ 를 (새 피드+아카이브)로 새로 채움
reapplyCuts (경계 편집) → 방금 채운 최신 _raw_ 복원 → 정상 동작 유지
```

`mergeArchiveIntoRooms()`의 복원 코드는 **건드리지 않는다.** init에서 2회 호출될 때
(#19) 조각 배열이 백업을 오염시키는 걸 막는 역할이 여전히 필요하다.

### Task 3 — 문서 갱신 (`hana-stay`)

- `CLAUDE.md`: `tr_splits` 절 → `tr_cuts` 구조로 교체, 청소앱 열 갱신
- `docs/03-data-model.md`: 동일
- `docs/04-apps-spec.md`: 청소앱 경계 반영 방식
- `docs/05-known-issues.md`: 신규 3건 추가
  - 트립 문의 단계 홀드 + 순차 해제 지연 (알고 써야 할 동작)
  - `_raw_` 복원이 새 피드를 덮는 버그 (해결됨)
  - 청소앱 경계 키 불일치 (해결됨)
- `E:\airbnb\app/` 구버전 사본 처리 — 삭제 권장 (오진 원인). 사장님 결정 필요

---

## 검증 계획

브라우저 도구로 배포본을 직접 열어 DOM을 읽어 대조한다 (화면 캡처 아닌 클래스명 확인).

### Task 1 검증 — 청소앱

| 확인 | 기대값 |
|---|---|
| 402호 8/10 | `c-tr-both` (현재 `c-tr-occupied`) |
| 402호 8/8 | `c-tr-checkin` (변화 없음) |
| 402호 8/12 | `c-tr-checkout` (변화 없음) |
| 601·603호 등 경계 없는 호실 | 수정 전과 동일 |
| KV `tr_cuts` | 청소앱 로드 후에도 **내용 불변** (쓰기 안 함 확인) |

### Task 2 검증 — 예약앱

| 확인 | 방법 |
|---|---|
| 경계 유지 | 402호 8/8·8/10 조각이 그대로 |
| 경계 편집 즉시 반영 | 임의 경계 추가/삭제 → 화면 즉시 변화 → 되돌리기 |
| 새 피드 반영 | 콘솔에서 `_raw_trBookings`에 가짜 항목을 심고 ↻ → 사라져야 함 |
| ⚠ 배지 | 잘못된 경고가 새로 생기지 않는지 |
| 아카이브 과거 | 7월 이전 달 표시가 수정 전과 동일 |

### 공통

- 두 앱 모두 오늘(8/6) 402호가 트립 없이 `아웃 + 블락`으로 보이는지
- 오버부킹 오탐(예약앱)이 늘지 않는지

---

## 순서 / 배포

1. Task 1 (청소앱) — 청소 일정이 지금 틀려 있어 최우선
2. Task 2 (예약앱)
3. 각각 검증 통과 후 커밋·푸시 (푸시 = 자동 배포이므로 검증 전 푸시 금지)
4. Task 3 문서

저장소가 3개(`jnjhana`, `booking`, `hana-stay`)로 갈리므로 커밋도 3개로 나눈다.

---

## 위험 / 주의

- **푸시 즉시 배포된다** (GitHub Pages / Cloudflare Pages). 검증을 배포 전에 끝낼 것
- 청소앱 경계 로직을 예약앱과 다르게 만들지 않는다 — 기준 앱은 예약앱
- 경계 데이터(`tr_cuts`)는 지금 402호 2건뿐이다. 작업 전 값을 기록해두고 작업 후 대조한다
  ```
  [{"roomName":"402 jnj","platform":"tr","y":2026,"m":7,"d":10},
   {"roomName":"402 jnj","platform":"tr","y":2026,"m":7,"d":8}]
  ```
- 워커(`ical-proxy`)는 이번 작업에서 **건드리지 않는다**

# 데이터 모델

## Room (호실)

### 저장 형태 (KV / localStorage)
```js
// localStorage key: 'hana_rooms'
// KV key: 'rooms' (Worker 내부)
[
  {
    name:  "302호",
    url:   "webcal://...",   // Airbnb iCal URL (없으면 '')
    bkUrl: "webcal://...",   // Booking.com iCal URL (없으면 '')
    trUrl: "webcal://...",   // Trip.com iCal URL (없으면 '')
    lvUrl: "webcal://...",   // 리브애니웨어 iCal URL (없으면 '')
    color: "#c8f07c"         // COLORS 배열에서 인덱스로 결정
  },
  ...
]
```

### 런타임 형태 (메모리)
저장 형태에 예약 배열 추가:
```js
{
  ...저장 형태,
  bookings:   [Booking, ...],  // Airbnb
  bkBookings: [Booking, ...],  // Booking.com
  trBookings: [Booking, ...],  // Trip.com
  lvBookings: [Booking, ...]   // 리브애니웨어
}
```
> 예약 배열은 **저장하지 않음** — 매 syncAll() 마다 iCal에서 새로 파싱

### 제약 조건
- 최대 9개 (`rooms.length >= 9` 체크)
- `name` 필수, 최대 20자
- 색상: `COLORS[rooms.length % 9]` — 추가 순서로 결정, 이후 변경 불가

### 기본 호실 (KV/localStorage 모두 없을 때)
```js
['302호','402호','501호','503호','601호','603호','701호','702호','703호']
// 모두 url/bkUrl/trUrl/lvUrl = '' (미연동 상태)
```

---

## Booking (예약)

```js
{
  cinY:  2025,  // 체크인 연도
  cinM:  5,     // 체크인 월 (0-indexed! 5 = 6월)
  cinD:  10,    // 체크인 일
  coutY: 2025,  // 체크아웃 연도
  coutM: 5,     // 체크아웃 월 (0-indexed)
  coutD: 13,    // 체크아웃 일
  src:   'booking' | 'trip' | 'lv'  // Airbnb는 src 없음
}
```

> **주의**: `cinM`, `coutM` 은 **0-indexed** (JavaScript Date와 동일).  
> 저장 시에도 0-indexed 그대로 저장됨.

### 날짜 범위 해석
- **숙박 기간**: `cin 이상 cout 미만` (cin <= day < cout)
- **체크인 날**: `day === cin`
- **체크아웃 날**: `day === cout`
- **both**: 어떤 예약의 cout이자 다른 예약의 cin인 날

---

## Extra (비밀번호 + 메모)

### localStorage 키 형식
```
hana_ex_<bkKey>
```

### bkKey 형식
```js
// Airbnb
`${roomName}|${cinY}${String(cinM+1).padStart(2,'0')}${String(cinD).padStart(2,'0')}`
// 예: "302호|20250610"

// Booking.com
`302호|20250610_bk`

// Trip.com
`302호|20250610_tr`

// 리브애니웨어
`302호|20250610_lv`
```

> **주의**: `cinM+1` — 저장할 때만 1-indexed로 변환.  
> **주의**: `roomName`이 키에 포함되므로 호실명 변경 시 기존 데이터 고아됨.

### Extra 값 구조
```js
{
  passwords: [
    { pw: "1234#", date: "2025.06.01 10:00" },
    { pw: "5678*", date: "2025.05.20 09:30" },
    // ... 최대 15개 (초과 시 가장 오래된 항목 제거)
  ],
  memos: [
    { text: "고양이 있는 게스트. 청소 시 확인", date: "2025.06.01 10:00" },
    { text: "에어컨 리모컨 위치 안내 필요", date: "2025.05.28 14:22 (수정됨)" },
  ]
}
```

### 저장/로드 방식
- **저장**: `setExtra(key, data)` → localStorage 즉시 + KV 비동기
- **로드**: `loadExtra(key)` → KV 시도 → 실패 시 `getExtra(key)` (localStorage)
- **읽기 (동기)**: `getExtra(key)` → localStorage만

---

---

## Push Event (알림 이벤트)

### 저장 위치
- PUSH_KV key: `events` (최대 50개, 초과 시 오래된 것 제거)

### 구조
```js
{
  id:       "1718000000000_ab1cd",  // `${Date.now()}_${random5}`
  type:     'new' | 'error' | 'recovered' | 'test',
  room:     "501 hana",             // 호실명
  platform: "Booking.com",          // 플랫폼 레이블
  cin:      "2026/06/10",           // new 타입만. 나머지는 ''
  cout:     "2026/06/13",           // new 타입만. 나머지는 ''
  ts:       1718000000000,          // Date.now()
  read:     false                   // 미확인 여부
}
```

### type별 발생 조건
| type | 발생 시점 |
|------|---------|
| `new` | 새 예약 감지 (6개월 이내) |
| `error` | 동일 호실·플랫폼 3회 연속 실패 (15분) |
| `recovered` | 미확인 `error`가 있는 호실·플랫폼이 동기화 성공 시 |
| `test` | `/push/test` 엔드포인트 호출 시 |

---

## 예약 아카이브 (booking_archive KV)

### 저장 위치
- KV key: `booking_archive`
- 예약앱 통계 뷰 전용. 달력 뷰(`synced_bookings`)와 완전 분리.

### 구조
```js
{
  "302호": {
    ab: [ {cinY, cinM, cinD, coutY, coutM, coutD, platform, summary}, ... ],
    bk: [ ... ],
    tr: [ ... ],
    lv: [ ... ]
  },
  "402호": { ... },
  ...
}
```

### 관리 규칙
- Sync 실행 시 자동 병합 (UID 기준 중복 제거)
- 블락 제외 필터는 **플랫폼별로 다름** (2026.07.08 수정 — known-issues #15):
  - `ab`: summary에 `not available` 포함 시 제외
  - `bk`: **전부 보존** (Booking.com은 실제 예약도 "CLOSED - Not available"로 옴)
  - `tr`/`lv`: summary가 `closed`일 때만 제외 (`not available`은 파서에서 이미 제거, 빈 summary는 실제 예약)
- append-only — 취소·날짜변경된 예약도 남음 (iCal로는 취소 구분 불가). 이 때문에 과거 날짜는 오버부킹 판정에서 제외함 (known-issues #16)
- **부분 스냅샷 자동 제거** (2026.07.08 — known-issues #17): 같은 플랫폼에서 다른 항목 범위에 완전히 포함되는 항목은 제거. Trip.com의 "오늘~체크아웃" 일일 스냅샷 적립 방지. 워커·예약앱 양쪽에 동일 규칙.
- 체크아웃 기준 13개월 이전 데이터 자동 정리
- 호실명이 키에 포함 → 호실명 변경 시 이전 이력 고아 (known-issues #1 동일)

---

## 트립 당일잠금 판정 (tr_daylock)

트립 자동 규칙 `전략 제어 – 당일 객실 마감`이 매일 KST `17:59:59`에 그날을 만실로 바꾸고 다음날 `07:00`에 되돌린다. 그 **가짜 하룻밤**이 이웃 예약과 날짜로 맞닿으면 한 덩어리 VEVENT로 합쳐져 나온다 → 입실이 하루 당겨지거나 퇴실이 하루 밀려 보인다 (known-issues #28).

파일 내용으로는 진짜 예약과 구분할 수 없다. **시각으로 가른다.**

### 저장 위치
- KV key: `tr_daylock` (⚠ `extra_` 접두사 **없음**)
- **워커만 쓴다. 두 앱은 읽지 않는다** → 기기 간 덮어쓰기(#22·#26) 위험 없음
- 진단용 조회: `GET /daylock`

### 구조
```js
{
  day:  "20260902",                     // KST 오늘. 롤오버 감지용
  seen: { "402 jnj": true, ... },        // 안전창 마지막 관측: 그날 밤이 피드에 덮여 있었나
  fake: { "402 jnj": { night: "20260901", min: 1080 } }  // 가짜 판정 + 잠긴 밤 + 감지 시각(KST 분)
}
```

⚠ **`fake`는 '오늘'이 아니라 '잠긴 밤'을 기억한다.** 잠금은 18:00에 생겨 다음날 07:00까지 살아 있어서, '오늘' 기준으로 만들면 자정에 초기화돼 **새벽 0~7시가 다시 틀린다.**
`fake` 항목은 피드가 그 밤을 더 이상 안 덮으면 **스스로 사라진다** (07:00 해제 시 자동 정리).

### 판정 규칙 — 자물쇠 둘 다 맞을 때만
```
① st.seen[room] === false                    안전창(KST 07:30~17:30) 관측에 그 밤이 비어 있었다
② KST 18:00 ~ 18:10 차례에 처음 나타났다        자동 규칙의 지문 (실측 18:00:57~18:01:01)
```
- `seen[room]`이 `undefined`(안전창에 피드를 못 받음)면 **교정하지 않는다**
- 어느 실패 경로든 "지금까지와 동일 동작"으로 떨어진다 → 오버부킹이 나는 방향으로 틀리지 않는다
- 트립 `예약 마감 시간`을 12:00으로 두면 17:30 이후 진짜 신규 예약이 불가능해져 판정이 더 안전해진다

### 적용 지점
| 대상 | 교정 | 비고 |
|---|---|---|
| `booking_archive` 저장 전 | ✔ | **최우선.** 안 막으면 가짜가 영구 고착 (#28-4) |
| `GET /bookings` | ✔ | 예약앱. `synced_bookings` **원본은 안 바꾼다** |
| `GET /?url=&fix=tr&room=` | ✔ | 청소앱 **옵트인**. `fix` 없으면 원본 바이트 그대로 |
| `GET /ical/...` 내보내기 | ✘ | **손대지 않는다** (2026-09-02 기준). 별건으로 검토 |

### 소급 청소
`POST /daylock/backfill` (`?dry=1` 예행) — 알림 로그의 `18:00~18:10` 도장으로 굳은 가짜를 찾아 **교정**한다(삭제 아님). 멱등.

---

## 예약 경계 (tr_cuts)

Trip.com이 연속 예약을 하나의 점유 블락으로 합쳐 보낼 때, 사용자가 실제 손님이 바뀌는 날을 표시해둔 것. 원본 예약은 건드리지 않고 이 날짜 메모만 별도 저장 → 로드 시 `trBookings`의 통짜를 조각으로 **치환**한다.

> **2026.08.06 이전에는 `tr_splits`** (기간 + 경계 목록)였다. 아래 "구버전과의 차이" 참조.

### 저장 위치
- KV key: `extra_tr_cuts` (Worker `/extra?key=tr_cuts`)
- localStorage key: `hana_cuts`
- 예약앱·청소앱이 **같은 키를 공유** → 예약앱에서 나눈 결과가 청소앱에도 반영
- **쓰기는 예약앱만.** 청소앱은 읽기 전용 (`saveCuts` 없음, 구버전 이관도 안 함)

### 구조
```js
// 배열 — 날짜 하나가 곧 하나의 경계
[
  { roomName: "402 jnj", platform: "tr", y: 2026, m: 7, d: 10 }   // m=0-indexed
]
```

### 적용 규칙 (오버부킹 방지)
- `applyCuts(room)`: 경계가 예약 **한가운데**(`cin < 경계 < cout`)일 때만 조각으로 치환 (원본 제거 + 조각 삽입, 추가 금지).
  경계가 `cin` 또는 `cout`과 같으면 이미 나뉜 상태 → 아무것도 안 함.
- 한 예약에 경계가 여러 개면 날짜순으로 정렬해 조각을 이어 만든다 (조각 = 경계 + 1개).
- 조각 경계는 `앞.cout === 뒤.cin` (cout-exclusive) → 경계일이 오버부킹으로 안 잡힘.
- 원본 배열은 `room['_raw_trBookings']`에 보관 → 경계 편집 시 `reapplyCuts()`가 복원해 재계산.
  - `mergeArchiveIntoRooms()`는 시작 시 이 raw를 먼저 복원 후 병합 (재호출 시 백업이 조각 배열로 오염되던 버그 수정, 2026.07.15).
  - ⚠ **`loadBookingsFromKV()`는 새 피드를 대입한 직후 이 백업을 반드시 `delete` 한다.** 안 그러면 위 복원이 새 피드를 옛 피드로 되돌려, ↻ 를 눌러도 취소된 예약이 사라지지 않는다 (known-issues #23, 2026.08.06).
  - 청소앱에는 `_raw_` 자체가 없다 — 경계 편집 경로가 없고 `syncAll()`이 매번 배열을 새로 채우기 때문.
- ~~은퇴(`retireCuts`)~~ **폐지 (2026.08.09)** — 경계는 **사용자가 해지할 때만** 사라진다.
  - 옛 동작: 그 날짜에 걸친 예약이 피드에서 전부 사라지면 경계를 자동으로 거두고 `saveCuts()`까지 했다.
  - 폐지 이유: 트립닷컴이 문의 단계 날짜를 넣었다 뺐다 하는(#25) 그 순간에 앱이 켜지면 **사용자가 손으로 만든 경계가 KV에서 영구 삭제**됐다. 예약이 돌아와도 경계는 돌아오지 않아 매번 다시 잘라야 했다 (known-issues #27).
  - 부수 효과(의도된 것): `loadBookingsFromKV`가 더는 `trCuts`를 건드리지 않으므로 그 경로의 `saveCuts()`도 없어졌다 → 피드를 못 읽은 상태에서 **부분 목록으로 KV를 덮어쓰는 경로**가 함께 막혔다.
  - 아무 예약에도 걸치지 않는 경계는 `applyCuts`가 그냥 지나치므로 화면에 무영향. 나중에 새 통짜 한가운데 걸리면 `detectCutAlerts`가 ⚠ 확인 카드를 띄운다.
- 호실명이 키에 포함 → 호실명 변경 시 고아 (known-issues #1 동일).

### 구버전(`tr_splits`)과의 차이 — 왜 바꿨나
옛 방식은 `origCin`~`origCout` 기간이 **완전 일치**해야 적용됐다. 손님이 하루 연장하거나 옆 예약이 취소되면 기간이 달라져 **정의 전체가 무효**가 됐고, 이를 살리려고 "자동 승계(split-inherit)"라는 별도 장치가 필요했다.

새 방식은 **날짜 하나만 기억**한다. 예약이 늘든 줄든 그 날짜가 여전히 예약 한가운데면 그대로 적용된다. 승계 로직이 통째로 불필요해졌고, 남은 건 "새 예약에 경계가 걸쳤으니 확인하라"는 ⚠ 안내뿐이다.

예약앱 `migrateSplitsToCuts()`가 구 정의의 `boundaries`에서 날짜만 뽑아 1회 이관한다 (**예약앱만** 수행).

---

## 피드 스냅샷 (tr_feed_prev) — ⚠ 확인 판정용

예약앱이 "직전에 본 Trip.com 원본 목록"을 기억해뒀다가, 다음 로드에서 비교해 **경계가 새로 통짜 한가운데에 걸리게 된 경우**를 찾아낸다. 상세: [features/split-inherit.md](features/split-inherit.md)

### 저장 위치 (예약앱 전용 — 청소앱은 안 읽음)
- KV key: `extra_tr_feed_prev` (Worker `/extra?key=tr_feed_prev`)
- localStorage key: `hana_feed_prev`
- KV POST는 **내용이 바뀐 경우에만** (KV write 한도 원칙)

### 구조
```js
{
  rooms: {                          // 호실별 직전 피드 목록 (Booking 객체 형식)
    "402 jnj": [ {cinY,cinM,cinD,coutY,coutM,coutD}, ... ]
  },
  alerts: [                         // 확인이 필요한 경계 (확인 전까지 ⚠ 유지)
    { roomName:"402 jnj", y:2026, m:7, d:10 }
  ]
}
```

### 판정 규칙 (`detectCutAlerts`)
호실 피드가 실제로 로드된 경우에만 호출한다 (스냅샷을 현재 피드로 갱신하므로).

- 오늘 이전 경계는 건너뛴다.
- 경계가 **지금 통짜 한가운데에 걸치지 않으면** 문제 없음 → 통과.
- 걸치더라도 **직전 피드에도 그 날짜를 덮는 예약이 있었으면** 원래 있던 예약이 합쳐진 것 → 정상 동작이므로 조용히 통과.
- 위 둘 다 아니면 = 그 자리에 없던 예약이 새로 생겨 경계를 삼킨 것 → `alerts`에 추가, ⚠ 표시.
- 경계가 삭제됐거나 더는 통짜에 안 걸치면 해당 ⚠ 자동 해제.

---

## 색상 팔레트

```js
const COLORS = [
  '#c8f07c',  // 0: 라임 그린
  '#6ecf8f',  // 1: 민트 그린
  '#5bc8d8',  // 2: 하늘색
  '#7fa8f5',  // 3: 코발트 블루
  '#c07aee',  // 4: 라벤더
  '#f07cc8',  // 5: 핑크
  '#f0b35b',  // 6: 오렌지
  '#f07070',  // 7: 코럴 레드
  '#a8d88a',  // 8: 올리브 그린
]
```

---

## 셀 상태 → CSS 클래스 매핑

| 플랫폼 | 상태 | CSS 클래스 |
|--------|------|-----------|
| Airbnb | 숙박중 | `c-occupied` |
| Airbnb | 체크아웃 | `c-checkout` |
| Airbnb | 체크인 | `c-checkin` |
| Airbnb | 동일날 인아웃 | `c-both` |
| Booking.com | 숙박중 | `c-bk-occupied` |
| Booking.com | 체크아웃 | `c-bk-checkout` |
| Booking.com | 체크인 | `c-bk-checkin` |
| Trip.com | 숙박중 | `c-tr-occupied` |
| Trip.com | 체크아웃 | `c-tr-checkout` |
| Trip.com | 체크인 | `c-tr-checkin` |
| 리브애니웨어 | 숙박중 | `c-lv-occupied` |
| 리브애니웨어 | 체크아웃 | `c-lv-checkout` |
| 리브애니웨어 | 체크인 | `c-lv-checkin` |
| 서로 다른 플랫폼 아웃+인 (1:1) | 반반 셀 | `c-mix-both` — 좌빨강/우초록, 아이콘 2개(아웃쪽 먼저) |
| 플랫폼 아웃 + 수동 블락 시작 | 반반 셀 | `c-mix-out-bl` — 좌빨강/우블락색 |
| 플랫폼 아웃 + 에어비앤비 블락 시작 | 반반 셀 | `c-mix-out-abblk` |
| 그 외 복수 이벤트 동시 (2:1 등) | 혼합 | `c-dual` + `d1-<소스> d2-<소스>` |

> `c-dual` 배경은 실제로 겹친 소스 색으로 상하 분할된다 (위=첫 소스, 아래=둘째 소스, 하나뿐이면 단색). 소스 키는 `ab / blk / bk / tr / lv / bl` 이며 CSS 변수 `--d1 / --d2`로 주입. 이전에는 고정 빨강/남색이라 Trip.com 예약이 Booking.com 숙박중 남색으로 보이는 문제가 있었다 (2026.08.05 수정).

> 아웃/인 반반 셀은 전부 에어비앤비 `c-both`와 동일 레이아웃(좌=아웃/우=인, `bothLbl()`) 사용. 아이콘만 해당 플랫폼(블락은 🔒, 에어비앤비 블락은 🚫). `blockTypeFor`는 블락 첫날에 `'start'` 반환 (2026.07.08).

---

## 날짜 유틸리티 함수

```js
today()           → { y, m, d }        // 오늘 날짜 (m은 0-indexed)
daysIn(y, m)      → number             // 해당 월 일수
dateStr(y, m, d)  → "2025.06.01"       // 표시용 문자열 (m+1)
nowStr()          → "2025.06.01 10:00" // 현재 시각 문자열
```

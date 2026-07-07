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

## 예약 분할 (tr_splits)

Trip.com이 연속 예약을 하나의 점유 블락으로 합쳐 보낼 때, 사용자가 실제 체크인/체크아웃 경계로 나눈 정의. 원본 예약은 건드리지 않고 이 정의만 별도 저장 → 로드 시 `trBookings`의 통짜를 조각으로 **치환**한다.

### 저장 위치
- KV key: `extra_tr_splits` (Worker `/extra?key=tr_splits`)
- localStorage key: `hana_splits`
- 예약앱·청소앱이 **같은 키를 공유** → 예약앱에서 나눈 결과가 청소앱에도 반영

### 구조
```js
// 배열
[
  {
    roomName: "402호",
    platform: "tr",                 // 현재 Trip.com만 (SPLIT_PLATFORMS)
    origCin:  {y:2026, m:6, d:9},    // 원본 통짜 체크인 (m=0-indexed)
    origCout: {y:2026, m:6, d:24},   // 원본 통짜 체크아웃
    decided:  true,                  // 경계 애매 시 사용자가 확정했는지
    boundaries: [                    // 내부 경계일들 (조각 = 경계+1개)
      {y:2026, m:6, d:18},
      {y:2026, m:6, d:20}
    ]
  }
]
```

### 적용 규칙 (오버부킹 방지)
- `applySplits(room)`: 예약이 split의 `origCin/origCout`와 **완전 일치**할 때만 조각으로 치환 (원본 제거 + 조각 삽입, 추가 금지).
- 조각 경계는 `앞.cout === 뒤.cin` (cout-exclusive) → 경계일이 오버부킹으로 안 잡힘.
- 원본이 1일이라도 바뀌면 매칭 실패 → **자동 무효**(원본 그대로 표시) + 안내. 단 원본이 비었으면(로드 실패) 판정 보류.
- 원본 배열은 `room['_raw_trBookings']`에 보관 → 분할 해제/재계산 시 복원.
- 호실명이 키에 포함 → 호실명 변경 시 고아 (known-issues #1 동일).

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
| 그 외 복수 이벤트 동시 (2:1 등) | 혼합 | `c-dual` |

> 아웃/인 반반 셀은 전부 에어비앤비 `c-both`와 동일 레이아웃(좌=아웃/우=인, `bothLbl()`) 사용. 아이콘만 해당 플랫폼(블락은 🔒, 에어비앤비 블락은 🚫). `blockTypeFor`는 블락 첫날에 `'start'` 반환 (2026.07.08).

---

## 날짜 유틸리티 함수

```js
today()           → { y, m, d }        // 오늘 날짜 (m은 0-indexed)
daysIn(y, m)      → number             // 해당 월 일수
dateStr(y, m, d)  → "2025.06.01"       // 표시용 문자열 (m+1)
nowStr()          → "2025.06.01 10:00" // 현재 시각 문자열
```

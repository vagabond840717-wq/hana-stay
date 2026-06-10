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
- `not available`, `closed`, 빈 summary 제외 — 실제 게스트 예약만 보존
- 체크아웃 기준 13개월 이전 데이터 자동 정리
- 호실명이 키에 포함 → 호실명 변경 시 이전 이력 고아 (known-issues #1 동일)

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
| 복수 플랫폼 동시 | 혼합 | `c-dual` |

---

## 날짜 유틸리티 함수

```js
today()           → { y, m, d }        // 오늘 날짜 (m은 0-indexed)
daysIn(y, m)      → number             // 해당 월 일수
dateStr(y, m, d)  → "2025.06.01"       // 표시용 문자열 (m+1)
nowStr()          → "2025.06.01 10:00" // 현재 시각 문자열
```

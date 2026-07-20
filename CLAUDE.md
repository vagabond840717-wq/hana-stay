# HANA STAY — Claude 작업 참조 파일

---

## 작업 워크플로우 (모든 기능 개발 시 필수 준수)

### 기능 개발 4단계 프로세스

```
1. /new-feature <설명>
   └→ 웹 검색 (베스트 프랙티스)
   └→ 영향 범위 분석
   └→ 스펙 초안 작성 (docs/features/DRAFT-xxx.md)
   └→ ⏸ 사용자 승인 대기

2. /plan <기능명>
   └→ 구현 계획서 작성 (docs/features/PLAN-xxx.md)
   └→ 파일별 변경 내용 상세화
   └→ ⏸ 사용자 승인 대기

3. /implement <기능명>
   └→ 계획서대로 Task 단위 구현
   └→ 각 Task 완료 후 보고
   └→ ⏸ 사용자 검수

4. /done <기능명>
   └→ 스펙 파일 확정 (DRAFT- 접두사 제거)
   └→ docs/ 전체 업데이트
   └→ CLAUDE.md 업데이트
   └→ ⏸ 사용자 확인 (문서 내용 맞는지 최종 검토)
```

**승인 없이 다음 단계로 넘어가지 않는다.**  
**큰 기능은 먼저 설계 제안, 구현은 반드시 승인 후.**

### 웹 검색 원칙 (리서치 단계)
- 구현 방법 2~3가지 옵션 검색
- 이 프로젝트는 **순수 HTML/JS/CSS** (프레임워크 없음) — 바닐라 구현 우선
- iCal 파싱, Cloudflare Workers, 모바일 UX 관련 내용 우선

---

## 프로젝트 구조

```
E:\airbnb\
├── CLAUDE.md                          ← 이 파일
├── .claude/
│   ├── settings.json                  ← 프로젝트 권한 설정
│   ├── settings.local.json            ← 로컬 전용 (git 제외)
│   └── commands/
│       ├── new-feature.md             ← /new-feature 커맨드
│       ├── plan.md                    ← /plan 커맨드
│       ├── implement.md               ← /implement 커맨드
│       └── done.md                    ← /done 커맨드
├── docs/
│   ├── 01-overview.md
│   ├── 02-architecture.md
│   ├── 03-data-model.md
│   ├── 04-apps-spec.md
│   ├── 05-known-issues.md
│   └── features/                      ← 기능별 스펙/계획서
│       ├── DRAFT-xxx.md               ← 작업 중인 스펙
│       ├── PLAN-xxx.md                ← 구현 계획서
│       └── xxx.md                     ← 완료된 스펙 (확정)
└── app/
    ├── JnJ/index.html                 ← 청소 스케줄 앱 (다크 테마)
    ├── JnJ booking/index.html         ← 예약현황 앱 (라이트/네이비 테마)
    ├── JnJ Price/index.html           ← 요금 계산기 (베이지 테마)
    └── parking-main/parking-main/
        ├── worker.js                  ← Cloudflare Worker (주차 KV)
        └── wrangler.toml
```

---

## 백엔드 (Cloudflare Workers)

**Proxy Worker URL**: `https://ical-proxy.vagabond1984.workers.dev`

| 메서드 | 경로 | 용도 |
|--------|------|------|
| GET | `/?url=<ical_url>` | iCal URL CORS 우회 프록시 |
| GET | `/rooms` | 호실 목록 KV에서 로드 |
| POST | `/rooms` | 호실 목록 KV에 저장 |
| GET | `/extra?key=<key>` | 비밀번호/메모 로드 |
| POST | `/extra` | 비밀번호/메모 저장 |
| GET | `/bookings` | 동기화된 예약 데이터 로드 (달력용) |
| POST | `/sync` | iCal 동기화 실행 |
| GET | `/archive` | 예약 이력 아카이브 로드 (통계용) |

---

## 핵심 데이터 구조

### Room 객체 (저장 형태)
```js
{
  name: "302호",
  url:   "webcal://...",  // Airbnb iCal (없으면 '')
  bkUrl: "webcal://...",  // Booking.com iCal
  trUrl: "webcal://...",  // Trip.com iCal
  lvUrl: "webcal://...",  // 리브애니웨어 iCal
  color: "#c8f07c"        // COLORS[i % 9] — 추가 순서 고정
}
```
런타임에는 `bookings[]`, `bkBookings[]`, `trBookings[]`, `lvBookings[]` 추가됨 (저장 안 함).

### Booking 객체
```js
{ cinY, cinM, cinD, coutY, coutM, coutD }
// cinM, coutM 은 0-indexed (5 = 6월)
```

### bkKey 형식 (비밀번호/메모 키)
```js
`${roomName}|${cinY}${String(cinM+1).padStart(2,'0')}${String(cinD).padStart(2,'0')}`
// Booking.com: + "_bk" | Trip.com: + "_tr" | 리브애니웨어: + "_lv"
// ⚠ roomName이 키에 포함 → 호실명 변경 시 데이터 고아됨
```

### Extra 데이터
```js
// localStorage: 'hana_ex_<bkKey>'
{
  passwords: [{ pw: "1234", date: "2025.06.01 10:00" }],  // 최대 15개
  memos:     [{ text: "...", date: "..." }]
}
```

### tr_splits (Trip.com 예약 분할)
```js
// localStorage: 'hana_splits' | KV: extra_tr_splits (/extra?key=tr_splits)
// 예약앱·청소앱 공유. 원본 미변경, 로드 시 trBookings 통짜를 조각으로 치환.
[{ roomName:"402호", platform:"tr",
   origCin:{y,m,d}, origCout:{y,m,d},  // m=0-indexed, 완전일치해야 적용
   decided:true, boundaries:[{y,m,d},...],
   inherited:true, autoEdges:[{y,m,d}] }]  // 자동 승계 잠정 분할에만 (decided:false → ⚠배지)
// applySplits: 원본 제거+조각 삽입(오버부킹 방지). 원본 바뀌면 승계 시도 → 불가 시 자동무효+안내.
// 원본 백업: room['_raw_trBookings'] — mergeArchiveIntoRooms는 시작 시 raw 복원 후 병합(오염 방지).
// 현재 SPLIT_PLATFORMS=['tr']. 상세: docs/features/booking-split.md
```

### tr_feed_prev (피드 스냅샷 — 분할 자동 승계, 예약앱 전용)
```js
// localStorage: 'hana_feed_prev' | KV: extra_tr_feed_prev (/extra?key=tr_feed_prev)
// 직전 Trip.com 원본 통짜 목록. 로드 시 diff → 통짜 합쳐짐이면 잠정 분할(decided:false) 자동 생성.
{ rooms:{ "402호":[{cinY..coutD},...] }, attention:[{roomName,cin,cout}] }
// KV write는 내용 변경 시에만. 잠정/attention은 사용자가 확인 카드에서 처리할 때까지 ⚠ 유지.
// 상세: docs/features/split-inherit.md
```

---

## 앱별 핵심 차이

| 기능 | JnJ (청소) | JnJ booking (예약) |
|------|-----------|-------------------|
| 테마 | 다크 `#0f0f0f` | 라이트 `#f4f6fb` |
| 오버부킹 감지 | ✗ | ✓ `.c-overbooking` |
| 블락 처리 | ✗ | ✓ `.c-ab-block` |
| 달력 스크롤 | 가로만 | 가로+세로 |
| PWA / 푸시 알림 | ✗ | ✓ |
| 비밀번호/메모 | ✓ | 확인 필요 |
| Trip.com 예약 분할 | 반영만(읽기전용) | ✓ 생성/편집 (`tr_splits`) |
| 분할 자동 승계 (⚠배지) | ✗ (결과 조각만 반영) | ✓ 감지/승계/확인 (`tr_feed_prev`) |

---

## 공통 로직 패턴

### 셀 상태 판별
```js
cellTypeFor(bookings, day, y, m)
// 반환: 'empty' | 'checkout' | 'checkin' | 'both' | 'occupied'
// ⚠ cout는 exclusive: cin < cur < cout (숙박) / cur === cout (체크아웃)
```

### iCal 필터 규칙 (플랫폼별 다름)
- Airbnb: `not available`, `airbnb (not available)` 제외
- Booking.com / Trip.com / LV: `not available`, `closed`, `''` 제외

### 저장 패턴
```js
localStorage 즉시 → fetch(KV) 비동기 + .catch(()=>{})
로드: KV 우선 → 실패 시 localStorage 폴백
```

### 렌더 사이클
```
render() → calendarArea.innerHTML 전체 교체 → attachCellClicks() 재등록
```

---

## 코드 수정 시 필수 체크리스트

수정 전:
- [ ] 청소 앱 / 예약 앱 둘 다 수정 필요한가?
- [ ] iCal 파서 4개 모두 확인했는가?
- [ ] bkKey 변경인가? → 마이그레이션 필요
- [ ] 날짜 월 값이 0-indexed인지 확인
- [ ] `render()` 호출 후 `attachCellClicks()` 체인 유지되는가?

수정 후:
- [ ] `docs/05-known-issues.md` 갱신
- [ ] 해당 앱 스펙 문서 갱신 (`docs/04-apps-spec.md`)
- [ ] 데이터 구조 변경 시 `docs/03-data-model.md` 갱신

---

## 기본값 / 상수

```js
COLORS = ['#c8f07c','#6ecf8f','#5bc8d8','#7fa8f5','#c07aee','#f07cc8','#f0b35b','#f07070','#a8d88a']
MONTHS_TO_SHOW = 6           // 월간/연속 뷰에서 표시 개월 수
MAX_ROOMS = 9                // 최대 호실 수
MAX_PW_HISTORY = 15          // 비밀번호 이력 최대 개수
DEFAULT_ROOMS = ['302호','402호','501호','503호','601호','603호','701호','702호','703호']
```

---

## 플랫폼 식별
- ✈ Airbnb | 🏨 Booking.com | 🌐 Trip.com | 🏡 리브애니웨어

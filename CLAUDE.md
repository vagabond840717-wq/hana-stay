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
├── ical-proxy/                        ← 별도 저장소 (git 제외). push → Cloudflare 자동 배포
│   ├── worker.js                      ← 프록시·동기화·아카이브·iCal 내보내기 전부
│   └── wrangler.toml
└── app/                               ← ⚠ 원본 아님. 검증 완료본 백업 (아래 참조)
    ├── JnJ/index.html                 ← 청소 스케줄 앱 (다크 테마)
    ├── JnJ booking/index.html         ← 예약현황 앱 (라이트/네이비 테마)
    ├── JnJ Price/index.html           ← 요금 계산기 (베이지 테마)
    └── parking-main/parking-main/
        ├── worker.js                  ← Cloudflare Worker (주차 KV)
        └── wrangler.toml
```

### ⚠ 앱 소스의 원본은 깃허브다 — `app/` 폴더를 참조하지 말 것

`E:\airbnb\app/` 는 **"마지막으로 검증된 정상본" 백업**이다. 롤백 지점 용도이며, 코드를 읽거나
고칠 때 쳐다보면 안 된다. 배포본보다 뒤처져 있을 수 있다.

| 앱 | 저장소 | 배포 |
|---|---|---|
| 예약현황 | `vagabond840717-wq/booking` | GitHub Pages — https://vagabond840717-wq.github.io/booking/ |
| 청소 스케줄 | `vagabond840717-wq/jnjhana` | Cloudflare Pages — https://jnjhana.pages.dev/ |
| 요금 계산기 | `vagabond840717-wq/price` | GitHub Pages |
| 주차 | `vagabond840717-wq/parking` | Cloudflare Worker |
| 문서 (이 폴더) | `vagabond840717-wq/hana-stay` | — |
| 프록시 워커 | `vagabond840717-wq/ical-proxy` | Cloudflare Workers |

**작업 절차**
1. `git clone --depth 1 <저장소>` 로 받는다. 두 앱 모두 main = 배포본 (2026-08-06 바이트 단위 확인)
2. `git log -1` 로 되돌아갈 커밋 해시를 기록해 사용자에게 알린다
3. **푸시 = 즉시 배포.** 검증을 배포 전에 끝낸다
4. 배포·확인 후 `app/` 백업을 그 검증본으로 덮어쓴다

**배포 전 검증 요령** — 정적 HTML이라 로컬 서버 없이도 실데이터로 검증할 수 있다.
배포본을 브라우저로 열고 `javascript_tool` 로 **수정한 함수만 갈아끼운 뒤** 화면을 다시 그린다.
전체 셀의 클래스명을 수정 전후로 덤프해 대조하면 부작용을 정확히 잡아낸다
(2026-08-06: 청소앱 1,656칸 중 1칸만 변경 / 예약앱 1,935칸 중 0칸 변경 확인).

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
| GET | `/ical/<호실명>` | **HANA STAY 통합 iCal 내보내기** — 아래 주의 |

**워커 소스**: `E:\airbnb\ical-proxy\worker.js` (별도 저장소 `vagabond840717-wq/ical-proxy`, git push → 자동 배포)

### ⚠ `/ical/<호실명>` 내보내기 — 전 채널 영향
**에어비앤비·부킹닷컴·리브애니웨어가 모두 이 주소를 구독한다.** 한 줄 고치면 전 채널이 동시에 바뀐다.
```js
// exportIcal — DTEND는 cout 그대로. 절대 하루 빼지 말 것.
const de = `${bk.coutY}${String(bk.coutM+1).padStart(2,'0')}${String(bk.coutD).padStart(2,'0')}`;
```
- iCal `DTEND;VALUE=DATE`는 **포함 안 되는 날** → `DTEND = cout` 만으로 체크아웃 당일이 판매 가능일이 된다
- 여기서 하루를 더 빼면 **마지막 숙박일(손님 투숙 중)이 전 채널에서 열린다** → 오버부킹. 2026-08-01~08-05 실제 발생, [05-known-issues.md](docs/05-known-issues.md) #21
- Trip.com·Booking.com도 **DTEND exclusive 표준**을 따른다 (실측 확인). 추측하지 말고 `/?url=` 프록시로 원본을 받아 대조할 것
- 에어비앤비 "Not available"은 내보내기에서 제외 (순환 방지) — 유지할 것
- 에어비앤비는 **가져온 블락을 자기 iCal로 되내보내지 않는다** → 반영 여부는 앱 달력 화면으로만 확인 가능

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

### tr_cuts (Trip.com 예약 경계) — 구 `tr_splits` 대체
```js
// localStorage: 'hana_cuts' | KV: extra_tr_cuts (/extra?key=tr_cuts)
// 예약앱·청소앱 공유. "이 호실 · 이 날짜는 손님이 바뀌는 날" 이라는 날짜 메모 하나뿐.
[{ roomName:"402 jnj", platform:"tr", y:2026, m:7, d:10 }]   // m=0-indexed
// applyCuts: 경계가 예약 한가운데(cin < 경계 < cout)일 때만 통짜를 조각으로 치환.
//   경계 == cin 또는 cout 이면 이미 나뉜 상태 → 아무것도 안 함. 원본 제거+조각 삽입(오버부킹 방지).
// 원본 백업: room['_raw_trBookings'] — 경계 편집 시 reapplyCuts()가 복원해 재계산.
//   ⚠ loadBookingsFromKV는 새 피드를 대입한 뒤 이 백업을 반드시 delete 한다.
//      안 그러면 mergeArchiveIntoRooms 첫머리의 복원이 새 피드를 옛 피드로 되돌린다(#23).
// ⚠ 자동 은퇴 없음(2026-08-09 폐지). 경계는 사용자가 해지할 때만 사라진다.
//   옛 retireCuts는 "그 날짜에 걸친 예약이 피드에 없으면 거둔다"였는데, 트립이 문의 날짜를
//   넣었다 뺐다 하는 순간(#25)에 앱이 켜지면 사용자 경계가 KV에서 영구 삭제됐다 → #27
//   ⇒ loadBookingsFromKV는 trCuts를 건드리지 않는다. 그 경로에 saveCuts()를 다시 넣지 말 것.
//   아무 예약에도 안 걸치는 경계는 화면에 무영향. 새 통짜에 걸치면 detectCutAlerts가 ⚠로 묻는다.
// 현재 SPLIT_PLATFORMS=['tr']. 상세: docs/features/booking-split.md
```
**구버전(`tr_splits`)과의 차이** — 옛 방식은 `origCin`~`origCout` 기간이 **완전일치**해야 적용됐다.
연장·단축이 한 번만 있어도 정의 전체가 무효가 됐다. 새 방식은 날짜 하나만 기억하므로 살아남는다.
예약앱 `migrateSplitsToCuts()`가 구 정의에서 경계 날짜만 뽑아 1회 이관한다(**예약앱만** 수행).

### tr_feed_prev (피드 스냅샷 — ⚠ 확인 판정, 예약앱 전용)
```js
// localStorage: 'hana_feed_prev' | KV: extra_tr_feed_prev (/extra?key=tr_feed_prev)
// 직전 Trip.com 원본 통짜 목록. 경계가 "새로" 통짜 한가운데에 걸리면 ⚠ 확인 항목 생성.
{ rooms:{ "402 jnj":[{cinY..coutD},...] }, alerts:[{roomName,y,m,d}] }
// 원래도 그 날짜를 덮던 예약이 합쳐진 것이면 정상 동작 → 조용히 통과.
// KV write는 내용 변경 시에만(한도 원칙). ⚠는 사용자가 확인 카드에서 처리할 때까지 유지.
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
| Trip.com 예약 경계 | 반영만(읽기전용) | ✓ 생성/편집 (`tr_cuts`) |
| 경계 확인 알림 (⚠배지) | ✗ (결과 조각만 반영) | ✓ 감지/확인 (`tr_feed_prev`) |

### ⚠ 기준 앱 = 예약앱 (JnJ booking)
**공통 로직·셀 표시가 두 앱에 다 있을 때는 예약앱을 먼저 고치고, 청소앱은 예약앱에 맞춘다.**
- 판정 함수(`cellTypeFor`, `blockTypeFor`, `bothLbl` 등)와 셀 라벨/색 규칙이 갈리면 **예약앱 쪽이 정답**
- 청소앱을 먼저 고치거나, 청소앱만 다르게 두는 선택은 하지 않는다
- 단, 청소앱에 없는 기능(오버부킹·PWA 등)까지 억지로 이식하지는 않는다 — **겹치는 부분만** 맞춤

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
// 단독 값(비밀번호·메모 등): localStorage 즉시 → fetch(KV) 비동기 + .catch(()=>{})
// 로드: KV 우선 → 실패 시 localStorage 폴백
```

**⚠ 공유 목록(`manual_blocks`, `tr_cuts`)은 위 패턴을 쓰면 안 된다.**
호실 구분 없이 배열 하나로 저장되므로, 통째 덮어쓰기는 **다른 기기·다른 호실의 항목을 지운다** (#22, #26).
```js
// 반드시 saveListMerged 경유 — 저장 직전 서버 최신 배열을 다시 읽고 '내 변경만' 얹는다
await saveBlocksMerged({type:'add'|'replace'|'remove', block, oldBlock})
await saveCutsMerged({type:'add'|'remove', cut|roomName+p})   // commitCuts(op)가 감싸고 있음
```
- **서버를 못 읽으면 쓰지 않는다.** 실패 시 토스트 + 화면의 낙관적 변경 되돌리기
- `localStorage`는 **서버 저장 성공 후에만** 갱신 (낡은 로컬본이 폴백으로 되살아나지 않게)
- **"못 읽었다"를 "비어 있다"로 해석하지 말 것** — #22·#26·#27이 전부 이 한 가지 실수에서 나왔다

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

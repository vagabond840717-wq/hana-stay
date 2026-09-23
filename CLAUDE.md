# HANA STAY — Claude 작업 참조 파일

---

## ⏩ "이어서 해" 규칙 — 제일 먼저 읽을 것

사용자가 목적어 없이 **"이어서 해"** · **"하던 거 계속"** · **"계속해"** 라고 하면,
**[`docs/NEXT.md`](docs/NEXT.md) 를 먼저 읽고** 거기 적힌 *"지금 바로 할 일"* 부터 시작한다.
**무엇을 할지 되묻지 않는다** — 목적어는 그 파일에 적혀 있다.

**⚠ 작업을 마칠 때마다 `docs/NEXT.md` 를 덮어쓴다.** 항상 최신 한 장만 유지한다.
안 고쳐두면 다음 세션이 엉뚱한 걸 이어받는다.

**⚠ 제목은 `MM/DD · 쉬운 이름` 형식으로 붙인다.** 사용자가 "9월 23일 그거" 하고 떠올릴 수 있어야 한다.
코딩 용어 금지 — `수첩이 진짜 예약을 지우던 문제` 처럼 사용자가 겪은 증상으로 적는다.

**⚠ 덮어썼으면 사용자에게 한 줄로 알린다** — 사용자가 세이브 포인트를 모르면 이 장치는 무용지물이다.
```
💾 세이브 포인트  09/24 · 트립 예약을 블락에 표시하기
   새 세션에서 "이어서 해" 라고만 하면 됩니다
```

**⚠ 세션 제목은 사용자에게 물어서 정한다.** 자동 생성된 제목은 사용자 기억과 안 맞는다.
할 일이 뚜렷해지는 첫 순간에 **후보 2~3개를 제안하고 고르게 한 뒤** 제목을 바꾼다.
형식은 세이브 포인트와 같다 — `MM/DD · 쉬운 이름`, 코딩 용어 금지.
(도구: `mcp__ccd_session_mgmt__set_session_title`)

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
| GET | `/daylock` | 트립 당일잠금 판정 상태 (진단용) |
| POST | `/daylock/backfill` | 굳은 가짜 소급 교정 (`?dry=1` 예행) |
| GET | `/ledger?from=&to=` | 과거 확정 원장 조회 (`YYYY-MM`, 생략 시 최근 13개월) |
| POST | `/ledger/freeze` | 지금 굳히기 (진단용. 안전창 밖이면 skip) |
| GET | `/changes?room=&ch=&limit=` | 피드 변화 기록 조회 — 무엇이 언제 나타나고 사라졌나 |
| GET | `/?url=<ical>&fix=tr\|bk&room=<호실>` | 청소앱용 — 교정본 iCal (아래 참조) |

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

### tr_daylock (트립 당일잠금 판정 — 워커 전용, 2026-09-02)
```js
// KV: tr_daylock  (⚠ extra_ 접두사 없음).  워커만 쓴다 — 두 앱은 읽지 않는다
{ day:"20260902",
  seen:{ "402 jnj":true, ... },                        // 안전창(07:30~17:30) 관측: 그날 밤이 덮여 있었나
  fake:{ "402 jnj":{ night:"20260901", min:1080 } } }  // 가짜 판정 + '잠긴 밤' + 감지 시각(KST 분)
// 왜: 트립 '전략 제어·당일 객실 마감'이 매일 17:59:59에 그날을 만실로 바꾸고 07:00에 되돌린다(#28).
//     그 가짜 하룻밤이 이웃 예약과 맞닿으면 한 덩어리로 합쳐져 입실이 당겨지거나 퇴실이 밀려 보인다.
// 자물쇠 2개가 모두 맞을 때만 걷어낸다:
//   ① seen[room] === false          (낮에 비어 있었다)
//   ② KST 18:00~18:10 에 처음 등장   (자동 규칙의 지문. 실측 18:00:57~18:01:01)
//   → 하나라도 어긋나면 원본 그대로. 오버부킹이 나는 방향으로는 틀리지 않는다
// ⚠ fake 는 '오늘'이 아니라 '잠긴 밤'을 기억한다. '오늘' 기준이면 자정에 초기화돼 새벽 0~7시가 다시 틀린다.
// 적용: 장부 저장 전 / GET /bookings(예약앱) / GET /?url=&fix=tr&room=(청소앱 옵트인)
//       / GET /ical/<호실>/ab (에어비앤비 전용 주소만, 2026-09-02)
// ⛔ 레거시 /ical/<호실> 에는 절대 적용 안 함 — 누가 구독 중인지 모르는 공용 문서다
// ⛔ /bk · /tr · /lv 도 적용 안 함 — 그 밤을 팔 수 있는 채널이 하나뿐이어야 채널 간 경쟁이 없다.
//    트립은 자기가 잠갔고 부킹·리브는 막힌 채로 둔다. 조건을 넓히려면 별도 승인
// ⛔ fix 없는 /?url= 은 원본 그대로 — #28 판별 순서 1번(원본 대조)을 오염시키지 말 것
// 소급 청소: POST /daylock/backfill (?dry=1 예행). 상세: docs/features/trip-daylock.md
```

### 앞잘림 복원 (untrim) — 새 저장소 없음, 아카이브를 읽기만 함 (2026-09-05)
```js
// 왜: 부킹·트립은 iCal 로 '예약'이 아니라 '지금부터 못 파는 날'을 보낸다.
//     SUMMARY 가 성격을 말한다 — bk 'CLOSED - Not available' / tr 'RoomStatus Fully booked' (재고)
//     vs ab 'Reserved' (예약). 재고를 보내는 두 채널만 지난 밤을 잘라낸다.
//     앱은 '막힘이 시작하는 칸'을 체크인으로 읽어서 → 어제 들어온 손님이 오늘 또 체크인으로 보인다.
// 교정 둘 (worker.js — 판정은 워커 한 곳에만 둔다):
//   ① untrimSegs        아카이브에 '끝 날짜 같고 시작만 더 이른' 조각 + 피드 시작이 오늘·어제 → 되돌림
//                       (bk·tr 만. ab 는 안 자름)
//   ② restoreTodayCheckouts  아카이브의 '퇴실 == 오늘' 조각이 피드에 없으면 되살림 (네 채널 전부)
// 적용: GET /bookings(예약앱) / GET /?url=&fix=tr|bk&room=(청소앱 옵트인)
// ⛔ exportIcal 은 synced_bookings 원본만 읽는다 → 내보내기 무영향. 되살린 조각은 절대 안 나간다
// ⛔ 앞날 것은 절대 되살리지 않는다 — 아카이브엔 취소분이 남아 있다 (#16)
// 안전 근거: 되살아나는 밤은 전부 지난 밤, 퇴실일은 cout-exclusive → 판매 가능일이 안 바뀐다
//            (시험: '오늘 이후 시작하는 조각이 새로 생기는지' 전수 0건)
// 상세: docs/05-known-issues.md #29
```

### 되살리기 유령 차단 (ghost-restore-guard) — 새 저장소 없음, 표시 단계에서만 거름 (2026-09-21)
```js
// 왜: 아카이브는 '한 번이라도 본 모든 모양'이 쌓이는 곳이다. 옛 버전·취소분·채널 간 메아리가 섞여 있다.
//     그걸 화면 메우는 데 쓰면서 "이미 있나"를 **날짜 완전일치**로만 봤다 → 하루만 줄어도 남남 취급.
//     601호 실측: 진짜 ab 9/18~9/20 옆에 아카이브 9/18~9/21 이 주입되고,
//     앱의 '포함 관계 정리'가 넓은 쪽(가짜)을 남기고 진짜를 지웠다.
//     9/20 칸에는 bk 9/17~9/20 · tr 9/9~9/20 까지 붙어 퇴실 도장이 3개 찍혔다.
//
// ⚖ 판정 근거는 하나다 — **한 방에 한 팀** (사용자 확인 2026-09-21).
//    같은 날 두 채널이 동시에 퇴실하는 일은 없다. 그래서 겹치면 진짜가 아니다.
//
// 교정 둘 (들어오는 문이 두 개다. 한 곳만 고치면 반만 막힌다):
//   ① worker.js  shouldRestoreSeg — '오늘 퇴실'분
//      (가) 그 방 **어느 채널이든** 겹치는 조각이 피드에 있으면 안 꺼낸다 (옛 '완전일치'를 대체)
//      (나) 그 방에 '오늘 퇴실'이 이미 표시돼 있으면 안 꺼낸다 (채널 무관)
//      ⛔ (가)를 '같은 채널'로 좁히지 말 것 — 603호를 놓친다
//         (bk 9/18~9/21 메아리가 ab 9/15~9/20 과 채널을 가로질러 겹침). 실제로 이렇게 짰다가 걸렸다
//   ② 예약앱 mergeArchiveIntoRooms — '지난 날짜'분
//      부킹·트립·리브 차단의 **퇴실일이 같은 방 에어비앤비 예약의 퇴실일과 같으면 메아리** → 안 꺼낸다
//      근거: 메아리는 채널이 **우리 퇴실일까지** 막아서 생긴다 → 끝 날짜가 반드시 일치한다.
//            시작은 채널이 이웃 블록과 뭉쳐 달라진다
//      ⛔ '겹치기만 하면 제외'로 넓히지 말 것 — 트립 뭉침 때문에 진짜 손님이 낀 덩어리까지 사라진다
//         (402호 tr 7/9~7/24 등 실측)
//
// 적용: GET /bookings · GET /?url=&fix=tr|bk&room= · 예약앱 과거 표시
// ⛔ exportIcal 무영향 (synced_bookings 원본만 읽음) / calcOccupancy 무영향 (아카이브 직접 읽음)
// ⛔ 청소앱은 아카이브를 아예 안 쓴다 — 고칠 게 없다 (확인함)
// 한계: 진짜 예약이 피드에서 이미 사라진 뒤면 겹칠 상대가 없어 못 막는다 (401호 9/3~9/18, #31)
//       → past-ledger 가 맡는다
// 상세: docs/features/ghost-restore-guard.md · docs/05-known-issues.md #32
```

### feed_changes (피드 변화 기록 — 워커 전용, 2026-09-22)
```js
// KV: feed_changes  (⚠ extra_ 접두사 없음). 워커만 쓴다 — 두 앱은 읽지 않는다
[{ ts:1790..., room:"603 jnj", ch:"ab", kind:"in"|"out", cin:"20260915", cout:"20260920", sum:"Reserved" }]
// 왜: '무엇이 언제 사라졌나'를 매 5분 계산해놓고 버려 왔다 (syncAllRooms 의 cancelled — 쓰는 데가 없었다).
//     예약에 이름표가 없어 생기는 사고(#29·#31·#32·#34)의 해법은 전부 '언제 무엇이 바뀌었나'인데
//     그 기록이 어디에도 없었다. 알림 로그(events)로는 대체 불가 — 50개 상한(≈11일치)이고,
//     'not available' 필터 때문에 부킹닷컴이 통째로 빠지며, 사라짐은 아예 안 적는다.
// 재료는 synced_bookings **전후 비교**뿐 — 아카이브·원장·경계·내보내기를 건드리지 않는다.
// ⛔ 사용자 알림(events)에 넣지 말 것 — 정상 퇴실도 피드에서 사라지므로 전부 '취소'로 오인된다.
//    화면에 안 나오는 관측 기록이다. 푸시·배지·달력 영향 0.
// ⛔ 변화가 있을 때만 KV 에 쓴다 (KV write 한도 원칙). 조용한 tick 은 읽지도 않는다.
// ⚠ 동기화가 짧은 간격으로 두 번 돌면 같은 변화가 두 줄 적힐 수 있다 (KV 읽기 시차). 기록만의 문제.
// 상한 3000줄 / 120일.  조회: GET /changes?room=&ch=&limit=  (최신이 앞)
```

### 아카이브 정리 규칙 — '앞잘림 지문'만 지운다 (2026-09-23, #34)
```js
// 지울 조건 — 세 개가 전부 맞을 때만:
//   ① 채널이 bk 또는 tr      (ab 는 앞을 안 자른다)
//   ② 끝날짜가 같다
//   ③ 시작만 늦어졌다
// ⛔ "더 넓은 것 안에 완전히 포함되면 지움"으로 되돌리지 말 것 — 진짜 예약을 지운다.
//    601호 부킹 7개월짜리 '안 파는 기간' 하나가 그 안의 진짜 예약 9건을 삼켰다 (실측 12건 복구).
// ⚠ 두 곳에 복제돼 있다 (#2) — worker.js 아카이브 병합 · 예약앱 mergeArchiveIntoRooms. 동시 수정.
```

### 입실일 복원 — 재료는 '어제 원장' (2026-09-23, #35)
```js
// ledgerCin(prevSegs, cinStr, todayStr, prevStr)
//   어제 원장에서 '어젯밤을 덮고 오늘 이후까지 이어지는' 조각을 찾아 그 입실일을 물려준다.
//   ⚠ buildDaySegs 의 입실일 상속과 **같은 규칙**이다. 판정은 이 함수 한 곳에만 (#2).
// ⛔ 피드의 퇴실일과 비교하지 않는다 — 투숙 중 연장·단축에 연결이 끊기면 안 된다.
//    (수첩 방식은 '끝날짜 같은 줄'로 찾아서 연장 시 2일 밀리고, 단축 시 복원 자체가 실패했다)
// ⛔ 어제 원장의 퇴실일이 '오늘'이면 잇지 않는다. '같은 손님의 당일 연장'과 '한 팀 나가고 새 팀
//    입실'은 날짜만으로 구분이 안 된다(입력이 완전히 동일). 잘못 이으면 **그날 교대 청소가
//    일정에서 사라진다.** 끊어 보는 쪽이 안전하다. 당일 연장은 수동 블락으로 잡는다.
// 적용: GET /bookings · GET /?url=&fix=.  원장에 어제 칸이 없으면 수첩(untrimSegs)으로 폴백.
// ⛔ exportIcal 무영향 — 앞잘림은 지난 밤만 깎는데 지난 밤은 팔 수 없다.
```

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
- [ ] **`toISOString()`으로 날짜 문자열을 만들지 않았는가?** → UTC라 KST 새벽~오전 9시에 하루 밀린다 (#30)
      `new Date('YYYY-MM-DD')` 도 UTC 자정 파싱이다. 앱 세 개 모두 `getFullYear/getMonth/getDate` 로컬 방식
- [ ] `render()` 호출 후 `attachCellClicks()` 체인 유지되는가?

수정 후:
- [ ] **예약앱을 고쳤다면 — 사용자 화면에 바로 안 뜬다.** GitHub Pages 가 `Cache-Control: max-age=600`
      으로 내려주므로 **브라우저 캐시가 최대 10분**이다. `?v=<난수>` 를 붙여 받으면 **새 코드가
      보이는 게 당연하므로**, 그것만으로 "배포 반영됨"이라고 말하면 사용자 화면과 다른 소리를
      하게 된다 (2026-09-21 실제로 어긋났다). 강력 새로고침(Ctrl+Shift+R)을 안내할 것.
      ⚠ 옛 기록의 "서비스워커가 붙들고 있다"는 **사실이 아니다** — `sw.js` 에 `fetch` 처리가
      아예 없어 문서를 캐시하지 못한다 (2026-09-23 확인). 푸시 알림 전용이다.
      청소앱(Cloudflare Pages)은 PWA가 아니라 무관
- [ ] **`docs/NEXT.md` 덮어쓰기** — 다음 세션이 "이어서 해" 로 이어받는 유일한 창구다
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

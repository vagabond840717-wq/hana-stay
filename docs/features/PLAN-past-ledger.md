# 과거 확정 원장 (past-ledger) 구현 계획

스펙: [DRAFT-past-ledger.md](DRAFT-past-ledger.md)

## 설계 확정 사항 (코드 확인 결과)

**끼워 넣을 지점이 한 곳뿐이다.** `cellTypeFor` 호출은 예약앱 전체에서 `cellClsAndLblBase()` 한 함수 안 5줄이 전부다.
→ 렌더 루프·색 규칙·`bothLbl`·이중셀(`c-dual`) 로직을 **전혀 건드리지 않는다.**

```js
// booking/index.html  cellClsAndLblBase() 현재
const ab=cellTypeFor(room.bookings||[],d,y,m);
const bk=cellTypeFor(room.bkBookings||[],d,y,m);
const tr=cellTypeFor(room.trBookings||[],d,y,m);
const lv=cellTypeFor(room.lvBookings||[],d,y,m);
```

**저장 형태는 "날짜 → 그 날에 걸친 조각들"** 로 한다. 체크아웃만 있는 날(`8/11` = 앞 손님 아웃 + 새 손님 인)을 잃지 않으려면
`cin ≤ X < cout` 만으로는 부족하고 **`cin ≤ X ≤ cout` 로 걸치는 조각을 전부** 담아야 한다.

```js
// KV: ledger_2026-08
{
  "601 jnj": {
    "20260811": { "tr": [ {"cin":"20260808","cout":"20260811"}, {"cin":"20260811","cout":"20260813"} ] },
    "20260812": { "tr": [ {"cin":"20260811","cout":"20260813"} ] }
  }
}
// KV: ledger_state   { "frozenThrough": "20260814" }
```

- 조각이 달을 넘겨도 **날짜 단위로 담기므로** 월 파일 소속이 모호해지지 않는다
- `cin` 을 담으므로 `bkKey`(비밀번호·메모) 복원 가능 — 조각별 분리 유지(#13)
- 수동 블락은 `manual_blocks` 가 이미 영구 보관 → **원장에 넣지 않는다**
- 용량: 9호실 × 31일 × 1~2조각 ≈ **월 20~25KB**

---

## 작업 단위 (순서대로)

### ▣ A단계 — 워커 (화면 무변화)

#### Task A1: 원장 조립 헬퍼 + `applyCuts` 이식
- **파일**: `ical-proxy/worker.js`
- **변경 내용**:
  - `applyCutsForRoom(segs, cuts)` 신설 — 예약앱 `applyCuts()` 와 **동일 규칙**: `cin < 경계 < cout` 일 때만 조각으로 치환, 경계가 `cin`·`cout` 과 같으면 무동작
  - `buildDaySegs(env, dayNum)` 신설 — 그 날짜에 걸친 조각을 호실×플랫폼별로 조립
    - 재료: `synced_bookings` + `booking_archive`(`cout < 오늘` 인 것) + `extra_tr_cuts`
    - 아카이브 병합·포함관계 정리는 **기존 `syncAllRooms` 의 규칙을 그대로 재사용**
- **예상 라인**: ~70줄

#### Task A2: 굳히기 실행 (안전창 + 멱등)
- **파일**: `ical-proxy/worker.js`
- **변경 내용**:
  - `freezeLedger(env)` 신설. `scheduled` 핸들러에서 `syncAllRooms` **완료 후** 호출
  - **안전창 검사**: KST `07:30 ~ 17:30` 이 아니면 즉시 반환
    - 근거: #28 — 트립이 `17:59:59` 에 그날을 만실로 바꾸고 다음날 `07:00` 에 되돌린다. 그 사이 피드는 퇴실일이 하루 밀려 있다
  - `ledger_state.frozenThrough` 다음날부터 **어제(KST)** 까지 순회. 이미 굳은 날은 건너뜀
  - **1회 실행당 최대 7일**로 제한 (장기 미가동 후 폭주 방지)
  - 굳힐 날이 없으면 **KV write 자체를 생략** (한도 원칙)
- **예상 라인**: ~60줄

#### Task A3: 조회 엔드포인트
- **파일**: `ical-proxy/worker.js`
- **변경 내용**: `GET /ledger?from=YYYY-MM&to=YYYY-MM` — 월 파일들을 합쳐 1회 응답. `cors()` 적용
- **예상 라인**: ~25줄

#### Task A4: 소급 채우기 (1회성)
- **파일**: `ical-proxy/worker.js`
- **변경 내용**:
  - `POST /ledger/backfill` — `booking_archive` 13개월치로 과거 원장 생성
  - **이미 굳은 날은 절대 덮어쓰지 않는다** (멱등)
  - 완료 후 `ledger_state.frozenThrough` 를 어제로 설정
- **예상 라인**: ~45줄
- **한계**: 아카이브에 취소분이 섞여 있어(601호 `7/30→8/3` 취소건 실재) 소급분은 부정확할 수 있다. 사용자 결정으로 표식·수정 UI는 만들지 않는다

---

### ▣ B단계 — 예약앱 (화면 변화 시작)

#### Task B1: 원장 로드 + 조회 헬퍼 + 전환 플래그
- **파일**: `booking/index.html`
- **변경 내용**:
  - `const LEDGER_ENABLED = true;` — **검증용 스위치.** 전/후 셀 덤프 대조 시 이 값만 바꿔 비교
  - `let ledgerData = {};`
  - `loadLedger()` — `GET /ledger?from&to` (13개월). `init()` 의 `Promise.all([triggerSync(), loadArchive()])` 에 합류
  - `ledgerSegs(room, y, m, d)` — 그 날짜가 **굳은 과거**면 `{ab,bk,tr,lv}` 반환, 아니면 `null`
    - 판정: `dnum < todayNum` **그리고** `dnum <= frozenThrough`
- **예상 라인**: ~45줄

#### Task B2: 셀 판정 분기 (핵심 1줄짜리 변경)
- **파일**: `booking/index.html` `cellClsAndLblBase()`
- **변경 내용**:
  ```js
  const L = LEDGER_ENABLED ? ledgerSegs(room,y,m,d) : null;
  const ab=cellTypeFor(L ? L.ab : (room.bookings||[]),d,y,m);
  const bk=cellTypeFor(L ? L.bk : (room.bkBookings||[]),d,y,m);
  const tr=cellTypeFor(L ? L.tr : (room.trBookings||[]),d,y,m);
  const lv=cellTypeFor(L ? L.lv : (room.lvBookings||[]),d,y,m);
  ```
  - `abBlocks`(에어비앤비 블락)·`blockTypeFor`(수동 블락)는 **그대로 둔다** — 원장 대상 아님
- **예상 라인**: ~6줄

#### Task B3: 청소 횟수 · 가동률을 원장 기준으로
- **파일**: `booking/index.html` `calcCleanings()`, `calcOccupancy()`
- **변경 내용**:
  - `calcCleanings`: 그 달 원장에서 **고유 조각**(`cin|cout` 중복 제거)을 모아 `cout` 이 그 달인 것을 센다
    - 현재는 `archiveData` 의 통짜를 세어 **601호 8월이 4회**(누락 2 + 허위 1). 원장 기준이면 **5회**
  - `calcOccupancy`: 원장의 날짜 키를 세는 방식으로 전환 (취소분 혼입 제거)
  - **폴백**: 그 달 원장이 없으면 기존 `archiveData` 경로 그대로 (`noData` 판정 유지)
- **예상 라인**: ~55줄

---

### ▣ C단계 — 문서

#### Task C1: 문서 갱신
- `docs/03-data-model.md` — `ledger_YYYY-MM`·`ledger_state` 구조 추가
- `docs/04-apps-spec.md` — 과거/미래 렌더 분리, **청소앱 범위 제외(의도)** 명시
- `docs/05-known-issues.md` — 신규 항목(트립 앞잘림 → 경계 무효화), 청소 횟수 축소집계 기록, #16 관련 갱신
- `CLAUDE.md` — 원장 키 + 안전창 규칙 추가

---

## 변경 파일 목록

- [ ] `ical-proxy/worker.js` — 원장 조립·굳히기·조회·소급채우기 (A1~A4)
- [ ] `booking/index.html` — 원장 로드·셀 판정 분기·통계 전환 (B1~B3)
- [ ] `docs/03-data-model.md` — 데이터 구조
- [ ] `docs/04-apps-spec.md` — 렌더 규칙 + 청소앱 제외 명시
- [ ] `docs/05-known-issues.md` — 신규/갱신 항목
- [ ] `CLAUDE.md` — 참조 갱신
- [ ] ~~`jnjhana/index.html` (청소앱)~~ — **변경 없음** (사용자 결정)

**청소앱을 안 건드려도 안전한 이유**: 원장은 **신규 KV 키**이고 청소앱은 이 키를 읽지 않는다.
기존 키(`tr_cuts`·`manual_blocks`·`synced_bookings`·`booking_archive`)는 **전부 무변경**이므로 #24 같은 조용한 어긋남이 생기지 않는다.

---

## 롤백 방법

| 단계 | 롤백 |
|---|---|
| A단계 (워커) | `git revert` 후 push → 자동 재배포. **원장 KV 키는 아무도 안 읽으므로 남아 있어도 무해** |
| B단계 (예약앱) | 1차: `LEDGER_ENABLED = false` 로 되돌려 push (즉시 기존 동작) / 2차: `git revert` |
| 소급 채우기 | 잘못 채워졌으면 `ledger_YYYY-MM` 키 삭제 + `ledger_state` 되돌린 뒤 재실행 |

**배포 전 각 저장소의 `git log -1` 해시를 기록해 사용자에게 알린다.**

---

## 테스트 시나리오

### A단계 (워커) — 배포 후
- [ ] 안전창 밖(예: KST 20시)에 cron이 돌면 **굳히기가 실행되지 않는다**
- [ ] 안전창 안에서 1회 실행 후 `GET /ledger` 에 어제 날짜가 생긴다
- [ ] 같은 날 두 번째 cron에서 **KV write가 발생하지 않는다** (멱등·한도)
- [ ] 601호 어제 칸의 조각이 트립 예약 목록과 일치한다
- [ ] `GET /ical/601 jnj` 응답이 **변경 전과 바이트 단위로 동일** (내보내기 무영향)

### 소급 채우기 후
- [ ] 601호 `8/11` = 아웃/인, `8/12` = 숙박중, `8/13` = 아웃/인 으로 복원
- [ ] 이미 굳은 날은 덮어써지지 않는다 (2회 실행해도 결과 동일)

### B단계 (예약앱) — 배포 **전** 검증
- [ ] 배포본에 수정 함수만 주입 → `LEDGER_ENABLED` false/true 로 **전체 셀 클래스 덤프 대조**
  - 오늘 이후 칸: **변경 0칸** 이어야 함
  - 과거 칸: 601호 트립 구간만 변경, 나머지 호실 변경 0칸
- [ ] `render()` 후 `attachCellClicks()` 체인 유지 — 과거 칸 클릭 시 비밀번호/메모 정상 조회
- [ ] 조각별 `bkKey` 분리 유지 — `8/11` 칸과 `8/13` 칸의 메모가 서로 다르게 열린다
- [ ] 601호 8월 청소 횟수 **4회 → 5회**, 청소비 **140,000 → 165,000원**
- [ ] 오버부킹 표시가 과거 날짜에 새로 생기지 않는다 (#16 규칙 유지 확인)
- [ ] 원장 로드 실패 시 기존 동작으로 폴백 (화면 안 깨짐)

---

## 예상 주의사항

| 항목 | 내용 |
|---|---|
| **#28 (최우선)** | **안전창을 어기면 틀린 상태가 영구 고착된다.** KST 18:00~07:00 굳히기 금지 |
| **UTC 시한** | 트립 `DTSTART = UTC 어제`. 날짜 `X` 는 KST `X+2` 09:00 이면 피드에서 사라진다 → `X+1` 낮에 굳히면 15~25시간 여유 |
| #2 | `applyCuts` 가 3번째 복제됨 (예약앱·청소앱·워커) — **수정 시 3곳 동시 확인** 필요. 문서에 명시 |
| #16 | 소급분에 취소 예약 혼입 → **과거 오버부킹 판정 제외는 그대로 둔다** |
| #23 | `_raw_trBookings` 무효화 경로는 **건드리지 않는다.** 원장은 별도 배열로만 쓴다 |
| #22·#26 | 원장은 **워커만 쓰고 앱은 읽기만** → 기기 간 덮어쓰기 위험 없음. `saveListMerged` 불필요 |
| KV 한도 | 굳힐 날 없으면 write 생략. 정상 운영 시 **하루 1회** |
| 렌더 체인 | `cellClsAndLblBase` 만 수정하므로 `render()` → `attachCellClicks()` 체인 영향 없음 |
| 배포 | 푸시 = 즉시 배포. A단계와 B단계를 **같은 날 함께 올리지 않는다** |

---

## 배포 순서

```
1. A1~A3 배포        → 화면 무변화. 며칠 쌓이는지 관찰
2. A4 소급 채우기 1회  → GET /ledger 로 601호 복구 확인 (아직 화면엔 무영향)
3. B1~B3 배포        → 전체 셀 덤프 대조 후 push
4. C1 문서 정리
```

**2번까지는 화면이 전혀 바뀌지 않으므로 언제든 중단할 수 있다.**

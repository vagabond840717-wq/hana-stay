# 분할 자동 승계 (split-inherit) 구현 계획

> 스펙: `DRAFT-split-inherit.md` (승인 완료)
> 수정 대상: `app/JnJ booking/index.html` **1개 파일만** (청소앱·워커 수정 없음)

## 설계 요점 (코드 기준)

- **스냅샷**: 매 로드 성공 시 호실별 Trip.com 원본 통짜 목록을 저장.
  - localStorage `hana_feed_prev` + KV `/extra?key=tr_feed_prev` (기존 `tr_splits` 저장 패턴 복제, L749~764)
  - 구조: `{ "402호": [ {cinY,cinM,cinD,coutY,coutM,coutD}, ... ], ... }` — 피드 원본만 (아카이브 병합 전)
  - KV POST는 **JSON이 직전과 다를 때만** (write 한도 원칙)
- **승계 판단 시점**: `loadBookingsFromKV()`(L1000)에서 `room.trBookings` 세팅 직후. 아카이브 병합(`mergeArchiveIntoRooms`)이 과거 예약을 섞기 전의 순수 피드로 비교.
- **잠정 분할** = 기존 `tr_splits` 항목에 `decided:false` + `autoEdges:[...]`(자동 추가된 경계 표시, "연장이에요" 버튼용). 구조 호환 — 청소앱은 기존 `applySplits`가 orig 일치로만 매칭하므로 **수정 없이 조각 표시됨**.
- **미승계 변화 기록(attention)**: 승계 근거가 없는 변화는 `hana_feed_prev`와 같은 저장소의 `attention` 배열에 기록 → 통짜에 ⚠ 배지. 사용자가 상세에서 처리하면 제거.
- **로드 실패 시**: `loadBookingsFromKV` try/catch로 조기 반환 → 승계·스냅샷 갱신·무효화 전부 실행 안 됨 (기억 보존 원칙 자동 충족).

## 작업 단위 (순서대로)

### Task 1: 스냅샷 저장/로드 기반
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - 전역 `feedPrev = {}`, `feedAttention = []` 선언 (L690 부근, trSplits 옆)
  - `saveFeedPrev()` / `loadFeedPrev()` 함수 추가 — `saveSplits`/`loadSplits`(L749~764) 패턴 복제, 단 `saveFeedPrev`는 직전 저장 JSON과 비교해 같으면 KV POST 생략
  - `init()`(L2222)의 `await loadSplits()` 다음에 `await loadFeedPrev()` 추가
- 예상 코드 라인: ~35줄

### Task 2: 승계 엔진 `inheritSplits()`
- 파일: `app/JnJ booking/index.html`
- 변경 내용: `applySplits` 위쪽(L779 부근)에 추가, `loadBookingsFromKV()`(L1000)의 trBookings 세팅 직후 호출
  - 호실별로 현재 피드 통짜(cur)와 스냅샷(prev) 비교 (날짜 비교는 기존 `dnum()` 재사용)
  - **cur의 각 통짜 L이 prev에 없던 모양이면**:
    1. L 안에 완전히 포함되는 prev 조각들의 안쪽 날짜(각 조각의 cin·cout 중 L 내부에 있는 것) + L 안에 포함되는 기존 split의 boundaries → 경계 후보로 수집·정렬·중복제거
    2. 경계 있음 → 잠정 split 생성/교체: `{roomName, platform:'tr', origCin:L.cin, origCout:L.cout, boundaries, decided:false, autoEdges:[새로 추가된 경계]}` → `saveSplits()`
    3. 경계 없음(근거 부족) → `feedAttention`에 `{roomName, cin, cout}` 기록 (시나리오 D)
  - L에 흡수된 기존 split은 잠정 split으로 교체됐으므로 제거 (무효 대상에서 제외)
  - 처리 후 스냅샷을 cur로 갱신 → `saveFeedPrev()`
  - 이 함수는 해당 호실 피드가 실제로 로드된 경우에만 동작 (빈 응답이면 건너뜀 — 기존 `if(!bks) return` 패턴 유지)
- 예상 코드 라인: ~80줄

### Task 3: `notifyStaleSplits()` 개편 (조용한 삭제 금지)
- 파일: `app/JnJ booking/index.html` (L821~828)
- 변경 내용:
  - Task 2에서 승계·교체된 split은 stale 판정에서 이미 빠짐
  - 남은 stale = 취소로 쪼개진 경우(시나리오 C) → 현행대로 삭제 + 토스트 유지
  - 잠정 split 생성 시 토스트 문구: `"OO호 예약이 바뀌어 경계를 잠정 유지했어요 — ⚠ 표시를 눌러 확인해 주세요"`
  - attention 기록 시 토스트: `"OO호 예약이 바뀌었어요 — ⚠ 표시를 눌러 확인해 주세요"`
- 예상 코드 라인: ~20줄

### Task 4: 달력 ⚠ 배지
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - CSS: `.attn-badge` (셀 우상단 작은 ⚠ 점, 네이비 테마에 맞는 주황/빨강 계열)
  - `needsAttention(roomName,y,m,d)` 헬퍼: 해당 날짜가 ① `decided:false` split 조각의 체크인일 또는 ② attention 통짜의 체크인일이면 true
  - 셀 생성부(`cellClsAndLbl` 결과 사용처 — 월간 `renderMonth`/주간 `renderWeek`/연속 `renderMulti`(L1418) 3곳 공통 지점)에 배지 span 추가
  - 확인 완료(`decided:true` 확정 or attention 제거) 시 다음 `render()`에서 자동 소멸
- 예상 코드 라인: ~30줄

### Task 5: 상세 모달 확인 카드
- 파일: `app/JnJ booking/index.html` (`openDetail` L1594~1618 분할 UI 부근)
- 변경 내용:
  - **잠정 split 조각 클릭 시** (`trSeg` 경로, 매칭 split이 `decided:false`): 기존 분할 편집기 위에 확인 카드 추가
    - `[✔ 이대로 확정]` → `commitSplit(..., decided:true)` (기존 함수 그대로, L872)
    - `[마지막 손님이 연장한 거예요]` → `autoEdges`에 해당하는 경계 제거 후 `decided:true` 저장 (경계가 하나도 안 남으면 split 삭제 = 통짜)
    - `[직접 수정]` → 기존 편집기 사용 (저장 시 `decided:true` — 기존 코드 L949 이미 true로 저장함)
    - `[통짜로 유지]` → 기존 분할 해제 버튼 재사용 + attention에도 안 남김
  - **attention 통짜 클릭 시** (`trOrig` 경로): 카드 `"⚠ 예약이 바뀌었어요"` + 기존 분할 편집기(이미 이 경로에 있음) + `[그대로 두기]` → attention 항목 제거 후 `saveFeedPrev()`
  - 카드에서 어떤 버튼이든 누르면 배지 해소 → `render()` + `attachCellClicks()` (commitSplit이 이미 수행, L877)
- 예상 코드 라인: ~70줄

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — Task 1~5 전부 구현 완료 (단일 파일)
  - 추가: 구현 중 발견한 **기존 잠재 버그 수정** — `mergeArchiveIntoRooms()` 재호출 시 `_raw_trBookings` 백업이 조각 배열로 덮어써져, 아카이브 로드 후 분할 수정이 다음 동기화까지 화면에 반영 안 되던 문제. 병합 시작 시 원본 복원 로직 추가.
- [x] `app/JnJ/index.html` — **변경 없음** 확인 (applySplits가 원본 일치로만 매칭 — decided/inherited 무관하게 조각 반영됨을 코드 검증)
- [x] `worker.js` — **변경 없음** (`/extra` 범용 키 재사용)

## 롤백 방법
- 예약앱 저장소에서 해당 커밋 1개 `git revert` → 즉시 이전 동작 복귀
- 데이터 잔여물 안전성:
  - `tr_feed_prev` 키: 구버전 앱은 이 키를 아예 읽지 않음 → 무해
  - `decided:false` 잠정 split: 구버전 앱에서는 일반 split과 동일하게 조각 표시됨 (배지만 없음) → 무해, 필요 시 기존 분할 해제 버튼으로 제거 가능

## 테스트 시나리오 (2026.07.15 셀프 테스트 — 스크래치 복사본 + 가짜 피드, 실서버 통신 0건 확인)
- [x] **A. 분할해 둔 통짜가 뒤로 연장**: 9~28 확장 시 경계 18·20 유지 + 24 자동 경계(autoEdges) + ⚠ 3+1개 + 토스트 ✓
- [x] **B. 분할 없던 두 조각 합쳐짐**: 9~24 통짜 도착 → 경계 18·20 잠정 분할(decided:false) + ⚠(8/9·18·20) ✓
- [x] **C. 취소로 쪼개짐**: split 삭제 + "분할이 해제됐어요" 토스트, 스냅샷 두 조각으로 갱신, 배지 없음 ✓
- [x] **C-2. 재채움**: 다시 9~24 → 경계 18·20 잠정 분할 + ⚠ (양옆 경계 보존, 가운데 확인 요청) ✓
- [x] **D. 근거 없는 변화**: 9~18 → 15~20 → attention 기록 + 8/15 ⚠ + [그대로 두기]로 해소 ✓
- [x] **확인 카드**: [이대로 확정](decided:true·배지 소멸) / [연장이에요](24 경계 제거, autoEdges 1개일 때만 노출) / 편집기 저장 / [분할 해제] 모두 동작 ✓
- [x] **배지 지속성**: 새로고침 반복에도 유지, 월간·주간·연속 3개 뷰 모두 표시 ✓
- [x] **로드 실패**: 전체 통신 차단 상태 새로고침 → split·스냅샷 무손실, 무효화 0건, 복구 후 배지 복귀 ✓
- [x] **오버부킹 오탐 없음**: 잠정 조각 상태에서 `.c-overbooking` 0개, 경계일은 `c-tr-both` 반반 셀 ✓
- [x] **KV write 절약**: 변화 없는 재로드에서 KV POST 0건 (변화 시에만 tr_splits/tr_feed_prev 각 1건) ✓
- [x] **청소앱 반영**: 코드 검증 — applySplits가 orig 일치로만 매칭, decided 무관 → 수정 없이 조각 반영 (실기기 확인은 사용자 검수 시)
- [x] **회귀**: 수첩 없을 때 자동 개입 없음, 수동 분할 생성(편집기)/해제 정상, 상세 모달 정상 ✓

## 예상 주의사항
- 월 값 **0-indexed** 일관성 (스냅샷·경계 전부 기존 Booking 객체 형식 그대로 사용)
- `render()` 후 `attachCellClicks()` 체인 — 기존 `commitSplit`(L877)과 `render()`(L1439)가 이미 처리, 새 코드에서 render 직접 호출 시 준수
- 배지 삽입 위치가 월간/주간/연속 **3개 뷰 모두** 커버하는지 구현 시 확인 (`cellClsAndLbl` 사용처 전수 확인)
- `mergeArchiveIntoRooms`가 과거 예약을 trBookings에 합치므로 스냅샷은 반드시 **그 이전**(loadBookingsFromKV 내부)에서 채취
- `reapplySplits()`(L811)는 `_raw_trBookings` 복원 기반 — 잠정 split도 동일 경로로 적용되므로 별도 처리 불필요
- known-issues #2 (두 앱 로직 분리): 승계 로직은 예약앱 전용임을 /done 단계에서 문서에 명시

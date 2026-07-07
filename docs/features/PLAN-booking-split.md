# booking-split 구현 계획 (Trip.com 합쳐진 예약 수동 분할)

기반 스펙: [DRAFT-booking-split.md](DRAFT-booking-split.md)

---

## 설계 요약 (구현 관점)
- 저장: `/extra` KV + localStorage에 새 키 `tr_splits` (배열). `manual_blocks` 패턴 그대로 복제.
- 가공: `applySplits()` — 데이터 로드 직후 `trBookings`의 통짜 1건을 조각 N건으로 **치환**.
- 기존 함수(`cellTypeFor`, `isOverbooking`, `openDetail`)는 가공된 배열을 보므로 **무수정**.
- 예약앱 = 생성/편집, 청소앱 = 반영만. 워커 = 수정 없음.

---

## 작업 단위 (순서대로)

### Task 1 — 예약앱: 분할 데이터 계층
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - 전역 `let trSplits = [];` 추가 (`manualBlocks` 옆, ~line 710).
  - `saveSplits()` / `loadSplits()` 추가 — `manual_blocks`용 `saveBlocks`/`loadBlocks`(line 751~763) 복제, 키만 `tr_splits`, localStorage 키 `hana_splits`.
  - 초기 로드 흐름에 `loadSplits()` 호출 추가 (기존 `loadBlocks()` 호출 지점 옆).
- 예상: ~25줄

### Task 2 — 예약앱: 가공 함수 applySplits
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - `applySplits(room)` 추가: `room.trBookings`를 순회, 각 예약이 `trSplits`의 `origCin/origCout`와 **완전 일치**하면 `boundaries`로 잘라 조각 배열로 치환. 불일치 split은 `invalidatedSplits`에 모아 안내용으로 표시.
  - 조각 생성 규칙: `[origCin, ...boundaries, origCout]` 인접 쌍 → `{cinY,cinM,cinD,coutY,coutM,coutD, platform:'trip', summary, _split:true}`. 앞.cout === 뒤.cin 보장.
  - `loadBookingsFromKV()`(line 805) 끝에서 `rooms.forEach(applySplits)` 호출 → 그 뒤 `render()`.
  - `triggerSync()`(line 792) 경로에서도 로드 후 `applySplits` 적용 확인.
- 예상: ~40줄

### Task 3 — 예약앱: 추천 계산 (이력 대조)
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - `fetchArchive()` 추가 — `${PROXY}/archive` GET, 결과 캐시(1회 로드 후 메모리 보관).
  - `recommendSplit(roomName, orig)` 추가: 아카이브의 해당 호실·`tr` 이력에서 `orig` 범위 안에 들어오는 조각 수집 → 경계 후보 반환. 빈틈/겹침/경계상충 판정하여 `{mode:'clean'|'ambiguous', boundaries, candidates}` 반환.
- 예상: ~50줄

### Task 4 — 예약앱: 분할 UI (openDetail 확장)
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - `openDetail()`(line 1308)에서 현재 셀이 **Trip.com 통짜 예약**(합쳐진 tr 예약, `_split` 아님)일 때만 분할 섹션 렌더.
  - `recommendSplit` 결과로 추천 카드: 이력 확인 조각 ✓ / 빠진 조각 ⚠️ 표시, 조각 날짜 편집 인풋, **[이대로 N건 나누기]** 버튼.
  - 이미 분할된 예약(`_split:true`)이면 **[분할 해제]** / **[분할 수정]** 버튼.
  - 버튼 핸들러: `trSplits`에 추가/수정/삭제 → `saveSplits()` → `applySplits` 재적용 → `render()` → `attachCellClicks()`.
- 예상: ~90줄 (UI 포함)

### Task 5 — 예약앱: 경계 변화 감지 + 확인 화면
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - Task 3의 `recommendSplit`이 `mode:'ambiguous'`(예전 빈칸이 채워짐 / 경계 버전 상충 / 기존 분할 원본 변경)일 때, Task 4에서 추천 카드 대신 **확인 화면**(후보: 연장형 / 원래대로 / 직접입력 / 통짜유지) 렌더.
  - 사용자가 고르면 `decided:true`로 `trSplits` 저장 → 다음 로드부터 재질문 안 함.
- 예상: ~40줄

### Task 6 — 예약앱: 자동 무효 안내
- 파일: `app/JnJ booking/index.html`
- 변경 내용:
  - `applySplits`에서 원본 불일치로 무효화된 split 발생 시, 해당 호실/기간에 안내 배지 또는 토스트("예약이 바뀌어 분할이 해제됐어요").
  - 무효 split은 `trSplits`에서 제거하거나 `stale:true` 표시(1회 안내 후 정리).
- 예상: ~20줄

### Task 7 — 청소앱: 반영 (읽기 전용)
- 파일: `app/JnJ/index.html`
- 변경 내용:
  - `let trSplits = [];` + `loadSplits()` 추가 (`loadBlocks` line 913~919 복제, 키 `tr_splits`).
  - `applySplits(room)` 동일 추가 (예약앱과 같은 로직, 안내/무효 처리는 생략 가능 — 표시만).
  - 데이터 로드 지점(line 904) 직후 `rooms.forEach(applySplits)` → 기존 `render()`(line 909) 유지.
  - 초기화에서 `loadSplits()` 호출 추가.
  - **분할 생성 UI 없음.**
- 예상: ~45줄

### Task 8 — 문서 갱신
- 파일: `docs/05-known-issues.md`, `docs/03-data-model.md`, `docs/04-apps-spec.md`
- 변경 내용: `tr_splits` 데이터 구조, 분할 동작, 알려진 한계(호실명 변경 시 고아, 타임스탬프 부재) 반영.
- 예상: ~30줄

---

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — 분할 생성/편집/감지/무효안내 (Task 1~6) ✅
- [x] `app/JnJ/index.html` — 분할 반영 (Task 7) ✅
- [x] `docs/*.md` — 문서 갱신 (Task 8) ✅ (03-data-model, 04-apps-spec, 05-known-issues, CLAUDE.md, 스펙 확정)
- 워커: 변경 없음 ✅

---

## 롤백 방법
- 분할 로직은 **가산적**(원본 미변경). 문제 시 `applySplits` 호출부만 주석 처리하면 즉시 기존 동작으로 복귀.
- 데이터는 별도 키 `tr_splits`에만 저장 → 삭제해도 기존 예약/블락/비번 데이터 무영향.
- git: 커밋 전이면 파일 되돌리기, 커밋 후면 해당 커밋 revert.

---

## 테스트 시나리오
- [ ] 402호 `7/9~24` 통짜 클릭 → 추천 카드에 `7/9-18`✓, `7/20-24`✓, 중간 ⚠️ 표시
- [ ] [나누기] → 달력에 3건, 경계일 아웃|인 반반 셀 정상
- [ ] **오버부킹 미발생**: 분할 후 경계일(7/18,7/20) `.c-overbooking` 안 뜸
- [ ] [분할 해제] → 통짜로 복귀
- [ ] [분할 수정] → 경계 변경 반영
- [ ] 청소앱 열기 → 같은 분할 반영, 청소 횟수 정확
- [ ] 원본 피드 변경 시뮬 → 분할 자동 무효 + 안내
- [ ] 떨어진 예약이 붙은 케이스 → 확인 화면 뜨고 선택대로 처리
- [ ] 분할 안 한 다른 호실/플랫폼 → 기존과 100% 동일
- [ ] Airbnb·Booking·리브 셀 → 분할 버튼 안 뜸

---

## 예상 주의사항
- **render() 후 attachCellClicks() 체인**: 분할 확정 핸들러에서 `render()` 호출 시 `attachCellClicks()` 재등록 필수 (예약앱·청소앱 공통).
- **월 0-indexed**: `origCin/origCout/boundaries`, 아카이브 대조, 조각 생성 전부 0-indexed 일관 유지.
- **오버부킹 안전**: 반드시 원본 *치환*(제거+삽입), 절대 *추가* 금지. 조각 경계 cout-exclusive.
- **완전 일치 매칭**: origCin/origCout이 1일이라도 다르면 미적용(자동 무효) — 억지 적용 금지.
- **아카이브 로드 실패**: `/archive` 실패 시 추천 없이 "직접 입력"만 제공(기능 자체는 동작).
- **bkKey 분리**: 조각별 cin이 달라 비번 키가 나뉨(의도됨) — 기존 통짜 비번이 있으면 첫 조각으로 승계할지 여부는 implement 시 확인.
- known-issues: 호실명 변경 시 `tr_splits` 고아 (기존 한계와 동일).

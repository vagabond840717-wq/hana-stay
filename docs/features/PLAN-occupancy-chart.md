# 점유율 그래프 뷰 (occupancy-chart) 구현 계획

스펙: `docs/features/DRAFT-occupancy-chart.md` (2026.07.21 승인: 합산 박수 · 등록 순서 · 주말=금·토 밤)

수정 파일은 `app/JnJ booking/index.html` 하나. 단일 파일 안에서 CSS → 계산 함수 → 렌더 순으로 4개 Task.

## 작업 단위 (순서대로)

### Task 1: CSS 추가
- 파일: `app/JnJ booking/index.html` (L518 부근, `.stats-clean-cost` 아래 / 데스크톱 미디어쿼리 앞)
- 변경 내용:
  - `.stats-chart-wrap` — 그래프 섹션 컨테이너 (padding 0 16px)
  - `.stats-chart` — flex 세로막대 컨테이너 (height 170px, align-items:flex-end, gap 4px, border-bottom)
  - `.stats-chart-col` / `.stats-chart-bar` / `.stats-chart-bar.sel` — 막대 (연한 파랑 `#93c5fd` / 선택 `var(--navy)`, radius 4px 4px 0 0)
  - `.stats-chart-val` (선택·최고·최저만 표시되는 값 라벨), `.stats-chart-xlab` (월 라벨, 선택 시 강조)
  - `.stats-summary` — 카드 4개 flex-wrap 행, `.stats-scard` (border, radius 8px, 큰 숫자 + 부가설명)
  - `.stats-roombars` / `.stats-roombar-row` — 호실명 + 가로막대(주중 `#93c5fd` / 주말 `#1e3a8a` 2px gap 분리) + `N박 · P%` 라벨
  - 데스크톱 미디어쿼리(`@media (min-width:768px)`)에 max-width 정렬 추가
- 예상 코드 라인: ~75줄

### Task 2: 계산 함수 `calcOccupancyDetail` 신설
- 파일: `app/JnJ booking/index.html` (L2094 `calcOccupancy` 바로 아래)
- 변경 내용: 기존 `calcOccupancy`와 동일한 날짜 순회에 요일 분해만 추가
  ```js
  calcOccupancyDetail(roomName, platform, year, month)
  // 반환: { nights, wkendNights, total, wkendTotal, noData }
  // wkdayNights = nights - wkendNights, wkdayTotal = total - wkendTotal 로 파생
  // 주말 밤 판정: new Date(y, m, d).getDay() === 5 || === 6  (금·토)
  // 점유 판정: cin <= cur < cout (기존과 동일, cout exclusive)
  ```
  - `calcOccupancy`는 수정하지 않음 (기존 그리드가 그대로 사용)
- 예상 코드 라인: ~30줄

### Task 3: 그래프 + 상세 섹션 렌더
- 파일: `app/JnJ booking/index.html` (`renderOccupancy` L2235~ 확장)
- 변경 내용:
  1. 상태 변수 `let statsSelMonth = null;` 추가 (L2023 `statsView` 옆) — `{y, m}` 저장
  2. `renderOccupancy()` 안에서 months 배열 계산 후:
     - `statsSelMonth`가 12개월 범위 밖이거나 null이면 이번 달로 초기화
     - 월별로 `calcOccupancyDetail` 합산 → 평균 % 배열 생성 (noData 달은 null)
     - 그래프 HTML: 값 라벨은 선택·최고·최저 달만, noData 달은 회색 "—"
     - 요약 카드 4개: 평균 점유율 / 객실당 평균 숙박 `N박 (M일 중)` / 총 숙박 `합계 N박 중 주중 X박 · 주말 Y박` / 주중 P% · 주말 Q%
     - 호실별 가로 막대: rooms 등록 순서, 주중/주말 세그먼트 + `N박 · P%`
     - 기존 그리드 테이블은 그 아래 그대로
  3. 렌더 후 그래프 막대 클릭 리스너 등록 (innerHTML 교체마다 재등록 — 기존 tooltip 패턴과 동일):
     탭 → `statsSelMonth` 갱신 → `renderOccupancy()` 재호출
  4. 데이터 전무 시 기존 empty 메시지 유지 (그래프도 미표시)
- 예상 코드 라인: ~120줄

### Task 4: 패널 열기 시 선택 초기화
- 파일: `app/JnJ booking/index.html` (`openStatsPanel` L2312)
- 변경 내용: 패널 열 때 `statsSelMonth = null;` → 항상 이번 달부터 시작. 플랫폼 탭 전환 시에는 선택 유지 (별도 수정 불필요 — `renderStats()`만 다시 타므로 자동)
- 예상 코드 라인: ~2줄

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — CSS ~80줄 + JS ~160줄 추가 (기존 코드 수정: `openStatsPanel`, `renderCleaningStats` 그래프 영역 정리, `.stats-body`에 `#statsChartArea` 추가)
- 청소앱(JnJ)·Worker·데이터 구조: 변경 없음

## 롤백 방법
- 단일 커밋으로 작업 → 문제 시 예약앱 저장소에서 `git revert <커밋>` 한 번으로 원복
- 데이터/KV 변경이 없으므로 롤백에 따른 데이터 정리 불필요

## 테스트 시나리오 (2026.07.21 실데이터로 검증 완료)
- [x] 점유율 탭 진입 시 그래프 표시, 기본 선택 = 이번 달(진한 네이비)
- [x] 막대 탭 → 요약 카드·호실별 막대가 그 달 기준으로 갱신 (6월 클릭: 156박=93+63 확인)
- [x] 값 라벨이 선택·최고·최저 달에만 표시 (동률 시 각 1개만 — 구현 중 0% 10개월 전부 라벨 붙는 문제 발견, first-occurrence로 수정)
- [x] 주중/주말 검증: 2026년 7월 금·토 밤 9일 → 주말 78%=63/81 손계산 일치, 6월 금·토 밤 8일 → 88%=63/72 일치
- [x] 총 숙박 = 주중+주말 합 일치 (7월 161=98+63)
- [x] 플랫폼 탭 전환 시 재계산·선택 달 유지 (Booking 필터: 유효 객실 4개로 재계산 확인)
- [x] 데이터 없는 호실 "—" 표시 (플랫폼 필터 시), 데이터 전무 시 empty 메시지 경로 유지
  - 참고: 과거 달은 기존 그리드와 동일하게 0%로 표시됨 (호실에 아카이브가 있으면 noData 아님 — 기존 동작과 일관)
- [x] 청소비용 탭 전환 시 그래프 영역 비움, 복귀 시 재표시
- [x] 모바일 폭(375px) 가로 넘침 없음, 카드 4개 줄바꿈 정상
- [x] 통계 닫고 달력 복귀 — 달력 정상 렌더 (통계 오버레이와 분리 확인)
- [x] 월 경계 걸친 예약: 날짜 단위 순회 방식이라 자동 분배 (7월 합계가 그리드 셀 값과 일치)

## 예상 주의사항
- 월 값 0-indexed (`getDay()`는 요일, 월 아님 — 혼동 주의)
- `renderOccupancy()`는 innerHTML 전체 교체 → 그래프 클릭 리스너도 매 렌더 후 재등록 (기존 tooltip 리스너와 같은 위치에서)
- 달력 `render()`/`attachCellClicks()` 체인과 무관 — 통계 오버레이 내부만 변경
- `closeStatsPanel`의 tooltip 정리 로직은 `#statsGrid` 대상이므로 그래프 섹션과 충돌 없음
- 완료 후: 예약앱 저장소 커밋·푸시 + hana-stay 저장소 STATUS.md 업데이트

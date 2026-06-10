# 예약 이력 아카이브 + 연간 점유율 뷰 구현 계획

## 핵심 설계 원칙 (충돌 방지)

`synced_bookings` KV 키는 절대 수정하지 않는다.
아카이브는 별도 KV 키 `booking_archive`에만 저장한다.
기존 달력 뷰, render(), cellTypeFor(), attachCellClicks()는 일절 건드리지 않는다.

---

## 작업 단위 (순서대로)

### Task 1: Worker — /archive 엔드포인트 추가
- **파일**: `ical-proxy/worker.js`
- **변경 내용**:
  1. `syncAllRooms()` 내부에 아카이브 병합 로직 추가
     - 기존 `booking_archive` KV 읽기
     - 이번 sync에서 얻은 예약들을 UID 기준으로 병합 (중복 제거)
     - UID = `${cinY}_${cinM}_${cinD}_${coutY}_${coutM}_${coutD}_${platform}`
     - 13개월(약 400일) 이전 체크아웃 예약은 정리
     - `booking_archive` KV에 저장 (synced_bookings는 수정하지 않음)
  2. `GET /archive` 엔드포인트 추가
     - `booking_archive` KV를 그대로 반환
- **예상 코드 라인**: ~40줄 추가

```js
// booking_archive 구조
{
  "302호": {
    ab: [ {cinY,cinM,cinD,coutY,coutM,coutD,platform,summary}, ... ],
    bk: [ ... ],
    tr: [ ... ],
    lv: [ ... ]
  },
  ...
}
```

### Task 2: 앱 — 통계 패널 CSS 추가
- **파일**: `app/JnJ booking/index.html` (CSS 영역)
- **변경 내용**:
  - `.stats-panel` — 전체화면 overlay (기존 panel-overlay 패턴 동일)
  - `.stats-grid` — 12열(월) × N행(호실+전체) 그리드
  - `.stats-cell` — 색상 강도로 점유율 표현 (CSS custom property 활용)
  - `.stats-platform-tabs` — 플랫폼 토글 버튼 (전체/✈/🏨/🌐/🏡)
  - 색상 scale: 빈달(회색) → 낮음(#dbeafe) → 높음(var(--navy))
- **예상 코드 라인**: ~80줄

### Task 3: 앱 — 통계 버튼 HTML 추가
- **파일**: `app/JnJ booking/index.html` (헤더 HTML 영역)
- **변경 내용**:
  - 헤더 `header-actions`에 `📊` 아이콘 버튼 추가 (notifyBtn 왼쪽)
  - 통계 패널 div 추가 (기존 panel-overlay 패턴 그대로)
- **예상 코드 라인**: ~50줄

### Task 4: 앱 — 통계 계산 및 렌더 JS
- **파일**: `app/JnJ booking/index.html` (script 영역)
- **변경 내용**:
  - `let archiveData = {}` 전역 변수 추가
  - `loadArchive()` — `GET /archive` 호출 후 archiveData 저장
  - `calcOccupancy(roomName, platform, year, month)` — 숙박일 계산
    - platform: 'all' | 'ab' | 'bk' | 'tr' | 'lv'
    - 블락/summary 포함 예약은 제외 (not available 등)
    - cin 포함, cout 미포함 (기존 cellTypeFor와 동일 규칙)
    - 반환: `{ days: 숙박일수, total: 해당월총일수 }`
  - `renderStats()` — 12개월 × 호실별 그리드 HTML 생성
    - 좌측: 호실명 고정 컬럼
    - 상단: 12개월 헤더 (오늘 기준 -11개월 ~ 현재월)
    - 마지막 행: 전체 합산
    - 셀 탭 → 인라인 툴팁 (N박/M일 · Z%)
  - `openStatsPanel()` / `closeStatsPanel()` — 패널 열기/닫기
- **예상 코드 라인**: ~130줄

### Task 5: 앱 — 이벤트 바인딩
- **파일**: `app/JnJ booking/index.html` (events 영역)
- **변경 내용**:
  - 📊 버튼 onclick → `openStatsPanel()`
  - 통계 패널 닫기 버튼 onclick → `closeStatsPanel()`
  - 플랫폼 탭 onclick → 현재 플랫폼 변수 변경 후 `renderStats()` 재호출
  - 앱 초기화(`init()`) 시 `loadArchive()` 병렬 호출 추가
- **예상 코드 라인**: ~20줄

---

## 변경 파일 목록

- [x] `ical-proxy/worker.js` — /archive 엔드포인트 + syncAllRooms 내 병합 로직
- [x] `app/JnJ booking/index.html` — 통계 버튼/패널/CSS/JS

## 변경하지 않는 파일

- `app/JnJ/index.html` (청소 앱) — 무관
- `app/JnJ Price/index.html` (요금 계산기) — 무관

---

## 롤백 방법

- Worker: `/archive` 엔드포인트 코드 제거, `syncAllRooms` 내 아카이브 저장 코드 제거
  - `synced_bookings`를 건드리지 않으므로 기존 달력 데이터는 100% 보존됨
- 앱: 통계 버튼/패널/JS 제거
  - 기존 달력 렌더링 코드를 수정하지 않으므로 달력 뷰는 완전 무결

---

## 테스트 시나리오

### 기존 기능 회귀 테스트 (충돌 확인)
- [ ] 월간/주간/연속 뷰에서 예약이 기존과 동일하게 표시되는가
- [ ] 셀 탭 → 상세 패널(비밀번호/메모) 정상 작동하는가
- [ ] 오버부킹 감지(빨간 ⚠️) 정상 표시되는가
- [ ] 블락 표시(🔒) 정상 표시되는가
- [ ] Sync 버튼 실행 후 달력 데이터 그대로인가
- [ ] 호실 추가/삭제 후 달력 정상 동작하는가

### 신규 기능 테스트
- [ ] 📊 버튼 탭 → 통계 패널 열림
- [ ] 통계 패널에 12개월 그리드 표시됨
- [ ] 호실별 행 + 전체 합산 행 표시됨
- [ ] 플랫폼 탭(전체/✈/🏨/🌐/🏡) 전환 시 수치 변경됨
- [ ] 셀 탭 → "N박/M일 · Z%" 툴팁 표시됨
- [ ] 데이터 없는 달은 회색으로 표시됨
- [ ] 패널 닫기 후 달력 뷰 그대로 유지됨
- [ ] Sync 후 /archive 데이터가 누적되는가 (Worker 로그로 확인)
- [ ] 모바일(iOS Safari)에서 스크롤/탭 정상 동작하는가

---

## 예상 주의사항

1. **Worker 배포 필요**: Worker 수정 후 `wrangler deploy` 또는 Cloudflare 대시보드에서 배포해야 반영됨
2. **초기 아카이브 데이터 없음**: 첫 Sync 전까지 통계 뷰는 빈 상태. "아직 데이터가 없어요. Sync를 실행하면 쌓이기 시작해요." 안내 문구 필요
3. **known-issues #5 (KV 저장 실패)**: 아카이브 저장 실패 시 조용히 무시함. 기존 방침 유지 (이번 범위에서 해결 안 함)
4. **known-issues #1 (호실명 변경 시 고아)**: 호실명이 아카이브 키로도 사용되므로 동일 문제 존재. 이번 범위 밖.
5. **render() 후 attachCellClicks() 체인**: 통계 패널은 render()를 호출하지 않으므로 해당 없음

# HANA STAY — 작업 현황

## 마지막 작업
- **날짜**: 2026-07-08
- **기기**: 사무실 컴퓨터 (Claude Code 데스크톱)
- **한 것**: 버그 3건 수정 (known-issues #15, #16, #17)
  - 워커 아카이브 필터 플랫폼별 규칙으로 변경 → 부킹닷컴 기록 안 남던 문제 해결 (`ical-proxy/worker.js`)
  - 지난 날짜 가짜 오버부킹 경고 제거 — `isOverbooking`에 과거 제외 조건 추가 (`app/JnJ booking/index.html`)
  - Trip.com 일일 스냅샷 중첩 제거 — 포함 관계 항목 자동 정리, 워커·예약앱 양쪽 (과거 달력 '인' 마커 도배 해결)
  - 아웃+인 조합 셀 통일 — 서로 다른 플랫폼/블락 조합도 에어비앤비식 반반 셀 + 플랫폼 아이콘 (`c-mix-*`, 기존 c-dual 남색 혼동 해소)
  - ⚠ 두 저장소(ical-proxy, booking) 푸시해야 배포 반영됨

## 다음에 할 것
- 집 컴퓨터에 Claude Code 설치 + 저장소 연결

## 진행 중인 기능
- 없음 (미완성 작업 없음)

## 미완성 / 보류 중
- `docs/features/DRAFT-cleaning-stats.md` — 청소 통계 기능 스펙 작성 중

---
> 새 세션 시작 시 Claude에게: "STATUS.md 읽고 어디까지 했는지 알려줘"

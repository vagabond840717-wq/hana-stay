# HANA STAY — 작업 현황

## 마지막 작업
- **날짜**: 2026-07-15
- **기기**: 사무실 컴퓨터 (Claude Code 데스크톱)
- **한 것**: 분할 자동 승계(split-inherit) 기능 완성 + 배포
  - Trip.com 피드가 합쳐져 와도 직전 스냅샷(`tr_feed_prev`) 비교로 알던 경계를 잠정 유지 + ⚠ 배지 (`app/JnJ booking/index.html`)
  - 확인 카드: 이대로 확정 / 마지막 손님 연장 / 직접 수정 / 분할 해제 — 확인 전까지 배지 유지
  - 통신 실패 시 판정 보류(기억 보존), KV write는 변경 시에만
  - 부수 수정: 아카이브 병합 재호출 시 분할 백업(_raw) 오염 버그 (known-issues #19 해결)
  - `booking` 저장소 push 완료(b3bc44c), Pages 반영 확인. 청소앱·워커 수정 없음
  - 문서 확정: `docs/features/split-inherit.md` + 03/04/05/CLAUDE.md 갱신

## 다음에 할 것
- **실전 검수 (사용자)**: 다음 트립닷컴 예약 변동 시 ⚠ 배지·잠정 분할이 실제로 뜨는지 확인, 처음 며칠은 트립닷컴 앱과 대조
- 집 컴퓨터에 Claude Code 설치 + 저장소 연결

## 진행 중인 기능
- 없음 (미완성 작업 없음)

## 미완성 / 보류 중
- `docs/features/DRAFT-cleaning-stats.md` — 청소 통계 기능 스펙 작성 중

---
> 새 세션 시작 시 Claude에게: "STATUS.md 읽고 어디까지 했는지 알려줘"

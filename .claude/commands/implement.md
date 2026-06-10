---
description: 승인된 계획서대로 구현. Task 단위로 진행하며 각 Task 완료 후 보고.
---

# 구현 진행: $ARGUMENTS

`docs/features/PLAN-$ARGUMENTS.md` 를 읽고 Task 순서대로 구현한다.

## 구현 원칙

1. **Task 단위로 진행** — 한 Task 완료 후 요약 보고, 다음으로 넘어감
2. **청소 스케줄 앱 ↔ 예약현황 앱 동시 수정** — 공통 로직은 두 파일 모두
3. **iCal 파서 4개 일관성 유지** — 하나 바꾸면 나머지도 확인
4. **render() 후 attachCellClicks() 체인 유지**
5. **날짜 계산**: 월은 항상 0-indexed
6. **bkKey 형식 유지**: `${roomName}|${cinY}${(cinM+1).padStart(2)}${cinD.padStart(2)}`
7. **KV 저장**: localStorage 먼저 → 비동기 KV

## 구현 완료 후

- 변경된 파일 목록과 변경 내용 요약 출력
- `docs/features/PLAN-$ARGUMENTS.md` 의 체크박스 업데이트
- 사용자에게 테스트 시나리오 안내
- `/done $ARGUMENTS` 명령어로 마무리 안내

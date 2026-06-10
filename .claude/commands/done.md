---
description: 기능 완료 처리. 문서 전체 업데이트, 스펙 파일 확정, known-issues 갱신.
---

# 기능 완료 처리: $ARGUMENTS

구현 및 검수가 끝난 후 문서를 최신화한다.

## Step 1 — 스펙 파일 확정

`docs/features/DRAFT-$ARGUMENTS.md` 를 `docs/features/$ARGUMENTS.md` 로 이름 변경.  
(DRAFT 접두사 제거 = 확정 스펙)

파일 상단에 완료 정보 추가:
```
완료일: YYYY.MM.DD
구현 파일: 파일1, 파일2, ...
```

## Step 2 — 관련 docs/ 업데이트

변경 사항에 따라 다음 파일들을 검토하고 업데이트:
- `docs/02-architecture.md` — 새 데이터 흐름이 생겼다면
- `docs/03-data-model.md` — 데이터 구조가 변경됐다면
- `docs/04-apps-spec.md` — 앱 기능 명세가 변경됐다면
- `docs/05-known-issues.md` — 해결된 이슈 체크, 새 이슈 추가

## Step 3 — CLAUDE.md 업데이트

새로 알게 된 패턴, 주의사항, 변경된 API가 있으면 CLAUDE.md에 반영.

## Step 4 — 완료 요약 출력 후 사용자 확인 요청

```
✅ 완료: [기능명]

구현 파일:
  - 파일1 (변경 요약)
  - 파일2 (변경 요약)

업데이트 문서:
  - docs/features/$ARGUMENTS.md (확정)
  - docs/XX.md (변경 내용)

다음 주의사항:
  - 있으면 기술
```

출력 후: "문서 내용이 맞는지 확인해 주세요. 수정이 필요하면 말씀해 주세요." 로 마무리.  
**사용자가 확인 완료라고 해야 해당 기능이 공식 완료.**

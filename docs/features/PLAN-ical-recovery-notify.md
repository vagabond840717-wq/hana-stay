# iCal 연결 오류 알림 자동 복구 처리 — 구현 계획

승인된 스펙: `DRAFT-ical-recovery-notify.md`  
방식: 옵션 B (미확인 오류 이벤트 기반) + 복구 시 푸시 발송

---

## 작업 단위 (순서대로)

### Task 1: 앱 — `recovered` 타입 렌더링 추가
- 파일: `app/JnJ booking/index.html`
- 변경 위치: `renderNotifyEvents()` 안의 타입 판별 블록 (line ~1891~1911)
- 변경 내용:
  - `isRecovered = e.type === 'recovered'` 변수 추가
  - 아이콘: `isRecovered ? '✅' : ...`
  - 제목 색상: `isRecovered ? '#166534' : ...`
  - 미확인 배경: `isRecovered ? '#f0fdf4' : ...`
  - body 문구: `isRecovered ? \`${e.platform} iCal 연결이 복구됐어요\` : ...`
- 예상 코드 라인: ~8줄 수정

### Task 2: Worker — `syncAllRooms()` 성공 분기에 복구 감지 추가
- 파일: `ical-proxy/worker.js`
- 변경 위치: `syncAllRooms()` → 플랫폼 루프 안 "성공(result !== null)" 분기 (line ~245~250)
- 변경 내용:
  1. 루프 시작 전 events KV를 1회 읽어 변수에 캐시 (`let events = null` → 처음 성공 시 lazy load)
  2. 성공 시: events 중 `e.type === 'error' && !e.read && e.room === room.name && e.platform === p.label` 필터
  3. 해당 이벤트 있으면:
     - 해당 이벤트들 `read: true` 처리 후 KV 저장
     - `saveEvent(env, { type: 'recovered', room: room.name, platform: p.label, ts: Date.now() })` 호출
     - `sendPushToAll(env, { title: \`✅ ${room.name} 다시 연결됨\`, body: \`${p.label} iCal 연결이 복구됐어요\`, room: room.name })` 호출
- 예상 코드 라인: ~20줄 추가

---

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — `recovered` 타입 렌더링 (먼저 배포)
- [x] `ical-proxy/worker.js` — 복구 감지·자동 확인·이벤트 저장·푸시 발송 (나중에 배포)

---

## 배포 순서 (중요)
1. `app/JnJ booking/index.html` 수정 → booking 저장소 push → Pages 자동 배포
2. `ical-proxy/worker.js` 수정 → ical-proxy 저장소 push → GitHub Actions 자동 배포

앱 먼저 배포해야 Worker가 `recovered` 이벤트를 저장했을 때 앱이 ❌ 대신 ✅로 올바르게 표시함.

---

## 롤백 방법
- 앱: booking 저장소에서 이전 커밋으로 revert → Pages 재배포
- Worker: ical-proxy 저장소에서 이전 커밋으로 revert → Actions 재배포
- KV 데이터: `recovered` 이벤트가 추가되어도 기존 기능에 영향 없음. 앱 롤백 시 `recovered` 타입은 '취소(❌)'로 보이지만 기능 오작동 없음.

---

## 테스트 시나리오
- [ ] 배포 후 다음 5분 동기화 타이밍에 "603 jnj", "501 hana" 미확인 오류 알림이 자동 확인됨 처리되고 "✅ 다시 연결됨" 알림이 새로 생기는지 확인
- [ ] 알림 배지 숫자가 오류 알림 수만큼 줄어드는지 확인
- [ ] 폰에 "✅ 다시 연결됨" 푸시 알림이 오는지 확인
- [ ] 정상 동기화(오류 알림 없을 때)에서는 복구 이벤트가 생기지 않는지 확인
- [ ] 새 예약·취소 알림 렌더링이 기존과 동일하게 동작하는지 확인 (회귀)

---

## 예상 주의사항
- events KV는 루프 안에서 여러 플랫폼이 동시에 성공할 수 있으므로, KV 읽기는 `syncAllRooms` 시작 시 1회 로드 후 메모리에서 참조. 수정 후 1회 저장.
- `saveEvent()` 내부에서도 events KV를 읽으므로 복구 감지 로직에서는 `saveEvent()` 대신 직접 이벤트 배열을 조작해 중복 KV 읽기 방지.
- 달력/예약 표시/블락/비밀번호 로직은 전혀 건드리지 않음 — render() / attachCellClicks() 체인 영향 없음.

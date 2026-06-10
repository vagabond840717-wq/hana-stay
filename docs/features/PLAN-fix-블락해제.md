# 예약앱 블락 해제 오작동 — 구현 계획

## 문제
블락 해제 버튼을 누르면 화면에서는 "해제 완료 ✓"가 뜨지만,
실제로는 서버(`PROXY/bookings/remove`)에 해당 기능이 없을 가능성 있음.
→ 앱을 새로 열면 블락이 다시 나타날 수 있음.

## 확인 필요 사항
`ical-proxy` Cloudflare Worker 코드에 다음 엔드포인트가 구현되어 있는지:
- `POST /bookings/remove`
- `POST /sync`
- `GET /bookings`

이 파일들은 로컬에 없어서 Cloudflare 대시보드에서 확인 필요.

---

## 작업 단위

### Task 1 — 서버 응답 실패 시 명확한 에러 메시지 표시
현재: 서버 실패해도 "해제 완료 ✓" 메시지 그대로 표시.
변경: 서버 실패 시 "서버 해제 실패 — 다시 시도해 주세요" 경고 표시.

- **파일**: `app/JnJ booking/index.html`
- **위치**: `addCancelOverride()` 함수 (601번째 줄 근처)
- **변경 내용**:
  ```js
  // 변경 전: 오류 무시
  try{
    await fetch(`${PROXY}/bookings/remove`, {...});
  }catch(e){}

  // 변경 후: 실패 감지
  let removeOk = false;
  try{
    const res = await fetch(`${PROXY}/bookings/remove`, {...});
    removeOk = res.ok;
  }catch(e){}
  if(!removeOk){
    showToast('⚠ 서버 해제 실패. 앱을 재시작하면 다시 나타날 수 있어요.');
  }
  ```

### Task 2 — cancelOverrides 로컬 저장 강화
서버가 없어도 앱 내에서 해제 상태를 유지하도록 localStorage 저장 확인.
현재도 저장되지만, 앱 재시작 시 KV 데이터가 cancelOverrides보다 늦게 적용되는 순서 문제 점검.

---

## 변경 파일
- [x] `app/JnJ booking/index.html` — addCancelOverride 함수

## 주의사항
- 서버(`ical-proxy` Worker)에 `/bookings/remove` 엔드포인트가 없다면
  이 수정만으로는 근본 해결이 안 됨.
- 서버 엔드포인트 존재 여부를 먼저 확인하는 것이 우선.
- 만약 없다면 Worker 코드 수정이 별도로 필요 (다음 기능으로 처리)

## 테스트 시나리오
- [ ] 블락 해제 버튼 클릭 → 성공 시 "해제 완료 ✓"
- [ ] 서버 오류 상황 시 경고 메시지 표시 확인
- [ ] 앱 새로고침 후 해제한 블락이 다시 나타나지 않는지 확인

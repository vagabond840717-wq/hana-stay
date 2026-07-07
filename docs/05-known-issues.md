# 알려진 문제 및 주의사항

이 문서는 코드 작업 시 반드시 확인해야 할 버그, 설계 취약점, 주의사항을 정리한다.

---

## 🔴 버그 / 데이터 손실 위험

### 1. 호실명 변경 시 비밀번호/메모 데이터 고아(orphan)
**증상**: 호실 이름을 수정하면 해당 호실의 모든 비밀번호·메모가 사라짐  
**원인**: `bkKey = "${roomName}|..."` — 키에 호실명 포함됨  
**해결 방향**: 호실에 고정 UUID 부여 후 키를 `"${roomId}|..."` 로 변경

### 2. 날짜 경계 계산 불일치 (청소 스케줄 vs 예약현황)
**증상**: 두 앱에서 같은 날짜가 다르게 표시될 수 있음  
**원인**: `cellTypeFor` 구현이 두 파일에 독립적으로 존재 → 한 쪽만 수정 시 불일치  
**해결 방향**: 공통 JS 유틸리티 파일 분리 (또는 두 파일 동시 수정)

---

## 🟡 설계 취약점

### 3. iCal 파서 4개 중복
**위치**: 청소 스케줄 앱 L747~824  
**문제**: `parseIcal`, `parseIcalBooking`, `parseIcalTrip`, `parseIcalLv` 가 거의 동일  
**위험**: 파싱 로직 수정 시 하나만 바꾸면 플랫폼별 동작 차이 발생  
**해결 방향**:
```js
// 현재
parseIcal(text)        // Airbnb: "not available", "airbnb (not available)" 필터
parseIcalBooking(text) // Booking: "closed", "not available", "" 필터
parseIcalTrip(text)    // Trip: "not available", "closed", "" 필터
parseIcalLv(text)      // LV: "not available", "closed", "" 필터

// 개선안
parseIcal(text, skipTitles) // 필터 목록을 인자로 받는 단일 함수
```

### 4. 색상 생성 시 고정 (수정 불가)
**문제**: `color = COLORS[rooms.length % 9]` — 추가 순서로만 결정  
**위험**: 호실 삭제 후 재추가 시 색이 바뀜, 사용자가 직접 지정 불가  
**해결 방향**: 설정 패널에 색상 선택 UI 추가

### 5. KV 저장 실패 감지 불가
**문제**: `fetch(PROXY_URL).catch(()=>{})` — 오류 무시  
**위험**: Worker 장애 시 데이터가 localStorage에만 있고 다른 기기에서 안 보임  
**해결 방향**: 저장 실패 시 UI에 표시 (toast 알림 등)

### ~~6. 예약앱 연속뷰 이전/다음 버튼 오작동~~ ✅ 해결됨 (2026.06.03)
`renderMulti()`의 날짜 기준을 `today()` → `curY/curM`으로 변경. commit ffaf6a3

### 6. 전체 재렌더링 성능
**문제**: `render()` 호출마다 달력 전체 HTML 재생성 + innerHTML 교체  
**위험**: 호실 9개 × 6개월 × 최대 31일 = 최대 1674개 셀 동시 생성  
**현재 상태**: 사용 중 체감 문제 없음, 단 저사양 기기에서 느릴 수 있음

### 7. 비밀번호 히스토리 최대 15개 하드코딩
```js
if (extra.passwords.length > 15) extra.passwords.pop();
```
**문제**: 설정으로 변경 불가, 문서화되지 않은 제한

---

## 🟢 알고 써야 할 동작 (버그 아님)

### 8. `cinM`은 0-indexed
모든 Booking 객체의 월 값은 JavaScript Date와 동일하게 0-indexed.  
`cinM = 5` → 6월. 표시할 때만 `cinM + 1`.  
bkKey 생성 시에는 `String(cinM+1).padStart(2,'0')` 로 변환.

### 9. 체크아웃 날은 숙박 중에 포함 안 됨
```js
if (cur > cin && cur < cout) occupied = true;  // strict inequality
```
체크아웃 날(`cur === cout`)은 `occupied` 가 아니라 `checkout` 상태.

### 10. 연속 뷰(multi)에서 네비게이션 숨김
```js
document.getElementById('prevBtn').style.visibility = isMulti ? 'hidden' : 'visible';
```
연속 뷰에서는 이전/다음 월 버튼이 숨겨짐. 현재 월(`curM`) 기준으로 6개월 표시.

### 11. syncAll 실패해도 이전 데이터 유지
`rooms[i].bookings`는 `syncAll()` 이전까지는 `[]` 상태.  
`syncAll()` 실패 시 해당 호실 예약이 안 보임 (이전 데이터로 롤백 없음).

### 12. 예약현황 앱의 `manifest.json` 미포함
PWA manifest를 선언하지만 `manifest.json` 파일이 별도로 없음.  
브라우저에서 설치 프롬프트 안 뜰 수 있음.

### 13. Trip.com 예약 분할(tr_splits)은 로드 시 치환 방식
Trip.com은 연속 예약을 통짜 블락으로 합쳐 보냄 → `tr_splits` 정의로 조각 치환(→ [03-data-model.md](03-data-model.md) 참고).  
- 원본을 **치환**(제거+삽입)하므로 오버부킹 안 남. 조각 경계는 cout-exclusive.
- 원본 피드가 바뀌면 분할 **자동 무효** + 안내(`notifyStaleSplits`). 데이터 로드 실패 시엔 무효 판정 보류(오삭제 방지).
- 분할 조각마다 체크인이 달라 **비밀번호/메모 키(bkKey)가 조각별로 분리됨** (의도된 동작 — 손님이 다름). 해제하면 통짜 키 하나로 복귀.
- 현재 대상은 **Trip.com만** (`SPLIT_PLATFORMS=['tr']`). 구조는 공통이라 확장 가능.

### ~~15. 아카이브에 Booking.com/Trip.com 예약이 저장 안 됨~~ ✅ 해결됨 (2026.07.08)
**증상**: 통계·과거 달력에 에어비앤비만 남고 부킹닷컴/트립닷컴 기록 없음  
**원인**: 워커 아카이브 필터가 `not available`/`closed`/`''` 제목을 일괄 제외 — Booking.com은 실제 예약도 "CLOSED - Not available"로, Trip.com은 빈 제목으로 옴  
**수정**: 플랫폼별 규칙으로 변경 (ab: not available만 제외 / bk: 전부 보존 / tr·lv: closed만 제외). worker.js `syncAllRooms` 아카이브 병합부.  
**주의**: 수정 이전에 피드에서 사라진 과거 bk/tr 예약은 소급 복구 불가 — 수정 시점부터 쌓임.

### ~~16. 지난 날짜에 가짜 오버부킹 경고~~ ✅ 해결됨 (2026.07.08)
**증상**: 이미 지난 날짜 셀에 ⚠️ 오버부킹 표시  
**원인**: 아카이브는 append-only라 취소·날짜변경된 옛 예약이 남음 → 과거 날짜에서 실제 예약과 겹쳐 `isOverbooking` 오감지  
**수정**: 예약앱 `isOverbooking`에 오늘 이전 날짜는 판정 제외 조건 추가. (근본 원인인 아카이브의 취소 예약 잔존은 통계 취지상 유지 — 취소 여부를 iCal에서 구분할 수 없음)

### ~~17. Trip.com 스냅샷 중첩 — 과거 달력에 매일 '인' 마커 도배~~ ✅ 해결됨 (2026.07.08)
**증상**: 지난 트립닷컴 숙박 구간에 날마다 체크인 마커가 뜨고 (#16의 오버부킹 도배도 실제 주범), 에어비앤비 날짜변경 예약이 두 장으로 남음  
**원인**: Trip.com iCal은 "오늘~체크아웃" 형태로 **DTSTART가 매일 당겨진** 피드를 보냄 → 아카이브가 시작일만 다른 같은 숙박을 매일 새 UID로 적립 (한 숙박이 8장까지 확인됨)  
**수정**: 같은 호실·같은 플랫폼에서 다른 항목 범위에 **완전히 포함**되는 항목은 부분 스냅샷으로 판단해 제거. 같은 플랫폼은 중복 판매가 불가능하므로 포함 관계 = 같은 숙박의 옛 버전이라 안전. 워커(`syncAllRooms` 아카이브 병합부)와 예약앱(`mergeArchiveIntoRooms`) 양쪽 적용 — 매 sync마다 실행되어 기존 적립분도 자동 정리됨.  
**한계**: 숙박을 **단축**한 변경(새 버전이 옛 버전에 포함됨)은 긴 옛 버전이 남음 — 통계용이라 허용.

### 14. `c-bk-both` / `c-lv-both`는 아직 단색 (both 반반셀 미적용)
분할 기능으로 `c-tr-both`는 아웃/인 반반 그라디언트로 수정했으나, Booking.com·리브애니웨어의 both 셀은 여전히 단색.  
해당 플랫폼은 아직 통짜 합침 사례가 없어 both가 드묾. 추후 그 플랫폼 분할을 켤 때 함께 수정 필요.

---

## 코드 수정 시 체크리스트

기능 추가 또는 수정 전에 반드시 확인:

- [ ] 청소 스케줄 앱과 예약현황 앱이 같은 로직을 쓰는가? → 두 파일 모두 수정
- [ ] iCal 파싱 관련 수정인가? → 4개 parser 모두 확인
- [ ] bkKey 관련 변경인가? → 기존 저장 데이터 마이그레이션 필요
- [ ] `render()` 를 호출하는가? → `attachCellClicks()` 가 같이 실행되는지 확인
- [ ] 날짜 비교 로직인가? → 0-indexed 월 일관성 확인
- [ ] KV 저장/로드인가? → localStorage 폴백 동작도 테스트

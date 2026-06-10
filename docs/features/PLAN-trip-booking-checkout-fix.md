# PLAN: Trip.com / Booking.com 체크아웃일 예약가능 처리

## 배경 (문제 분석 완료)

### 원인
- Trip.com, Booking.com iCal의 DTEND = 체크아웃 당일 (마지막 막힌 날, 포함)
- HANA STAY가 이 DTEND를 그대로 Airbnb에 내보냄
- Airbnb가 DTEND를 포함으로 읽어서 체크아웃 당일을 블락
- Airbnb가 "Not Available" 생성 (DTSTART=체크아웃일, DTEND=체크아웃일+1)
- HANA STAY가 이걸 다시 읽어서 체크아웃일+1에 회색+빨강 표시
- 이 abBlock이 다시 내보내져서 Trip.com에도 체크아웃일+1이 블락됨

### 데이터로 확인된 사실
- 402 jnj, Trip.com: 체크인 7/20, 체크아웃 7/24, 4박 → 7/24에 "Airbnb 블락" 자동 생성
- 603 jnj, Booking.com: 체크인 6/21, 체크아웃 7/3, 12박 → 같은 문제 가능성

---

## 수정 방향

**핵심**: 체크아웃일을 "예약 가능일"로 인식하여 내보내기에서 제외

---

## 수정 파일 및 내용

### 1. `ical-proxy/worker.js` — exportIcal 함수

**현재 코드 (354번째 줄 근처):**
```js
for (const [key, bks] of Object.entries(roomBookings)) {
  for (const bk of bks) {
    const ds = `${bk.cinY}${String(bk.cinM+1).padStart(2,'0')}${String(bk.cinD).padStart(2,'0')}`;
    const de = `${bk.coutY}${String(bk.coutM+1).padStart(2,'0')}${String(bk.coutD).padStart(2,'0')}`;
    events += `BEGIN:VEVENT\r\n...DTSTART;VALUE=DATE:${ds}\r\nDTEND;VALUE=DATE:${de}\r\n...`;
  }
}
```

**수정 내용:**
```js
for (const [key, bks] of Object.entries(roomBookings)) {
  for (const bk of bks) {
    // Airbnb "Not Available" 항목은 내보내기에서 제외 (순환 문제 방지)
    if (bk.summary?.toLowerCase().includes('not available')) continue;

    const ds = `${bk.cinY}${String(bk.cinM+1).padStart(2,'0')}${String(bk.cinD).padStart(2,'0')}`;

    // Trip.com, Booking.com은 DTEND가 체크아웃 당일(포함)
    // Airbnb가 이를 포함으로 읽어 체크아웃일을 블락하는 문제 방지
    // → 체크아웃일은 예약가능일로 내보내기 위해 DTEND = cout - 1일 사용
    let deDate = new Date(bk.coutY, bk.coutM, bk.coutD);
    if (key === 'tr' || key === 'bk') {
      deDate.setDate(deDate.getDate() - 1);
    }
    const de = `${deDate.getFullYear()}${String(deDate.getMonth()+1).padStart(2,'0')}${String(deDate.getDate()).padStart(2,'0')}`;

    events += `BEGIN:VEVENT\r\n...DTSTART;VALUE=DATE:${ds}\r\nDTEND;VALUE=DATE:${de}\r\n...`;
  }
}
```

---

### 2. `ical-proxy/worker.js` — parseIcal 함수

**수정 내용**: Airbnb "Not Available" DTEND는 exclusive(제외)로 처리

**현재 코드 (329번째 줄):**
```js
const cin = pd(dtstart), cout = pd(dtend);
```

**수정 내용:**
```js
let cin = pd(dtstart), cout = pd(dtend);
// Airbnb "Not Available"의 DTEND는 exclusive (첫 번째 빈 날)
// → 하루 빼서 실제 마지막 블락 날을 cout으로 저장
if (platform === 'airbnb' && summary.toLowerCase().includes('not available')) {
  const coutDate = new Date(cout.y, cout.m, cout.d);
  coutDate.setDate(coutDate.getDate() - 1);
  cout = { y: coutDate.getFullYear(), m: coutDate.getMonth(), d: coutDate.getDate() };
}
```

**효과**: Airbnb "Not Available" (7/24~7/25) → cin=7/24, cout=7/24 → 7/24에만 표시, 7/25는 빈칸

---

### 3. 배포

- `ical-proxy/worker.js` 수정 후 Cloudflare에 `wrangler deploy`
- 앱은 `app/JnJ booking/index.html` 수정 불필요 (워커만 수정)
- GitHub에 커밋 & 푸시

---

## 예상 결과

| 항목 | 수정 전 | 수정 후 |
|------|---------|---------|
| 402호 7/25 표시 | 회색+빨강 (abBlock 체크아웃) | 빈칸 (정상) |
| Trip.com 7/25 블락 | 블락됨 | 해제됨 |
| Airbnb 7/24 블락 | Not Available 생성 | 생성 안 됨 |
| 체크아웃일 앱 표시 | 아웃(체크아웃) | 동일 (아웃 표시 유지) |

---

## 확인 필요

- 수정 후 `wrangler deploy` 실행
- 앱에서 동기화(새로고침) 후 402호 7/25 확인
- Booking.com도 동일하게 동작하는지 확인

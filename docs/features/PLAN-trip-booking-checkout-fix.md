# PLAN: Trip.com / Booking.com 체크아웃일 예약가능 처리

> ## ⛔ 이 계획의 2번째 수정(DTEND −1)은 철회됨 (2026-08-05)
>
> 아래 "원인"의 첫 줄 전제가 **틀렸다.** Trip.com·Booking.com도 표준대로 **DTEND exclusive**로 보낸다
> (실측: 트립 예약관리 `체크인 8/17 / 체크아웃 8/22` ↔ 트립 iCal `DTSTART:20260817 / DTEND:20260822`).
> 체크아웃 당일은 원래부터 예약 가능일로 나가고 있었고, 여기서 하루를 더 빼는 바람에
> **마지막 숙박일까지 전 채널에서 판매 가능** 상태가 됐다 → 오버부킹.
>
> `exportIcal`의 −1은 제거했다(commit 23992de). `DTEND = cout` 그대로가 정답.
> 함께 넣은 **"Not available 내보내기 제외"(순환 방지)는 유효하므로 유지**한다.
> 전말은 [05-known-issues.md #21](../05-known-issues.md) 참고.

## 배경 (문제 분석 완료)

### 원인
- ~~Trip.com, Booking.com iCal의 DTEND = 체크아웃 당일 (마지막 막힌 날, 포함)~~ ← **오판**
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

## 구현 상태

- [x] `ical-proxy/worker.js` `parseIcal` — 에어비앤비 "Not Available" DTEND exclusive 처리 (기존 적용됨)
- [x] `ical-proxy/worker.js` `exportIcal` — Not available 제외 (순환 방지) — **유지**
- [x] ~~`exportIcal` tr/bk 체크아웃일 -1일 내보내기 (2026-08-01 적용)~~ → **철회·제거 (2026-08-05, commit 23992de)**

## 확인 필요

- [x] 배포 — GitHub Actions 자동 배포 확인 (`/ical/402 jnj` 응답이 트립 원본과 DTEND 일치)
- [ ] 각 채널이 새 피드를 다시 읽어간 뒤, 402호 **8/21 잠김 / 8/22 열림** 확인
- [ ] `parseIcal`의 에어비앤비 "Not available" −1 처리 재검토 — 내보내기에서 제외되므로 당장 피해는 없으나,
      이 경로만 `cout`이 inclusive라 앱 화면에서 에어비앤비 블락이 하루 짧게 보일 가능성

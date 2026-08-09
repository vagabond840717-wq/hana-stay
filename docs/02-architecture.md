# 시스템 아키텍처

## 전체 흐름

```
[브라우저 / 모바일]
        │
        │  fetch (CORS 우회)
        ▼
[Cloudflare Worker]  ← ical-proxy.vagabond1984.workers.dev
  ├── GET /?url=     → Airbnb/Booking/Trip/LV iCal 서버에 요청
  ├── GET /rooms     → KV에서 호실 목록 반환
  ├── POST /rooms    → KV에 호실 목록 저장
  ├── GET /extra     → KV에서 비밀번호/메모 반환
  ├── POST /extra    → KV에 비밀번호/메모 저장
  ├── GET /bookings  → synced_bookings KV 반환 (달력 표시용)
  ├── POST /sync     → iCal 동기화 실행
  ├── GET /archive   → booking_archive KV 반환 (통계 뷰용)
  └── GET /ical/<호실명> → HANA STAY 통합 iCal 내보내기 (각 플랫폼이 구독)
        │
        ▼
[Cloudflare KV]      ← 서버 측 영속 저장소
```

## Cloudflare Worker 엔드포인트

### iCal 프록시
```
GET https://ical-proxy.vagabond1984.workers.dev/?url=<encoded_ical_url>
```
- 브라우저에서 직접 `webcal://` URL을 fetch하면 CORS 오류 발생
- Worker가 서버 측에서 해당 URL을 가져와 응답으로 전달
- `webcal://` → `https://` 로 변환 처리 필요 (Worker 내부)

### 호실 데이터
```
GET  /rooms          → 저장된 호실 배열 반환 (JSON)
POST /rooms          → 호실 배열 저장 (body: JSON 배열)
```

### Extra 데이터 (비밀번호/메모)
```
GET  /extra?key=<bkKey>   → 해당 키의 { passwords, memos } 반환
POST /extra               → { key, data } 저장
```

### iCal 내보내기 (⚠ 전 채널 영향)
```
GET https://ical-proxy.vagabond1984.workers.dev/ical/<호실명>
예: /ical/402%20jnj
```
- `synced_bookings` KV + `extra_manual_blocks` KV를 합쳐 하나의 iCal로 내보냄
- **에어비앤비·부킹닷컴·리브애니웨어가 각각 이 주소를 구독**한다 → 여기를 고치면 전 채널이 동시에 바뀐다
- `DTEND = cout` **그대로** 내보낸다. iCal의 `DTEND;VALUE=DATE`는 포함 안 되는 날이므로 이것만으로
  체크아웃 당일이 판매 가능일이 된다. **여기서 하루를 빼면 마지막 숙박일이 열려 오버부킹**
  ([05-known-issues.md #21](05-known-issues.md))
- 에어비앤비 "Not available" 항목은 내보내지 않는다 (되받아 읽으며 블락이 번지는 순환 방지)
- 수동 블락은 `end`가 inclusive 저장이라 내보낼 때만 `+1일`

> **검증 방법**: 에어비앤비는 **가져온 달력의 블락을 자기 iCal로 다시 내보내지 않는다.**
> 따라서 "우리가 보낸 게 제대로 반영됐는지"는 피드 대조로 알 수 없고 **에어비앤비 앱 달력 화면**을 봐야 한다.
> 반대로 "우리가 뭘 보내고 있는지"는 위 주소를 직접 받아보면 즉시 확인된다.

## 데이터 저장 이중화 전략

```
사용자 액션
    │
    ├─[1]→ localStorage.setItem(...)   ← 즉시, 동기
    └─[2]→ fetch(PROXY_URL, POST)      ← 비동기, 오류 무시(.catch)
```

**로드 순서**:
1. Worker `/rooms` 응답 성공 → KV 데이터 사용 + localStorage 갱신
2. 실패 → localStorage 폴백

**목적**: 오프라인 또는 Worker 장애 시에도 동작 유지

## 앱 간 데이터 공유

- 세 앱은 **같은 Cloudflare KV**를 공유
- 호실 목록(`/rooms`)은 청소 스케줄 앱과 예약현황 앱이 동일하게 사용
- 비밀번호/메모(`/extra`)는 청소 스케줄 앱에서 주로 사용
- 요금 계산기는 외부 저장소 사용 안 함 (순수 계산기)

## 예약 아카이브 흐름

```
POST /sync 실행
        │
        ▼
iCal 파싱 결과 (최근 1개월 이후 예약만 포함)
        │
        ├─[1]→ synced_bookings KV 갱신  ← 달력 표시용 (기존)
        │
        └─[2]→ booking_archive KV 병합  ← 통계용 (신규)
                  UID 기준 중복 제거
                  not available / closed 제외
                  13개월 이전 자동 정리
```

- `synced_bookings`와 `booking_archive`는 완전히 별도 KV 키
- 달력 뷰는 `synced_bookings`만 사용, 통계 뷰는 `booking_archive`만 사용

## iCal 데이터 흐름

```
각 플랫폼 iCal URL (webcal://)
        │
        ▼ (Worker 프록시)
[iCal 텍스트 수신]
        │
        ▼
parseIcal() / parseIcalBooking() / parseIcalTrip() / parseIcalLv()
        │
        ▼
Booking[] 배열 → room.bookings / bkBookings / trBookings / lvBookings
        │
        ▼
cellTypeFor(bookings, day, y, m) → 'empty'|'checkout'|'checkin'|'both'|'occupied'
        │
        ▼
cellClsAndLbl(room, day, y, m) → { cls: CSS클래스, lbl: HTML }
        │
        ▼
HTML 렌더링
```

## 랜더링 아키텍처

- **상태**: `rooms[]` 배열 (전역), `curY`, `curM`, `viewMode`
- **렌더 트리거**: `render()` — 모든 뷰 재생성
- **DOM 업데이트**: `innerHTML` 전체 교체 (Virtual DOM 없음)
- **이벤트**: `render()` 후 `attachCellClicks()` 재등록

```
render()
  ├── renderMonth()   or
  ├── renderWeek()    or
  └── renderMulti() → renderMonthBlock() × N개월
        │
        └── cellClsAndLbl() × (rows × days)
```

## 주차 Worker (별도)

```
GET  /kv?key=<key>      → KV 값 조회
POST /kv                → { key, value } KV 저장
```
- `PARKING_KV` 바인딩 사용
- 주차 앱 전용, 다른 앱과 KV 네임스페이스 분리

완료일: 2026.06.08
구현 파일: ical-proxy/worker.js, app/JnJ booking/index.html

# 예약 이력 아카이브 + 연간 점유율 뷰 스펙

## 요약
Sync 시 지난 예약 데이터를 KV에 누적 보존하고, 예약앱에서 12개월치 호실별 점유율을 한눈에 볼 수 있는 통계 뷰를 추가한다.

## 목적 및 배경
iCal 피드는 과거 예약을 포함하지 않는다. Sync를 실행하면 직전 달 이전 데이터가 KV에서 사라진다.
결과적으로 "작년 여름에 예약이 얼마나 됐는지", "비수기가 언제인지" 같은 운영 판단을 위한 데이터가 없다.
예약이 발생할 때마다 누적 저장하고, 시각적으로 성수기/비수기 패턴을 파악할 수 있도록 한다.

## 기능 상세

### 사용자 시나리오
- Sync를 실행할 때마다 예약 데이터가 자동으로 아카이브됨 (별도 조작 불필요)
- 예약앱 헤더 우측 📊 버튼을 탭하면 통계 뷰로 전환
- 각 호실별로 월별 점유율(%)을 색상 강도로 표시
- 플랫폼(전체/✈/🏨/🌐/🏡) 탭으로 필터 전환 가능
- 셀 탭 시 "N박/M일 · Z%" 상세 표시

### 동작 규칙
- 점유율 = 숙박 일수 / 해당 월 총 일수 × 100
- 숙박 일수: cin 포함, cout 미포함 (기존 cellTypeFor와 동일)
- 블락(Not Available, 수동 블락) 제외 — 실제 게스트 예약만 집계
- 아카이브 보존 기간: 최근 13개월 (자동 정리)
- 데이터 없는 달: "—" 표시 (회색)

## 기술 구현

### Worker (ical-proxy/worker.js)
- `GET /archive` 엔드포인트 추가 → `booking_archive` KV 반환
- `syncAllRooms()` 내 아카이브 병합 로직 추가
  - 기존 `booking_archive`를 읽어 새 예약과 UID 기준 병합 (중복 제거)
  - UID = `cinY_cinM_cinD_coutY_coutM_coutD`
  - not available, closed, 빈 summary 제외
  - 13개월 이전 체크아웃 예약 자동 정리
  - `synced_bookings` 키는 수정하지 않음 (기존 달력과 완전 분리)

### 앱 (app/JnJ booking/index.html)
- 헤더에 📊 버튼 추가
- 통계 패널 (전체화면 슬라이드업 오버레이)
- `loadArchive()` — 앱 초기화 시 `/archive` 호출
- `calcOccupancy(roomName, platform, year, month)` — 월별 숙박일 계산
- `renderStats()` — 12개월 × 호실 그리드 렌더링
- 플랫폼 탭 전환 시 즉시 재계산

## 데이터 구조 (booking_archive KV)

```js
{
  "302호": {
    ab: [ {cinY, cinM, cinD, coutY, coutM, coutD, platform, summary}, ... ],
    bk: [ ... ],
    tr: [ ... ],
    lv: [ ... ]
  },
  ...
}
```

## 주의사항
- 처음 배포 후 첫 Sync 전까지 통계 화면은 빈 상태 ("아직 데이터가 없어요" 안내)
- 호실명 변경 시 이전 이력 고아 (known-issues #1과 동일 문제, 이번 범위 밖)
- KV 저장 실패 시 조용히 무시 (known-issues #5와 동일 방침)

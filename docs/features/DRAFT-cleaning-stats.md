# 월별 청소 횟수 / 비용 통계 스펙

## 요약
예약앱 📊 통계 패널에 "청소비용" 뷰를 추가해 월별·호실별 청소 횟수와 청소비용을 한눈에 파악한다.
단기(1~3박) / 장기(4박+) 단가를 별도 적용하며, 단가는 UI에서 수정 가능하다.

## 목적 및 배경
- 체크아웃 1건 = 청소 1회 = 청소비용 1건 지급
- 박수에 따라 청소 난이도/시간이 달라 단기·장기 단가가 다름
- 현재 archiveData(KV `/archive`)에 모든 예약의 체크인/아웃 날짜가 저장되어 있어 **추가 API 없이** 바로 집계 가능

## 기능 상세

### 단기 / 장기 구분 기준
| 구분 | 박수 | 기본 단가 |
|------|------|----------|
| 단기 | 1~3박 | 25,000원 |
| 장기 | 4박 이상 | 45,000원 |

> 박수 계산: `cout 날짜 - cin 날짜` (일 수)

### 사용자 시나리오
1. 📊 버튼 탭 → 통계 패널 오픈 (기존 동일)
2. 패널 상단 **점유율 | 청소비용** 뷰 토글 탭 노출
3. "청소비용" 탭 탭 → 테이블이 청소비용 뷰로 전환
4. 상단 단가 입력 행: **단기 [ 25,000 ]원 / 장기 [ 45,000 ]원** (각각 수정 가능)
   - 입력 변경 즉시 테이블 재계산 + localStorage 저장
5. 기존 플랫폼 탭(전체/✈/🏨/🌐/🏡) 그대로 유지
6. 셀 탭 → 툴팁: "단기 N회 + 장기 M회 · 합계 ₩XXX,XXX"

### UI 레이아웃
```
[ 점유율 | 청소비용 ]             ← 뷰 토글 (신규)
[ 전체 | ✈ | 🏨 | 🌐 | 🏡 ]     ← 기존 플랫폼 탭

단기(~3박) [25,000] 원  장기(4박~) [45,000] 원   ← 청소비용 뷰일 때만 노출

        25년1월  2월   3월  ...  6월(이번달)
302호    5회    4회   6회        7회
402호    3회    3회   4회        5회
...
합계     8회    7회  10회       12회
       ₩285k ₩255k ₩360k     ₩435k
```

### 셀 표시
- 본문: `N회` (총 청소 횟수 — 단기 + 장기 합산)
- 셀 하단 소 텍스트: `₩XXX,XXX`
- 셀 색상: 횟수 기준 강도 (0회=회색, 1~2=연주황, 3~4=중간, 5~6=진한, 7+=가장 진한)
- 툴팁(탭 시): `단기 N회 · 장기 M회 · 합계 ₩XXX,XXX`

### 합계 행
- 각 월 하단 합계 행: 전체 호실 합산 횟수 + 총 비용
- 예: `12회 · ₩435,000`

## 기술 구현 방안 (옵션 A 채택)

### 기존 statsPanel에 뷰 토글 추가
- 별도 패널/아이콘 버튼 불필요
- 플랫폼 탭은 두 뷰 공통 사용

### 핵심 집계 함수
```js
function calcCleanings(roomName, platform, year, month) {
  const roomData = archiveData[roomName];
  if (!roomData) return { short: 0, long: 0, noData: true };
  const keys = platform === 'all' ? ['ab','bk','tr','lv'] : [platform];
  const hasAny = keys.some(k => (roomData[k]||[]).length > 0);
  if (!hasAny) return { short: 0, long: 0, noData: true };
  let shortCnt = 0, longCnt = 0;
  for (const key of keys) {
    for (const bk of (roomData[key]||[])) {
      if (bk.coutY === year && bk.coutM === month) {
        const nights = Math.round(
          (new Date(bk.coutY, bk.coutM, bk.coutD) - new Date(bk.cinY, bk.cinM, bk.cinD))
          / 86400000
        );
        if (nights <= 3) shortCnt++;
        else longCnt++;
      }
    }
  }
  return { short: shortCnt, long: longCnt, noData: false };
}
```

### 단가 저장
```js
// localStorage keys
'hana_clean_fee_short'  // 기본값 25000
'hana_clean_fee_long'   // 기본값 45000
```

## 영향 받는 파일
- `app/JnJ booking/index.html` — 유일한 수정 파일
  - CSS: 뷰 토글, 단가 입력 행, 청소 셀 색상 클래스
  - HTML: statsPanel 내 토글 + 단가 입력 행 추가
  - JS: `calcCleanings()`, `renderCleaningStats()` 추가, renderStats 분기

## 데이터 변경
- 없음 (archiveData 재사용)
- 단가 2개만 localStorage에 신규 저장

## 제약 및 주의사항
- `coutM`은 0-indexed — 비교 시 `bk.coutM === month`
- 아카이브에 없는 기간(sync 미실행)은 "—" 표시
- 청소 스케줄 앱(`app/JnJ/index.html`) 수정 불필요

## 미결정 사항
- 없음 (사용자 확인 완료)

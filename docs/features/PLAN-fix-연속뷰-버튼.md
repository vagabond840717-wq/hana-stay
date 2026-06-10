# 예약앱 연속뷰 버튼 오작동 — 구현 계획

완료일: 2026.06.03
구현 파일: app/JnJ booking/index.html
배포: GitHub push → Cloudflare 자동 배포 완료 (commit ffaf6a3)

## 문제
예약현황 앱에서 "연속" 뷰를 선택하면 이전/다음 버튼이 보이지만,
눌러도 달력이 항상 오늘 기준 6개월로 고정됨.

## 원인
`renderMulti()` 함수 안에서 날짜 기준을 `curY, curM` (현재 선택 월) 대신
`t2.y, t2.m` (오늘 날짜)로 고정해서 계산하고 있음.

## 작업 단위

### Task 1 — renderMulti 날짜 기준 수정
- **파일**: `app/JnJ booking/index.html`
- **위치**: 약 982번째 줄 `renderMulti()` 함수
- **변경 전**:
  ```js
  const t2=today();
  // 연속뷰는 항상 오늘 달 기준으로 6개월
  for(let i=0;i<MONTHS_TO_SHOW;i++){
    let y=t2.y, m=t2.m+i;
  ```
- **변경 후**:
  ```js
  const t2=today();
  // curY, curM 기준으로 6개월 (이전/다음 버튼 반응)
  for(let i=0;i<MONTHS_TO_SHOW;i++){
    let y=curY, m=curM+i;
  ```

### Task 2 — 연속뷰에서 이전/다음 버튼 표시
- **파일**: `app/JnJ booking/index.html`
- **위치**: 약 1029번째 줄 `render()` 함수
- **변경 전**:
  ```js
  document.getElementById('prevBtn').style.visibility=isMulti?'hidden':'visible';
  document.getElementById('nextBtn').style.visibility=isMulti?'hidden':'visible';
  ```
- **변경 후**:
  ```js
  document.getElementById('prevBtn').style.visibility='visible';
  document.getElementById('nextBtn').style.visibility='visible';
  ```

## 변경 파일
- [x] `app/JnJ booking/index.html` — renderMulti, render 함수 2곳 ✅ 완료 (commit ffaf6a3)

## 롤백 방법
두 줄만 바꾸는 수정이라 원래 코드로 되돌리기 쉬움.

## 테스트 시나리오
- [ ] 연속 뷰에서 "다음" 버튼 클릭 → 달력이 한 달 앞으로 이동
- [ ] 연속 뷰에서 "이전" 버튼 클릭 → 달력이 한 달 뒤로 이동
- [ ] 오늘 버튼 클릭 → 오늘 기준으로 돌아옴
- [ ] 월간/주간 뷰에서 이전/다음 버튼 정상 작동 확인

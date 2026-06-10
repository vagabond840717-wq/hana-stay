# 요금계산기 0% 입력 오류 — 구현 계획

## 문제
리브애니웨어 수수료를 0%로 입력하면 실제로는 10%로 계산됨.
```js
const lvComm = +document.getElementById('lv_comm').value || 10;
// 0을 입력하면 +0 = 0, 0은 falsy → fallback 10이 적용됨
```
같은 패턴이 다른 입력값에도 있을 수 있음.

---

## 작업 단위

### Task 1 — 모든 입력값의 falsy fallback 패턴 수정
- **파일**: `app/JnJ Price/index.html`
- **위치**: `calc()` 함수 안 (373번째 줄 근처)
- **대상**: `|| <숫자>` 패턴 전수 검토
- **변경 방식**:
  ```js
  // 변경 전 (0 입력 시 기본값으로 대체됨)
  const lvComm = +document.getElementById('lv_comm').value || 10;

  // 변경 후 (빈칸일 때만 기본값, 0은 0으로 인식)
  const lvCommEl = document.getElementById('lv_comm').value;
  const lvComm = lvCommEl === '' ? 3.3 : +lvCommEl;
  ```

수정 대상 변수:
- `abFee` — 기본값 15.5
- `abPrmo` — 기본값 0
- `bkComm` — 기본값 17.5
- `bkPrmo` — 기본값 0
- `trComm` — 기본값 15
- `trPrmo` — 기본값 0
- `lvComm` — 기본값 3.3 (현재 fallback이 10으로 잘못됨)
- `lvPrmo` — 기본값 0
- `clean` — 기본값 0 (0이 유효한 값)
- `extra` — 기본값 0

---

## 변경 파일
- [x] `app/JnJ Price/index.html` — calc() 함수

## 테스트 시나리오
- [ ] 리브애니웨어 수수료 0% 입력 → 0%로 계산되는지 확인
- [ ] 청소비 0원 입력 → 0원으로 계산되는지 확인
- [ ] 입력창 완전히 비움 → 기본값으로 계산되는지 확인

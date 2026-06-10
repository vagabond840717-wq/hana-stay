# 주차앱 글자 안 보이는 문제 — 구현 계획

## 문제
대기 예약 수 표시(`+1`, `+2` 등)가 CSS에서 `color: var(--purple)` 을 쓰는데
`:root`에 `--purple` 변수가 정의되어 있지 않아 글자가 보이지 않음.

---

## 작업 단위

### Task 1 — --purple CSS 변수 추가
- **파일**: `app/parking-main/parking-main/index.html`
- **위치**: `:root` 블록 (9번째 줄 근처)
- **변경 전**:
  ```css
  :root {
    --bg: #0f1117;
    --surface: #1a1d27;
    --surface2: #22263a;
    --border: #2e3248;
    --accent: #4f8ef7;
    --green: #3ecf8e;
    --red: #f75c5c;
    --yellow: #f7c94f;
    --text: #e8eaf2;
    --text2: #8b90a8;
    --text3: #555a72;
  }
  ```
- **변경 후**:
  ```css
  :root {
    --bg: #0f1117;
    --surface: #1a1d27;
    --surface2: #22263a;
    --border: #2e3248;
    --accent: #4f8ef7;
    --green: #3ecf8e;
    --red: #f75c5c;
    --yellow: #f7c94f;
    --purple: #a78bfa;   ← 추가
    --text: #e8eaf2;
    --text2: #8b90a8;
    --text3: #555a72;
  }
  ```

---

## 변경 파일
- [x] `app/parking-main/parking-main/index.html` — :root CSS 변수 1개 추가

## 테스트 시나리오
- [ ] 대기 예약이 있는 슬롯에서 `+1`, `예약1` 등 보라색 글자 보이는지 확인

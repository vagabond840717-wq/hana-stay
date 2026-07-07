# 월별 청소 통계 구현 계획

## 작업 단위 (순서대로)

---

### Task 1: CSS 추가
- **파일**: `app/JnJ booking/index.html`
- **위치**: 기존 `.stats-empty strong { ... }` 블록 바로 뒤 (line ~455)
- **변경 내용**: 아래 스타일 클래스 추가

```css
/* 뷰 토글 탭 */
.stats-view-tabs {
  display: flex; gap: 4px; padding: 10px 16px 0;
}
.stats-vtab {
  padding: 5px 14px; border-radius: 20px; font-size: 12px; font-weight: 500;
  border: 1px solid var(--border2); background: var(--surface); color: var(--text2);
  cursor: pointer; font-family: 'DM Sans', sans-serif; transition: all 0.15s;
}
.stats-vtab.active { background: var(--navy); color: #fff; border-color: var(--navy); }

/* 단가 입력 행 */
.stats-fee-row {
  display: flex; align-items: center; gap: 12px;
  padding: 8px 16px; flex-wrap: wrap;
}
.stats-fee-label { font-size: 11px; color: var(--text2); white-space: nowrap; }
.stats-fee-input {
  width: 80px; padding: 5px 8px; font-size: 12px; font-family: 'DM Mono', monospace;
  background: var(--surface); border: 1px solid var(--border2); border-radius: var(--r);
  color: var(--navy); outline: none; text-align: right;
}
.stats-fee-input:focus { border-color: var(--navy); }

/* 청소 셀 — 2줄 레이아웃 */
.stats-cell.clean-cell {
  flex-direction: column; gap: 1px; height: 42px;
}
.stats-clean-count { font-size: 10px; font-weight: 700; line-height: 1; }
.stats-clean-cost  { font-size: 8px; font-weight: 500; line-height: 1; opacity: 0.85; }

/* 청소 셀 색상 (횟수 기준) */
.clean-0 { background: var(--surface2); color: var(--text3); }
.clean-1 { background: #fff7ed; color: #c2410c; }
.clean-2 { background: #ffedd5; color: #c2410c; }
.clean-3 { background: #fed7aa; color: #9a3412; }
.clean-4 { background: #fb923c; color: #fff; }
.clean-5 { background: #ea580c; color: #fff; }
.clean-6 { background: #c2410c; color: #fff; }
```

- **예상 추가 라인**: ~40줄

---

### Task 2: HTML 수정 — statsPanel 구조 변경
- **파일**: `app/JnJ booking/index.html`
- **위치**: `<div class="stats-overlay" id="statsPanel">` 블록 (line ~615)

**변경 1**: 타이틀 텍스트 변경
```html
<!-- 변경 전 -->
<div class="stats-title">📊 점유율 통계</div>
<!-- 변경 후 -->
<div class="stats-title">📊 통계</div>
```

**변경 2**: 플랫폼 탭 위에 뷰 토글 탭 삽입
```html
<!-- stats-header 닫는 태그 바로 뒤에 추가 -->
<div class="stats-view-tabs" id="statsViewTabs">
  <button class="stats-vtab active" data-view="occupancy">점유율</button>
  <button class="stats-vtab" data-view="cleaning">청소비용</button>
</div>
```

**변경 3**: 플랫폼 탭과 stats-body 사이에 단가 입력 행 추가 (기본 숨김)
```html
<div class="stats-fee-row" id="statsFeeRow" style="display:none;">
  <span class="stats-fee-label">단기(~3박)</span>
  <input class="stats-fee-input" id="feeShortInput" type="number" value="25000" min="0">
  <span class="stats-fee-label">원</span>
  <span class="stats-fee-label" style="margin-left:8px;">장기(4박~)</span>
  <input class="stats-fee-input" id="feeLongInput" type="number" value="45000" min="0">
  <span class="stats-fee-label">원</span>
</div>
```

- **예상 추가 라인**: ~12줄

---

### Task 3: JS — 상태변수 및 단가 초기화
- **파일**: `app/JnJ booking/index.html`
- **위치**: `// ── STATS ──` 섹션 상단 (line ~1521, `let archiveData = {}` 위)

```js
let statsView = 'occupancy';
let cleanFeeShort = parseInt(localStorage.getItem('hana_clean_fee_short') || '25000', 10);
let cleanFeeLong  = parseInt(localStorage.getItem('hana_clean_fee_long')  || '45000', 10);
```

- **예상 추가 라인**: 3줄

---

### Task 4: JS — calcCleanings() 함수 추가
- **파일**: `app/JnJ booking/index.html`
- **위치**: `calcOccupancy()` 함수 바로 뒤

```js
function calcCleanings(roomName, platform, year, month) {
  const roomData = archiveData[roomName];
  if (!roomData) return { short: 0, long: 0, noData: true };
  const keys = platform === 'all' ? ['ab','bk','tr','lv'] : [platform];
  if (!keys.some(k => (roomData[k]||[]).length > 0)) return { short: 0, long: 0, noData: true };
  let shortCnt = 0, longCnt = 0;
  for (const key of keys) {
    for (const bk of (roomData[key]||[])) {
      if (bk.coutY === year && bk.coutM === month) {
        const nights = Math.round(
          (new Date(bk.coutY, bk.coutM, bk.coutD) - new Date(bk.cinY, bk.cinM, bk.cinD))
          / 86400000
        );
        if (nights <= 3) shortCnt++; else longCnt++;
      }
    }
  }
  return { short: shortCnt, long: longCnt, noData: false };
}
```

- **예상 추가 라인**: 16줄

---

### Task 5: JS — renderCleaningStats() 함수 추가
- **파일**: `app/JnJ booking/index.html`
- **위치**: `renderStats()` 함수 바로 앞

핵심 로직:
- 월 목록(최근 12개월)은 renderStats()와 동일하게 생성
- hasData 확인: archiveData에 예약이 하나라도 있어야 함
- 각 셀: `calcCleanings()` → 횟수 + 금액 계산
- 금액 표시: `₩` + 천 단위 콤마 (예: `₩45,000`)
- 셀 색상 클래스: `cleanClass(total)` 함수 (0→clean-0, 1~2→clean-1 or 2, 3~4→clean-3, 5~6→clean-4, 7+→clean-5 or 6)
- 합계 행: 각 월 전체 호실 횟수 합 + 비용 합

```js
function cleanClass(n) {
  if (n === 0) return 'clean-0';
  if (n <= 2)  return 'clean-' + n;       // 1 or 2
  if (n <= 4)  return 'clean-3';
  if (n <= 6)  return 'clean-' + (n - 1); // 4 or 5
  return 'clean-6';
}

function fmtWon(n) {
  return '₩' + n.toLocaleString('ko-KR');
}

function renderCleaningStats() {
  const grid = document.getElementById('statsGrid');
  const t = today();
  const months = [];
  for (let i = 11; i >= 0; i--) {
    let y = t.y, m = t.m - i;
    while (m < 0) { m += 12; y--; }
    months.push({ y, m });
  }

  const hasData = Object.keys(archiveData).length > 0 &&
    rooms.some(r => {
      const rd = archiveData[r.name];
      return rd && ['ab','bk','tr','lv'].some(k => (rd[k]||[]).length > 0);
    });

  if (!rooms.length || !hasData) {
    grid.innerHTML = `<div class="stats-empty"><strong>🧹</strong>아직 데이터가 없어요.<br>Sync를 실행하면 이번 달부터 쌓이기 시작해요.</div>`;
    return;
  }

  const thead = `<tr>
    <th class="stats-room-th">호실</th>
    ${months.map(({y,m}) => {
      const isCur = y===t.y && m===t.m;
      const label = y!==t.y ? `${String(y).slice(2)}/${m+1}` : `${m+1}월`;
      return `<th class="stats-month-th${isCur?' cur-month':''}">${label}</th>`;
    }).join('')}
  </tr>`;

  const roomRows = rooms.map(r => {
    const cells = months.map(({y,m}) => {
      const {short, long, noData} = calcCleanings(r.name, statsPlatform, y, m);
      if (noData) return `<td class="stats-cell-td"><div class="stats-cell no-data">—</div></td>`;
      const total = short + long;
      const cost  = short * cleanFeeShort + long * cleanFeeLong;
      const cls   = cleanClass(total);
      const tip   = `단기 ${short}회 · 장기 ${long}회 · ${fmtWon(cost)}`;
      const costLabel = total > 0 ? fmtWon(cost).replace('₩','₩') : '';
      return `<td class="stats-cell-td"><div class="stats-cell clean-cell ${cls}" data-tip="${tip}">
        <span class="stats-clean-count">${total}회</span>
        <span class="stats-clean-cost">${total > 0 ? '₩'+Math.round(cost/1000)+'k' : ''}</span>
      </div></td>`;
    }).join('');
    return `<tr><td class="stats-room-td"><span class="room-dot-s" style="background:${r.color}"></span>${r.name}</td>${cells}</tr>`;
  }).join('');

  const totalCells = months.map(({y,m}) => {
    let sumShort = 0, sumLong = 0, validCount = 0;
    for (const r of rooms) {
      const {short, long, noData} = calcCleanings(r.name, statsPlatform, y, m);
      if (!noData) { sumShort += short; sumLong += long; validCount++; }
    }
    if (!validCount) return `<td class="stats-total-cell-td"><div class="stats-cell no-data">—</div></td>`;
    const total = sumShort + sumLong;
    const cost  = sumShort * cleanFeeShort + sumLong * cleanFeeLong;
    const cls   = cleanClass(Math.min(total, 6));
    return `<td class="stats-total-cell-td"><div class="stats-cell clean-cell ${cls}" data-tip="${total}회 · ${fmtWon(cost)}">
      <span class="stats-clean-count">${total}회</span>
      <span class="stats-clean-cost">₩${Math.round(cost/1000)}k</span>
    </div></td>`;
  }).join('');

  const totalRow = `<tr><td class="stats-total-td">합계</td>${totalCells}</tr>`;
  grid.innerHTML = `<table class="stats-table"><thead>${thead}</thead><tbody>${roomRows}${totalRow}</tbody></table>`;

  // 툴팁 이벤트 — 기존 renderStats()와 동일 패턴
  grid.querySelectorAll('.stats-cell[data-tip]').forEach(cell => {
    cell.addEventListener('click', e => {
      const wasActive = cell.classList.contains('tip-active');
      grid.querySelectorAll('.stats-cell.tip-active').forEach(c => {
        c.classList.remove('tip-active');
        c.querySelector('.stats-tooltip')?.remove();
      });
      if (!wasActive) {
        cell.classList.add('tip-active');
        const tip = document.createElement('div');
        tip.className = 'stats-tooltip';
        tip.textContent = cell.dataset.tip;
        cell.appendChild(tip);
      }
      e.stopPropagation();
    });
  });
}
```

- **예상 추가 라인**: ~55줄

---

### Task 6: JS — renderStats() 분기 처리
- **파일**: `app/JnJ booking/index.html`
- **위치**: 기존 `function renderStats()` 앞에 이름 변경 + wrapper 추가

1. 기존 `function renderStats() {` → `function renderOccupancy() {` 로 이름 변경
2. 새 `renderStats()` wrapper 함수 추가:

```js
function renderStats() {
  if (statsView === 'cleaning') renderCleaningStats();
  else renderOccupancy();
}
```

- **변경 라인**: 기존 1줄 수정 + 5줄 추가

---

### Task 7: JS — 이벤트 바인딩 추가 및 수정
- **파일**: `app/JnJ booking/index.html`
- **위치**: `// ── EVENTS ──` 섹션, 기존 statsPlatformTabs 이벤트 바로 뒤

**추가 1**: 뷰 토글 탭 클릭 이벤트
```js
document.getElementById('statsViewTabs').addEventListener('click', e => {
  const btn = e.target.closest('.stats-vtab');
  if (!btn) return;
  document.querySelectorAll('.stats-vtab').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  statsView = btn.dataset.view;
  document.getElementById('statsFeeRow').style.display = statsView === 'cleaning' ? 'flex' : 'none';
  renderStats();
});
```

**추가 2**: 단가 입력 change 이벤트 (양쪽 input 공통)
```js
['feeShortInput','feeLongInput'].forEach(id => {
  document.getElementById(id).addEventListener('input', () => {
    cleanFeeShort = parseInt(document.getElementById('feeShortInput').value, 10) || 0;
    cleanFeeLong  = parseInt(document.getElementById('feeLongInput').value, 10) || 0;
    localStorage.setItem('hana_clean_fee_short', cleanFeeShort);
    localStorage.setItem('hana_clean_fee_long',  cleanFeeLong);
    if (statsView === 'cleaning') renderCleaningStats();
  });
});
```

**수정**: `openStatsPanel()` — 단가 input 초기값 동기화
```js
function openStatsPanel() {
  document.getElementById('statsPanel').classList.add('open');
  document.getElementById('feeShortInput').value = cleanFeeShort;
  document.getElementById('feeLongInput').value  = cleanFeeLong;
  renderStats();
}
```

**수정**: `closeStatsPanel()` — 뷰 상태 초기화 (선택)  
현재 뷰 상태를 유지하는 게 자연스러우므로 초기화 없이 그대로 둠.

- **예상 추가/수정 라인**: ~20줄

---

## 변경 파일 목록
- [x] `app/JnJ booking/index.html` — CSS/HTML/JS 전부 이 파일 하나

## 롤백 방법
- git checkout로 `app/JnJ booking/index.html` 단일 파일 되돌리기
- localStorage의 `hana_clean_fee_short`, `hana_clean_fee_long` 키는 삭제해도 무방 (기본값으로 동작)

## 테스트 시나리오
- [ ] 📊 버튼 탭 → 통계 패널 열림, 기본 "점유율" 뷰 표시
- [ ] "청소비용" 탭 탭 → 테이블 전환, 단가 입력 행 노출
- [ ] "점유율" 탭 탭 → 다시 점유율 뷰, 단가 입력 행 사라짐
- [ ] 단가 변경 → 즉시 테이블 재계산
- [ ] 단가 localStorage 저장 확인 → 앱 재시작 후 값 유지
- [ ] 플랫폼 탭(✈/🏨 등) 전환 → 청소비용 뷰에서도 필터 작동
- [ ] 셀 탭 → 툴팁에 "단기 N회 · 장기 M회 · ₩XXX,XXX" 표시
- [ ] 합계 행 → 월별 전체 횟수 합산, 금액 올바른지 확인
- [ ] 데이터 없는 호실/월 → "—" 표시
- [ ] 기존 점유율 통계 정상 동작 확인 (회귀 테스트)

## 예상 주의사항
- `coutM`은 0-indexed — calcCleanings 내 `bk.coutM === month` 비교 시 그대로 사용
- `renderOccupancy()` 이름 변경 시 `renderStats()` wrapper가 호출하므로 외부 호출부 수정 불필요
- 툴팁 이벤트는 renderCleaningStats() 내부에서 직접 등록 (기존 renderOccupancy와 동일 패턴)
- `cleanClass(n)` 에서 n이 0일 때 `clean-0` (회색)으로 표시 — 데이터 있지만 해당 월 청소 없음과 noData를 시각적으로 구분할 것

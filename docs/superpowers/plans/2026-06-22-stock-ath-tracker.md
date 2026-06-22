# Stock ATH Tracker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `stock_tracker.html` 단일 파일 — 종목 등록 후 Yahoo Finance 데이터로 ATH 대비 현재가 % 표시

**Architecture:** 단일 정적 HTML 파일에 CSS·JS 인라인 포함. Yahoo Finance v8 Chart API 직접 호출 (CORS 실패 시 allorigins.win 프록시 폴백). watchlist는 localStorage 저장.

**Tech Stack:** HTML5, CSS3 (Grid/Flexbox), Vanilla JS (ES2020+), Yahoo Finance v8 Chart API

## Global Constraints

- 외부 라이브러리·CDN 없음 — 순수 HTML/CSS/JS
- API 키 불필요
- 단일 파일 `stock_tracker.html` (index.html 변경 금지)
- localStorage key: `ath_watchlist` (JSON 배열)
- Yahoo Finance 엔드포인트: `https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1mo&range=max`
- CORS 프록시 폴백: `https://api.allorigins.win/raw?url={encodedUrl}`

---

### Task 1: HTML 구조 + CSS 스타일

**Files:**
- Create: `stock_tracker.html`

**Produces:**
- 빈 카드 그리드, 입력창, 버튼이 스타일된 정적 페이지
- 각 CSS 클래스: `.stock-card`, `.ticker`, `.current-price`, `.pct-badge.negative`, `.pct-badge.positive`, `.progress-bar`, `.spinner`

- [ ] **Step 1: `stock_tracker.html` 파일 생성 — HTML 뼈대 + 전체 CSS**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>주가 ATH 트래커</title>
  <style>
    *, *::before, *::after { margin: 0; padding: 0; box-sizing: border-box; }

    body {
      background: #0f172a;
      color: #e2e8f0;
      font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
      min-height: 100vh;
      padding: 24px 16px;
    }

    header { text-align: center; margin-bottom: 32px; }

    h1 {
      font-size: 2rem;
      font-weight: 700;
      background: linear-gradient(135deg, #6366f1, #8b5cf6);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
      margin-bottom: 6px;
    }

    .subtitle { color: #64748b; font-size: 0.875rem; }

    .add-section {
      display: flex;
      gap: 10px;
      max-width: 480px;
      margin: 0 auto 24px;
    }

    .add-section input {
      flex: 1;
      background: #1e293b;
      border: 1px solid #334155;
      border-radius: 10px;
      padding: 11px 16px;
      color: #e2e8f0;
      font-size: 1rem;
      text-transform: uppercase;
      outline: none;
      transition: border-color 0.2s;
    }
    .add-section input::placeholder { text-transform: none; color: #475569; }
    .add-section input:focus { border-color: #6366f1; }

    .btn-add {
      background: linear-gradient(135deg, #6366f1, #8b5cf6);
      color: #fff;
      border: none;
      border-radius: 10px;
      padding: 11px 22px;
      font-size: 0.95rem;
      font-weight: 600;
      cursor: pointer;
      transition: opacity 0.2s, transform 0.1s;
      white-space: nowrap;
    }
    .btn-add:hover { opacity: 0.88; }
    .btn-add:active { transform: scale(0.97); }
    .btn-add:disabled { opacity: 0.45; cursor: not-allowed; }

    .controls {
      display: flex;
      justify-content: space-between;
      align-items: center;
      max-width: 1100px;
      margin: 0 auto 14px;
    }

    .stock-count { color: #64748b; font-size: 0.82rem; }

    .btn-refresh {
      display: flex;
      align-items: center;
      gap: 6px;
      background: #1e293b;
      border: 1px solid #334155;
      border-radius: 8px;
      padding: 7px 14px;
      color: #94a3b8;
      font-size: 0.82rem;
      cursor: pointer;
      transition: background 0.2s, color 0.2s;
    }
    .btn-refresh:hover { background: #334155; color: #e2e8f0; }
    .btn-refresh:disabled { opacity: 0.5; cursor: not-allowed; }
    .btn-refresh .icon { display: inline-block; }
    .btn-refresh.spinning .icon { animation: spin 0.9s linear infinite; }

    @keyframes spin { to { transform: rotate(360deg); } }

    .stock-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
      gap: 14px;
      max-width: 1100px;
      margin: 0 auto;
    }

    .stock-card {
      background: #1e293b;
      border: 1px solid #334155;
      border-radius: 14px;
      padding: 20px;
      transition: border-color 0.2s, transform 0.2s, opacity 0.2s;
    }
    .stock-card:hover { border-color: #475569; transform: translateY(-2px); }
    .stock-card.loading { opacity: 0.65; }
    .stock-card.error { border-color: rgba(239,68,68,0.4); background: rgba(239,68,68,0.06); }

    .card-header {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      margin-bottom: 16px;
    }

    .ticker { font-size: 1.25rem; font-weight: 700; color: #f1f5f9; }

    .company-name {
      font-size: 0.78rem;
      color: #64748b;
      margin-top: 3px;
      max-width: 200px;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .btn-delete {
      background: none;
      border: none;
      color: #475569;
      cursor: pointer;
      font-size: 1rem;
      line-height: 1;
      padding: 2px 4px;
      transition: color 0.2s;
    }
    .btn-delete:hover { color: #ef4444; }

    .price-row {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 14px;
    }

    .current-price { font-size: 1.55rem; font-weight: 700; color: #f1f5f9; }

    .pct-badge {
      font-size: 1rem;
      font-weight: 700;
      padding: 5px 11px;
      border-radius: 8px;
    }
    .pct-badge.negative { background: rgba(239,68,68,0.14); color: #f87171; }
    .pct-badge.positive { background: rgba(34,197,94,0.14); color: #4ade80; }

    .ath-row {
      display: flex;
      justify-content: space-between;
      font-size: 0.78rem;
      color: #64748b;
      margin-bottom: 10px;
    }
    .ath-value { color: #94a3b8; font-weight: 600; }

    .progress-track {
      background: #0f172a;
      border-radius: 4px;
      height: 5px;
      overflow: hidden;
    }
    .progress-bar {
      height: 100%;
      border-radius: 4px;
      transition: width 0.5s ease;
    }
    .progress-bar.negative { background: linear-gradient(90deg, #dc2626, #f87171); }
    .progress-bar.positive { background: linear-gradient(90deg, #16a34a, #4ade80); }

    .spinner-wrap { display: flex; justify-content: center; padding: 18px 0; }
    .spinner {
      width: 26px; height: 26px;
      border: 3px solid #334155;
      border-top-color: #6366f1;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }

    .error-msg { color: #f87171; font-size: 0.84rem; text-align: center; padding: 10px 0; }

    .empty-state {
      text-align: center;
      padding: 52px 20px;
      color: #475569;
      grid-column: 1 / -1;
    }
    .empty-state .icon { font-size: 2.8rem; margin-bottom: 10px; }
    .empty-state h3 { font-size: 1rem; color: #64748b; margin-bottom: 6px; }
    .empty-state p { font-size: 0.82rem; }

    .last-updated { text-align: center; color: #475569; font-size: 0.73rem; margin-top: 22px; }

    .toast {
      position: fixed; bottom: 22px; right: 22px;
      background: #1e293b; border: 1px solid #334155;
      border-radius: 10px; padding: 11px 18px; font-size: 0.88rem;
      animation: slideIn 0.25s ease; z-index: 999;
    }
    .toast.error { border-color: rgba(239,68,68,0.5); color: #f87171; }
    .toast.success { border-color: rgba(34,197,94,0.5); color: #4ade80; }

    @keyframes slideIn {
      from { transform: translateX(80px); opacity: 0; }
      to   { transform: translateX(0);   opacity: 1; }
    }
  </style>
</head>
<body>
  <header>
    <h1>📈 주가 ATH 트래커</h1>
    <p class="subtitle">역대 최고가(ATH) 대비 현재 주가 위치 확인</p>
  </header>

  <div class="add-section">
    <input id="tickerInput" type="text" placeholder="종목 코드 입력 (예: AAPL, TSLA, NVDA)" maxlength="10">
    <button class="btn-add" id="addBtn" onclick="addStock()">+ 추가</button>
  </div>

  <div class="controls">
    <span class="stock-count" id="stockCount"></span>
    <button class="btn-refresh" id="refreshBtn" onclick="refreshAll()">
      <span class="icon">↻</span> 전체 새로고침
    </button>
  </div>

  <div class="stock-grid" id="stockGrid"></div>
  <p class="last-updated" id="lastUpdated"></p>

  <script>
    /* JS는 Task 2~4에서 추가 */
  </script>
</body>
</html>
```

- [ ] **Step 2: 브라우저에서 열어 UI 확인**

  파일 경로를 브라우저에서 열거나 Live Server로 확인.
  체크 항목:
  - 다크 배경, 보라색 그라디언트 제목 표시
  - 입력창 + 추가 버튼 중앙 정렬
  - 빈 상태 (카드 없음) 정상 렌더링

- [ ] **Step 3: 커밋**

```bash
git add stock_tracker.html
git commit -m "feat: stock ATH tracker — HTML 구조 및 CSS 스타일"
```

---

### Task 2: localStorage Watchlist + 토스트

**Files:**
- Modify: `stock_tracker.html` — `<script>` 블록

**Interfaces:**
- Produces:
  - `watchlist: string[]` — 전역 배열 (ticker 심볼 목록)
  - `saveWatchlist(): void`
  - `updateControls(): void`
  - `showToast(msg: string, type: 'success' | 'error'): void`
  - `renderEmpty(): void`

- [ ] **Step 1: `<script>` 블록에 watchlist 관리 코드 추가**

  Task 1에서 작성한 `/* JS는 Task 2~4에서 추가 */` 주석을 아래 코드로 교체:

```javascript
// ── State ──────────────────────────────────────────────
let watchlist = JSON.parse(localStorage.getItem('ath_watchlist') || '[]');
const stockCache = {};  // symbol → { currentPrice, ath, athDate, pctFromAth, currency, companyName }

// ── Persistence ────────────────────────────────────────
function saveWatchlist() {
  localStorage.setItem('ath_watchlist', JSON.stringify(watchlist));
}

function updateControls() {
  document.getElementById('stockCount').textContent =
    watchlist.length > 0 ? `총 ${watchlist.length}개 종목` : '';
}

// ── Toast ───────────────────────────────────────────────
function showToast(msg, type = 'success') {
  const el = document.createElement('div');
  el.className = `toast ${type}`;
  el.textContent = msg;
  document.body.appendChild(el);
  setTimeout(() => el.remove(), 3000);
}

// ── Empty state ─────────────────────────────────────────
function renderEmpty() {
  const grid = document.getElementById('stockGrid');
  if (watchlist.length > 0 || grid.querySelector('.stock-card')) return;
  grid.innerHTML = `
    <div class="empty-state">
      <div class="icon">📊</div>
      <h3>아직 추가된 종목이 없습니다</h3>
      <p>위에서 종목 코드를 입력해 추가해보세요 (예: AAPL, TSLA, NVDA, MSFT)</p>
    </div>`;
}

// ── Enter key ───────────────────────────────────────────
document.getElementById('tickerInput')
  .addEventListener('keydown', e => { if (e.key === 'Enter') addStock(); });

// placeholder — Task 3, 4에서 구현
function addStock() {}
function refreshAll() {}
```

- [ ] **Step 2: 브라우저 콘솔에서 동작 확인**

  개발자 도구 콘솔에서 실행:
  ```javascript
  watchlist.push('TEST'); saveWatchlist();
  localStorage.getItem('ath_watchlist'); // → '["TEST"]'
  showToast('토스트 테스트', 'success');  // 우하단 초록 토스트
  showToast('에러 테스트', 'error');      // 우하단 빨간 토스트
  ```

- [ ] **Step 3: 커밋**

```bash
git add stock_tracker.html
git commit -m "feat: watchlist localStorage 저장 및 토스트 알림"
```

---

### Task 3: Yahoo Finance API 호출 + ATH 계산

**Files:**
- Modify: `stock_tracker.html` — `<script>` 블록 (Task 2 코드 아래 추가)

**Interfaces:**
- Consumes: 없음 (외부 API)
- Produces:
  - `fetchStockData(symbol: string): Promise<StockData>`
  - `StockData = { currentPrice: number, ath: number, athDate: string, pctFromAth: number, currency: string, companyName: string }`

- [ ] **Step 1: API 호출 + ATH 계산 함수 추가** (`renderEmpty()` 아래에 삽입)

```javascript
// ── API ─────────────────────────────────────────────────
async function fetchYahoo(url) {
  // 직접 호출 시도
  try {
    const res = await fetch(url, { signal: AbortSignal.timeout(8000) });
    if (res.ok) return await res.json();
  } catch (_) { /* CORS or timeout → proxy */ }

  // CORS 프록시 폴백
  const proxy = `https://api.allorigins.win/raw?url=${encodeURIComponent(url)}`;
  const res = await fetch(proxy, { signal: AbortSignal.timeout(14000) });
  if (!res.ok) throw new Error('데이터를 가져올 수 없습니다');
  return res.json();
}

async function fetchStockData(symbol) {
  const url = `https://query1.finance.yahoo.com/v8/finance/chart/${encodeURIComponent(symbol)}?interval=1mo&range=max`;
  const data = await fetchYahoo(url);

  const result = data?.chart?.result?.[0];
  if (!result) throw new Error('종목을 찾을 수 없습니다');

  const meta       = result.meta;
  const highs      = result.indicators?.quote?.[0]?.high ?? [];
  const timestamps = result.timestamp ?? [];

  // ATH: 월봉 high 배열 전체에서 최댓값
  let ath = 0, athTimestamp = 0;
  highs.forEach((h, i) => {
    if (h != null && h > ath) { ath = h; athTimestamp = timestamps[i]; }
  });

  if (ath === 0) throw new Error('가격 데이터가 없습니다');

  const currentPrice = meta.regularMarketPrice;
  const currency     = meta.currency ?? 'USD';
  const companyName  = meta.longName ?? meta.shortName ?? symbol;
  const athDate      = athTimestamp
    ? new Date(athTimestamp * 1000).toLocaleDateString('ko-KR', { year: 'numeric', month: 'long', day: 'numeric' })
    : '-';
  const pctFromAth   = ((currentPrice - ath) / ath) * 100;

  return { currentPrice, ath, athDate, pctFromAth, currency, companyName };
}
```

- [ ] **Step 2: 브라우저 콘솔에서 실제 API 응답 확인**

```javascript
fetchStockData('AAPL').then(d => console.log(d));
// 예상 출력:
// { currentPrice: 213.55, ath: 260.10, athDate: '2025년 X월 X일',
//   pctFromAth: -17.9, currency: 'USD', companyName: 'Apple Inc.' }
```
  - `pctFromAth` 가 음수 (ATH 달성 후 하락 상태)인지 확인
  - `ath` 값이 `currentPrice` 보다 크거나 같은지 확인

- [ ] **Step 3: 커밋**

```bash
git add stock_tracker.html
git commit -m "feat: Yahoo Finance API 호출 및 ATH 계산 로직"
```

---

### Task 4: 카드 렌더링 + addStock + refreshAll

**Files:**
- Modify: `stock_tracker.html` — `<script>` 블록 (Task 3 코드 아래 추가, placeholder 함수 교체)

**Interfaces:**
- Consumes:
  - `fetchStockData(symbol)` → `StockData` (Task 3)
  - `saveWatchlist()`, `updateControls()`, `showToast()`, `renderEmpty()` (Task 2)
  - `watchlist`, `stockCache` (Task 2 전역)
- Produces:
  - `renderCard(symbol, data, loading, error)` — 카드 DOM 생성/갱신
  - `addStock()` — 입력창 → watchlist 추가
  - `refreshAll()` — 전체 카드 새로고침

- [ ] **Step 1: `formatPrice` 유틸 + `renderCard` 추가** (Task 3 코드 아래)

```javascript
// ── Utils ────────────────────────────────────────────────
function formatPrice(price, currency) {
  const symbol = currency === 'KRW' ? '₩' : '$';
  let formatted;
  if (price >= 1000)       formatted = price.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
  else if (price >= 1)     formatted = price.toFixed(2);
  else                     formatted = price.toFixed(4);
  return `${symbol}${formatted}`;
}

// ── Card renderer ─────────────────────────────────────────
function renderCard(symbol, data, loading, errorMsg) {
  const grid = document.getElementById('stockGrid');

  // 빈 상태 제거
  const empty = grid.querySelector('.empty-state');
  if (empty) empty.remove();

  let card = document.getElementById(`card-${symbol}`);
  if (!card) {
    card = document.createElement('div');
    card.id = `card-${symbol}`;
    grid.appendChild(card);
  }

  if (loading) {
    card.className = 'stock-card loading';
    card.innerHTML = `
      <div class="card-header">
        <div><div class="ticker">${symbol}</div><div class="company-name">불러오는 중...</div></div>
        <button class="btn-delete" onclick="removeStock('${symbol}')">✕</button>
      </div>
      <div class="spinner-wrap"><div class="spinner"></div></div>`;
    return;
  }

  if (errorMsg) {
    card.className = 'stock-card error';
    card.innerHTML = `
      <div class="card-header">
        <div><div class="ticker">${symbol}</div></div>
        <button class="btn-delete" onclick="removeStock('${symbol}')">✕</button>
      </div>
      <div class="error-msg">⚠️ ${errorMsg}</div>`;
    return;
  }

  const { currentPrice, ath, athDate, pctFromAth, currency, companyName } = data;
  const negative   = pctFromAth < 0;
  const pctText    = `${pctFromAth >= 0 ? '+' : ''}${pctFromAth.toFixed(2)}%`;
  const progressW  = Math.min(100, Math.max(2, (currentPrice / ath) * 100)).toFixed(1);

  card.className = 'stock-card';
  card.innerHTML = `
    <div class="card-header">
      <div>
        <div class="ticker">${symbol}</div>
        <div class="company-name" title="${companyName}">${companyName}</div>
      </div>
      <button class="btn-delete" onclick="removeStock('${symbol}')">✕</button>
    </div>
    <div class="price-row">
      <div class="current-price">${formatPrice(currentPrice, currency)}</div>
      <div class="pct-badge ${negative ? 'negative' : 'positive'}">${pctText}</div>
    </div>
    <div class="ath-row">
      <span>역대 최고가 <span class="ath-value">${formatPrice(ath, currency)}</span></span>
      <span>${athDate}</span>
    </div>
    <div class="progress-track">
      <div class="progress-bar ${negative ? 'negative' : 'positive'}" style="width:${progressW}%"></div>
    </div>`;
}
```

- [ ] **Step 2: placeholder `addStock`, `refreshAll` 교체 + `removeStock` + 초기화**

  Task 2에서 작성한 `function addStock() {}` 와 `function refreshAll() {}` 를 아래로 교체:

```javascript
// ── Actions ───────────────────────────────────────────────
async function addStock() {
  const input  = document.getElementById('tickerInput');
  const addBtn = document.getElementById('addBtn');
  const symbol = input.value.trim().toUpperCase();
  if (!symbol) return;
  if (watchlist.includes(symbol)) {
    showToast(`${symbol}은(는) 이미 추가되어 있습니다`, 'error');
    return;
  }

  input.value = '';
  addBtn.disabled = true;
  watchlist.push(symbol);
  saveWatchlist();
  updateControls();
  renderCard(symbol, null, true);

  try {
    const data = await fetchStockData(symbol);
    stockCache[symbol] = data;
    renderCard(symbol, data, false);
    showToast(`${symbol} 추가 완료`, 'success');
  } catch (e) {
    renderCard(symbol, null, false, e.message);
  } finally {
    addBtn.disabled = false;
  }
}

function removeStock(symbol) {
  watchlist = watchlist.filter(s => s !== symbol);
  delete stockCache[symbol];
  saveWatchlist();
  updateControls();
  document.getElementById(`card-${symbol}`)?.remove();
  renderEmpty();
}

async function refreshAll() {
  if (watchlist.length === 0) return;
  const btn = document.getElementById('refreshBtn');
  btn.classList.add('spinning');
  btn.disabled = true;

  await Promise.all(watchlist.map(async symbol => {
    renderCard(symbol, stockCache[symbol] ?? null, true);
    try {
      const data = await fetchStockData(symbol);
      stockCache[symbol] = data;
      renderCard(symbol, data, false);
    } catch (e) {
      renderCard(symbol, null, false, e.message);
    }
  }));

  btn.classList.remove('spinning');
  btn.disabled = false;
  document.getElementById('lastUpdated').textContent =
    `마지막 업데이트: ${new Date().toLocaleTimeString('ko-KR')}`;
}

// ── Init ──────────────────────────────────────────────────
(async function init() {
  updateControls();
  if (watchlist.length === 0) { renderEmpty(); return; }
  await refreshAll();
})();

// 5분마다 자동 새로고침
setInterval(refreshAll, 5 * 60 * 1000);
```

- [ ] **Step 3: 브라우저 통합 테스트**

  1. `AAPL` 입력 → 추가 → 로딩 스피너 표시 후 카드 렌더링 확인
  2. `TSLA`, `NVDA` 추가 → 카드 3개 그리드 확인
  3. `INVALID_XYZ` 입력 → 빨간 에러 카드 확인
  4. 페이지 새로고침 → 3개 종목 복원 확인 (localStorage)
  5. 카드 삭제(✕) → 카드 제거 + localStorage 반영 확인
  6. 전체 새로고침 버튼 → 스피너 회전 후 데이터 갱신 확인
  7. % 뱃지: ATH 이하 종목 빨간색, ATH 초과 종목(있다면) 초록색 확인
  8. progress bar: ATH에 가까울수록 더 길게 표시 확인

- [ ] **Step 4: 커밋**

```bash
git add stock_tracker.html
git commit -m "feat: 카드 렌더링, 종목 추가/삭제/새로고침, 자동 갱신 완성"
```

---

## Self-Review

**Spec coverage:**
- [x] 종목 등록 → `addStock()`
- [x] ATH 계산 → `fetchStockData()` 내 `max(highs)`
- [x] ATH 대비 % 표시 → `pctFromAth`, `.pct-badge`
- [x] localStorage 저장 → `saveWatchlist()`
- [x] 자동 새로고침 → `setInterval(refreshAll, 300_000)`
- [x] CORS 프록시 폴백 → `fetchYahoo()`
- [x] 단일 파일, 외부 의존성 없음

**Type consistency:**
- `renderCard(symbol, data, loading, errorMsg)` — Task 4 Step 1, 2 모두 동일 시그니처
- `stockCache` — Task 2에서 선언, Task 4에서 사용
- `fetchStockData` — Task 3에서 정의, Task 4에서 `await fetchStockData(symbol)` 사용
- `formatPrice(price, currency)` — Task 4에서 정의 및 호출 모두 두 인자 사용

**Placeholder scan:** 없음

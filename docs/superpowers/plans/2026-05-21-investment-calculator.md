# Investment Calculator: Stocks vs. Real Estate — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained `index.html` file that compares a $100K S&P 500 investment vs. a leveraged rental property over 30 years, with live-recalculating inputs, a dual-line chart, and a year-by-year table.

**Architecture:** One HTML file, inline CSS, inline JS, Chart.js 4.x loaded from CDN. All financial logic in a single `calculate()` function that returns a `results` object consumed by `renderSummary()`, `renderChart()`, and `renderTable()`. Every input change triggers a full recalculate + re-render cycle.

**Tech Stack:** Vanilla HTML/CSS/JS, Chart.js 4.4.x (CDN), no build step, no npm.

---

## File Map

| File | Responsibility |
|---|---|
| `index.html` | Everything — HTML structure, inline CSS, inline JS |

The JS section is organized as:
1. `CONFIG` — all default values
2. `TOOLTIPS` — tooltip text per input key
3. `buildAmortSchedule(principal, annualRate, termMonths)` — amortization
4. `calculate()` — reads CONFIG, returns `{ rows[], summary }` 
5. `formatMoney(n)` / `formatPct(n)` / `formatNum(n, decimals)` — formatters
6. `renderSummary(summary)` — updates 3 summary cards
7. `renderChart(rows)` — creates/updates Chart.js instance
8. `renderTable(rows, summary)` — builds the year-by-year table
9. `recalculate()` — calls calculate() then all three renderers
10. Input wiring — event listeners on all `+` / `−` buttons

---

## Task 1: HTML Scaffold + CSS Layout

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with full page skeleton**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Stocks vs Real Estate Calculator</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <style>
    /* ── Reset & Base ─────────────────────────────── */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
    body { background: #0d0d0d; color: #ccc; font-size: 13px; }

    /* ── Custom Properties ────────────────────────── */
    :root {
      --bg: #0d0d0d;
      --surface: #111;
      --surface2: #1a1a1a;
      --border: #2a2a2a;
      --text: #ccc;
      --muted: #888;
      --faint: #555;
      --blue: #4da6ff;
      --blue-bg: #0d1a2e;
      --blue-border: #1a3a5a;
      --blue-dim: #0f2240;
      --amber: #f0a030;
      --amber-bg: #1a0d00;
      --amber-border: #5a2e00;
      --amber-dim: #2a1400;
      --red: #e05050;
      --green: #50c878;
      --sidebar-w: 260px;
    }

    /* ── App Shell ────────────────────────────────── */
    #app {
      display: flex;
      height: 100vh;
      overflow: hidden;
    }

    /* ── Sidebar ──────────────────────────────────── */
    #sidebar {
      width: var(--sidebar-w);
      flex-shrink: 0;
      background: #0a0a0a;
      border-right: 1px solid var(--border);
      overflow-y: auto;
      padding: 12px 10px;
      display: flex;
      flex-direction: column;
      gap: 10px;
    }

    /* ── Main Area ────────────────────────────────── */
    #main {
      flex: 1;
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }

    /* ── Topbar ───────────────────────────────────── */
    #topbar {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 10px 16px;
      border-bottom: 1px solid var(--border);
      flex-shrink: 0;
    }
    #topbar h1 {
      font-size: 14px;
      font-weight: 600;
      color: var(--text);
      letter-spacing: 0.3px;
    }
    #assumptions-btn {
      background: #1e1e1e;
      border: 1px solid var(--border);
      color: var(--muted);
      border-radius: 6px;
      padding: 5px 12px;
      cursor: pointer;
      font-size: 12px;
    }
    #assumptions-btn:hover { border-color: var(--blue); color: var(--blue); }

    /* ── Content Scroll Area ──────────────────────── */
    #content {
      flex: 1;
      overflow-y: auto;
      padding: 16px;
      display: flex;
      flex-direction: column;
      gap: 16px;
    }

    /* ── Summary Cards ────────────────────────────── */
    #summary {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 12px;
      flex-shrink: 0;
    }
    .summary-card {
      border-radius: 10px;
      padding: 16px;
      text-align: center;
    }
    .summary-card .label {
      font-size: 10px;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 1px;
      margin-bottom: 8px;
    }
    .summary-card .value {
      font-size: 26px;
      font-weight: 700;
      color: #fff;
      margin-bottom: 4px;
    }
    .summary-card .sub {
      font-size: 11px;
    }
    #card-stocks { background: var(--blue-bg); border: 1px solid var(--blue-border); }
    #card-stocks .label, #card-stocks .sub { color: var(--blue); }
    #card-re { background: var(--amber-bg); border: 1px solid var(--amber-border); }
    #card-re .label, #card-re .sub { color: var(--amber); }
    #card-winner { background: var(--surface); border: 1px solid var(--border); }
    #card-winner .label, #card-winner .sub { color: var(--muted); }

    /* ── Chart ────────────────────────────────────── */
    #chart-container {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 10px;
      padding: 16px;
      flex-shrink: 0;
    }
    #chart-container .section-title {
      font-size: 10px;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: var(--muted);
      margin-bottom: 12px;
    }
    #chart-wrap { position: relative; height: 220px; }

    /* ── Table ────────────────────────────────────── */
    #table-container {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 10px;
      overflow: hidden;
    }
    #table-container .section-title {
      font-size: 10px;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: var(--muted);
      padding: 12px 16px 8px;
    }
    #results-table {
      width: 100%;
      border-collapse: collapse;
      font-size: 11px;
    }
    #results-table th {
      padding: 7px 10px;
      text-align: right;
      font-weight: 600;
      font-size: 10px;
      text-transform: uppercase;
      letter-spacing: 0.5px;
      border-bottom: 1px solid var(--border);
      white-space: nowrap;
      background: #0f0f0f;
    }
    #results-table th:first-child { text-align: center; }
    #results-table td {
      padding: 6px 10px;
      text-align: right;
      border-bottom: 1px solid #161616;
      white-space: nowrap;
    }
    #results-table td:first-child { text-align: center; color: var(--muted); }
    #results-table tr:last-child td { border-bottom: none; }
    #results-table tr:hover td { background: #161616; }
    .th-stocks { color: var(--blue) !important; }
    .th-re { color: var(--amber) !important; }
    .pos { color: var(--green); }
    .neg { color: var(--red); }
    .highlight-stocks { color: var(--blue); }
    .highlight-re { color: var(--amber); }
    .refi-row td { border-top: 1px dashed var(--amber-border) !important; }

    /* ── Assumptions Modal ────────────────────────── */
    #modal-overlay {
      display: none;
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.7);
      z-index: 100;
      align-items: center;
      justify-content: center;
    }
    #modal-overlay.open { display: flex; }
    #modal-box {
      background: #161616;
      border: 1px solid var(--border);
      border-radius: 12px;
      padding: 24px;
      max-width: 560px;
      width: 90%;
      max-height: 80vh;
      overflow-y: auto;
    }
    #modal-box h2 { font-size: 15px; margin-bottom: 16px; color: #fff; }
    #modal-box h3 { font-size: 11px; text-transform: uppercase; letter-spacing: 1px; color: var(--muted); margin: 14px 0 6px; }
    #modal-box ul { list-style: none; display: flex; flex-direction: column; gap: 5px; }
    #modal-box li { font-size: 12px; color: var(--text); padding-left: 12px; position: relative; }
    #modal-box li::before { content: '·'; position: absolute; left: 0; color: var(--muted); }
    #modal-close {
      position: absolute;
      top: 16px; right: 16px;
      background: none; border: none; color: var(--muted); font-size: 20px; cursor: pointer;
    }
    #modal-box { position: relative; }
  </style>
</head>
<body>
  <div id="app">

    <!-- SIDEBAR -->
    <div id="sidebar">
      <!-- Input groups injected here by Task 2 -->
    </div>

    <!-- MAIN -->
    <div id="main">
      <div id="topbar">
        <h1>Stocks vs Real Estate Calculator</h1>
        <button id="assumptions-btn">📋 Assumptions</button>
      </div>
      <div id="content">
        <div id="summary">
          <div class="summary-card" id="card-stocks">
            <div class="label">S&amp;P 500 After Tax</div>
            <div class="value" id="stocks-value">—</div>
            <div class="sub" id="stocks-sub">—</div>
          </div>
          <div class="summary-card" id="card-re">
            <div class="label">Real Estate After Tax</div>
            <div class="value" id="re-value">—</div>
            <div class="sub" id="re-sub">—</div>
          </div>
          <div class="summary-card" id="card-winner">
            <div class="label">Winner</div>
            <div class="value" id="winner-value">—</div>
            <div class="sub" id="winner-sub">—</div>
          </div>
        </div>
        <div id="chart-container">
          <div class="section-title">Portfolio Value Over Time (Pre-Tax)</div>
          <div id="chart-wrap">
            <canvas id="chart"></canvas>
          </div>
        </div>
        <div id="table-container">
          <div class="section-title">Year-by-Year Breakdown</div>
          <table id="results-table">
            <thead id="table-head"></thead>
            <tbody id="table-body"></tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- ASSUMPTIONS MODAL -->
    <div id="modal-overlay">
      <div id="modal-box">
        <button id="modal-close">×</button>
        <h2>Model Assumptions</h2>
        <h3>Stocks Track</h3>
        <ul>
          <li>Annual Total Return includes price appreciation + dividends reinvested (not separately taxed during hold)</li>
          <li>Position not sold until end of horizon; all gains taxed as LTCG in final year</li>
          <li>Additional out-of-pocket RE costs (closing costs, negative cash flow years) are also invested in stocks to maintain apples-to-apples comparison</li>
        </ul>
        <h3>Real Estate — Property</h3>
        <ul>
          <li>Building value = 80% of purchase price (IRS land/improvement split for depreciation)</li>
          <li>Depreciation: straight-line over 27.5 years (residential rental, IRS Section 168)</li>
          <li>Purchase closing costs: 2.5% of purchase price (added to stocks track as equivalent investment)</li>
          <li>Selling costs: 6% of sale price (deducted from sale proceeds)</li>
        </ul>
        <h3>Real Estate — Mortgage</h3>
        <ul>
          <li>Standard fixed-rate amortization (30yr)</li>
          <li>Refinance uses remaining term (same payoff date — does not restart 30yr clock)</li>
          <li>Refinance cost: % of remaining balance at refi year, counted as negative cash flow</li>
        </ul>
        <h3>Real Estate — Tax</h3>
        <ul>
          <li>Schedule E deductions: mortgage interest, property tax, insurance, maintenance, CapEx, management, depreciation</li>
          <li>Passive activity losses applied fully in year incurred (no PAL carryforward limitation — simplified)</li>
          <li>Depreciation recapture at sale: IRS Section 1250, 25% rate on accumulated depreciation (capped at actual gain)</li>
          <li>Remaining gain above recapture amount taxed at RE LTCG rate</li>
        </ul>
        <h3>Cash Flow Recycling</h3>
        <ul>
          <li>Positive net cash flow → invested in RE's S&amp;P bucket (same annual return, LTCG at sale)</li>
          <li>Negative net cash flow → out-of-pocket; same dollar amount added to stocks track as additional investment</li>
        </ul>
        <h3>Scope Exclusions</h3>
        <ul>
          <li>No inflation adjustment (all figures in nominal dollars)</li>
          <li>No state income tax, AMT, or 1031 exchange</li>
          <li>No mortgage PMI (assumes ≥20% down payment removes PMI)</li>
          <li>No HOA fees</li>
        </ul>
      </div>
    </div>

  </div>
  <script>
    // JS injected in later tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` in browser**

Expected: Dark page with sidebar (empty), topbar with title + button, three empty summary cards, chart container placeholder, table container. No errors in console.

- [ ] **Step 3: Commit**

```bash
cd /Users/eyal/dev/stocks-vs-realestate
git init
git add index.html
git commit -m "feat: scaffold HTML structure and CSS layout"
```

---

## Task 2: Sidebar Input Groups

**Files:**
- Modify: `index.html` — replace `<!-- Input groups injected here by Task 2 -->` with full sidebar HTML, and add sidebar CSS

- [ ] **Step 1: Add input group CSS inside `<style>` (before the closing `</style>` tag)**

```css
/* ── Input Groups ─────────────────────────────── */
.input-group {
  border-radius: 8px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.input-group.group-general { background: #161616; border: 1px solid #333; }
.input-group.group-stocks { background: var(--blue-bg); border: 1px solid var(--blue-border); }
.input-group.group-re { background: var(--amber-bg); border: 1px solid var(--amber-border); }

.group-header {
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 1px;
  text-transform: uppercase;
  padding-bottom: 4px;
  border-bottom: 1px solid;
  margin-bottom: 2px;
}
.group-general .group-header { color: var(--text); border-color: #333; }
.group-stocks .group-header { color: var(--blue); border-color: var(--blue-border); }
.group-re .group-header { color: var(--amber); border-color: var(--amber-border); }

.input-row { display: flex; flex-direction: column; gap: 3px; }

.input-label-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.input-label {
  font-size: 10px;
  color: var(--muted);
  display: flex;
  align-items: center;
  gap: 4px;
}
.input-hint { font-size: 9px; color: var(--faint); }

.input-ctrl {
  display: flex;
  align-items: center;
  gap: 4px;
}
.input-ctrl button {
  width: 24px;
  height: 26px;
  border-radius: 4px;
  border: 1px solid;
  cursor: pointer;
  font-size: 14px;
  font-weight: 700;
  line-height: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: opacity 0.1s;
}
.input-ctrl button:active { opacity: 0.7; }
.group-general .input-ctrl button { background: #222; border-color: #444; color: var(--text); }
.group-stocks .input-ctrl button { background: var(--blue-dim); border-color: var(--blue-border); color: var(--blue); }
.group-re .input-ctrl button { background: var(--amber-dim); border-color: var(--amber-border); color: var(--amber); }

.input-display {
  flex: 1;
  background: #070707;
  border: 1px solid;
  border-radius: 4px;
  padding: 4px 8px;
  color: #fff;
  text-align: right;
  font-size: 12px;
  font-weight: 500;
}
.group-general .input-display { border-color: #333; }
.group-stocks .input-display { border-color: var(--blue-border); }
.group-re .input-display { border-color: var(--amber-border); }

/* ── Tooltip ──────────────────────────────────── */
.tip {
  position: relative;
  display: inline-flex;
  cursor: help;
}
.tip-icon {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #2a2a2a;
  color: var(--muted);
  font-size: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-style: normal;
  user-select: none;
}
.tip:hover .tip-icon { background: #3a3a3a; color: var(--text); }
.tip-text {
  display: none;
  position: absolute;
  left: 50%;
  bottom: calc(100% + 6px);
  transform: translateX(-50%);
  width: 220px;
  background: #1e1e1e;
  border: 1px solid #444;
  border-radius: 6px;
  padding: 8px 10px;
  font-size: 10px;
  line-height: 1.5;
  color: var(--text);
  z-index: 50;
  pointer-events: none;
  white-space: normal;
}
.tip:hover .tip-text { display: block; }
```

- [ ] **Step 2: Replace `<!-- Input groups injected here by Task 2 -->` with the full sidebar HTML**

```html
<!-- GENERAL -->
<div class="input-group group-general">
  <div class="group-header">⚙ General</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Initial Investment
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Total cash deployed in each track. Both start with exactly this amount. RE closing costs and negative cash flows are added as extra stock investments to maintain fairness.</span></span>
      </span>
      <span class="input-hint">step $10K</span>
    </div>
    <div class="input-ctrl">
      <button data-key="initialInvestment" data-dir="-1">−</button>
      <span class="input-display" id="disp-initialInvestment">$100,000</span>
      <button data-key="initialInvestment" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Horizon
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Years to hold both investments before selling. Typical: 15–30yr for RE, matching the mortgage term. Shorter horizons favor stocks (RE has high upfront costs).</span></span>
      </span>
      <span class="input-hint">step 5 yrs</span>
    </div>
    <div class="input-ctrl">
      <button data-key="horizon" data-dir="-1">−</button>
      <span class="input-display" id="disp-horizon">30 years</span>
      <button data-key="horizon" data-dir="1">+</button>
    </div>
  </div>
</div>

<!-- S&P 500 -->
<div class="input-group group-stocks">
  <div class="group-header">📈 S&amp;P 500</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Annual Total Return
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Total return including dividends reinvested. 30yr avg (1994–2024): ~10.5%. Bear case: 7%. Bull case: 12–13%. Conservative planning: 8–9%.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="stockReturn" data-dir="-1">−</button>
      <span class="input-display" id="disp-stockReturn">10.5%</span>
      <button data-key="stockReturn" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        LTCG Tax Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Long-term capital gains rate at sale. 0% (income &lt;$47K), 15% ($47K–$518K), 20% (above). Most investors: 15%.</span></span>
      </span>
      <span class="input-hint">step 1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="stockLtcg" data-dir="-1">−</button>
      <span class="input-display" id="disp-stockLtcg">15%</span>
      <button data-key="stockLtcg" data-dir="1">+</button>
    </div>
  </div>
</div>

<!-- RE: PROPERTY -->
<div class="input-group group-re">
  <div class="group-header">🏠 Property</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Purchase Price
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Full property price. Down payment % of this determines the loan. Rent starts at rent-% × this value.</span></span>
      </span>
      <span class="input-hint">step $50K</span>
    </div>
    <div class="input-ctrl">
      <button data-key="purchasePrice" data-dir="-1">−</button>
      <span class="input-display" id="disp-purchasePrice">$500,000</span>
      <button data-key="purchasePrice" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Annual Appreciation
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Case-Shiller national HPI 20yr avg: ~3.8%. Coastal metros (NYC, LA): 5–7%. Midwest: 2–3%. Conservative: 3%. Real appreciation (inflation-adjusted) is ~1.5%.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="appreciation" data-dir="-1">−</button>
      <span class="input-display" id="disp-appreciation">3.8%</span>
      <button data-key="appreciation" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Starting Rent (% of value/mo)
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Monthly rent ÷ property value. "1% rule" (B/C class): 0.67–1%. Luxury/coastal markets: 0.3–0.5%. Example: 0.3% of $500K = $1,500/mo.</span></span>
      </span>
      <span class="input-hint">step 0.05%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="rentPct" data-dir="-1">−</button>
      <span class="input-display" id="disp-rentPct">0.30%</span>
      <button data-key="rentPct" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Annual Rent Growth
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Historical avg rent growth: ~3% (tracks inflation). Hot markets: 5–8%. Conservative: 2%. This compounds — after 30yr at 3%, rent doubles.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="rentGrowth" data-dir="-1">−</button>
      <span class="input-display" id="disp-rentGrowth">3.0%</span>
      <button data-key="rentGrowth" data-dir="1">+</button>
    </div>
  </div>
</div>

<!-- RE: MORTGAGE -->
<div class="input-group group-re">
  <div class="group-header">🏠 Mortgage &amp; Financing</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Down Payment %
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">% of purchase price paid upfront. 20% avoids PMI. Less down = more leverage but higher monthly payment. Model assumes no PMI regardless.</span></span>
      </span>
      <span class="input-hint">step 5%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="downPct" data-dir="-1">−</button>
      <span class="input-display" id="disp-downPct">20%</span>
      <button data-key="downPct" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Mortgage Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Current 30yr fixed: ~7.0–7.5% (2024–2025). Historical avg since 1990: ~6.5%. Pre-2022 era: 3–4%. Investor loans run ~0.5–0.75% above primary-home rates.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="mortgageRate" data-dir="-1">−</button>
      <span class="input-display" id="disp-mortgageRate">7.25%</span>
      <button data-key="mortgageRate" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Refi Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Rate you expect when refinancing mid-term. If rates drop from current 7%+ back toward historical avg (~5%), a 4–5% refi is plausible within 5–15 years.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="refiRate" data-dir="-1">−</button>
      <span class="input-display" id="disp-refiRate">4.5%</span>
      <button data-key="refiRate" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Refi Year
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Year to refinance. Set to 0 or beyond horizon to skip. Refi uses same payoff date (does not restart 30yr clock). Mid-term (yr 7–15) is most common.</span></span>
      </span>
      <span class="input-hint">step 1 yr · 0 = no refi</span>
    </div>
    <div class="input-ctrl">
      <button data-key="refiYear" data-dir="-1">−</button>
      <span class="input-display" id="disp-refiYear">Year 10</span>
      <button data-key="refiYear" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Refi Cost %
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Closing costs as % of remaining loan balance. Typical: 1.5–3%. "No-cost" refis exist but usually roll costs into rate. Set to 0 for no-cost refi.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="refiCost" data-dir="-1">−</button>
      <span class="input-display" id="disp-refiCost">2.0%</span>
      <button data-key="refiCost" data-dir="1">+</button>
    </div>
  </div>
</div>

<!-- RE: EXPENSES -->
<div class="input-group group-re">
  <div class="group-header">🏠 Expenses (% of property value/yr)</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Maintenance
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Routine repairs: paint, plumbing, HVAC filters. Rule of thumb: 1%/yr. Older homes (20+ yrs): 1.5–2%. New construction: 0.5%. Applied to current property value each year.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="maintenance" data-dir="-1">−</button>
      <span class="input-display" id="disp-maintenance">1.0%</span>
      <button data-key="maintenance" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        CapEx
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Capital expenditures: roof, HVAC, water heater, appliances. Budget separately from maintenance. Rule of thumb: 1%/yr. Older properties: 1.5–2%.</span></span>
      </span>
      <span class="input-hint">step 0.1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="capex" data-dir="-1">−</button>
      <span class="input-display" id="disp-capex">1.0%</span>
      <button data-key="capex" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Property Tax
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Annual property tax as % of current assessed value. National avg: 0.8–1.1%. High: NJ 2.5%, TX 2%. Low: CA ~0.75% (Prop 13 limits growth). Varies widely by state.</span></span>
      </span>
      <span class="input-hint">step 0.05%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="propTax" data-dir="-1">−</button>
      <span class="input-display" id="disp-propTax">0.80%</span>
      <button data-key="propTax" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Insurance
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Landlord insurance (not homeowner's). Avg: 0.5–1%/yr of value. Coastal/flood/hurricane zones: 1.5–2.5%. Includes liability coverage.</span></span>
      </span>
      <span class="input-hint">step 0.05%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="insurance" data-dir="-1">−</button>
      <span class="input-display" id="disp-insurance">0.75%</span>
      <button data-key="insurance" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Vacancy Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">% of potential rent lost to vacancies (turnover, gaps). Rule of thumb: 5% ≈ 18 days/yr. Tight markets: 2–3%. Soft markets: 8–12%. Applied to gross rent.</span></span>
      </span>
      <span class="input-hint">step 1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="vacancy" data-dir="-1">−</button>
      <span class="input-display" id="disp-vacancy">5%</span>
      <button data-key="vacancy" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Property Management
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">PM fee as % of collected rent. Standard: 8–12%. Full-service (leasing + maintenance coordination): 10–15%. Self-manage: 0% (but include your time cost).</span></span>
      </span>
      <span class="input-hint">step 1% of rent</span>
    </div>
    <div class="input-ctrl">
      <button data-key="mgmt" data-dir="-1">−</button>
      <span class="input-display" id="disp-mgmt">10%</span>
      <button data-key="mgmt" data-dir="1">+</button>
    </div>
  </div>
</div>

<!-- RE: TAX -->
<div class="input-group group-re">
  <div class="group-header">🏠 Tax</div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Marginal Tax Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Your ordinary income marginal rate — used to value Schedule E deductions. 22% ($89–$190K MFJ), 24% ($190–$364K), 32% ($364–$462K), 35% ($462–$693K), 37% above.</span></span>
      </span>
      <span class="input-hint">step 1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="marginalRate" data-dir="-1">−</button>
      <span class="input-display" id="disp-marginalRate">35%</span>
      <button data-key="marginalRate" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        Depreciation Recapture Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">IRS Section 1250 recapture rate on accumulated depreciation at sale. Fixed at 25% by law. Only changes via new tax legislation. Applied to the lesser of accumulated depreciation or actual gain.</span></span>
      </span>
      <span class="input-hint">step 1% · IRS: 25%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="recaptureRate" data-dir="-1">−</button>
      <span class="input-display" id="disp-recaptureRate">25%</span>
      <button data-key="recaptureRate" data-dir="1">+</button>
    </div>
  </div>

  <div class="input-row">
    <div class="input-label-row">
      <span class="input-label">
        RE LTCG Rate
        <span class="tip"><span class="tip-icon">i</span><span class="tip-text">Long-term capital gains rate on RE appreciation above the depreciation recapture portion. Same brackets as stocks LTCG. Most investors: 15%.</span></span>
      </span>
      <span class="input-hint">step 1%</span>
    </div>
    <div class="input-ctrl">
      <button data-key="reLtcg" data-dir="-1">−</button>
      <span class="input-display" id="disp-reLtcg">15%</span>
      <button data-key="reLtcg" data-dir="1">+</button>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Verify in browser**

Open `index.html`. Expected: fully styled sidebar with all input groups, +/− buttons, ⓘ tooltips that appear on hover. No JS errors in console.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sidebar input groups with tooltips"
```

---

## Task 3: CONFIG, Formatters, and Amortization

**Files:**
- Modify: `index.html` — replace `// JS injected in later tasks` with the first JS block

- [ ] **Step 1: Write console.assert sanity tests (paste into `<script>` block, BEFORE any functions)**

```javascript
// ── Sanity tests (run on load) ─────────────────
function runTests() {
  // Amortization: $400K at 7.25% for 30yr → monthly payment ≈ $2,729–$2,733
  const sched = buildAmortSchedule(400000, 0.0725, 360);
  const pmt = sched[0].interest + sched[0].principal; // approx annual / 12 not exact, so check annual
  console.assert(sched.length === 30, 'Schedule should have 30 annual entries');
  console.assert(sched[29].endBal < 1, 'Balance after 30yr should be ~0');
  console.assert(sched[0].interest > 28000 && sched[0].interest < 30000, 
    'Year1 interest on $400K@7.25% should be ~$28,800');

  // Formatters
  console.assert(formatMoney(1000000) === '$1.00M', 'formatMoney 1M');
  console.assert(formatMoney(842000) === '$842K', 'formatMoney 842K');
  console.assert(formatMoney(-4200) === '-$4K', 'formatMoney negative');
  console.assert(formatPct(0.105) === '10.5%', 'formatPct 10.5');

  console.log('%c✓ All sanity tests passed', 'color: #50c878');
}
```

- [ ] **Step 2: Run `index.html` in browser to see "runTests is not defined" error (tests fail as expected)**

Expected: Console error `runTests is not defined` or `buildAmortSchedule is not defined`.

- [ ] **Step 3: Implement CONFIG, formatters, and buildAmortSchedule**

```javascript
// ── CONFIG: all defaults ────────────────────────
const CONFIG = {
  initialInvestment: 100000,
  horizon: 30,
  // Stocks
  stockReturn: 0.105,
  stockLtcg: 0.15,
  // RE — Property
  purchasePrice: 500000,
  appreciation: 0.038,
  rentPct: 0.003,      // monthly as fraction of purchase price
  rentGrowth: 0.03,
  // RE — Mortgage
  downPct: 0.20,
  mortgageRate: 0.0725,
  refiRate: 0.045,
  refiYear: 10,        // 0 = no refi
  refiCost: 0.02,
  // RE — Expenses (as fraction of property value per year, except mgmt)
  maintenance: 0.01,
  capex: 0.01,
  propTax: 0.008,
  insurance: 0.0075,
  vacancy: 0.05,
  mgmt: 0.10,          // fraction of collected rent
  // RE — Tax
  marginalRate: 0.35,
  recaptureRate: 0.25,
  reLtcg: 0.15,
};

// ── INPUT METADATA ───────────────────────────────
// Defines step, min, max, format for each CONFIG key
const INPUT_META = {
  initialInvestment: { step: 10000, min: 10000, max: 2000000, fmt: 'money' },
  horizon:           { step: 5,     min: 5,     max: 40,      fmt: 'years' },
  stockReturn:       { step: 0.001, min: 0,     max: 0.20,    fmt: 'pct1' },
  stockLtcg:         { step: 0.01,  min: 0,     max: 0.20,    fmt: 'pct0' },
  purchasePrice:     { step: 50000, min: 100000,max: 5000000, fmt: 'money' },
  appreciation:      { step: 0.001, min: 0,     max: 0.15,    fmt: 'pct1' },
  rentPct:           { step: 0.0005,min: 0.001, max: 0.02,    fmt: 'pct2' },
  rentGrowth:        { step: 0.001, min: 0,     max: 0.15,    fmt: 'pct1' },
  downPct:           { step: 0.05,  min: 0.05,  max: 0.50,    fmt: 'pct0' },
  mortgageRate:      { step: 0.001, min: 0.01,  max: 0.15,    fmt: 'pct2' },
  refiRate:          { step: 0.001, min: 0.01,  max: 0.15,    fmt: 'pct2' },
  refiYear:          { step: 1,     min: 0,     max: 40,      fmt: 'refiyear' },
  refiCost:          { step: 0.001, min: 0,     max: 0.10,    fmt: 'pct1' },
  maintenance:       { step: 0.001, min: 0,     max: 0.10,    fmt: 'pct1' },
  capex:             { step: 0.001, min: 0,     max: 0.10,    fmt: 'pct1' },
  propTax:           { step: 0.0005,min: 0,     max: 0.05,    fmt: 'pct2' },
  insurance:         { step: 0.0005,min: 0,     max: 0.05,    fmt: 'pct2' },
  vacancy:           { step: 0.01,  min: 0,     max: 0.50,    fmt: 'pct0' },
  mgmt:              { step: 0.01,  min: 0,     max: 0.30,    fmt: 'pct0' },
  marginalRate:      { step: 0.01,  min: 0,     max: 0.55,    fmt: 'pct0' },
  recaptureRate:     { step: 0.01,  min: 0,     max: 0.30,    fmt: 'pct0' },
  reLtcg:            { step: 0.01,  min: 0,     max: 0.25,    fmt: 'pct0' },
};

// ── FORMATTERS ──────────────────────────────────
function formatMoney(n) {
  const abs = Math.abs(n);
  const sign = n < 0 ? '-' : '';
  if (abs >= 1e6) return sign + '$' + (abs / 1e6).toFixed(2) + 'M';
  if (abs >= 1000) return sign + '$' + Math.round(abs / 1000) + 'K';
  return sign + '$' + Math.round(abs).toLocaleString();
}

function formatPct(n, decimals = 1) {
  return (n * 100).toFixed(decimals) + '%';
}

function formatDisplayValue(key, val) {
  const fmt = INPUT_META[key]?.fmt;
  switch (fmt) {
    case 'money':    return '$' + val.toLocaleString();
    case 'years':    return val + ' years';
    case 'refiyear': return val === 0 ? 'No refi' : 'Year ' + val;
    case 'pct0':     return (val * 100).toFixed(0) + '%';
    case 'pct1':     return (val * 100).toFixed(1) + '%';
    case 'pct2':     return (val * 100).toFixed(2) + '%';
    default:         return String(val);
  }
}

// ── AMORTIZATION ────────────────────────────────
// Returns array of { endBal, interest, principal, payment } indexed [0]=year1 ... [n-1]=yearN
function buildAmortSchedule(principal, annualRate, termMonths) {
  if (principal <= 0 || annualRate <= 0 || termMonths <= 0) return [];
  const r = annualRate / 12;
  const n = termMonths;
  const pmt = principal * r * Math.pow(1 + r, n) / (Math.pow(1 + r, n) - 1);
  const numYears = Math.ceil(n / 12);
  const annual = [];
  let bal = principal;

  for (let y = 0; y < numYears; y++) {
    let annInt = 0, annPri = 0;
    const monthsThisYear = Math.min(12, n - y * 12);
    for (let m = 0; m < monthsThisYear; m++) {
      const int = bal * r;
      const pri = Math.min(pmt - int, bal);
      annInt += int;
      annPri += pri;
      bal = Math.max(bal - pri, 0);
    }
    annual.push({ endBal: bal, interest: annInt, principal: annPri, payment: annInt + annPri });
  }
  return annual;
}
```

- [ ] **Step 4: Add `runTests()` call at bottom of script and open in browser**

```javascript
// Call at the very bottom of <script>
runTests();
```

Expected: Console shows `✓ All sanity tests passed` in green.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add CONFIG, formatters, and amortization schedule"
```

---

## Task 4: Main calculate() Function

**Files:**
- Modify: `index.html` — add `calculate()` function inside `<script>`

- [ ] **Step 1: Add targeted test for calculate() output shape**

Add to `runTests()`:
```javascript
  // calculate() smoke test
  const res = calculate();
  console.assert(res.rows.length === CONFIG.horizon, 'rows length should equal horizon');
  console.assert(res.rows[0].year === 1, 'first row year should be 1');
  console.assert(res.summary.stocksAfterTax > 0, 'stocks after tax should be positive');
  console.assert(res.summary.reAfterTax > 0, 're after tax should be positive');
  console.assert(res.rows[29].stocksTotal > res.rows[0].stocksTotal, 'stocks grows over time');
```

- [ ] **Step 2: Implement calculate()**

```javascript
function calculate() {
  const H = CONFIG.horizon;
  const loanAmount = CONFIG.purchasePrice * (1 - CONFIG.downPct);
  const purchaseClosingCost = CONFIG.purchasePrice * 0.025;

  // Build full 30yr amort (always 30yr mortgage regardless of horizon)
  let amort = buildAmortSchedule(loanAmount, CONFIG.mortgageRate, 360);

  // Apply refi: replace schedule entries from refiYear onward
  let refiCostAmount = 0;
  if (CONFIG.refiYear >= 1 && CONFIG.refiYear <= Math.min(H, 30)) {
    // Balance at START of refi year = endBal after (refiYear-1) full years
    const balBeforeRefi = CONFIG.refiYear === 1
      ? loanAmount
      : amort[CONFIG.refiYear - 2].endBal;
    refiCostAmount = balBeforeRefi * CONFIG.refiCost;
    const remainingYears = 30 - CONFIG.refiYear; // same payoff date
    if (remainingYears > 0 && balBeforeRefi > 0) {
      const refiSched = buildAmortSchedule(balBeforeRefi, CONFIG.refiRate, remainingYears * 12);
      for (let i = 0; i < refiSched.length && (CONFIG.refiYear - 1 + i) < amort.length; i++) {
        amort[CONFIG.refiYear - 1 + i] = refiSched[i];
      }
    }
  }

  // Depreciation
  const buildingValue = CONFIG.purchasePrice * 0.80;
  const annualDepr = buildingValue / 27.5;

  // Stocks track — starts with initial investment
  // Year 0: purchase closing costs treated as negative RE CF → added to stocks track
  let stocksValue = CONFIG.initialInvestment + purchaseClosingCost;
  let stocksCostBasis = CONFIG.initialInvestment + purchaseClosingCost;

  // RE S&P bucket
  let reSpxBucket = 0;
  let reSpxCostBasis = 0;

  let accumDepr = 0;
  const rows = [];

  for (let y = 1; y <= H; y++) {
    // ── Stocks track: grow by return ────────────
    stocksValue *= (1 + CONFIG.stockReturn);

    // ── Property value ───────────────────────────
    const propVal = CONFIG.purchasePrice * Math.pow(1 + CONFIG.appreciation, y);

    // ── Mortgage ────────────────────────────────
    // amort is 0-indexed: amort[y-1] = year y data.
    // If year > 30, mortgage is paid off.
    const amortRow = (y <= 30 && amort[y - 1]) ? amort[y - 1] : { endBal: 0, interest: 0, principal: 0, payment: 0 };
    const mortPayment = amortRow.payment;
    const mortInterest = amortRow.interest;
    const loanBal = amortRow.endBal;

    // ── Refi cost (one-time in refi year) ────────
    const thisRefiCost = (y === CONFIG.refiYear && CONFIG.refiYear >= 1) ? refiCostAmount : 0;

    // ── Rent ─────────────────────────────────────
    const monthlyRent = CONFIG.purchasePrice * CONFIG.rentPct * Math.pow(1 + CONFIG.rentGrowth, y - 1);
    const grossRent = monthlyRent * 12;
    const collectedRent = grossRent * (1 - CONFIG.vacancy);

    // ── Operating expenses ───────────────────────
    const mgmtFee = collectedRent * CONFIG.mgmt;
    const maintenanceCost = propVal * CONFIG.maintenance;
    const capexCost = propVal * CONFIG.capex;
    const propTaxCost = propVal * CONFIG.propTax;
    const insuranceCost = propVal * CONFIG.insurance;

    // ── Operating cash flow (pre-tax) ───────────
    const operatingCF = collectedRent
      - mortPayment
      - mgmtFee
      - maintenanceCost
      - capexCost
      - propTaxCost
      - insuranceCost
      - thisRefiCost;

    // ── Depreciation ─────────────────────────────
    accumDepr += annualDepr;

    // ── Schedule E taxable income ────────────────
    // Deductions: mortgage interest, prop tax, insurance, maintenance, capex, mgmt, depreciation
    const scheduleEIncome = collectedRent
      - mortInterest
      - mgmtFee
      - maintenanceCost
      - capexCost
      - propTaxCost
      - insuranceCost
      - annualDepr;

    // Tax effect: negative scheduleEIncome = paper loss = tax savings at marginal rate
    const taxEffect = -scheduleEIncome * CONFIG.marginalRate;

    // ── Net cash flow ────────────────────────────
    const netCF = operatingCF + taxEffect;

    // ── Route cash flows ─────────────────────────
    if (netCF >= 0) {
      // Positive: invest in RE's S&P bucket
      reSpxBucket = reSpxBucket * (1 + CONFIG.stockReturn) + netCF;
      reSpxCostBasis += netCF;
    } else {
      // Negative: out of pocket; grow RE bucket without addition, add |netCF| to stocks track
      reSpxBucket = reSpxBucket * (1 + CONFIG.stockReturn);
      const outOfPocket = -netCF;
      stocksValue += outOfPocket;
      stocksCostBasis += outOfPocket;
    }

    // ── Equity ───────────────────────────────────
    const equity = propVal - loanBal;
    const reTotal = equity + reSpxBucket;

    rows.push({
      year: y,
      // Stocks
      stocksTotal: stocksValue,
      // RE
      propVal,
      loanBal,
      equity,
      grossRent,
      collectedRent,
      mortPayment,
      mortInterest,
      maintenanceCost,
      capexCost,
      propTaxCost,
      insuranceCost,
      mgmtFee,
      thisRefiCost,
      operatingCF,
      annualDepr,
      scheduleEIncome,
      taxEffect,
      netCF,
      accumDepr,
      reSpxBucket,
      reTotal,
      isRefiYear: y === CONFIG.refiYear && CONFIG.refiYear >= 1,
    });
  }

  // ── SALE CALCULATIONS ────────────────────────────
  const lastRow = rows[H - 1];

  // Stocks after tax
  const stocksGain = lastRow.stocksTotal - stocksCostBasis;
  const stocksTax = Math.max(stocksGain, 0) * CONFIG.stockLtcg;
  const stocksAfterTax = lastRow.stocksTotal - stocksTax;

  // RE property sale
  const grossSalePrice = lastRow.propVal;
  const sellingCosts = grossSalePrice * 0.06;
  const netSaleProceeds = grossSalePrice - sellingCosts;
  const mortPayoff = lastRow.loanBal;
  const equityAfterSale = netSaleProceeds - mortPayoff;

  const adjBasis = CONFIG.purchasePrice - lastRow.accumDepr;
  const totalGain = netSaleProceeds - adjBasis;
  const recapturedAmount = Math.min(lastRow.accumDepr, Math.max(totalGain, 0));
  const deprRecaptureTax = recapturedAmount * CONFIG.recaptureRate;
  const capitalGain = Math.max(totalGain - recapturedAmount, 0);
  const capitalGainsTax = capitalGain * CONFIG.reLtcg;

  // RE S&P bucket after tax
  const reSpxGain = lastRow.reSpxBucket - reSpxCostBasis;
  const reSpxTax = Math.max(reSpxGain, 0) * CONFIG.reLtcg;
  const reSpxAfterTax = lastRow.reSpxBucket - reSpxTax;

  const reAfterTax = equityAfterSale - deprRecaptureTax - capitalGainsTax + reSpxAfterTax;

  return {
    rows,
    summary: {
      stocksAfterTax,
      reAfterTax,
      stocksFinalValue: lastRow.stocksTotal,
      reFinalValue: lastRow.reTotal,
      stocksGain: stocksAfterTax - CONFIG.initialInvestment,
      reGain: reAfterTax - CONFIG.initialInvestment,
      // Sale detail (for modal/debug)
      equityAfterSale,
      deprRecaptureTax,
      capitalGainsTax,
      reSpxAfterTax,
      stocksTax,
    },
  };
}
```

- [ ] **Step 3: Open in browser — verify tests pass**

Console should show `✓ All sanity tests passed`. No errors.

- [ ] **Step 4: Quick manual sanity in console**

Open browser console and run:
```javascript
const r = calculate();
console.log('Stocks after tax:', formatMoney(r.summary.stocksAfterTax));
console.log('RE after tax:', formatMoney(r.summary.reAfterTax));
console.log('Year 1 net CF:', formatMoney(r.rows[0].netCF));
console.log('Year 30 equity:', formatMoney(r.rows[29].equity));
```

Expected (approximate with defaults):
- Stocks after tax: ~$740K–$900K
- RE after tax: depends on refi and cash flow, typically $900K–$1.4M
- Year 1 net CF: should be negative (around -$15K to -$25K with 7.25% mortgage)
- Year 30 equity: property ~$500K × 1.038^30 ≈ $1.5M, minus ~$0 remaining loan

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: implement core calculate() financial model"
```

---

## Task 5: Render Summary Cards

**Files:**
- Modify: `index.html` — add `renderSummary()` inside `<script>`

- [ ] **Step 1: Implement renderSummary()**

```javascript
function renderSummary(summary) {
  const { stocksAfterTax, reAfterTax } = summary;
  const stocksReturn = ((stocksAfterTax - CONFIG.initialInvestment) / CONFIG.initialInvestment * 100).toFixed(0);
  const reReturn = ((reAfterTax - CONFIG.initialInvestment) / CONFIG.initialInvestment * 100).toFixed(0);

  document.getElementById('stocks-value').textContent = formatMoney(stocksAfterTax);
  document.getElementById('stocks-sub').textContent =
    `+${formatMoney(stocksAfterTax - CONFIG.initialInvestment)} gain · ${stocksReturn}% return`;

  document.getElementById('re-value').textContent = formatMoney(reAfterTax);
  document.getElementById('re-sub').textContent =
    `+${formatMoney(reAfterTax - CONFIG.initialInvestment)} gain · ${reReturn}% return`;

  const delta = reAfterTax - stocksAfterTax;
  const winner = delta >= 0 ? 'Real Estate' : 'S&P 500';
  const winnerColor = delta >= 0 ? 'var(--amber)' : 'var(--blue)';
  document.getElementById('winner-value').textContent =
    (delta >= 0 ? 'RE' : 'S&P') + ' +' + formatMoney(Math.abs(delta));
  document.getElementById('winner-value').style.color = winnerColor;
  document.getElementById('winner-sub').textContent = `${winner} outperforms by ${formatMoney(Math.abs(delta))}`;
}
```

- [ ] **Step 2: Wire up initial render at bottom of script**

Replace the existing `runTests()` call with:
```javascript
runTests();
const _initial = calculate();
renderSummary(_initial.summary);
```

- [ ] **Step 3: Open in browser and verify**

Expected: Three summary cards show actual dollar amounts. RE card shows amber values, Stocks card shows blue values, Winner card shows which is ahead.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: render summary cards with after-tax values"
```

---

## Task 6: Render Chart

**Files:**
- Modify: `index.html` — add `renderChart()` and chart instance variable

- [ ] **Step 1: Add chart instance variable above the CONFIG block**

```javascript
let chartInstance = null;
```

- [ ] **Step 2: Implement renderChart()**

```javascript
function renderChart(rows) {
  const labels = rows.map(r => `Yr ${r.year}`);
  const stocksData = rows.map(r => Math.round(r.stocksTotal));
  const reData = rows.map(r => Math.round(r.reTotal));

  if (chartInstance) {
    chartInstance.data.labels = labels;
    chartInstance.data.datasets[0].data = stocksData;
    chartInstance.data.datasets[1].data = reData;
    chartInstance.update();
    return;
  }

  const ctx = document.getElementById('chart').getContext('2d');
  chartInstance = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [
        {
          label: 'S&P 500 Track',
          data: stocksData,
          borderColor: '#4da6ff',
          backgroundColor: 'rgba(77, 166, 255, 0.08)',
          fill: true,
          tension: 0.35,
          pointRadius: 2,
          pointHoverRadius: 5,
          borderWidth: 2,
        },
        {
          label: 'Real Estate Track',
          data: reData,
          borderColor: '#f0a030',
          backgroundColor: 'rgba(240, 160, 48, 0.08)',
          fill: true,
          tension: 0.35,
          pointRadius: 2,
          pointHoverRadius: 5,
          borderWidth: 2,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: { mode: 'index', intersect: false },
      plugins: {
        legend: {
          labels: { color: '#888', font: { size: 11 }, boxWidth: 20 },
        },
        tooltip: {
          backgroundColor: '#1e1e1e',
          borderColor: '#444',
          borderWidth: 1,
          titleColor: '#ccc',
          bodyColor: '#aaa',
          callbacks: {
            label: ctx => ` ${ctx.dataset.label}: ${formatMoney(ctx.raw)}`,
          },
        },
      },
      scales: {
        x: {
          ticks: { color: '#666', font: { size: 10 }, maxTicksLimit: 10 },
          grid: { color: '#1e1e1e' },
        },
        y: {
          ticks: {
            color: '#666',
            font: { size: 10 },
            callback: val => formatMoney(val),
          },
          grid: { color: '#1e1e1e' },
        },
      },
    },
  });
}
```

- [ ] **Step 3: Update initial render call**

```javascript
runTests();
const _initial = calculate();
renderSummary(_initial.summary);
renderChart(_initial.rows);
```

- [ ] **Step 4: Verify in browser**

Expected: Dual-line chart showing both tracks from Year 1 to Year 30. Blue line = stocks, amber line = RE. Hover shows tooltip with values. Chart is 220px tall.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add Chart.js dual-line portfolio chart"
```

---

## Task 7: Render Year-by-Year Table

**Files:**
- Modify: `index.html` — add `renderTable()` inside `<script>`

- [ ] **Step 1: Implement renderTable()**

```javascript
function renderTable(rows, summary) {
  // Header
  const thead = document.getElementById('table-head');
  thead.innerHTML = `<tr>
    <th>Yr</th>
    <th class="th-stocks">Stocks Total</th>
    <th class="th-re">Prop Value</th>
    <th class="th-re">Loan Bal</th>
    <th class="th-re">Equity</th>
    <th class="th-re">Gross Rent</th>
    <th class="th-re">Net Rent</th>
    <th class="th-re">Mort. Payment</th>
    <th class="th-re">Oper. CF</th>
    <th class="th-re">Tax Effect</th>
    <th class="th-re">Net CF</th>
    <th class="th-re">RE S&P Bucket</th>
    <th class="th-re">RE Total</th>
  </tr>`;

  // Body
  const tbody = document.getElementById('table-body');
  tbody.innerHTML = rows.map(r => {
    const cfClass = r.netCF >= 0 ? 'pos' : 'neg';
    const taxClass = r.taxEffect >= 0 ? 'pos' : 'neg'; // positive = savings
    const operClass = r.operatingCF >= 0 ? 'pos' : 'neg';
    const refiMark = r.isRefiYear ? ' title="Refinance year"' : '';
    return `<tr class="${r.isRefiYear ? 'refi-row' : ''}"${refiMark}>
      <td>${r.year}${r.isRefiYear ? ' 🔄' : ''}</td>
      <td class="highlight-stocks">${formatMoney(r.stocksTotal)}</td>
      <td>${formatMoney(r.propVal)}</td>
      <td>${formatMoney(r.loanBal)}</td>
      <td class="highlight-re">${formatMoney(r.equity)}</td>
      <td>${formatMoney(r.grossRent)}</td>
      <td>${formatMoney(r.collectedRent)}</td>
      <td>${formatMoney(r.mortPayment)}</td>
      <td class="${operClass}">${formatMoney(r.operatingCF)}</td>
      <td class="${taxClass}">${formatMoney(r.taxEffect)}</td>
      <td class="${cfClass}">${formatMoney(r.netCF)}</td>
      <td class="highlight-re">${formatMoney(r.reSpxBucket)}</td>
      <td class="highlight-re">${formatMoney(r.reTotal)}</td>
    </tr>`;
  }).join('');

  // Final row: after-tax values
  tbody.innerHTML += `<tr style="border-top: 2px solid #444; font-weight: 600;">
    <td style="color:#ccc">Sale</td>
    <td class="highlight-stocks">${formatMoney(summary.stocksAfterTax)} <span style="font-size:9px;color:var(--muted)">(after tax)</span></td>
    <td colspan="3"></td>
    <td colspan="8" class="highlight-re" style="text-align:right">${formatMoney(summary.reAfterTax)} <span style="font-size:9px;color:var(--muted)">(after tax)</span></td>
  </tr>`;
}
```

- [ ] **Step 2: Update initial render call**

```javascript
runTests();
const _initial = calculate();
renderSummary(_initial.summary);
renderChart(_initial.rows);
renderTable(_initial.rows, _initial.summary);
```

- [ ] **Step 3: Verify in browser**

Expected: Table with 30 data rows + 1 "Sale" row. Year column shows refi year with 🔄 icon. Cash flow columns are red (negative) or green (positive). Refi year row has dashed amber top border.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add year-by-year breakdown table"
```

---

## Task 8: Assumptions Modal

**Files:**
- Modify: `index.html` — add modal open/close JS

- [ ] **Step 1: Add modal wiring to the script**

```javascript
// ── Modal ────────────────────────────────────────
document.getElementById('assumptions-btn').addEventListener('click', () => {
  document.getElementById('modal-overlay').classList.add('open');
});
document.getElementById('modal-close').addEventListener('click', () => {
  document.getElementById('modal-overlay').classList.remove('open');
});
document.getElementById('modal-overlay').addEventListener('click', e => {
  if (e.target === e.currentTarget) {
    document.getElementById('modal-overlay').classList.remove('open');
  }
});
document.addEventListener('keydown', e => {
  if (e.key === 'Escape') document.getElementById('modal-overlay').classList.remove('open');
});
```

- [ ] **Step 2: Verify in browser**

Click "📋 Assumptions" button → modal opens with all assumption text. Click × or click outside or press Escape → closes.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add assumptions modal with open/close"
```

---

## Task 9: Input Wiring — Live Recalculation

**Files:**
- Modify: `index.html` — add `recalculate()`, `updateDisplay()`, and input event listeners

- [ ] **Step 1: Implement recalculate() and updateDisplay()**

```javascript
// ── Recalculate + re-render everything ───────────
function recalculate() {
  const result = calculate();
  renderSummary(result.summary);
  renderChart(result.rows);
  renderTable(result.rows, result.summary);
}

// ── Update display span for a given key ──────────
function updateDisplay(key) {
  const el = document.getElementById('disp-' + key);
  if (el) el.textContent = formatDisplayValue(key, CONFIG[key]);
}
```

- [ ] **Step 2: Add event listeners for all +/− buttons**

```javascript
// ── Input button event listeners ─────────────────
document.querySelectorAll('[data-key][data-dir]').forEach(btn => {
  btn.addEventListener('click', () => {
    const key = btn.dataset.key;
    const dir = parseInt(btn.dataset.dir, 10); // +1 or -1
    const meta = INPUT_META[key];
    if (!meta) return;

    const newVal = CONFIG[key] + dir * meta.step;
    // Clamp to min/max, round to avoid floating point drift
    const decimals = meta.step < 0.01 ? 4 : (meta.step < 1 ? 3 : 0);
    CONFIG[key] = Math.round(
      Math.min(Math.max(newVal, meta.min), meta.max)
      * Math.pow(10, decimals)
    ) / Math.pow(10, decimals);

    updateDisplay(key);
    recalculate();
  });
});
```

- [ ] **Step 3: Update the initial render block at the bottom of script to use recalculate()**

Replace the existing initial render lines:
```javascript
runTests();
recalculate();
```

- [ ] **Step 4: Verify in browser — full interaction test**

Test each of the following and confirm the chart + table + cards update live:
1. Click `+` on Initial Investment (goes from $100K to $110K) — stocks value should increase
2. Click `−` on Mortgage Rate (from 7.25% to 7.15%) — RE cash flow should improve slightly
3. Click `+` on Refi Year past horizon — RE refi year becomes "No refi" equivalent behavior
4. Click `−` on Horizon to 25 — table shortens to 25 rows
5. Click `+` on Annual Appreciation — RE total jumps significantly

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: wire all inputs to live recalculation"
```

---

## Task 10: Polish — Sidebar Scrollbar, Table Overflow, Responsive Touch

**Files:**
- Modify: `index.html` — small CSS additions

- [ ] **Step 1: Add scrollbar styling and table horizontal scroll**

Add inside `<style>`:
```css
/* ── Scrollbar styling ───────────────────────── */
::-webkit-scrollbar { width: 5px; height: 5px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #333; border-radius: 3px; }
::-webkit-scrollbar-thumb:hover { background: #555; }

/* ── Table scroll wrapper ────────────────────── */
#table-container { overflow-x: auto; }

/* ── Sidebar header ──────────────────────────── */
#sidebar-header {
  font-size: 11px;
  color: var(--muted);
  font-weight: 600;
  letter-spacing: 0.5px;
  padding: 4px 2px;
}

/* ── Refi year special display ───────────────── */
.refi-badge {
  display: inline-block;
  background: var(--amber-dim);
  color: var(--amber);
  border: 1px solid var(--amber-border);
  border-radius: 3px;
  font-size: 8px;
  padding: 1px 4px;
  margin-left: 4px;
  vertical-align: middle;
}
```

- [ ] **Step 2: Add sidebar header label above first input group**

Add `<div id="sidebar-header">INPUTS</div>` as the first child of `#sidebar`.

- [ ] **Step 3: Verify in browser**

Scroll the table horizontally — all 13 columns visible. Sidebar shows thin custom scrollbar when scrolling tall content. No layout breaks.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: polish scrollbars, table overflow, sidebar header"
```

---

## Task 11: Final Verification

**Files:**
- Read: `index.html`

- [ ] **Step 1: Open fresh browser tab and run full functional checklist**

```
□ Page loads without console errors
□ Summary cards show 3 non-zero values
□ Chart shows two colored lines with legend
□ Table shows correct number of rows matching Horizon input
□ Refi year row has 🔄 marker and dashed border
□ Tax Effect column: green in early years (depreciation loss), red when RE profitable
□ Net CF column: negative early years, turns positive as rent grows
□ Increasing Annual Appreciation → RE line rises steeply in chart
□ Setting Refi Year to 0 → no refi (no 🔄 row in table)
□ Assumptions modal: opens, closes with ×, Escape, and click-outside
□ Tooltips appear on hover for every ⓘ icon
□ +/− buttons clamp at min/max (e.g. LTCG can't go below 0% or above 20%)
```

- [ ] **Step 2: Console cross-check of sale math**

In browser console:
```javascript
const r = calculate();
const s = r.summary;
// Verify: RE after-tax = equity-after-sale − taxes + S&P bucket after-tax
console.log('RE equity after sale:', formatMoney(s.equityAfterSale));
console.log('Depreciation recapture tax:', formatMoney(s.deprRecaptureTax));
console.log('Capital gains tax:', formatMoney(s.capitalGainsTax));
console.log('RE S&P bucket after-tax:', formatMoney(s.reSpxAfterTax));
const reCalc = s.equityAfterSale - s.deprRecaptureTax - s.capitalGainsTax + s.reSpxAfterTax;
console.assert(Math.abs(reCalc - s.reAfterTax) < 1, 'RE after-tax math checks out');
console.log('%c✓ Sale math verified', 'color: #50c878');
```

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "feat: complete investment calculator — stocks vs real estate"
```

---

## Self-Review Against Spec

| Spec Requirement | Task |
|---|---|
| Single HTML file, no build | Task 1 |
| Chart.js dual-line chart | Task 6 |
| Sidebar with grouped inputs | Task 2 |
| +/− increment buttons with steps | Task 9 |
| Tooltips per input with guidance | Task 2 (tip-text), Task 3 (INPUT_META) |
| Assumptions dialog | Task 1 (HTML), Task 8 (JS) |
| S&P total return (incl. dividends) | Task 4 (CONFIG.stockReturn) |
| LTCG tax only at end of horizon | Task 4 (calculate, sale section) |
| Case-Shiller 3.8% default | Task 3 (CONFIG.appreciation) |
| Amortization with refi at year 10 | Task 3 (buildAmortSchedule), Task 4 (refi logic) |
| Rent 0.3% of value, 3% growth | Task 4 |
| All expense categories | Task 4 |
| Schedule E deductions + tax effect | Task 4 |
| Depreciation (27.5yr straight-line) | Task 4 |
| Positive CF → RE S&P bucket | Task 4 |
| Negative CF → added to stocks track | Task 4 |
| Purchase closing cost 2.5% | Task 4 (year 0) |
| Sale closing cost 6% | Task 4 (sale section) |
| Depreciation recapture (25%, capped at gain) | Task 4 (sale section) |
| LTCG on remaining gain | Task 4 (sale section) |
| RE S&P bucket taxed at LTCG at sale | Task 4 (sale section) |
| Summary cards with after-tax totals | Task 5 |
| Year-by-year table with cash flow columns | Task 7 |
| Color coding: blue=stocks, amber=RE | Task 1 (CSS vars), Task 2 (group classes) |
| Refi year marked in table | Task 7 (refi-row class) |

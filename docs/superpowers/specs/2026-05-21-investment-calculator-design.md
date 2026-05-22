# Investment Calculator: Stocks vs. Real Estate — Design Spec

**Date:** 2026-05-21  
**Status:** Approved

---

## 1. Overview

A single self-contained HTML file (no build step, no server) that lets a user compare two parallel $100K investments over a configurable horizon:

- **Track A — S&P 500:** Lump-sum invested, dividends reinvested, sold at end, taxed as LTCG.
- **Track B — Real Estate:** 20% down on a rental property, 30yr mortgage, rented out, sold at end with full tax accounting.

The calculator recalculates live on every input change and shows:
- Summary cards (after-tax final values + winner delta)
- A dual-line chart (portfolio value over time)
- A year-by-year breakdown table (with cash flow columns)
- An "Assumptions" dialog for hardcoded model parameters

---

## 2. Architecture

**Single HTML file** with inline CSS and inline JavaScript. No npm, no framework, no build step.

**External CDN dependency (only one):**
- [Chart.js 4.x](https://cdn.jsdelivr.net/npm/chart.js) — for the dual-line portfolio chart. Loaded from CDN via `<script>` tag.

**Structure of the file:**
```
index.html
├── <head>  — inline CSS, Chart.js CDN link
├── <body>
│   ├── #sidebar        — grouped input panels
│   ├── #main
│   │   ├── #summary    — 3 summary cards
│   │   ├── #chart      — Chart.js canvas
│   │   └── #table      — year-by-year table
│   └── #assumptions-modal  — hidden dialog, toggled by button
└── <script>  — all financial logic + UI wiring
```

---

## 3. Visual Design

### Color palette

| Role | Color | Usage |
|---|---|---|
| Background | `#0d0d0d` | Page background |
| Surface | `#111` / `#1a1a1a` | Cards, panels |
| Border | `#2a2a2a` | Dividers |
| **S&P 500 (blue)** | `#4da6ff` | Stock inputs, chart line, table headers, summary card accent |
| **Real Estate (amber)** | `#f0a030` | RE inputs, chart line, table headers, summary card accent |
| Neutral | `#ccc` / `#888` | General text, labels |
| Negative cash flow | `#e05050` | Table cash flow cells when negative |
| Positive cash flow | `#50c878` | Table cash flow cells when positive |

### Layout

```
┌────────────────────────────────────────────────────────────────┐
│  [?] Assumptions                              Investment Calculator │
├──────────────┬─────────────────────────────────────────────────┤
│              │  [Stocks $842K]  [RE $1.12M]  [Winner: RE +$278K] │
│   SIDEBAR    ├─────────────────────────────────────────────────┤
│   (240px)    │              Chart.js dual-line chart             │
│              │                (30 years, both tracks)            │
│   Inputs     ├─────────────────────────────────────────────────┤
│   grouped    │              Year-by-Year Table                   │
│   by section │              (scrollable, all 30 rows)            │
│              │                                                   │
└──────────────┴─────────────────────────────────────────────────┘
```

### Sidebar input groups

Each group is a card with a colored header. Inputs have `−` / `+` buttons flanking a read-only display field.

| Group | Color | Inputs |
|---|---|---|
| **General** | Neutral | Initial Investment, Horizon (years) |
| **S&P 500** | Blue | Annual Total Return (incl. dividends reinvested), LTCG Tax Rate |
| **RE — Property** | Amber | Purchase Price, Annual Appreciation, Starting Rent (% of value), Annual Rent Growth |
| **RE — Mortgage & Financing** | Amber | Down Payment %, Mortgage Rate, Refi Rate, Refi Year, Refi Cost % |
| **RE — Expenses** | Amber | Maintenance %, CapEx %, Property Tax %, Insurance %, Vacancy %, Management % |
| **RE — Tax** | Amber | Marginal Tax Rate, Depreciation Recapture Rate, LTCG Rate (RE) |

### Input increment steps

| Input | Step |
|---|---|
| Initial Investment | $10,000 |
| Purchase Price | $50,000 |
| Horizon | 5 years |
| All rates / percentages | 0.1% |
| Refi Year | 1 year |

### Tooltips

Every input label has a `ⓘ` icon. On hover, a tooltip appears with:
- The default/recommended value
- Its source (e.g., "30yr S&P avg", "Case-Shiller national 20yr avg")
- The valid range

Implemented as CSS `::after` pseudo-element on a `.tooltip` wrapper — no JS required.

### Assumptions dialog

A fixed `?` button in the top-left corner opens a modal overlay listing all hardcoded model parameters:

- Building value = 80% of purchase price (IRS land/improvement split for depreciation)
- Depreciation schedule: straight-line over 27.5 years (residential rental, IRS)
- Dividends are reinvested annually and included in the total return rate (not separately taxed during holding period — simplified)
- Mortgage amortization: standard fixed-rate formula
- Closing costs at purchase: 2.5% of purchase price (additional cash out, treated as negative cash flow Year 0)
- Closing costs at sale: 6% of sale price (deducted from proceeds)
- Refi cost: 2% of remaining balance at refi year (rolled into negative cash flow that year)
- Tax deductions: Schedule E (interest, taxes, insurance, maintenance, CapEx, management, depreciation)
- Passive activity loss: applied fully in year incurred (no PAL limitation — simplification)
- Positive cash flow → invested in RE's S&P bucket at the same annual return rate
- Negative cash flow → out-of-pocket; same amount added to stocks track as additional investment
- Property tax deductibility: Schedule A (applies to marginal rate)
- Depreciation recapture at sale: IRS Section 1250, 25% rate on accumulated depreciation
- Capital gain at sale: (net proceeds − adjusted basis − mortgage payoff), LTCG rate on remainder after recapture

---

## 4. Financial Model

### 4.1 Stocks Track

**Each year:**
```
portfolio_value[y] = portfolio_value[y-1] × (1 + total_return_rate)
```
(Total return = price appreciation + dividends reinvested. Dividends not separately taxed during hold.)

**At sale (Year N):**
```
gross_gain = final_value - cost_basis
cost_basis = initial_investment + sum(additional_investments_from_negative_RE_cashflow)
tax_owed = gross_gain × ltcg_rate
after_tax = final_value - tax_owed
```

**Stocks track capital (each year):**
```
stocks_capital[y] = portfolio_value[y]  (pre-tax; shown as-is, tax applied only at end)
```

---

### 4.2 Real Estate Track

#### Mortgage amortization

Standard fixed-rate monthly payment:
```
P = loan_amount
r = monthly_rate = annual_rate / 12
n = total_months = 30 × 12

monthly_payment = P × r × (1+r)^n / ((1+r)^n − 1)
```

At refi year: remaining balance is paid off, refi cost (2% of balance) is added to that year's cash outflow. New loan = remaining balance. New payment recalculated for remaining term.

#### Property value

```
property_value[y] = purchase_price × (1 + appreciation_rate)^y
```

#### Rent

```
monthly_rent[y] = (purchase_price × rent_pct) × (1 + rent_growth)^(y-1)
annual_gross_rent[y] = monthly_rent[y] × 12
annual_collected_rent[y] = annual_gross_rent[y] × (1 − vacancy_rate)
```

#### Annual operating expenses

```
management_fee[y]  = annual_collected_rent[y] × mgmt_pct
maintenance[y]     = property_value[y] × maintenance_pct
capex[y]           = property_value[y] × capex_pct
property_tax[y]    = property_value[y] × prop_tax_pct
insurance[y]       = property_value[y] × insurance_pct
mortgage_payment[y] = monthly_payment × 12  (full P+I)
mortgage_interest[y] = from amortization schedule
mortgage_principal[y] = mortgage_payment[y] - mortgage_interest[y]
```

#### Operating cash flow (pre-tax)

```
operating_cf[y] = annual_collected_rent[y]
                - mortgage_payment[y]
                - management_fee[y]
                - maintenance[y]
                - capex[y]
                - property_tax[y]
                - insurance[y]
```

#### Depreciation

```
building_value = purchase_price × 0.80
annual_depreciation = building_value / 27.5
accumulated_depreciation[y] = annual_depreciation × y
```

#### Schedule E taxable income

```
schedule_e_income[y] = annual_collected_rent[y]
                     - mortgage_interest[y]
                     - management_fee[y]
                     - maintenance[y]
                     - capex[y]
                     - property_tax[y]
                     - insurance[y]
                     - annual_depreciation

tax_effect[y] = -schedule_e_income[y] × marginal_rate
  (negative schedule_e_income = loss = tax savings = positive tax_effect)
  (positive schedule_e_income = income = tax owed = negative tax_effect)
```

#### Net cash flow

```
net_cf[y] = operating_cf[y] + tax_effect[y]
```

- If `net_cf[y] > 0`: add to **RE S&P bucket** (invested at stock return rate)
- If `net_cf[y] < 0`: out of pocket → add `|net_cf[y]|` to **stocks track** as additional investment

#### Equity

```
remaining_balance[y] = from amortization schedule
equity[y] = property_value[y] - remaining_balance[y]
```

#### RE total capital (each year, pre-sale)

```
re_spx_bucket[y] = re_spx_bucket[y-1] × (1 + stock_return) + max(net_cf[y], 0)
re_spx_bucket_cost_basis[y] = re_spx_bucket_cost_basis[y-1] + max(net_cf[y], 0)
re_total_capital[y] = equity[y] + re_spx_bucket[y]
```

#### Sale (Year N)

```
gross_sale_price = property_value[N]
selling_costs = gross_sale_price × 0.06
net_sale_proceeds = gross_sale_price - selling_costs
mortgage_payoff = remaining_balance[N]
equity_after_sale = net_sale_proceeds - mortgage_payoff

adjusted_basis = purchase_price - accumulated_depreciation[N]
total_gain = net_sale_proceeds - adjusted_basis
recaptured_amount = min(accumulated_depreciation[N], max(total_gain, 0))  // capped at actual gain
depreciation_recapture_tax = recaptured_amount × recapture_rate  (25%)
capital_gain = max(total_gain - recaptured_amount, 0)
capital_gains_tax = capital_gain × ltcg_rate_re

re_spx_bucket_gain = re_spx_bucket[N] - re_spx_bucket_cost_basis
re_spx_tax = re_spx_bucket_gain × ltcg_rate_re
re_spx_after_tax = re_spx_bucket[N] - re_spx_tax

after_tax_re = equity_after_sale - depreciation_recapture_tax - capital_gains_tax + re_spx_after_tax
```

---

## 5. Year-by-Year Table Columns

| Column | Notes |
|---|---|
| Year | 1–30 |
| Stocks Value | Pre-tax portfolio value |
| RE Property Value | Appreciated value |
| RE Equity | Property value − loan balance |
| Gross Rent | Before vacancy |
| Net Rent Collected | After vacancy |
| Operating Cash Flow | Before tax effect |
| Tax Effect | Savings (green) or owed (red) |
| Net Cash Flow | After tax; red if negative |
| RE S&P Bucket | Accumulated reinvested surplus |
| RE Total Capital | Equity + S&P bucket |
| Stocks Track Total | After additional investments from negative RE years |

---

## 6. Default Values

| Input | Default | Source |
|---|---|---|
| Initial Investment | $100,000 | User-specified |
| Horizon | 30 years | User-specified |
| S&P Annual Total Return | 10.5% | 30yr S&P 500 total return avg incl. dividends reinvested (1994–2024) |
| LTCG Tax Rate (stocks) | 15% | Standard bracket for most investors |
| Purchase Price | $500,000 | User-specified |
| Annual Appreciation | 3.8% | Case-Shiller national HPI 20yr avg |
| Starting Rent | 0.3% of value/mo | Rule of thumb |
| Annual Rent Growth | 3.0% | Historical avg |
| Down Payment | 20% | User-specified |
| Mortgage Rate | 7.25% | Current 30yr fixed (2024–2025) |
| Refi Rate | 4.5% | Assumed mid-term opportunity |
| Refi Year | 10 | Mid-term assumption |
| Refi Cost | 2% of balance | Typical closing cost |
| Maintenance | 1.0% | Rule of thumb |
| CapEx | 1.0% | Rule of thumb |
| Property Tax | 0.80% | National avg |
| Insurance | 0.75% | National avg |
| Vacancy | 5% | Rule of thumb |
| Management | 10% of rent | Standard PM fee |
| Marginal Tax Rate | 35% | User-specified |
| Depreciation Recapture | 25% | IRS Section 1250 |
| LTCG Rate (RE) | 15% | Standard bracket |

---

## 7. Scope Exclusions

- No inflation adjustment (nominal dollars throughout)
- No state income tax
- No AMT consideration
- No 1031 exchange
- No PAL (Passive Activity Loss) carryforward — losses applied in full in year incurred
- No HOA fees (not in original spec)
- No mortgage points / origination fees beyond the refi cost
- No opportunity cost on the down payment closing costs delta (the 2.5% purchase closing cost is treated as Year 0 negative cash flow)

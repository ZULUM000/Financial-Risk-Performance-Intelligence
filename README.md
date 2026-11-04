# Financial Risk & Performance Intelligence

An end-to-end, **SQL-first** analytics project that turns raw financial datasets into a Power BI dashboard covering credit risk, fraud detection, portfolio performance and cash flow.

**Stack:** PostgreSQL (pgAdmin, psql) · SQL (CTEs, window functions, views, percentiles) · Power BI (DAX)
**Status:** Modules 1–2 complete · Module 3 in progress · Module 4 and the Executive Summary page planned

<!-- INSERT HERE (optional): a hero image of your best dashboard page, e.g.
![Dashboard preview](images/dashboard-preview.png) -->

---

## Why this project

Financial teams need to answer four questions: who is likely to default, which transactions look fraudulent, how is the portfolio performing, and where is cash going. This project answers each with transparent, rule-based SQL (no black-box model) and presents the results in one five-page dashboard.

## Architecture

```
Raw CSV  ->  raw table  ->  clean view  ->  flags / metrics table  ->  Power BI page
```

Each module follows the same pipeline: load unchanged data into a raw table, clean and validate it in a view, build rule-based flags or metrics in a table, then connect Power BI and refresh to see changes live.

## Dashboard pages

| # | Page | Status |
|---|------|--------|
| 1 | Executive Summary | Planned |
| 2 | Credit Risk | Built |
| 3 | Fraud Detection | Built and polished |
| 4 | Portfolio Performance | In progress (SQL layer under way) |
| 5 | Cash Flow | Not started |

---

## Module 1: Credit Risk (complete)

**Dataset:** UCI "Default of Credit Card Clients" (30,000 credit card clients, Taiwan)
**Script:** `sql/01_credit_risk.sql`
**Objects:** `credit_clients_raw` (table) -> `credit_clients_clean` (view) -> `credit_risk_flags` (table)

**Data quality:** no nulls in key fields and no duplicate client IDs. Education and marriage contain undocumented codes (education 0, 5, 6; marriage 0), which were bucketed into "Other/Unknown" rather than deleted.

**Baseline default rate:** 22.12%

**Findings**

| Question | Result |
|----------|--------|
| Default rate by credit-limit quartile | 31.92% (lowest limits) falling to 14.27% (highest limits) |
| Late-payment months vs default (strongest finding) | 11.71% with 0 late months, rising to 70.32% with 6 late months |
| Credit utilization, defaulters vs non-defaulters (median) | 46.27% vs 24.17%, nearly double |
| Default rate by education | High School 25.16%, University 23.73%, Graduate School 19.23% |
| Default rate by sex and marriage | Male 24.17% vs Female 20.78%; Married 23.47% vs Single 20.93% (smaller gaps) |
| Rule-based risk tiers | Low 11.44%, Medium 25.48%, High 54.60% default rate |

Tier sizes: Medium 17,234 · Low 10,949 · High 1,817. The rule-based tiers separate risk cleanly.

<!-- INSERT HERE: screenshot of the Credit Risk page of your Power BI report.
Save it as images/credit-risk-page.png, then keep the line below. -->
<img width="821" height="412" alt="image" src="https://github.com/user-attachments/assets/bb207536-4f48-40a3-8c8b-94c7f93caf2f" />

---

## Module 2: Fraud Detection (complete)

**Dataset:** PaySim synthetic mobile-money transactions, 6,362,620 rows
**Script:** `sql/02_fraud_detection.sql`
**Objects:** `transactions_raw` (table) -> `transactions_clean` (view) -> `fraud_flags` (table)

### Method

- **Clean view:** keeps rows with `amount > 0` and adds balance-reconciliation differences. For `CASH_IN` the sender balance formula runs in the opposite direction, so a `CASE` handles it.
- **Rule 1, zero-balance destination:** a TRANSFER or CASH_OUT whose destination balance is 0 before and after (money that appears to vanish, consistent with a mule account).
- **Rule 2, unusually large amount:** amount above the 99th percentile for that transaction type (`PERCENTILE_CONT`, cast to `NUMERIC` for rounding).
- **`rule_flag`:** Rule 1 OR Rule 2.
- **Validation:** the rule flag is compared with the ground-truth `isFraud` label using a confusion matrix, giving precision and recall.

### Data quality notes

- No nulls in key fields.
- 16 rows have `amount = 0`. **All 16 are fraud** (verified: 16 excluded rows, 16 fraud). They are excluded from the clean view, so the analysis covers **6,362,604 valid transactions with 8,197 fraud cases**. Including them, the raw dataset has 8,213 fraud cases (0.1291%).
- Fraud occurs **only** in TRANSFER and CASH_OUT.
- Merchant (M-prefixed) destination balances are never tracked (always 0/0).
- Many balances do not reconcile. This is a simulator limitation, reported as a note and not "corrected".
- Fraud amounts cap at exactly 10,000,000 (a simulator artifact).
- The simulator's own `isFlaggedFraud` never fires on legitimate transactions, but it catches very little fraud.
- The first import was silently truncated to 1,048,575 rows by Excel's row limit. The data was reloaded from the original CSV with `psql \copy` (the pgAdmin Query Tool does not support `\copy`).

### Dashboard results

| Metric | Value |
|--------|-------|
| Fraud rate | 0.13% |
| Rule precision | 8.83% |
| Rule recall | 71.68% |
| Total transaction value | 1.14T |
| Repeat destination rate | 0.53% |
| Fraud count by type | CASH_OUT 4,100 · TRANSFER 4,097 |
| Fraud amount by type | TRANSFER 6.1bn · CASH_OUT 6.0bn |
| Average amount | Fraud 1.47M vs legitimate 0.18M (about 8 times larger) |

**Reading the precision/recall trade-off:** the rules catch about 72% of fraud (recall) but only about 9% of flagged transactions are truly fraudulent (precision). Because Rule 2 flags the top 1% of every type, it produces many false positives. That is acceptable for a first-pass screening layer where analysts review alerts, but it would need tuning before automated blocking.

**Other observations**

- Fraud is spread across the simulation, with daily counts mostly between about 220 and 320 (simulation day = 24 steps).
- Fraudulent transactions are on average roughly 8 times larger than legitimate ones.
- No fraud was recorded in PAYMENT, DEBIT or CASH_IN.

<!-- INSERT HERE: screenshot of the Fraud Detection page of your Power BI report.
Save it as images/fraud-detection-page.png, then keep the line below. -->
<img width="814" height="413" alt="image" src="https://github.com/user-attachments/assets/e02b9a3d-3b55-4181-9edc-f0a117836749" />


---

## Module 3: Portfolio Performance (in progress)

**Dataset:** Kaggle "S&P 500 stock data" (daily prices, February 2013 to February 2018), plus the public S&P 500 constituents list for sectors
**Script:** `sql/03_portfolio_performance.sql` (in progress)
**Objects so far:** `stock_prices_raw` (table), `sectors_raw` (table), `daily_returns` (view), `return_exclusions` (table), `daily_returns_clean` (view), `sample_portfolio` (table)

### What is done

- **Load:** 619,040 daily price rows, 505 symbols, 2013-02-08 to 2018-02-07 (1,259 trading days). The CSV used month/day/year dates, so the load needed `SET datestyle = 'MDY'`.
- **Sectors:** 503 symbols across 11 sectors. Only **353 of the 505** price symbols match a sector, because the constituents file is today's index and the rest are companies delisted, acquired or renamed since (for example FB to META). Sector analysis therefore uses those 353, which introduces some survivorship bias, noted as a limitation. The full-market benchmark uses all stocks with complete history.
- **Data quality:** no duplicates, no null or non-positive closing prices. 11 rows are missing open/high/low, 1 row has high below low, and 4 have zero volume; none affects close-to-close returns. Six stocks have a few missing days inside their history and many stocks joined or left the index mid-period.
- **Daily returns:** `(close - previous close) / previous close` per stock, using `LAG` over each symbol (618,535 returns).
- **Corporate-action artifacts:** the prices are not adjusted for every spin-off or split. Reviewing all moves beyond 25% either way found 19 returns that were artifacts (spin-offs, a special dividend, an unadjusted 2-for-1 split and one-day bad prices). They are listed with reasons in `return_exclusions` and removed in `daily_returns_clean` (618,516 returns). Large genuine moves, such as earnings shocks, were kept.
- **Sample portfolio:** 12 stocks chosen by a fixed rule, not by hindsight: the most liquid stock in each sector (highest average daily dollar volume) with the full 1,259-day history, plus a second Information Technology stock. The stocks are GOOGL, AMZN, PG, XOM, BAC, GILD, GE, AAPL, MSFT, FCX, SPG and DUK.

### Still to do

- Portfolio and benchmark daily returns, with cumulative return (equal-weighted, rebalanced daily)
- Return and volatility by sector and by stock
- Largest single-day drop and the sectors behind it
- Sector vs volatility comparison
- Power BI page

---

## Modules 4 (planned)

- **Module 4: Cash Flow / Expense.** Inflows, outflows and expense categories.

It will follow the same pipeline and feed the Executive Summary.

## Repository structure

```
README.md
sql/
  01_credit_risk.sql
  02_fraud_detection.sql
  03_portfolio_performance.sql   (in progress)
  04_cash_flow.sql               (planned)
images/
  credit-risk-page.png
  fraud-detection-page.png
powerbi/
  Financial_Risk_Dashboard.pbix  (optional)
```

## How to reproduce

1. Create a PostgreSQL database `financial_risk_performance`.
2. Download the datasets: UCI Default of Credit Card Clients; PaySim (Kaggle); S&P 500 stock data (Kaggle, `all_stocks_5yr.csv`); S&P 500 constituents (`datasets/s-and-p-500-companies` on GitHub).
3. Run each module script in order. Load large CSVs with `psql \copy`, not the pgAdmin import wizard, and verify the row counts (30,000; 6,362,620; 619,040).
4. Open the Power BI file, point the PostgreSQL connection at your server and refresh.

## Design decisions

- **SQL-first, no ML layer:** rules are transparent and explainable, which suits risk and compliance work.
- **Document, don't silently fix:** data-quality quirks (undocumented codes, non-reconciling balances, capped amounts, unadjusted spin-offs) are recorded as notes or in exclusion tables with reasons.
- **Validate every rule:** flags are tested against ground truth with precision and recall.
- **Avoid hindsight bias:** the sample portfolio is chosen by a fixed liquidity rule, not by past performance.

## Skills demonstrated

SQL (CTEs, window functions, views, percentiles, conditional logic) · data cleaning and validation · rule-based risk and fraud flagging · precision/recall evaluation · financial returns and cumulative return calculation · Power BI (DAX measures, interactive dashboards, KPI cards) · data storytelling

## Data sources

- UCI Machine Learning Repository: Default of Credit Card Clients
- PaySim: synthetic financial transactions dataset (Kaggle)
- S&P 500 stock data, 2013–2018 (Kaggle, `camnugent/sandp500`)
- S&P 500 constituents list (`datasets/s-and-p-500-companies`, GitHub)

# Financial Risk & Performance Intelligence

An end-to-end, **SQL-first** analytics project that turns raw financial datasets into a Power BI dashboard covering credit risk, fraud detection, portfolio performance and cash flow.

**Stack:** PostgreSQL (pgAdmin, psql) · SQL (CTEs, window functions, views, percentiles) · Power BI (DAX)
**Status:** Complete. Four SQL modules feeding a five-page Power BI dashboard.

<!-- SCREENSHOT 1 of 5 (hero image): export the finished Executive Summary page and save it as `images/executive-summary-page.png`. Keep the line below. Delete this note once the image is in place. -->

![Executive Summary dashboard page](images/executive-summary-page.png)

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
| 1 | Executive Summary | Headline KPIs, one finding per module, portfolio vs benchmark chart |
| 2 | Credit Risk | Default rate by limit, late payments, utilisation and risk tier |
| 3 | Fraud Detection | Fraud rate, rule precision and recall, fraud by type |
| 4 | Portfolio Performance | Portfolio vs benchmark, volatility, sector returns |
| 5 | Cash Flow | Income, spending, savings rate, categories, payment mode |

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

<!-- SCREENSHOT 2 of 5: the Credit Risk page of your Power BI report. Save as `images/credit-risk-page.png`. Keep the line below. -->
![Credit Risk dashboard page](images/credit-risk-page.png)

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

<!-- SCREENSHOT 3 of 5: the Fraud Detection page. Save as `images/fraud-detection-page.png`. Keep the line below. -->
![Fraud Detection dashboard page](images/fraud-detection-page.png)

---

## Module 3: Portfolio Performance (complete)

**Dataset:** Kaggle "S&P 500 stock data" (daily prices, February 2013 to February 2018), plus the public S&P 500 constituents list for sectors
**Script:** `sql/03_portfolio_performance.sql`
**Objects:** `stock_prices_raw`, `sectors_raw` (tables) -> `daily_returns` (view) -> `return_exclusions` (table) -> `daily_returns_clean` (view) -> `sample_portfolio` (table) -> `portfolio_daily`, `benchmark_daily`, `portfolio_vs_benchmark`, `sample_stock_stats`, `sector_stats` (views)

### Method and data quality

- **Load:** 619,040 daily price rows, 505 symbols, 2013-02-08 to 2018-02-07 (1,259 trading days). The CSV used month/day/year dates, so the load needed `SET datestyle = 'MDY'`.
- **Sectors:** 503 symbols across 11 sectors. Only **353 of the 505** price symbols match a sector, because the constituents file is today's index and the rest were delisted, acquired or renamed since (for example FB to META). Sector results therefore carry some survivorship bias.
- **Data quality:** no duplicates, no null or non-positive closing prices. 11 rows are missing open/high/low, 1 row has high below low and 4 have zero volume; none affects close-to-close returns. Six stocks have a few missing days inside their history.
- **Daily returns:** `(close - previous close) / previous close` using `LAG` over each symbol (618,535 returns).
- **Corporate-action artifacts:** the prices are not adjusted for every spin-off or split. Reviewing all moves beyond 25% either way found 19 returns that were artifacts (spin-offs, a special dividend, an unadjusted 2-for-1 split and one-day bad prices). They are listed with reasons in `return_exclusions` and removed in `daily_returns_clean` (618,516 returns). Genuine large moves, such as earnings shocks, were kept.
- **Sample portfolio:** 12 stocks chosen by a fixed rule, not hindsight: the most liquid stock in each sector (highest average daily dollar volume) with the full history, plus a second Information Technology stock. GOOGL, AMZN, PG, XOM, BAC, GILD, GE, AAPL, MSFT, FCX, SPG and DUK.
- **Cumulative return:** PostgreSQL has no running product, so it is computed as `EXP(SUM(LN(1 + r))) OVER (ORDER BY date) - 1`.

### Findings

| Question | Result |
|----------|--------|
| Sample portfolio vs market (cumulative, equal-weighted, rebalanced daily) | Portfolio +75.18% vs benchmark +92.80% over the period |
| Most and least volatile stocks in the portfolio | FCX 54.2% a year, PG 14.2% a year |
| Best and worst total return in the portfolio | AMZN +441%, FCX -50% |
| Best and worst sector (average annualised return) | Information Technology +20.8% a year, Energy -0.5% a year |
| Sector vs volatility | Higher in Energy (30.3%), Materials (27.4%) and Information Technology (26.4%); lowest in Utilities (18.4%) and Consumer Staples (19.3%) |
| Largest single-day portfolio drop | 24 Aug 2015: portfolio -4.67% vs benchmark -4.15% |
| What drove it | A market-wide fall: every sector dropped 3-5.5%, led by Energy (-5.5%), Real Estate (-4.9%) and Materials (-4.8%); Information Technology and Consumer Staples fell least |

**Reading the portfolio vs benchmark result:** six of the 12 stocks more than doubled, yet the daily-rebalanced portfolio trailed the market sample. The simple average of the 12 stocks' total returns is about +96% (buy and hold). The lower +75% reflects daily rebalancing back to equal weights and a concentrated, volatile 12-stock sample, with FCX, GE and XOM as the main drags. The benchmark uses the same method, so the comparison is consistent, but it shows how much the method matters.

**Limitations:** survivorship bias in the sector list; Communication Services has only 8 stocks; the portfolio is 12 large, liquid stocks, so conclusions are descriptive and not a prediction.

<!-- SCREENSHOT 4 of 5: the Portfolio Performance page. Save as `images/portfolio-performance-page.png`. Keep the line below. -->
![Portfolio Performance dashboard page](images/portfolio-performance-page.png)

---

## Module 4: Cash Flow / Expense (complete)

**Dataset:** Kaggle "Daily Household Transactions" (`prasad22/daily-transactions-dataset`), 2,461 personal transactions, January 2015 to September 2018. **All amounts are in Indian rupees (₹).**
**Script:** `sql/04_cash_flow.sql`
**Objects:** `household_transactions_raw` (table, all TEXT) -> `household_transactions_clean` (view) -> `monthly_cash_flow`, `category_monthly`, `category_summary`, `payment_mode_spend` (views)

### Method and data quality

- **Load:** all columns loaded as text with `psql \copy`, then typed in the clean view. Dates are `DD/MM/YYYY`, some with a time part, so they are parsed with `to_date(split_part(txn_date, ' ', 1), 'DD/MM/YYYY')`. Range 2015-01-01 to 2018-09-20; September 2018 is partial and is excluded from monthly trends with an `is_full_month` flag.
- **No nulls or zero amounts.** 9 exact duplicate rows were kept (identical purchases on the same day are plausible).
- **Income, expense or transfer:** the source labels money moving between the person's own accounts as expenses or income. The clean view reclassifies these as `Transfer` so they do not distort spending: expense rows in Money transfer, Investment, Public Provident Fund and Recurring Deposit (₹990,386.90), income rows labelled Maturity amount and Petty cash (₹395,962.00), and every transfer-out row. **Assumption:** "Money transfer" (₹606,528.90) is treated as moving money between own accounts, not spending.
- **After cleaning:** Expense 2,000 rows / ₹967,003.63; Income 112 / ₹2,646,435.35; Transfer In 13 / ₹395,962.00; Transfer Out 336 / ₹2,761,167.80. The views reconcile exactly (₹967,003.63 over 45 months and 23 categories).
- **Uneven logging:** only 12–33 rows a month were logged in 2015 to mid-2016, against 80–130 a month in 2017–18. The apparent jump in spending from 2016 to 2017 is mostly better logging, not a real change, so trend and seasonality statements use 2017 onwards.

### Findings

| Question | Result |
|----------|--------|
| Total income, spending and net cash flow | Income ₹2,646,435 · spend ₹967,004 · net ₹1,679,432 |
| Savings rate | 63.5% of income (spend is only 36.5% of income) |
| Typical monthly spend, 2017 onwards | About ₹34,000 to ₹37,000 a month, steady |
| Biggest categories | Transportation 17.48%, Household 16.72%, Subscription 11.85%, Food 9.97% |
| Concentration | Top 4 categories = 56% of spend; top 5 = 65% |
| Most frequent category | Food (907 transactions) |
| Payment mode | Bank account 63.0% of spend (801 txns), Card 20.3% (163), Cash 16.7% (1,036 txns, the most frequent) |
| Seasonality (2017-01 to 2018-08, indicative) | Highest in January (₹64,939), August (₹56,350) and December (₹46,805); lowest in April (₹13,834) and March (₹19,949) |
| Savings scenario: lift net cash flow by 10% (₹167,943) | Cutting Transportation alone would need a 99.3% cut; cutting the top 4 categories needs 31%. Transportation is the first place to look |

**Limitations:** the data is one household's self-logged records, so it shows habits, not a market-wide pattern. Seasonality rests on only one or two observations per month. Early-period spending is understated because logging was sparse.

<!-- SCREENSHOT 5 of 5: the Cash Flow page, the finished version. Save as `images/cash-flow-page.png`. Keep the line below. -->

![Cash Flow dashboard page](images/cash-flow-page.png)

---

## Executive Summary page

The first page of the dashboard pulls the four modules into one view for a non-technical reader:

- **Headline KPIs:** default rate 22.1% · fraud rate 0.13% · portfolio return +75.2% vs benchmark +92.8% · net cash flow ₹1.68M
- **One line per module** with a button that jumps to its page
- **Portfolio vs benchmark** cumulative return line chart
- **Method note:** all flags are rule-based SQL, not machine learning, and cash flow excludes transfers between own accounts

---

## Repository structure

```
README.md
sql/
  01_credit_risk.sql
  02_fraud_detection.sql
  03_portfolio_performance.sql
  04_cash_flow.sql
images/
  executive-summary-page.png
  credit-risk-page.png
  fraud-detection-page.png
  portfolio-performance-page.png
  cash-flow-page.png
powerbi/
  Financial_Risk_Dashboard.pbix  (optional)
```

## How to reproduce

1. Create a PostgreSQL database `financial_risk_performance`.
2. Download the datasets: UCI Default of Credit Card Clients; PaySim (Kaggle); S&P 500 stock data (Kaggle, `all_stocks_5yr.csv`); S&P 500 constituents (`datasets/s-and-p-500-companies` on GitHub); Daily Household Transactions (Kaggle, `prasad22/daily-transactions-dataset`).
3. Run each module script in order. Load large CSVs with `psql \copy`, not the pgAdmin import wizard, and verify the row counts (30,000; 6,362,620; 619,040; 2,461).
4. Open the Power BI file, point the PostgreSQL connection at your server and refresh.

## Design decisions

- **SQL-first, no ML layer:** rules are transparent and explainable, which suits risk and compliance work.
- **Document, don't silently fix:** data-quality quirks (undocumented codes, non-reconciling balances, capped amounts, unadjusted spin-offs) are recorded as notes or in exclusion tables with reasons.
- **Validate every rule:** flags are tested against ground truth with precision and recall.
- **Avoid hindsight bias:** the sample portfolio is chosen by a fixed liquidity rule, not by past performance.
- **Separate spending from transfers:** moving money between own accounts is not an expense, so it is reclassified and the assumption is stated.

## Skills demonstrated

SQL (CTEs, window functions, views, percentiles, conditional logic) · data cleaning and validation · rule-based risk and fraud flagging · precision/recall evaluation · financial returns and cumulative return calculation · cash-flow and expense analysis · Power BI (DAX measures, interactive dashboards, KPI cards) · data storytelling

## Data sources

- UCI Machine Learning Repository: Default of Credit Card Clients
- PaySim: synthetic financial transactions dataset (Kaggle)
- S&P 500 stock data, 2013–2018 (Kaggle, `camnugent/sandp500`)
- S&P 500 constituents list (`datasets/s-and-p-500-companies`, GitHub)
- Daily Household Transactions (Kaggle, `prasad22/daily-transactions-dataset`)

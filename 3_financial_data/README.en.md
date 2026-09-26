# `3_financial_data/` — Financial Statements

**One file per instrument**: each symbol has its own `{Symbol}.parquet` holding its full history since 2016
(by reporting period for the statements, by ex-rights event for dividend factors).

```
3_financial_data/
├── balance/          balance sheet
├── income/           income statement
├── cashflow/         cash flow statement
├── capital/          share capital structure
├── holder_num/       shareholder counts
├── per_share/        per-share metrics (TTM)
├── pershare_index/   per-share financial ratios
└── dividend_factors/ dividend / bonus-share adjustment factors
```

Layout: [中文](README.md) · [back to top](../README.en.md)

## Measured columns and size

Counts verified by reading real files (600519.SH, 000001.SZ, 300750.SZ) — all agree with `preview/`:

| Table | Columns | Sample size (1 stock) | Periods | Files |
|---|---:|---:|---:|---:|
| `balance` | 75 | 60.8 KB | 42 | 5559 |
| `income` | 41 | 36.7 KB | 42 | 5559 |
| `cashflow` | 68 | 59.0 KB | 42 | — |
| `capital` | 10 | 7.7 KB | 50 | — |
| `holder_num` | 38 | 28.4 KB | 42 | — |
| `per_share` | 10 | 9.6 KB | 42 | — |
| `pershare_index` | 99 | 86.3 KB | 42 | — |
| `dividend_factors` | 7 | 5.0 KB | 30 events | — |

One stock across all eight tables ≈ **290 KB**; the full 8 × 5,559 ≈ 44,000 files is roughly 1.5 GB
(extrapolated from samples, not a measured total).

## Common columns

Every table except `dividend_factors` starts with:

| Column | Type | Notes |
|---|---|---|
| `m_timetag` | object, `YYYYMMDD` **string** | reporting period end; sample range 20160331–20260630 |
| `m_anntime` | object, `YYYYMMDD` string | **announcement date**; sample max 20260815 |
| `Symbol` | object | code with market suffix, e.g. `600519.SH` |

> **`m_anntime` is the point-in-time key that makes backtests honest.** A figure is only usable from its
> announcement date, not from its reporting period — otherwise you act on the annual report months before
> it exists. Both columns are strings, so convert before comparing:
> `merge_asof(kline.sort_values('time'), fin.sort_values('m_anntime'), left_on='time', right_on='m_anntime')`

`holder_num` differs: it leads with `declareDate`, `endDate`, `Symbol` and is disclosed at a different cadence.

## Fields

<!-- financial_en: 30 行 | 译自 quantdb.cn §4 财务数据 -->
| Field | Type | Accounting item |
|---|---|---|
| m_timetag | date | Reporting period end date (YYYY-MM-DD) |
| m_anntime | date | Actual public announcement date of the report |
| cash_equivalents | float | Cash and bank balances |
| account_receivable | float | Accounts receivable |
| total_current_assets | float | Total current assets |
| fix_assets | float | Fixed assets |
| tot_assets | float | Total assets |
| shortterm_loan | float | Short-term borrowings |
| accounts_payable | float | Accounts payable |
| total_current_liability | float | Total current liabilities |
| long_term_loans | float | Long-term borrowings |
| tot_liab | float | Total liabilities |
| undistributed_profit | float | Retained earnings |
| tot_shrhldr_eqy_excl_min_int | float | Equity attributable to owners of the parent |
| total_equity | float | Total shareholders' equity |
| m_timetag / m_anntime | date | Reporting period / announcement date |
| revenue | float | Total operating revenue |
| total_operating_cost | float | Total operating cost |
| cost_of_goods_sold | float | Cost of goods sold |
| research_expenses | float | R&D expenses |
| oper_profit | float | Operating profit |
| tot_profit | float | Total profit |
| net_profit_excl_min_int_inc | float | Net profit attributable to owners of the parent |
| s_fa_eps_basic | float | Basic earnings per share (EPS) |
| s_fa_bps | float | Book value per share (BPS) |
| s_fa_eps_basic | float | Basic earnings per share (EPS) — duplicated upstream |
| du_return_on_equity | float | Return on equity, diluted (ROE) |
| du_profit_rate | float | Net profit margin on sales |
| inc_revenue_rate | float | YoY revenue growth (%) |
| inc_net_profit_rate | float | YoY net profit growth (%) |

> **`m_timetag` and `m_anntime` are `YYYYMMDD` strings in the Parquet, not dates** — convert before comparing.
> `m_anntime` is the point-in-time key: financials become usable only on their announcement date.

The remaining columns follow native TDX/Wind-style naming. For example `income` begins
`m_timetag, m_anntime, revenue, operating_revenue, total_operating_cost, cost_of_goods_sold, …`,
while `pershare_index` uses `s_fa_*` prefixed names (`s_fa_ocfps`, `s_fa_bps`, `s_fa_eps_basic`,
`s_fa_eps_diluted`, …). **Most of these have no per-field upstream definition — the biggest semantic gap in
the dataset.** Read the actual column list with `pd.read_parquet(...).columns`.

### `dividend_factors` (7 columns)

```
time, type, interest, allotPrice, stockBonus, allotment, Symbol
```

Per-10-share dividend/bonus/rights records, used to synthesise price adjustment factors.

| Column | Meaning |
|---|---|
| `time` | ex-dividend / ex-rights date |
| `type` | distribution type |
| `interest` | cash dividend per share (per-10-share basis) |
| `allotPrice` | rights issue price |
| `stockBonus` | bonus share ratio |
| `allotment` | rights ratio |

## Download

Per-stock file counts are large (5,559 files per table), so **download by symbol** where possible:

```bash
# A few names, full financial history
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  "3_financial_data/income/600519.SH.parquet" \
  "3_financial_data/balance/600519.SH.parquet" \
  --local_dir ./quantdb

# One table market-wide (note: 5559 files)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "3_financial_data/income/*" --local_dir ./quantdb --max-workers 16
```

## Reading

```python
import pandas as pd, glob

income = pd.read_parquet("quantdb/3_financial_data/income/600519.SH.parquet")
income["m_anntime"] = pd.to_datetime(income["m_anntime"], format="%Y%m%d")
income["m_timetag"] = pd.to_datetime(income["m_timetag"], format="%Y%m%d")

# Market-wide panel from per-stock files (5559 files — expect it to take a while)
frames = [pd.read_parquet(p) for p in glob.glob("quantdb/3_financial_data/income/*.parquet")]
panel = pd.concat(frames, ignore_index=True)
```

## Caveats

- Updated on reporting-period disclosure; latest is the 2026 interim reports (sample max `m_timetag` 20260630).
- Period counts vary by stock (recent IPOs, delisted names; `capital` has 50 rows where
  `dividend_factors` has 30 events) — do not assume a balanced panel after concatenation.
- `holder_num` is disclosed on a different cadence; align it on announcement date, not reporting period.
- Amount fields are in **CNY**; per-share fields in **CNY/share**.

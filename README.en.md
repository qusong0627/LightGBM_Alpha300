# QuantDB — China A-Share Quantitative Dataset (300+ Factors)

[![Dataset on ModelScope](https://img.shields.io/badge/Dataset-ModelScope-blue)](https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300)
[![License Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-green)](LICENSE)
[![中文文档](https://img.shields.io/badge/%E4%B8%AD%E6%96%87-README-orange)](README.md)

A China A-share quantitative research dataset covering the whole market (main boards / ChiNext / STAR Market),
roughly 5,500 instruments from 2016 to date: three adjustment schemes of daily OHLCV, index quotes,
financial statements, margin trading and securities lending, technical indicators, valuation and
order-book sentiment, plus **110 L1 factors / 211 L2 high-frequency factors / a 329-column merged table** —
ready for multi-factor modelling and LightGBM training.

> **The data itself lives on ModelScope**: <https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300>
> This repository ships **no data files** (56 GB / ~74,000 Parquet files). Its directory layout mirrors the
> data layout one-to-one; each directory's README documents that part's structure, measured size and field semantics.
>
> **Update cadence: synced at the end of each week.** Daily bars, derived indicators and factor tables append
> per trading day; financial statements update on reporting-period disclosure; margin trading is disclosed T+1.

Other languages: [中文](README.md)

## Dataset navigation

| Directory | Contents | Layout | Measured size | Doc |
|---|---|---|---:|---|
| `1_kline_data/` | Unadjusted / forward / backward daily bars, index bars | 2,608 day partitions | 0.12–0.22 MB/day | [EN](1_kline_data/README.en.md) · [中文](1_kline_data/README.md) |
| `2_base_sector/` | Margin trading, instrument master, sectors, calendar, index weights | Mixed | 3.2 MB (master table) | [EN](2_base_sector/README.en.md) · [中文](2_base_sector/README.md) |
| `3_financial_data/` | Balance sheet / income / cash flow / capital / holders / per-share / dividends | One file per stock | 5,559 files per table | [EN](3_financial_data/README.en.md) · [中文](3_financial_data/README.md) |
| `4_bond_etf/` | Convertible bonds, ETF creation/redemption lists | Snapshot files | ~1.2 MB | [EN](4_bond_etf/README.en.md) · [中文](4_bond_etf/README.md) |
| `5_technical_derived/` | Technical indicators (35), valuation (16), sentiment (17) | Day partitions | 0.45–1.12 MB/day | [EN](5_technical_derived/README.en.md) · [中文](5_technical_derived/README.md) |
| `6_ml_datasets/` | `features_daily` 78 / `l1_factors` 119 / `l2_factors` 219 / `l1_l2_factors` 329 | Day partitions | 1.74–14.39 MB/day | [EN](6_ml_datasets/README.en.md) · [中文](6_ml_datasets/README.md) |

## Quick download

Prerequisite: `pip install modelscope pandas pyarrow` (modelscope ≥ 1.40; every command below was executed and verified)

```bash
# Everything (~56 GB — check your disk and bandwidth first)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --local_dir ./quantdb --max-workers 8
```

**Look at samples first** (27 tables × 300 rows, ~3.1 MB total, a few seconds):

```bash
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "preview/*" --local_dir ./quantdb-preview
```

**One table, or one time range** (download granularity = a single trading-day partition):

```bash
# A single factor table
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_factors/*" --local_dir ./quantdb

# The merged L1+L2 wide table for 2026 only
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/dt=2026*/*" --local_dir ./quantdb

# A single concrete file (paths containing dt= work fine)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  "1_kline_data/daily_unadjusted/dt=20260924/data.parquet" --local_dir ./quantdb
```

Equivalent Python:

```python
from modelscope import snapshot_download

path = snapshot_download(
    repo_id="qusong0627/LightGBM_Alpha300",
    repo_type="dataset",
    local_dir="./quantdb",
    allow_patterns=["6_ml_datasets/l1_factors/dt=2026*/*"],
)
```

> ⚠️ The flag is `--repo-type dataset`, **not** `--dataset`. The `--dataset` form you find in many
> tutorials fails outright on the current version.

## How much to download (measured)

Measured on each table's own latest partition; a year assumes ~244 trading days:

| Dataset | Per day | Per year (est.) | Partitions | Latest |
|---|---:|---:|---:|---|
| `1_kline_data/daily_unadjusted` | 0.12 MB | ~29 MB | 2608 | 20260924 |
| `1_kline_data/daily_forward` | 0.13 MB | ~32 MB | 2608 | 20260924 |
| `1_kline_data/daily_backward` | 0.22 MB | ~54 MB | 2608 | 20260924 |
| `5_technical_derived/technical_indicators` | 1.12 MB | ~273 MB | 2608 | 20260924 |
| `5_technical_derived/valuation` | 0.46 MB | ~112 MB | 2608 | 20260924 |
| `5_technical_derived/market_sentiment` | 0.52 MB | ~127 MB | 2608 | 20260924 |
| `6_ml_datasets/features_daily` | 1.74 MB | ~0.42 GB | 2608 | 20260924 |
| `6_ml_datasets/l1_factors` | 3.69 MB | ~0.90 GB | 2608 | 20260924 |
| `6_ml_datasets/l2_factors` | 9.92 MB | ~2.42 GB | 2120 | 20260924 |
| `6_ml_datasets/l1_l2_factors` | 14.39 MB | ~3.51 GB | 2120 | 20260924 |
| `preview/*` | — | 3.1 MB total | — | — |
| **Whole dataset** | — | **~56 GB** | 2608 trading days | 20260924 |

Daily cross-sections come in two universes (measured 2026-09-24): the daily bars and all three tables under `5_technical_derived/` have **5,570 rows**, including 347 Beijing Stock Exchange (`.BJ`) symbols; the factor tables have **no `.BJ`** — `features_daily` 5,223, `l1_factors` 5,208, `l2_factors` 5,210, `l1_l2_factors` 5,196 rows (14 MB × 329 columns). Joining across the two universes on `symbol` always drops the BSE rows.

## Global conventions

### Stock code format

<!-- code-format_en: 3 行 | 译自 quantdb.cn §股票代码格式 -->
| Exchange | Suffix | Example codes |
|---|---|---|
| Shanghai Stock Exchange (SSE) | .SH | 600000.SH , 688000.SH |
| Shenzhen Stock Exchange (SZSE) | .SZ | 000001.SZ , 300001.SZ |
| Beijing Stock Exchange (BSE) | .BJ | 830000.BJ , 430000.BJ |

### Price adjustment

<!-- adj-types_en: 3 行 | 译自 quantdb.cn §复权处理说明 -->
| Adjustment | Parameter | Algorithm | Typical use |
|---|---|---|---|
| Unadjusted | `unadjusted` | Raw traded prices, including ex-dividend/ex-rights gaps | Real trading-cost analysis, dividend return reconstruction |
| Forward | `forward` | Anchored on the latest price; history shifted downward | Quant backtests, technical indicators (most common) |
| Backward | `backward` | Anchored on the first trading day of 2016 (local history start); latest price shifted upward | Long-horizon compound returns, cumulative historical gain |

Two implementation notes:

- `daily_forward` is a **proportional** forward adjustment (scaled so the latest trading day's close is
  unchanged, so history stays strictly positive). `daily_backward` is a **proportional** backward adjustment
  (anchored at the window's first day, so new corporate actions never shift history).
- **Technical indicators are computed on the backward-adjusted series; valuation and sentiment on unadjusted
  raw prices.** OHLCV inside the factor tables (`l1_factors` onward) is **backward-adjusted** — on 2026-09-24,
  `000001.SZ` closes at 11.30 unadjusted but 18.65 in `l1_factors`. Confirm the basis before joining tables.

### Units

<!-- units_en: 12 行 | 译自 quantdb.cn §单位说明 -->
| Data type | Unit | Notes |
|---|---|---|
| Price (open/high/low/close) | CNY | Precise to fen/li |
| Daily volume (daily_* / features / L1 / technical) | shares | `volume` is in shares; identical across all three adjustment schemes |
| Daily turnover value (daily_* / features / L1) | 10k CNY | `amount` is in units of 10,000 CNY |
| Index daily bars (index_daily) | shares / 10k CNY | `volume` in shares, `amount` in 10k CNY; average price = amount\*1e4/volume |
| Market cap (total_mv / float_mv) | CNY | market cap = share capital × close |
| Share capital (total_capital) | shares | Total / circulating share count |
| Financial statement data | CNY | Standard accounting convention unless otherwise noted |

## Reading the data

Directory names are already `dt=` partitions, so DuckDB / pyarrow can prune them directly:

```python
import pandas as pd

kline = pd.read_parquet("quantdb/1_kline_data/daily_unadjusted/dt=20260924/data.parquet")
# columns: symbol, time, open, high, low, close, volume, amount
```

```python
import glob
frames = [pd.read_parquet(p) for p in sorted(glob.glob(
    "quantdb/6_ml_datasets/l1_factors/dt=2026*/data.parquet"))]
train = pd.concat(frames, ignore_index=True)   # verified: 4 partitions = 20832 rows × 119 cols
```

```python
import duckdb
df = duckdb.sql('''
  SELECT symbol, close, mom_ret_20d, micro_vpin_20
  FROM read_parquet('quantdb/6_ml_datasets/l1_l2_factors/dt=*/data.parquet',
                    hive_partitioning=true)
  WHERE dt >= 20260101
''').df()
```

## Known data conditions

All of the following were **measured** on real partitions:

1. **Two factors stopped updating.** `ind_netflow_rank_20` and `concept_flow_rank` are 100% null on
   2026-09-24 (degenerate to a constant), yet were healthy on 2026-06-01 — the upstream industry/concept
   money-flow source stopped feeding at some point. **Drop columns by null-rate threshold at runtime;
   do not hard-code a column list.**
2. **Two symbol universes.** The daily bars and the three `5_technical_derived/` tables carry 5,570 rows per
   day, including **347 Beijing Stock Exchange (`.BJ`)** symbols; every table under `6_ml_datasets/` excludes
   `.BJ` (5,196–5,223 rows). `margin_trading` does include BSE (341 symbols on 2026-09-23). Decide which
   universe a cross-section needs before joining — the two row sets are not the same length.
3. **Two key-column spellings.** Every table uses lowercase `symbol`. The date column is `time` in the
   daily bars, `5_technical_derived/` and `features_daily`, and `date` in `l1_factors` / `l2_factors` /
   `l1_l2_factors` (`l1_factors` carries both). Both are real `datetime64`; `YYYYMMDD` strings appear only
   as `TradingDate` / `m_timetag` / `m_anntime` in the base and financial tables.
4. **Forward-return labels are prefixed**: `future_return_1d`…`future_return_60d`, present in both
   `features_daily` and `technical_indicators`.
5. **L2 starts in 2018 only** (same for the merged table, 2120 trading days) — beware of structural breaks
   across that boundary.
6. **High-null L2 factors**: `micro_jump_skew` ~54%, `micro_jump_recovery_time` ~48%,
   `micro_price_impact_large` ~24%; the median null rate of other `micro_*`/`flow_*` columns is ~0.
7. **Valuation factors have extreme values**: `fun_pe` spanned -1206 to +1069 in a single day. LightGBM's
   rank-based splits are fairly tolerant; winsorize or quantile-transform before neutralisation or linear models.
8. **⚠️ Features vs labels.** `future_return_*` are forward N-day returns — ML **labels**, not features.
   Leaving them in `feature_name` is direct label leakage.

## Documentation principle: the data wins

The field tables in each directory README are translated excerpts of
<https://www.quantdb.cn/docs/fields.html>, audited column by column against the real Parquet files:
column names, counts and semantics are always written as the data has them. The adjustment basis of
`technical_indicators.close`, for instance, was settled by matching 300 stocks against all three daily
series (see [`5_technical_derived/`](5_technical_derived/README.en.md)).

## Licence and citation

- Data licence: **Apache License 2.0** (see [LICENSE](LICENSE))
- For quantitative research and education only; **not investment advice**
- Copyright of the raw market data belongs to the exchanges and upstream vendors; do not commercially redistribute
- Validate accuracy independently; the maintainers accept no liability for losses from errors or omissions

For downloads, updates and feedback use the ModelScope dataset page:
<https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300>

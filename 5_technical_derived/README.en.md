# `5_technical_derived/` — Derived Indicators

Three **day-partitioned** derived-metric tables. Narrow and cheap — good candidates for a full download.

```
5_technical_derived/
├── technical_indicators/dt=YYYYMMDD/data.parquet   # 35 cols, 1.12 MB/day
├── valuation/dt=YYYYMMDD/data.parquet              # 16 cols, 0.45 MB/day
└── market_sentiment/dt=YYYYMMDD/data.parquet       # 17 cols, 0.52 MB/day
```

Layout: [中文](README.md) · [back to top](../README.en.md)

## Measured size — and a synchronisation lag

| Sub-directory | Columns | Partitions | Latest | Per day | Per year (est.) |
|---|---:|---:|---|---:|---:|
| `technical_indicators` | 35 | 2608 | **20260924** | 1.12 MB | ~273 MB |
| `valuation` | 16 | **2604** | **20260918** | 0.45 MB | ~110 MB |
| `market_sentiment` | 17 | **2604** | **20260918** | 0.52 MB | ~127 MB |

> ⚠️ **`valuation` and `market_sentiment` trail the rest of the dataset by four trading days**
> (2604 vs 2608 partitions). A script that hard-codes "the latest trading day" reads an empty frame
> for these two — resolve each table's own newest partition instead.

## Download

```bash
# All three tables (~0.5 GB total — the easiest option)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "5_technical_derived/*" --local_dir ./quantdb

# Just the valuation history
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "5_technical_derived/valuation/dt=202*/*" --local_dir ./quantdb
```

## Computation basis

- **`technical_indicators` is computed on the backward-adjusted series (`daily_backward`).**
- **`valuation` and `market_sentiment` are computed on unadjusted raw prices.**

All three tables lead with `symbol, time, close`, but the two `close` values are not the same basis —
never cross-validate one against the other.

> ⚠️ **The upstream spec is wrong here.** `fields.html` labels this table's `close` as 前复权
> (forward-adjusted). Measured on 2026-09-24 across 300 stocks: `technical_indicators.close` equals
> `daily_backward.close` in **300/300 cases with 0.000 mean relative deviation**, and does not match
> `daily_forward`. **It is backward-adjusted**, agreeing with the ModelScope README and contradicting the
> spec. Example: `000001.SZ` closes at 11.30 unadjusted/forward, 18.647061 backward — this table says
> 18.647061. Where documentation and data disagree, **trust the data**.

## 1. `technical_indicators` (35 columns, measured)

```
symbol, time, close,
ma5, ma10, ma20, ma60, ma_gap_5, ma_gap_10, ma_gap_20,
rsi_6, rsi_14, kdj_k, kdj_d, kdj_j,
macd_dif, macd_dea, macd_hist,
vol_std_5, vol_std_20, vol_std_60, vol_atr_14,
vol_to_ma5, vol_to_ma20, volume_ma_3, amount_ma_5, volume_trend_3d,
future_return_1d, future_return_3d, future_return_5d,
future_return_10d, future_return_20d, future_return_60d,
pct_change, beta_20
```

| Family | Columns | Notes |
|---|---|---|
| Moving averages | `ma5/10/20/60`, `ma_gap_5/10/20` | close SMA and its bias |
| Oscillators | `rsi_6`, `rsi_14`, `kdj_k/d/j` | RSI, KDJ |
| Trend | `macd_dif`, `macd_dea`, `macd_hist` | MACD |
| Volatility | `vol_std_5/20/60`, `vol_atr_14` | return std dev, average true range |
| Volume energy | `vol_to_ma5`, `vol_to_ma20`, `volume_ma_3`, `amount_ma_5`, `volume_trend_3d` | volume ratios and means |
| Returns | `pct_change`, `beta_20` | daily change, 20-day beta |
| **Labels** | `future_return_1d … 60d` | **forward N-day returns — ML labels, not features** |

<!-- technical_en: 7 行 | 译自 quantdb.cn §6 技术衍生指标 -->
| Field | Type | Computation |
|---|---|---|
| close | float | Close price — upstream says "forward-adjusted", **actually backward-adjusted (see note)** |
| ma5 / ma10 / ma20 / ma60 | float | 5/10/20/60-day simple moving averages |
| rsi_6 / rsi_14 | float | RSI (mainstream ewm smoothing) |
| kdj_k / kdj_d / kdj_j | float | KDJ (SMA(RSV,3,1)) |
| macd_dif / macd_dea / macd_hist | float | MACD (hist = 2 × (DIF − DEA)) |
| vol_atr_14 | float | 14-day ATR (true range, Wilder RMA) |
| beta_20 | float | 20-day CAPM beta versus CSI 300 |

> ⚠️ **Upstream error.** The spec labels `close` as 前复权 (forward-adjusted).
> Measured against the actual files for 300 stocks on 2026-09-24:
> `technical_indicators.close` equals `daily_backward.close` in **300/300** cases (0.000 relative deviation),
> and does not match `daily_forward`. **The indicators are computed on the backward-adjusted series.**

> **⚠️ Label leakage.** The six `future_return_*` columns sit in the same table as the features. If you
> auto-generate a feature list as "everything except the key", you feed the model its own answer and
> backtest metrics become fiction. Exclude them explicitly.

## 2. `valuation` (16 columns, measured)

```
symbol, time, close, total_capital, circulating_capital, total_mv, float_mv,
net_profit_ttm, revenue_ttm, equity, annual_net_profit,
pe_ttm, pe_static, pb, ps_ttm, dividend_rate
```

| Column | Unit | Meaning |
|---|---|---|
| `total_capital` / `circulating_capital` | shares | total / circulating share count |
| `total_mv` / `float_mv` | CNY | total / free-float market value |
| `net_profit_ttm` / `revenue_ttm` / `equity` / `annual_net_profit` | CNY | TTM profit / revenue / parent equity / latest annual profit |
| `pe_ttm` / `pe_static` / `pb` / `ps_ttm` / `dividend_rate` | × / % | valuation ratios |

<!-- valuation_en: 6 行 | 译自 quantdb.cn §7 估值指标 -->
| Field | Type | Formula |
|---|---|---|
| total_mv / float_mv | float | Total / free-float market cap (share count × close) |
| net_profit_ttm | float | TTM net profit attributable to parent (single-quarter differenced, rolling 4 quarters) |
| pe_ttm | float | Trailing P/E (total_mv / net_profit_ttm) |
| pb | float | P/B (total_mv / parent equity) |
| ps_ttm | float | Trailing P/S (total_mv / revenue_ttm) |
| dividend_rate | float | Trailing 12-month cumulative dividend yield (DPS / close) |

> Only 7 of the 16 shipped columns are documented upstream; `pe_static`, `annual_net_profit`,
> `revenue_ttm`, `equity`, `total_capital`, `circulating_capital`, `close`, `symbol`, `time` are not listed.

> Denominators can be negative or near zero, so `pe_ttm` / `ps_ttm` produce large and negative outliers
> (`fun_pe` in the factor tables measured -1206 to +1069 in a single day). Winsorize, or use reciprocals
> (EP/BP), before computing factor exposures.

## 3. `market_sentiment` (17 columns, measured)

```
symbol, time, close, price_range, upper_shadow, lower_shadow, body_ratio,
amount_per_trade, liquidity_score, intraday_vol, gap_up_down,
buy_pressure, sell_pressure, momentum_1d, momentum_3d, am_pm_trend, volume_concentration
```

| Column | Meaning |
|---|---|
| `price_range` / `upper_shadow` / `lower_shadow` / `body_ratio` | amplitude, shadows, real-body share (candle anatomy) |
| `amount_per_trade` | **CNY/share** — implied volume-weighted average price for the day |
| `liquidity_score` / `intraday_vol` | liquidity score, intraday volatility |
| `gap_up_down` | opening gap |
| `buy_pressure` / `sell_pressure` | order-book pressure |
| `momentum_1d` / `momentum_3d` / `am_pm_trend` | short momentum, morning/afternoon trend |
| `volume_concentration` | intraday volume concentration |

> **`market_sentiment` has no section in the upstream spec at all.** The meanings above are inferred from
> the column names; the exact formulas for `liquidity_score`, `buy_pressure` and `sell_pressure` are
> **not published anywhere** — reconstruct and verify them yourself before relying on them.

## Reading

```python
import pandas as pd

tech = pd.read_parquet("quantdb/5_technical_derived/technical_indicators/dt=20260924/data.parquet")
val  = pd.read_parquet("quantdb/5_technical_derived/valuation/dt=20260918/data.parquet")  # 4 days behind

feat  = tech.drop(columns=[c for c in tech.columns if c.startswith("future_return_")])
labels = tech[["symbol", "time", "future_return_5d"]]
```

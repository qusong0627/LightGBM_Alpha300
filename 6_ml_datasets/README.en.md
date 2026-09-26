# `6_ml_datasets/` — Machine-Learning Datasets (300+ Factors)

The **modelling core**: four day-partitioned wide tables, one full-market cross-section per day
(~5,200 rows). 321 L1+L2 factors in total, each with a published definition — reproduced in full below,
translated, and verified column by column against the data.

```
6_ml_datasets/
├── features_daily/dt=YYYYMMDD/data.parquet      #  78 cols, 1.74 MB/day, 2608 parts
├── l1_factors/dt=YYYYMMDD/data.parquet          # 119 cols, 3.69 MB/day, 2608 parts
├── l2_factors/dt=YYYYMMDD/data.parquet          # 219 cols, 9.92 MB/day, 2120 parts (from 2018)
└── l1_l2_factors/dt=YYYYMMDD/data.parquet       # 329 cols, 14.39 MB/day, 2120 parts (from 2018)
```

Layout: [中文](README.md) · [back to top](../README.en.md)

## Measured size

| Table | Columns | Partitions | Per day | Per year (est.) | Full history (est.) |
|---|---:|---:|---:|---:|---:|
| `features_daily` | 78 | 2608 | 1.74 MB | ~0.42 GB | ~4.5 GB |
| `l1_factors` | 119 | 2608 | 3.69 MB | ~0.90 GB | ~9.6 GB |
| `l2_factors` | 219 | 2120 | 9.92 MB | ~2.42 GB | ~21 GB |
| `l1_l2_factors` | 329 | 2120 | 14.39 MB | ~3.51 GB | ~30.5 GB |

Measured on 2026-09-24: `l1_factors` 5,208 rows, `l1_l2_factors` 5,196 rows (slightly fewer after the inner join).

## Which table do you need

- Daily features + labels for single-factor tests → `features_daily`
- Multi-factor modelling from 2016 → `l1_factors`
- Microstructure / money-flow factors from 2018 → `l1_l2_factors` (already merged; no need for both l1 and l2)
- Raw L2 high-frequency factors only → `l2_factors`

## Download

```bash
# Samples first (27 tables × 300 rows, 3.1 MB)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "preview/*" --local_dir ./quantdb-preview

# One year of L1 factors (~0.9 GB)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_factors/dt=2026*/*" --local_dir ./quantdb

# The merged wide table since 2018 (~30 GB, the modelling workhorse)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/*" --local_dir ./quantdb --max-workers 16

# Only a training window
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/dt=202[3-6]*/*" --local_dir ./quantdb
```

```python
from modelscope import snapshot_download
snapshot_download(
    repo_id="qusong0627/LightGBM_Alpha300", repo_type="dataset",
    local_dir="./quantdb",
    allow_patterns=["6_ml_datasets/l1_l2_factors/dt=2026*/*"],
)
```

## Keys and price basis

| Table | Key columns | Price columns |
|---|---|---|
| `features_daily` | `symbol, time` | `close` |
| `l1_factors` / `l2_factors` / `l1_l2_factors` | `symbol, date` (plus `time`) | backward-adjusted `open/high/low/close/volume/amount` |

> **OHLCV in the factor tables is backward-adjusted, not traded price.** On 2026-09-24, `000001.SZ`
> closed at 11.30 unadjusted but appears as 18.65 here. Do not use it for price display or limit-up/down checks.

## 1. `features_daily` (78 columns)

Daily feature wide table = technical indicators + valuation + sentiment + instrument attributes
+ six forward-return labels. Full column list:

```
symbol, time, close, ma5, ma10, ma20, ma60, ma_gap_5, ma_gap_10, ma_gap_20,
rsi_6, rsi_14, kdj_k, kdj_d, kdj_j, macd_dif, macd_dea, macd_hist,
vol_std_5, vol_std_20, vol_std_60, vol_atr_14, vol_to_ma5, vol_to_ma20,
volume_ma_3, amount_ma_5, volume_trend_3d,
future_return_1d, future_return_3d, future_return_5d, future_return_10d, future_return_20d, future_return_60d,
pct_change, beta_20,
total_capital, circulating_capital, total_mv, float_mv, net_profit_ttm, revenue_ttm, equity,
annual_net_profit, pe_ttm, pe_static, pb, ps_ttm, dividend_rate,
in_hs300, is_hsgt, is_margin, is_kcb_creatable, is_st, is_quit_risk, is_hk,
industry_code, industry_name, sector_code, region_area_code, region_area_name,
main_business, list_date, total_cap_yi, float_mv_yi, free_float_shares, ipo_price,
zt_price, dt_price, hs_turnover, seal_strength, zaf, beta_now, dyna_pe, static_pe_ttm,
div_yield, pb_mrq, ever_zt_count, year_zt_days
```

<!-- factors_features_en: 45 行 | 译自 quantdb.cn §8.1 features_daily -->
| Field | Label | Definition |
|---|---|---|
| time / Symbol | Trading date / stock code | Row index; `{6 digits}.{SH/SZ/BJ}` |
| close | Close (CNY, backward-adjusted) | Backward-adjusted close; base series for all technical indicators |
| ma5 | MA5 | N-day simple moving average of the backward-adjusted close |
| ma10 | MA10 | N-day simple moving average of the backward-adjusted close |
| ma20 | MA20 | N-day simple moving average of the backward-adjusted close |
| ma60 | MA60 | N-day simple moving average of the backward-adjusted close |
| ma_gap_5 | MA bias 5 (%) | (close − MA) / MA × 100 |
| ma_gap_10 | MA bias 10 (%) | (close − MA) / MA × 100 |
| ma_gap_20 | MA bias 20 (%) | (close − MA) / MA × 100 |
| rsi_6 / rsi_14 | RSI6 / RSI14 | Mainstream convention: avg_gain/avg_loss via ewm(alpha=1/N); all-up = 100, flat = 50, all-down = 0, first row = NaN |
| kdj_k | KDJ-K | RSV = (C − LLV(L,9)) / (HHV(H,9) − LLV(L,9)) × 100; K/D = SMA(RSV,3,1) = ewm(com=2); J = 3K − 2D; RSV = 50 for zero-range windows |
| kdj_d | KDJ-D | See kdj_k |
| kdj_j | KDJ-J | See kdj_k |
| macd_dif | MACD-DIF | DIF = EMA12 − EMA26, DEA = EMA9 of DIF, histogram = 2 × (DIF − DEA) |
| macd_dea | MACD-DEA | See macd_dif |
| macd_hist | MACD histogram | See macd_dif |
| vol_std_5 | 5-day volatility | N-day rolling std dev of the daily percentage return series |
| vol_std_20 | 20-day volatility | N-day rolling std dev of the daily percentage return series |
| vol_std_60 | 60-day volatility | N-day rolling std dev of the daily percentage return series |
| vol_atr_14 | 14-day ATR | Wilder RMA of true range (ewm alpha = 1/14) |
| vol_to_ma5 | Volume ratio MA5 | Today's volume / N-day mean volume (including today) — not the order-book turnover ratio |
| vol_to_ma20 | Volume ratio MA20 | Today's volume / N-day mean volume (including today) |
| volume_ma_3 | 3-day mean volume (shares) | 3-day rolling mean of volume |
| amount_ma_5 | 5-day mean amount (10k CNY) | 5-day rolling mean of turnover value |
| volume_trend_3d | 3-day volume trend | 3-day mean of the volume period-over-period change rate |
| return_1d | Forward 1-day return (LABEL) (%) | (close.shift(−d)/close − 1) × 100, on the **unadjusted** close (reflects genuinely tradable return; undistorted across ex-rights windows) |
| return_3d | Forward 3-day return (LABEL) (%) | Same, d = 3 |
| return_5d | Forward 5-day return (LABEL) (%) | Same, d = 5 |
| return_10d | Forward 10-day return (LABEL) (%) | Same, d = 10 |
| return_20d | Forward 20-day return (LABEL) (%) | Same, d = 20 |
| return_60d | Forward 60-day return (LABEL) (%) | Same, d = 60 |
| pct_change | Daily change (%) | Close relative to the previous close |
| beta_20 | 20-day Beta | CAPM beta versus CSI 300 (000300.SH), on unadjusted returns |
| total_capital | Total share capital | Latest share count aligned to the announcement date (market cap uses raw price × shares) |
| circulating_capital | Circulating share capital | Announcement-date aligned |
| total_mv / float_mv | Total / float market cap (CNY) | Total shares × close / circulating shares × close |
| net_profit_ttm | TTM net profit to parent (CNY) | Single-quarter differenced, rolling 4-quarter sum |
| revenue_ttm | TTM revenue (CNY) | Single-quarter differenced, rolling 4-quarter sum |
| equity | Parent equity (CNY) | Total equity attributable to owners of the parent (announcement date) |
| annual_net_profit | Annual report net profit (CNY) | Latest annual figure; the static P/E denominator |
| pe_ttm | PE (TTM) | total_mv / net_profit_ttm |
| pe_static | PE (static) | total_mv / annual_net_profit |
| pb | PB | total_mv / equity |
| ps_ttm | PS (TTM) | total_mv / revenue_ttm |
| dividend_rate | Dividend yield | Trailing 12-month cumulative DPS / close |

> ⚠️ **Naming mismatch with the data.** Upstream calls the labels `return_1d` … `return_60d`;
> **the actual columns are `future_return_1d` … `future_return_60d`**. Upstream also writes `Symbol`,
> while the file uses lowercase `symbol`. These six columns are **labels, not features**.
> 37 further columns present in the data are not documented upstream.

> ⚠️ **Spec vs data (data wins).**
> - The spec names the labels `return_1d`…`return_60d`; **the actual columns are `future_return_1d`…`future_return_60d`**.
> - The spec writes `Symbol`; the file uses lowercase `symbol`.
> - The spec covers 48 fields; the table has 78. The **37 undocumented** ones are mostly instrument
>   attributes (`industry_code/name`, `region_area_code/name`, `main_business`, `list_date`, `is_st`,
>   `is_hsgt`, `is_margin`, `in_hs300`, `is_kcb_creatable`, `is_quit_risk`, `is_hk`) and TDX-style derived
>   columns (`zt_price`, `dt_price`, `seal_strength`, `hs_turnover`, `zaf`, `beta_now`, `dyna_pe`,
>   `static_pe_ttm`, `div_yield`, `pb_mrq`, `ever_zt_count`, `year_zt_days`, `total_cap_yi`,
>   `float_mv_yi`, `free_float_shares`, `ipo_price`).

> **⚠️ `future_return_*` are labels, not features.** Six forward-return columns share the table with the
> features; auto-generating a feature list as "everything but the key" leaks the target.

## 2. `l1_factors` (119 columns = 9 keys + 110 daily factors)

Keys and prices: `symbol, date, time, open, high, low, close, volume, amount` (backward-adjusted).
The 110 factors are organised by prefix:

| Prefix | Count | Family |
|---|---:|---|
| `turn_*` | 16 | turnover rate |
| `amt_*` | 16 | turnover value |
| `mom_*` | 16 | momentum |
| `ind_*` | 14 | industry |
| `fun_*` | 11 | fundamentals & valuation |
| `concept_*` | 10 | concept themes |
| `vol_*` | 8 | volatility |
| `tech_*` | 6 | technical patterns |
| `chip_*` | 6 | holder cost distribution ("chips") |
| `style_*` | 5 | style |
| `mfi_*` + `obv_*` | 2 | classic money flow |

### All 110 L1 factor definitions (verified 110/110 against the actual columns)

<!-- factors_l1_en: 110 行 | 译自 quantdb.cn/docs/fields.html §8.2 -->
| Field | Label | Definition |
|---|---|---|
| turn_1 | 1-day turnover | Turnover rate on the current day |
| turn_3 | 3-day turnover | N-day rolling mean of the turnover rate |
| turn_5 | 5-day turnover | N-day rolling mean of the turnover rate |
| turn_10 | 10-day turnover | N-day rolling mean of the turnover rate |
| turn_20 | 20-day turnover | N-day rolling mean of the turnover rate |
| turn_60 | 60-day turnover | N-day rolling mean of the turnover rate |
| turn_std_20 | 20-day turnover std dev | 20-day rolling standard deviation of turnover |
| turn_z_20 | 20-day turnover z-score | (today − 20-day mean) / 20-day std dev |
| turn_ratio_1_5 | 1/5-day turnover ratio | Today / 5-day mean |
| turn_ratio_1_20 | 1/20-day turnover ratio | Today / 20-day mean |
| turn_trend_5_20 | 5/20-day turnover trend | 5-day mean / 20-day mean |
| turn_acc_5 | 5-day turnover acceleration | 5-day rolling sum of turnover |
| turn_acc_20 | 20-day turnover acceleration | 20-day rolling sum of turnover |
| turn_breakout_20 | 20-day turnover breakout | Today / 20-day maximum turnover |
| turn_hl_pos_20 | 20-day turnover range position | (today − 20-day min) / (20-day max − 20-day min) |
| turn_high_days_10 | High-turnover days in 10 | Days in the last 10 (excluding today) whose turnover exceeded the 20-day mean |
| amt_log | Log turnover value | log(turnover value of the day) |
| amt_ma_5 | 5-day mean turnover (log) | log(N-day rolling mean of turnover value) |
| amt_ma_20 | 20-day mean turnover (log) | log(N-day rolling mean of turnover value) |
| amt_ma_60 | 60-day mean turnover (log) | log(N-day rolling mean of turnover value) |
| amt_z_20 | 20-day turnover z-score | (log amount − 20-day mean) / 20-day std dev |
| amt_ratio_1_5 | 1/5-day amount ratio | Today / 5-day mean |
| amt_ratio_1_20 | 1/20-day amount ratio | Today / 20-day mean |
| amt_ratio_5_20 | 5/20-day amount ratio | 5-day mean / 20-day mean |
| amt_skew_20 | 20-day turnover skewness | 20-day rolling skewness of turnover value |
| amt_close_pos | Amount-weighted close position | (close − low) / (high − low); close's relative position within the day's range |
| amt_net_flow_5 | 5-day net money flow | (OBV − OBV.shift(N)) / N-day cumulative volume; negative = net outflow |
| amt_net_flow_20 | 20-day net money flow | (OBV − OBV.shift(N)) / N-day cumulative volume; negative = net outflow |
| amt_up_ratio_5 | 5-day up-day amount share | Turnover value on up days / total turnover value over N days |
| amt_up_ratio_20 | 20-day up-day amount share | Turnover value on up days / total turnover value over N days |
| amt_high_days_10 | High-amount days in 10 | Days in the last 10 (excluding today) whose turnover value exceeded the 20-day mean |
| mfi_14 | 14-day money flow index | 14-day MFI computed from typical price × volume |
| obv_slope_20 | 20-day OBV slope | 20-day ordinary-least-squares slope of OBV / 20-day mean volume |
| amt_vol_ratio_20 | 20-day return-volume correlation | 20-day rolling correlation between daily return and volume |
| mom_ret_1d | 1-day return | close/close.shift(N) − 1; **backward**-looking N-day return (opposite direction to the forward-return labels in §8.1) |
| mom_ret_3d | 3-day return | close/close.shift(N) − 1, backward-looking |
| mom_ret_5d | 5-day return | close/close.shift(N) − 1, backward-looking |
| mom_ret_10d | 10-day return | close/close.shift(N) − 1, backward-looking |
| mom_ret_20d | 20-day return | close/close.shift(N) − 1, backward-looking |
| mom_ret_60d | 60-day return | close/close.shift(N) − 1, backward-looking |
| mom_ma_gap_5 | MA5 bias | close / SMA(close, N) − 1 |
| mom_ma_gap_20 | MA20 bias | close / SMA(close, N) − 1 |
| mom_macd_dif | MACD fast line | Same definition as §8.1 (EMA12 − EMA26; DEA = EMA9; histogram = 2 × difference) |
| mom_macd_dea | MACD slow line | Same definition as §8.1 |
| mom_macd_hist | MACD histogram | Same definition as §8.1 |
| mom_rsi_6 | 6-day RSI | Same definition as §8.1 (standard ewm smoothing) |
| mom_rsi_14 | 14-day RSI | Same definition as §8.1 |
| mom_kdj_k | KDJ-K | Same definition as §8.1 (9-day RSV, SMA(3,1)) |
| mom_kdj_d | KDJ-D | Same definition as §8.1 |
| mom_kdj_j | KDJ-J | Same definition as §8.1 |
| vol_std_5 | 5-day return std dev | N-day rolling std dev of backward-adjusted daily returns |
| vol_std_10 | 10-day return std dev | N-day rolling std dev of backward-adjusted daily returns |
| vol_std_20 | 20-day return std dev | N-day rolling std dev of backward-adjusted daily returns |
| vol_atr_14 | 14-day ATR | ATR(14) of true range, same definition as §8.1 |
| vol_parkinson_20 | 20-day Parkinson volatility | High-low based Parkinson estimator, 20-day window |
| vol_gk_20 | 20-day Garman-Klass volatility | OHLC based Garman-Klass estimator, 20-day window |
| vol_amp_1 | 1-day amplitude | (high − low) / close |
| vol_amp_20 | 20-day mean amplitude | 20-day rolling mean of the daily amplitude |
| tech_bb_width | Bollinger band width | Upper-minus-lower width of Bollinger(20, 2σ) |
| tech_bb_pos | Bollinger band position | Close's relative position between the upper and lower bands |
| tech_cci_20 | 20-day CCI | CCI(20) |
| tech_adx_14 | 14-day ADX | ADX(14); trend strength |
| tech_close_to_high_20 | Distance from 20-day high | 1 − close / rolling_max(high, 20) |
| tech_max_drawdown_20 | 20-day max drawdown | 1 − close / rolling_max(close, 20) |
| fun_float_mv | Log free-float market cap | log(raw price × circulating shares) |
| fun_total_mv | Log total market cap | log(raw price × total shares) |
| fun_mv_rank | Market-cap cross-section percentile | Cross-sectional percentile rank of free-float market cap on the day |
| fun_pe | PE (TTM) | Total market cap / TTM net profit attributable to parent |
| fun_pb | PB | Total market cap / equity attributable to parent |
| fun_bp | Book-to-price (BP) | 1 / PB |
| fun_ep | Earnings yield (EP, TTM) | TTM net profit attributable to parent / total market cap |
| fun_value_zscore | Composite value z-score | (zscore(EP) + zscore(BP)) / 2, computed cross-sectionally each day |
| fun_roe | ROE (%) | TTM net profit attributable to parent / equity × 100 |
| fun_peg | PEG | PE / YoY net-profit growth rate |
| fun_np_growth | YoY net-profit growth | TTM net profit year-over-year (derived from the income statement) |
| chip_profit_ratio_20 | 20-day profitable-chip ratio | Share of chips whose cost basis is below the current price (over the window) |
| chip_profit_ratio_60 | 60-day profitable-chip ratio | Share of chips whose cost basis is below the current price (over the window) |
| chip_concentration_20 | 20-day chip concentration | (P95 − P5) / current price; relative width of the cost distribution |
| chip_floating_ratio | 5-day floating-chip ratio | Chips from the last 5 days / total chips |
| chip_cost_90_width | 90% chip cost width (CNY) | Absolute width P95 − P5 |
| chip_profit_delta_5 | 5-day change in profitable chips | ratio_20(t) − ratio_20(t−5) |
| style_beta_20 | 20-day beta vs CSI 500 | Rolling Cov(stock, benchmark) / Var(benchmark) |
| style_beta_60 | 60-day beta vs CSI 500 | Rolling Cov(stock, benchmark) / Var(benchmark) |
| style_idio_vol_20 | 20-day idiosyncratic vol | N-day rolling std dev of the CAPM residual (r − beta × rb) |
| style_idio_vol_60 | 60-day idiosyncratic vol | N-day rolling std dev of the CAPM residual (r − beta × rb) |
| style_residual_ret_20 | 20-day residual momentum | 20-day rolling sum of the CAPM residual |
| ind_ret_5 | Industry 5-day return percentile | Cross-industry percentile rank of the mean backward-looking return of the industry's constituents |
| ind_ret_10 | Industry 10-day return percentile | Cross-industry percentile rank of the mean backward-looking return of the industry's constituents |
| ind_ret_20 | Industry 20-day return percentile | Cross-industry percentile rank of the mean backward-looking return of the industry's constituents |
| ind_rotation_speed_20 | Industry rotation speed | \|short-term rank − long-term rank\| / number of industries (5-day vs 20-day) |
| ind_strength_20 | 20-day relative industry strength | Stock's 20-day return − mean 20-day return of its industry |
| ind_dispersion_20 | Intra-industry dispersion | Std dev of the 20-day returns of the industry's constituents |
| ind_breadth_up_20 | Industry advancing breadth | Share of constituents with a 20-day return > 0 |
| ind_volume_ratio_20 | Industry turnover share percentile | Cross-industry percentile rank of the industry's total turnover value |
| ind_crowding_20 | Industry crowding percentile | Cross-industry rank of industry turnover / market-wide turnover |
| ind_netflow_rank_20 | Industry money-flow percentile | Cross-industry rank of the sum of L2 net inflow (`flow_net_amount`) within the industry — **currently unpopulated, see caveats** |
| ind_relative_pe | Industry relative-PE percentile | Cross-industry rank of (industry median PE / market-wide median PE) |
| ind_concentration | Industry concentration | Top-5 free-float market cap within the industry / industry total free-float cap |
| ind_relative_momentum_20 | Industry relative excess momentum | Mean 20-day industry return − mean 20-day market-wide return |
| ind_momentum_decay | Industry momentum decay | Mean 5-day industry return − mean 20-day industry return |
| concept_hot_score | Concept heat score | Number of the stock's concepts with a 20-day return > 3% |
| concept_momentum_top3 | Top-3 concept momentum score | Mean of the top-3 concepts ranked by 20-day return among those the stock belongs to |
| concept_exposure_top1 | Primary concept exposure | The stock's turnover-value percentile inside its strongest concept |
| concept_rotation_score | Concept rotation score | Mean of \|5-day rank − 20-day rank\| across the stock's concepts |
| concept_crowding_max | Max concept crowding | Maximum turnover-share percentile among the stock's concepts |
| concept_diversity | Concept diversity index | Number of concepts the stock belongs to |
| concept_flow_rank | Associated concept money-flow percentile | Mean of the L2 net-inflow percentiles across the stock's concepts — **currently unpopulated, see caveats** |
| concept_leader_score | Concept leader score | Mean of the stock's turnover-value percentile within each of its concepts |
| concept_cross_sector | Cross-sector concept strength | Whether the stock's concepts span ≥ 2 first-level industries (1/0) |
| concept_volume_ratio | Concept turnover share | Mean of (concept turnover / market-wide turnover) across the stock's concepts |

## 3. `l2_factors` (219 columns = 8 keys + 211 high-frequency factors)

Computed from intraday **tick-by-tick trades and orders**. **2018 onwards only** (2120 trading days).

| Prefix | Count | Family |
|---|---:|---|
| `micro_*` | 139 | microstructure: quoted/effective spreads, VPIN (8/20/50/100 buckets), order imbalance, 10-level depth, price impact (Amihud, Kyle's λ), cancellation behaviour, inter-trade intervals and sequence entropy, jumps, session decomposition (T1–T8), liquidity family, adverse selection, information share |
| `flow_*` | 44 | money flow: net/buy/sell amounts and ratios, super-large/large/medium/small order structure, order-flow toxicity, consistency, big-trade share |
| `vol_*` | 28 | realized volatility: RV/RRV/skewness/kurtosis, 5/10/15-minute and AM/PM decomposition, jump component |

### All 211 L2 factor definitions (verified 211/211 against the actual columns)

<!-- factors_l2_en: 211 行 | 译自 quantdb.cn/docs/fields.html §8.3 -->
| Field | Label | Definition |
|---|---|---|
| micro_qsp_equal | Equal-weighted quoted spread | qsp = (ask1 − bid1) / mid × 100, equal-weighted mean, in percent |
| micro_qsp_time | Time-weighted quoted spread | qsp weighted by snapshot time intervals |
| micro_qsp_volume | Volume-weighted quoted spread | qsp weighted by snapshot volume |
| micro_esp_equal | Equal-weighted effective spread | esp = 2 × \|trade price − mid at trade time\| / mid × 100, equal-weighted mean |
| micro_esp_time | Time-weighted effective spread | esp weighted by trade time intervals |
| micro_esp_volume | Volume-weighted effective spread | esp weighted by trade volume |
| micro_aqsp | Absolute quoted spread (CNY) | Equal-weighted mean of ask1 − bid1, in CNY |
| micro_aesp | Absolute effective spread (CNY) | Equal-weighted mean of 2 × \|trade price − mid\|, in CNY |
| micro_qsp_vol_20 | Quoted-spread volatility | Std dev of qsp across the whole day |
| micro_esp_vol_20 | Effective-spread volatility | Std dev of esp across the whole day |
| micro_spread_slope_1_5 | Depth slope, levels 1–5 | Volume-weighted regression slope of price over depth for levels 1–5, averaged intraday |
| micro_spread_slope_5_10 | Depth slope, levels 5–10 | Same as above for levels 5–10 |
| micro_spread_slope_1_10 | Depth slope, levels 1–10 | Same as above for levels 1–10 |
| micro_bid_ask_vol_ratio | Level-1 bid/ask volume ratio | mean(bid_vol1) / mean(ask_vol1) |
| micro_spread_recovery | Spread recovery speed | Average number of snapshots for qsp to fall back within 1.1× the mean after exceeding mean + 1σ |
| micro_spread_open | Opening quoted spread | Mean qsp during the opening session |
| micro_spread_mid | Continuous-hours quoted spread | Mean qsp during the continuous session |
| micro_spread_close | Closing quoted spread | Mean qsp during the closing session |
| micro_spread_vol_ratio | Spread volatility ratio | Equal-weighted qsp / per-tick return volatility proxy |
| micro_spread_U_shape | Spread U-shape | (open + close) / 2 / mid − 1, clipped to [−5, 5] |
| vol_realized_rv | Realized variance | Sum of squared 5-minute-bucket VWAP log returns, r = 100 × Δlog(VWAP) |
| vol_realized_rrv | Realized range variance | Sum of squared 5-minute-bucket high-low ranges |
| vol_realized_rskew | Realized skewness | √m × Σr³ / RV^1.5 |
| vol_realized_rkurt | Realized kurtosis | m × Σr⁴ / RV² |
| vol_realized_5min | 5-minute realized volatility | RV over 300 s buckets |
| vol_realized_10min | 10-minute realized volatility | RV over 600 s buckets |
| vol_realized_15min | 15-minute realized volatility | RV over 900 s buckets |
| vol_realized_am | Morning realized volatility | 5-minute-bucket RV over the morning session |
| vol_realized_pm | Afternoon realized volatility | 5-minute-bucket RV over the afternoon session |
| vol_realized_opening | Opening realized volatility | 5-minute-bucket RV over the opening session |
| vol_realized_closing | Closing realized volatility | 5-minute-bucket RV over the closing session |
| vol_realized_jump | Realized jump | max(RV − BV, 0), where BV is bipower variation |
| vol_realized_sjv | Signed jump variation | (RV − BV) / RV × √m, a standardized jump statistic |
| flow_net_amount | Net money inflow (CNY) | Active buy amount − active sell amount |
| flow_buy_amount | Active buy amount | Sum of trade amount where side = 0 |
| flow_sell_amount | Active sell amount | Sum of trade amount where side = 1 |
| flow_net_ratio | Net inflow ratio | (buy − sell) / (buy + sell), on an amount basis |
| flow_buy_ratio | Buy-trade ratio | Buy trade count / total trade count |
| flow_sell_ratio | Sell-trade ratio | Sell trade count / total trade count |
| flow_super_net | Super-large order net inflow | Buy − sell amount for single trades ≥ 1M CNY |
| flow_large_net | Large order net inflow | Buy − sell amount for single trades of 200k–1M CNY |
| flow_medium_net | Medium order net inflow | Buy − sell amount for single trades of 50k–200k CNY |
| flow_small_net | Small order net inflow | Buy − sell amount for single trades < 50k CNY |
| flow_large_ratio | Large order ratio | Large order (buy − sell) / (buy + sell) |
| flow_medium_ratio | Medium order ratio | Medium order (buy − sell) / (buy + sell) |
| flow_small_ratio | Small order ratio | Small order (buy − sell) / (buy + sell) |
| flow_large_pct | Large-trade amount share | (super-large + large) amount / whole-day amount |
| flow_imbalance_volume | Volume imbalance | (buy volume − sell volume) / (buy volume + sell volume) |
| flow_imbalance_open | Opening imbalance | Amount imbalance during the opening session |
| flow_imbalance_mid_am | Morning continuous imbalance | Amount imbalance during the morning continuous session |
| flow_imbalance_mid_pm | Afternoon continuous imbalance | Amount imbalance during the afternoon continuous session |
| flow_imbalance_close | Closing imbalance | Amount imbalance during the closing session |
| flow_imbalance_autocorr | Imbalance autocorrelation | Lag-1 autocorrelation of the 5-minute-bucket amount imbalance series |
| flow_imbalance_revert_speed | Imbalance reversion speed | Average number of buckets to fall from imbalance > 0.3 back below 0.1 |
| flow_consistency | Money-flow consistency | Share of adjacent trades in the same direction |
| flow_money_flow_index | Money flow index | (up-tick amount − down-tick amount) / (sum of both) |
| flow_big_trade_ratio | Big-trade ratio | Trades ≥ 500k CNY / total trade count |
| micro_vpin_8 | VPIN, 8 buckets | Mean buy/sell volume imbalance over volume-equal buckets (N = 8) |
| micro_vpin_20 | VPIN, 20 buckets | Same, N = 20 |
| micro_vpin_50 | VPIN, 50 buckets | Same, N = 50; bucket size = total volume / 50 |
| micro_vpin_ma_5 | 5-bucket VPIN mean | Mean of the last 5 buckets of the N = 50 VPIN series |
| micro_vpin_ma_20 | 20-bucket VPIN mean | Mean of the last 20 buckets of the N = 50 VPIN series |
| micro_vpin_delta_5 | 5-bucket VPIN change | Last bucket − mean of the previous 5 buckets |
| micro_vpin_zscore_20 | VPIN z-score | (last bucket − 20-bucket mean) / 20-bucket std dev |
| micro_pin | PIN (probability of informed trading) | \|buy volume − sell volume\| / (buy volume + sell volume), whole-day totals |
| micro_order_flow_toxicity | Order-flow toxicity | VPIN50 × whole-day price direction (±1) |
| micro_volume_sync | Volume-return comovement | corr(\|per-tick return\|, per-tick volume) |
| micro_vpin_surge | VPIN surge count | Number of buckets with VPIN > mean + 2σ |
| micro_vpin_trend | VPIN trend | Simple linear regression slope of the bucketed VPIN series |
| micro_vpin_vol_ratio | VPIN volatility ratio | VPIN50 / sum-of-squared-log-returns volatility proxy |
| micro_vpin_amount_ratio | VPIN amount ratio | VPIN50 / log(whole-day turnover value) |
| micro_depth_bid | Bid depth | log1p(mean of total bid volume across 10 levels) |
| micro_depth_ask | Ask depth | log1p(mean of total ask volume across 10 levels) |
| micro_depth_ratio_1 | Level-1 depth ratio | mean(bid1 volume / ask1 volume) |
| micro_depth_ratio_5 | Level 1–5 depth ratio | mean(bid 1–5 volume / ask 1–5 volume) |
| micro_depth_ratio_10 | Level 1–10 depth ratio | mean(bid 1–10 volume / ask 1–10 volume) |
| micro_depth_weighted_bid | Volume-weighted bid price | Volume-weighted average bid price, averaged intraday |
| micro_depth_weighted_ask | Volume-weighted ask price | Volume-weighted average ask price, averaged intraday |
| micro_depth_concentration_bid | Bid-side concentration | Mean Herfindahl index of bid volume over the top 5 levels |
| micro_depth_concentration_ask | Ask-side concentration | Mean Herfindahl index of ask volume over the top 5 levels |
| micro_depth_slope | Depth slope | Volume-weighted regression slope of bid price over level (winsorized to 1–99%) |
| micro_depth_effective | Effective depth | mean(min(bid1 volume, ask1 volume)) |
| micro_depth_recovery | Depth recovery speed | Average snapshots for bid depth to return to 0.9× its mean after falling below mean − 0.5σ |
| micro_depth_open | Opening depth ratio | mean(total bid volume / total ask volume) during the opening session |
| micro_depth_mid | Continuous-hours depth ratio | Same during the continuous session |
| micro_depth_close | Closing depth ratio | Same during the closing session |
| micro_depth_volatility | Depth volatility | Coefficient of variation (std / mean) of the bid-to-ask total volume ratio |
| micro_depth_extreme | Depth extreme value | (last snapshot bid1 volume − mean) / std dev |
| micro_depth_imbalance_1 | Level-1 depth imbalance | mean((bid1 − ask1) / (bid1 + ask1)) |
| micro_depth_imbalance_5 | Level 1–5 depth imbalance | Same over aggregate top-5 volume |
| micro_depth_imbalance_10 | Level 1–10 depth imbalance | Same over aggregate top-10 volume |
| flow_cancel_rate | Cancellation rate | Cancelled orders / new orders |
| flow_cancel_buy_rate | Buy-side cancellation rate | Cancelled buys / new buys |
| flow_cancel_sell_rate | Sell-side cancellation rate | Cancelled sells / new sells |
| flow_cancel_imbalance | Cancellation imbalance | (cancelled buys − cancelled sells) / (cancelled buys + cancelled sells) |
| flow_cancel_depth_dist | Cancellation depth distribution | Mean relative distance of cancelled prices from the median new-order price |
| flow_cancel_lifetime | Order lifetime (s) | Approximated by the mean time interval between adjacent orders |
| flow_cancel_hft_ratio | HFT cancellation ratio | Cancellations with interval < 1 s / all cancellations |
| flow_cancel_spoof | Spoofing cancellations (count) | Number of cancellations whose size exceeds 5× the mean new-order size |
| flow_order_arrival_rate | Order arrival rate (orders/min) | Total orders / 240 minutes |
| flow_order_imbalance | Order imbalance | (new buy volume − new sell volume) / (sum of both) |
| flow_order_book_rebuild | Order-book rebuild | New orders per minute containing cancellations / total cancellations |
| flow_order_size_skew | Order size skewness | Skewness of new order volumes |
| flow_order_large_ratio | Large order ratio | Share of orders with volume > mean + 3σ |
| flow_cancel_price_deviation | Cancellation price deviation | Distance from cancelled price to mean new-order price, in units of 0.1% tick |
| flow_cancel_post_price_change | Post-cancellation price change (CNY) | Mean trade-price change in the 5 s window after a cancellation (vectorized searchsorted) |
| flow_order_type_diversity | Order type diversity | Shannon entropy of the order-type distribution |
| flow_order_quality | Order quality | (new orders − cancellations) / total orders |
| flow_order_duration_p50 | Order interval P50 (s) | Median interval between adjacent orders |
| flow_order_duration_p90 | Order interval P90 (s) | 90th percentile interval between adjacent orders |
| flow_cancel_cluster | Cancellation clustering | std / mean of per-minute cancellation counts |
| micro_zone_vol_ratio_T1 | Call auction volume share | 09:15–09:25 volume / whole-day volume (snapshots + ticks combined) |
| micro_zone_vol_ratio_T2 | Opening volume share | 09:30–10:00 volume / whole-day volume |
| micro_zone_vol_ratio_T3 | Late-morning volume share | 10:00–11:00 volume / whole-day volume |
| micro_zone_vol_ratio_T4 | Pre-lunch volume share | 11:00–11:30 volume / whole-day volume |
| micro_zone_vol_ratio_T5 | Early-afternoon volume share | 13:00–13:30 volume / whole-day volume |
| micro_zone_vol_ratio_T6 | Mid-afternoon volume share | 13:30–14:30 volume / whole-day volume |
| micro_zone_vol_ratio_T7 | Pre-close volume share | 14:30–14:57 volume / whole-day volume |
| micro_zone_vol_ratio_T8 | Closing auction volume share | 14:57–15:00 volume / whole-day volume |
| micro_zone_rv_ratio_AM | Morning volatility share | Morning 5-minute-bucket RV / whole-day RV |
| micro_zone_rv_ratio_PM | Afternoon volatility share | Afternoon 5-minute-bucket RV / whole-day RV |
| micro_zone_rv_ratio_open | Opening volatility share | Opening session RV / whole-day RV |
| micro_zone_rv_ratio_close | Closing volatility share | Closing session RV / whole-day RV |
| micro_zone_spread_ratio_open | Opening spread share | Opening mean qsp / whole-day mean qsp |
| micro_zone_spread_ratio_mid | Continuous-hours spread share | Continuous-session mean qsp / whole-day mean qsp |
| micro_zone_spread_ratio_close | Closing spread share | Closing mean qsp / whole-day mean qsp |
| micro_open_gap | Opening gap | (first snapshot price − previous close) / previous close |
| micro_call_auction_vol_ratio | Call auction volume ratio | 09:15–09:25 (incl. result snapshot) volume / whole-day volume |
| micro_close_auction_impact | Closing auction impact | (final price − second-to-last snapshot price) / second-to-last price |
| micro_close_squeeze | Closing squeeze | (last 10 trades' return − last 30 trades' return) / last 30 trades' return |
| micro_lunch_break_gap | Lunch-break gap | (first afternoon price − last morning price) / last morning price |
| micro_zone_vpin_open | Opening VPIN | 8-bucket VPIN during the opening session |
| micro_zone_imbalance_am | Morning imbalance | Morning amount (buy − sell) / (buy + sell) |
| micro_zone_imbalance_pm | Afternoon imbalance | Afternoon amount (buy − sell) / (buy + sell) |
| micro_zone_morning_evening_ratio | Morning/afternoon ratio | Morning amount / afternoon amount |
| micro_zone_distribution | Session distribution concentration | Shannon entropy of the volume distribution across the 8 sessions |
| micro_jump_count_1pct | Jump count (>1%) | Number of ticks with \|return\| > 1% |
| micro_jump_count_05pct | Jump count (>0.5%) | Number of ticks with \|return\| > 0.5% |
| micro_jump_size_mean | Mean jump size | Mean \|size\| of jumps > 0.5%; 0 when no jump |
| micro_jump_size_max | Max jump size | Max \|size\| of jumps > 0.5%; 0 when no jump |
| micro_jump_skew | Jump skewness | (up-jump count − down-jump count) / total, at a 0.5% threshold |
| micro_jump_recovery_time | Jump recovery time (s) | Mean seconds to return within ±0.1% after a jump, within the next 500 ticks |
| micro_jump_cluster | Jump clustering | Share of 5-minute buckets containing ≥ 2 jumps |
| micro_amihud_illiquidity | Amihud illiquidity | mean(\|log return\| / turnover value) × 1e6 |
| micro_kyle_lambda | Kyle's lambda | Absolute regression slope of Δprice on signed volume |
| micro_price_impact_large | Large-order price impact | Mean return over the 5 trades following trades > 500k CNY |
| micro_impact_elasticity | Price impact elasticity | Regression slope of \|Δprice\| on log(volume) |
| micro_impact_cost | Impact cost (CNY) | Mean spread + mean \|per-tick spread\| |
| micro_impact_permanent_ratio | Permanent impact ratio | \|spread\| 10 trades after a large trade / (after 10 + current), for amounts > 3× the mean |
| micro_impact_decay_half_life | Impact decay half-life (trades) | Mean number of trades for the impact to decay by half |
| micro_jump_flag | Jump flag | 1 if any jump > 0.5% occurred, else 0 |
| micro_trade_interval_mean | Mean trade interval (s) | Mean per-second difference between adjacent ticks |
| micro_trade_interval_median | Median trade interval (s) | Median per-second difference between adjacent ticks |
| micro_trade_interval_cv | Trade interval CV | std / mean of trade intervals |
| micro_trade_zero_interval_ratio | Zero-interval ratio | Ticks with interval = 0 / total intervals |
| micro_trade_arrival_rate | Trade arrival rate (trades/min) | Total trades / actual trading minutes |
| micro_trade_sign_autocorr | Trade sign autocorrelation | Lag-1 correlation of the trade direction series (±1) |
| micro_trade_sign_transition_BB | Buy→buy transition rate | Share of buys still followed by buys |
| micro_trade_sign_transition_EB | Sell→buy transition rate | Share of sells followed by buys |
| micro_trade_continuation | Trade continuation (trades) | Mean length of same-direction runs |
| micro_trade_seq_entropy | Trade sequence entropy | Shannon entropy over the BB/BE/EB/EE states |
| micro_trade_run_length | Longest same-direction run | Longest streak of consecutive same-direction trades |
| micro_trade_buy_pressure | Buy pressure (CNY) | Sum of buy amount × upward deviation above mid |
| micro_trade_sell_pressure | Sell pressure (CNY) | Sum of sell amount × downward deviation below mid |
| micro_liquidity_amihud_5 | Amihud illiquidity (last 5 buckets) | Mean Amihud over the last five 5-minute buckets × 1e6 |
| micro_liquidity_amihud_20 | Amihud illiquidity (last 20 buckets) | Same over the last 20 buckets (or all if fewer) |
| micro_liquidity_ratio | Liquidity ratio | Return volatility × level-1 depth / mean spread |
| micro_liquidity_turnover_decay | Turnover decay | (09:30–10:00 volume share) / 12.5% |
| micro_liquidity_volume_depth_ratio | Volume-to-depth ratio | Whole-day volume / mean level-1 quoted volume |
| micro_liquidity_volatility | Liquidity volatility | Std dev of bucketed Amihud |
| micro_liquidity_extreme | Liquidity extremes (count) | Buckets with Amihud > mean + 3σ |
| micro_liquidity_close | Closing liquidity | Mean bucketed Amihud during the closing session |
| micro_liquidity_roll | Roll spread estimate | 2 × √max(0, −first-order autocovariance), on a per-tick spread basis |
| micro_liquidity_cost | Liquidity cost (CNY) | Mean quoted spread + mean \|per-tick spread\| |
| micro_liquidity_twa | Time-weighted liquidity | Whole-day mean of bucketed Amihud |
| micro_liquidity_concentration | Liquidity concentration | Largest 5-minute bucket volume / whole-day volume |
| micro_liquidity_daily_pattern | Intraday liquidity pattern | (first 2 + last 2 buckets) / 2 / whole-day mean — a U-shape proxy |
| micro_liquidity_relative_spread | Relative quoted spread (%) | Mean of (ask1 − bid1) / mid × 100 |
| micro_liquidity_holden | Holden spread estimate | Mean of the available terms: median Amihud and mean spread |
| micro_vpin_100 | VPIN, 100 buckets | Mean buy/sell imbalance over 100 count-equal buckets |
| micro_vpin_kyle | Kyle VPIN | Mean signed-volume imbalance over 20 buckets |
| micro_vpin_quantile_ratio | VPIN quantile ratio | Share of the 50 buckets with VPIN > 0.5 |
| micro_vpin_tail_risk | VPIN tail risk | 95th percentile of the 50-bucket VPIN |
| micro_vpin_entropy | VPIN entropy | Normalized Shannon entropy of the 50-bucket VPIN |
| micro_adverse_selection | Adverse selection cost | Over the first 500 trades: mean \|trade price − mid 5 s later\| / trade price |
| micro_realized_spread | Realized spread | Mean of direction × (trade price − mid 5 s later) / trade price |
| micro_adverse_selection_cost | Adverse selection cost ratio | Adverse selection cost / mean relative spread |
| micro_spread_decomposition | Spread decomposition | \|realized spread\| / mean effective spread — the adverse-selection component |
| micro_information_share | Information share | corr(signed volume, price change 5 trades later) |
| micro_toxicity_persistence | Toxicity persistence | Lag-1 autocorrelation of the bucketed VPIN series |
| micro_vpin_hurst | VPIN Hurst exponent | R/S Hurst exponent of the bucketed VPIN series |
| micro_vpin_conviction | VPIN conviction | Runs persistently above the VPIN median / total buckets |
| micro_toxicity_asymmetry | Toxicity asymmetry | Mean bucketed buy ratio − mean sell ratio |
| micro_informed_ratio | Informed trading ratio | P90 large-order amount share × direction concentration |
| micro_liquidity_mining | Liquidity mining factor | Mean impact over the 5 trades after P90 large orders / large-order share |
| micro_vpin_bayesian | Bayesian VPIN | Posterior mean VPIN after shrinking toward a 0.5 prior |
| vol_turnover_total | Total volume (shares) | Sum of tick volumes; divide by free-float cap to get turnover |
| vol_turnover_open | Opening volume share | 09:30–10:00 volume / whole-day volume |
| vol_turnover_close | Closing volume share | 14:30–15:00 volume / whole-day volume |
| vol_turnover_am_pm | Morning/afternoon volume ratio | Morning volume / afternoon volume |
| vol_turnover_concentration | Turnover concentration | Largest of 10 count-equal buckets / mean bucket |
| vol_price_corr | Volume-price correlation | Correlation of minute-bucket volume with minute returns |
| vol_price_divergence | Volume-price divergence | Share of minutes where volume change and price change disagree in sign |
| vol_weighted_price | Price deviation from VWAP | (close − VWAP) / VWAP |
| vol_up_down_ratio | Up/down volume ratio | Up-tick volume / down-tick volume |
| vol_large_trade_intensity | Large-trade intensity | P90 large-order amount share / trade-count share |
| vol_tick_density | Tick density (ticks/s) | Total ticks / actual trading seconds |
| vol_skew | Volume skewness | Skewness of minute-bucket volume |
| vol_kurtosis | Volume kurtosis | Kurtosis of minute-bucket volume |
| vol_gini | Volume Gini coefficient | Inequality of minute-bucket volume, 0–1 |
| vol_persistence | Volume persistence | Lag-1 autocorrelation of minute-bucket volume |

## 4. `l1_l2_factors` (329 columns)

`symbol, date` + backward-adjusted OHLCV + 110 L1 + 211 L2, inner-joined on `(symbol, date)`,
OHLCV taken from the L1 side. **This is the table to use for multi-factor modelling.**

<details>
<summary>Show all 329 column names</summary>

**All 329 columns**

```
symbol, date, open, high, low, close, volume, amount, turn_1, turn_3, turn_5, turn_10, turn_20,
turn_60, turn_std_20, turn_z_20, turn_ratio_1_5, turn_ratio_1_20, turn_trend_5_20, turn_acc_5,
turn_acc_20, turn_breakout_20, turn_hl_pos_20, turn_high_days_10, amt_log, amt_ma_5, amt_ma_20,
amt_ma_60, amt_z_20, amt_ratio_1_5, amt_ratio_1_20, amt_ratio_5_20, amt_skew_20, amt_close_pos,
amt_net_flow_5, amt_net_flow_20, amt_up_ratio_5, amt_up_ratio_20, amt_high_days_10, mfi_14,
obv_slope_20, amt_vol_ratio_20, mom_ret_1d, mom_ret_3d, mom_ret_5d, mom_ret_10d, mom_ret_20d,
mom_ret_60d, mom_ma_gap_5, mom_ma_gap_20, mom_macd_dif, mom_macd_dea, mom_macd_hist, mom_rsi_6,
mom_rsi_14, mom_kdj_k, mom_kdj_d, mom_kdj_j, vol_std_5, vol_std_10, vol_std_20, vol_atr_14,
vol_parkinson_20, vol_gk_20, vol_amp_1, vol_amp_20, tech_bb_width, tech_bb_pos, tech_cci_20,
tech_adx_14, tech_close_to_high_20, tech_max_drawdown_20, fun_float_mv, fun_total_mv,
fun_mv_rank, fun_pe, fun_pb, fun_bp, fun_ep, fun_value_zscore, fun_roe, fun_peg, fun_np_growth,
chip_profit_ratio_20, chip_profit_ratio_60, chip_concentration_20, chip_floating_ratio,
chip_cost_90_width, chip_profit_delta_5, style_beta_20, style_beta_60, style_idio_vol_20,
style_idio_vol_60, style_residual_ret_20, ind_ret_5, ind_ret_10, ind_ret_20,
ind_rotation_speed_20, ind_strength_20, ind_dispersion_20, ind_breadth_up_20,
ind_volume_ratio_20, ind_crowding_20, ind_netflow_rank_20, ind_relative_pe, ind_concentration,
ind_relative_momentum_20, ind_momentum_decay, concept_hot_score, concept_momentum_top3,
concept_exposure_top1, concept_rotation_score, concept_crowding_max, concept_diversity,
concept_flow_rank, concept_leader_score, concept_cross_sector, concept_volume_ratio,
micro_qsp_equal, micro_qsp_time, micro_qsp_volume, micro_esp_equal, micro_esp_time,
micro_esp_volume, micro_aqsp, micro_aesp, micro_qsp_vol_20, micro_esp_vol_20,
micro_spread_slope_1_5, micro_spread_slope_5_10, micro_spread_slope_1_10,
micro_bid_ask_vol_ratio, micro_spread_recovery, micro_spread_open, micro_spread_mid,
micro_spread_close, micro_spread_vol_ratio, micro_spread_U_shape, vol_realized_rv,
vol_realized_rrv, vol_realized_rskew, vol_realized_rkurt, vol_realized_5min, vol_realized_10min,
vol_realized_15min, vol_realized_am, vol_realized_pm, vol_realized_opening,
vol_realized_closing, vol_realized_jump, vol_realized_sjv, flow_net_amount, flow_buy_amount,
flow_sell_amount, flow_net_ratio, flow_buy_ratio, flow_sell_ratio, flow_super_net,
flow_large_net, flow_medium_net, flow_small_net, flow_large_ratio, flow_medium_ratio,
flow_small_ratio, flow_large_pct, flow_imbalance_volume, flow_imbalance_open,
flow_imbalance_mid_am, flow_imbalance_mid_pm, flow_imbalance_close, flow_imbalance_autocorr,
flow_imbalance_revert_speed, flow_consistency, flow_money_flow_index, flow_big_trade_ratio,
micro_vpin_8, micro_vpin_20, micro_vpin_50, micro_vpin_ma_5, micro_vpin_ma_20,
micro_vpin_delta_5, micro_vpin_zscore_20, micro_pin, micro_order_flow_toxicity,
micro_volume_sync, micro_vpin_surge, micro_vpin_trend, micro_vpin_vol_ratio,
micro_vpin_amount_ratio, micro_depth_bid, micro_depth_ask, micro_depth_ratio_1,
micro_depth_ratio_5, micro_depth_ratio_10, micro_depth_weighted_bid, micro_depth_weighted_ask,
micro_depth_concentration_bid, micro_depth_concentration_ask, micro_depth_slope,
micro_depth_effective, micro_depth_recovery, micro_depth_open, micro_depth_mid,
micro_depth_close, micro_depth_volatility, micro_depth_extreme, micro_depth_imbalance_1,
micro_depth_imbalance_5, micro_depth_imbalance_10, flow_cancel_rate, flow_cancel_buy_rate,
flow_cancel_sell_rate, flow_cancel_imbalance, flow_cancel_depth_dist, flow_cancel_lifetime,
flow_cancel_hft_ratio, flow_cancel_spoof, flow_order_arrival_rate, flow_order_imbalance,
flow_order_book_rebuild, flow_order_size_skew, flow_order_large_ratio,
flow_cancel_price_deviation, flow_cancel_post_price_change, flow_order_type_diversity,
flow_order_quality, flow_order_duration_p50, flow_order_duration_p90, flow_cancel_cluster,
micro_zone_vol_ratio_T1, micro_zone_vol_ratio_T2, micro_zone_vol_ratio_T3,
micro_zone_vol_ratio_T4, micro_zone_vol_ratio_T5, micro_zone_vol_ratio_T6,
micro_zone_vol_ratio_T7, micro_zone_vol_ratio_T8, micro_zone_rv_ratio_AM,
micro_zone_rv_ratio_PM, micro_zone_rv_ratio_open, micro_zone_rv_ratio_close,
micro_zone_spread_ratio_open, micro_zone_spread_ratio_mid, micro_zone_spread_ratio_close,
micro_open_gap, micro_call_auction_vol_ratio, micro_close_auction_impact, micro_close_squeeze,
micro_lunch_break_gap, micro_zone_vpin_open, micro_zone_imbalance_am, micro_zone_imbalance_pm,
micro_zone_morning_evening_ratio, micro_zone_distribution, micro_jump_count_1pct,
micro_jump_count_05pct, micro_jump_size_mean, micro_jump_size_max, micro_jump_skew,
micro_jump_recovery_time, micro_jump_cluster, micro_amihud_illiquidity, micro_kyle_lambda,
micro_price_impact_large, micro_impact_elasticity, micro_impact_cost,
micro_impact_permanent_ratio, micro_impact_decay_half_life, micro_jump_flag,
micro_trade_interval_mean, micro_trade_interval_median, micro_trade_interval_cv,
micro_trade_zero_interval_ratio, micro_trade_arrival_rate, micro_trade_sign_autocorr,
micro_trade_sign_transition_BB, micro_trade_sign_transition_EB, micro_trade_continuation,
micro_trade_seq_entropy, micro_trade_run_length, micro_trade_buy_pressure,
micro_trade_sell_pressure, micro_liquidity_amihud_5, micro_liquidity_amihud_20,
micro_liquidity_ratio, micro_liquidity_turnover_decay, micro_liquidity_volume_depth_ratio,
micro_liquidity_volatility, micro_liquidity_extreme, micro_liquidity_close,
micro_liquidity_roll, micro_liquidity_cost, micro_liquidity_twa, micro_liquidity_concentration,
micro_liquidity_daily_pattern, micro_liquidity_relative_spread, micro_liquidity_holden,
micro_vpin_100, micro_vpin_kyle, micro_vpin_quantile_ratio, micro_vpin_tail_risk,
micro_vpin_entropy, micro_adverse_selection, micro_realized_spread,
micro_adverse_selection_cost, micro_spread_decomposition, micro_information_share,
micro_toxicity_persistence, micro_vpin_hurst, micro_vpin_conviction, micro_toxicity_asymmetry,
micro_informed_ratio, micro_liquidity_mining, micro_vpin_bayesian, vol_turnover_total,
vol_turnover_open, vol_turnover_close, vol_turnover_am_pm, vol_turnover_concentration,
vol_price_corr, vol_price_divergence, vol_weighted_price, vol_up_down_ratio,
vol_large_trade_intensity, vol_tick_density, vol_skew, vol_kurtosis, vol_gini, vol_persistence
```

</details>

## Known data conditions (measured)

1. **Two factors stopped updating**: `ind_netflow_rank_20` and `concept_flow_rank` are 100% null and constant
   on 2026-09-24, though healthy on 2026-06-01. **Filter by column null rate at runtime, not by hard-coded name.**
2. **High-null L2 factors**: `micro_jump_skew` ~54%, `micro_jump_recovery_time` ~48%,
   `micro_price_impact_large` ~24%; other `micro_*`/`flow_*` columns are near-zero median null.
3. **`fun_pe` extremes**: measured single-day range -1206 to +1069. Prefer `fun_ep`/`fun_bp`, or winsorize.
4. **Amount columns in `micro_*`/`flow_*` are in 10k CNY**; the remaining ratio-type factors are dimensionless.
5. **The 2018 boundary**: L2 and the merged table have no earlier data; watch for structural breaks in
   cross-boundary training windows.

## A minimal modelling skeleton

```python
import glob, pandas as pd

paths = sorted(glob.glob("quantdb/6_ml_datasets/l1_l2_factors/dt=202[3-6]*/data.parquet"))
df = pd.concat([pd.read_parquet(p) for p in paths], ignore_index=True)

# Drop factors that stopped being populated, by null rate rather than by name
null_rate = df.isna().mean()
dead = null_rate[null_rate > 0.30].index.tolist()

keys = {"symbol", "date", "time", "open", "high", "low", "close", "volume", "amount"}
feat = [c for c in df.columns
        if c not in keys and c not in dead and not c.startswith("future_return_")]
print(f"{len(feat)} features, {len(dead)} dropped as mostly-null: {dead[:5]}")
```

> `l1_l2_factors` **carries no label column of its own**. Pull `future_return_*` from `features_daily` or
> `5_technical_derived/technical_indicators` and align on `(symbol, time/date)`.

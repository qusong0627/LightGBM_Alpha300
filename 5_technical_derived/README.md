# `5_technical_derived/` — 衍生指标

三张**按交易日分区**的派生指标表。列数不大，体积很小，很适合整表全量下载。

```
5_technical_derived/
├── technical_indicators/dt=YYYYMMDD/data.parquet   # 35 列，1.12 MB/日
├── valuation/dt=YYYYMMDD/data.parquet              # 16 列，0.46 MB/日
└── market_sentiment/dt=YYYYMMDD/data.parquet       # 17 列，0.52 MB/日
```

## 实测体积与单日截面

| 子目录 | 列数 | 分区数 | 最新分区 | 单日 | 一年（估） |
|---|---:|---:|---|---:|---:|
| `technical_indicators` | 35 | 2608 | 20260924 | 1.12 MB | ~273 MB |
| `valuation` | 16 | 2608 | 20260924 | 0.46 MB | ~112 MB |
| `market_sentiment` | 17 | 2608 | 20260924 | 0.52 MB | ~127 MB |

三张表分区进度一致，都跟日线齐平（2608 个分区，最新 `dt=20260924`）。

> ⚠️ **这三张表的单日截面是 5570 行，含 347 只北交所（`.BJ`）标的**，和日线表一致；
> 但 `6_ml_datasets/` 下的因子表（`l1_factors` 5208 行、`l1_l2_factors` 5196 行）**不含 `.BJ`**。
> 用 `symbol` 把估值/情绪/技术指标与因子表拼接时，北交所那部分标的必然落空——按因子表为主表做内连接即可。

## 下载

```bash
# 三张表全量（合计约 0.5 GB，最省事的方案）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "5_technical_derived/*" --local_dir ./quantdb

# 只要估值表的历史
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "5_technical_derived/valuation/dt=202*/*" --local_dir ./quantdb
```

## 计算口径

- **`technical_indicators` 基于后复权（`daily_backward`）序列计算。**
- **`valuation` 与 `market_sentiment` 基于不复权真实价计算。**

三张表首列都是 `symbol, time, close`，但两个 `close` 不是同一口径——横向拼接时不要拿它们互相校验。

> ⚠️ **本表 `close` 是后复权口径。**
> 实测方法：取 2026-09-24 的 300 只标的，把 `technical_indicators.close` 分别与三套日线的 `close` 比对——
> 与 **`daily_backward` 300/300 完全相等，平均相对偏差 0.000**；与 `daily_unadjusted` 仅 37/300 相等
> （即无除权历史的标的），与 `daily_forward` 同样只有部分相等。
> 参考例：`000001.SZ` 当日不复权/前复权 11.30，后复权 18.647061，本表 `close` = 18.647061。

## 1. `technical_indicators`（35 列，实测列名）

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

| 字段族 | 列 | 说明 |
|---|---|---|
| 均线 | `ma5/10/20/60`、`ma_gap_5/10/20` | 收盘价均线及偏离 |
| 振荡 | `rsi_6`、`rsi_14`、`kdj_k/d/j` | RSI、KDJ |
| 趋势 | `macd_dif`、`macd_dea`、`macd_hist` | MACD 三线 |
| 波动率 | `vol_std_5/20/60`、`vol_atr_14` | 收益标准差、真实波幅 |
| 量能 | `vol_to_ma5`、`vol_to_ma20`、`volume_ma_3`、`amount_ma_5`、`volume_trend_3d` | 量比、均量 |
| 收益 | `pct_change`、`beta_20` | 当日涨跌幅、20 日 beta |
| **标签** | `future_return_1d ~ 60d` | **未来 N 日收益，是 ML 标签，不是特征** |

口径摘录：

<!-- technical: 7 行 -->
| 字段名称 | 数据类型 | 计算口径与指标说明 |
|---|---|---|
| close | float | 收盘价（后复权，实测） |
| ma5 / ma10 / ma20 / ma60 | float | 5/10/20/60 日简单移动平均线 |
| rsi_6 / rsi_14 | float | 相对强弱指标 RSI（主流 SMA 平滑口径） |
| kdj_k / kdj_d / kdj_j | float | 随机指标 KDJ（SMA(RSV,3,1)） |
| macd_dif / macd_dea / macd_hist | float | MACD 指标（hist = 2×(DIF-DEA)） |
| vol_atr_14 | float | 14日 ATR（平均真实波幅，Wilder RMA） |
| beta_20 | float | 20日 CAPM Beta 敏感度（相对沪深 300） |

> **⚠️ 标签泄漏**：`future_return_*` 混在特征表里，做 LightGBM 时如果按「除 symbol/time 外全当特征」自动生成特征列，
> 会把 6 个未来收益直接喂进模型，回测指标会假性极好。**必须显式排除。**

## 2. `valuation`（16 列，实测列名）

```
symbol, time, close, total_capital, circulating_capital, total_mv, float_mv,
net_profit_ttm, revenue_ttm, equity, annual_net_profit,
pe_ttm, pe_static, pb, ps_ttm, dividend_rate
```

| 列 | 单位 | 说明 |
|---|---|---|
| `total_capital` / `circulating_capital` | 股 | 总股本 / 流通股本 |
| `total_mv` / `float_mv` | 元 | 总市值 / 流通市值 |
| `net_profit_ttm` / `revenue_ttm` / `equity` / `annual_net_profit` | 元 | TTM 净利润 / 营收 / 净资产 / 年报净利润 |
| `pe_ttm` / `pe_static` / `pb` / `ps_ttm` / `dividend_rate` | 倍 / % | 估值比率 |

口径摘录：

<!-- valuation: 6 行 -->
| 字段名称 | 数据类型 | 计算公式与说明 |
|---|---|---|
| total_mv / float_mv | float | 总市值 / 流通市值（总股本 × 收盘价） |
| net_profit_ttm | float | 归母净利润 TTM（近 4 季度单季差分滚动求和） |
| pe_ttm | float | 滚动市盈率 TTM（total_mv / net_profit_ttm） |
| pb | float | 市净率 PB（total_mv / 归母权益） |
| ps_ttm | float | 市销率 PS TTM（total_mv / revenue_ttm） |
| dividend_rate | float | 近 12 个月累计股息率（每股分红 / 收盘价） |

> 估值分母可能为负或极小，`pe_ttm` / `ps_ttm` 会出现异常大值与负值（对照 [`6_ml_datasets`](../6_ml_datasets/README.md) 里 `fun_pe` 实测单日跨度 -1206 ~ +1069）。
> 做因子暴露前请 winsorize 或取倒数（EP/BP）。

## 3. `market_sentiment`（17 列，实测列名）

```
symbol, time, close, price_range, upper_shadow, lower_shadow, body_ratio,
amount_per_trade, liquidity_score, intraday_vol, gap_up_down,
buy_pressure, sell_pressure, momentum_1d, momentum_3d, am_pm_trend, volume_concentration
```

| 列 | 说明 |
|---|---|
| `price_range` / `upper_shadow` / `lower_shadow` / `body_ratio` | 振幅、上下影线、实体占比（K 线形态） |
| `amount_per_trade` | **元/股**，当日隐含均价（成交额/成交量） |
| `liquidity_score` / `intraday_vol` | 流动性打分、日内波动 |
| `gap_up_down` | 跳空幅度 |
| `buy_pressure` / `sell_pressure` | 买卖压力（盘口委托口径） |
| `momentum_1d` / `momentum_3d` / `am_pm_trend` | 短期动量、上下午趋势 |
| `volume_concentration` | 成交集中度 |

> 上表列名取自实际 Parquet，含义按列名推断；`liquidity_score`、`buy_pressure`、`sell_pressure`
> 这类合成指标**没有公开公式可查**，用前建议自行反推验证。

## 读取

```python
import pandas as pd

tech = pd.read_parquet("quantdb/5_technical_derived/technical_indicators/dt=20260924/data.parquet")
val  = pd.read_parquet("quantdb/5_technical_derived/valuation/dt=20260924/data.parquet")

feat = tech.drop(columns=[c for c in tech.columns if c.startswith("future_return_")])
label = tech[["symbol", "time", "future_return_5d"]]
```

按日分区拼接同 [`1_kline_data`](../1_kline_data/README.md)：`glob("*/dt=*/data.parquet")` + `concat`，
分区信息在文件内的 `time` 列上，不必额外解析目录名。

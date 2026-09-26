# `1_kline_data/` — Daily OHLCV

Root of the quote data. **Partitioned by trading day**: each partition holds one `data.parquet`
containing the whole-market cross-section for that day.

Layout: [中文](README.md) · [back to top](../README.en.md)

## Structure

```
1_kline_data/
├── daily_unadjusted/dt=YYYYMMDD/data.parquet   # unadjusted
├── daily_forward/dt=YYYYMMDD/data.parquet      # proportional forward adjustment
├── daily_backward/dt=YYYYMMDD/data.parquet     # proportional backward adjustment
└── index_daily/dt=YYYYMMDD/data.parquet        # index bars (broad-based / style / industry / theme)
```

## Measured size

| Sub-directory | Partitions | Since | Per day | Latest | Columns |
|---|---:|---|---:|---|---:|
| `daily_unadjusted` | 2608 | 2016-01-04 | 0.12 MB | 20260924 | 8 |
| `daily_forward` | 2608 | 2016-01-04 | 0.13 MB | 20260924 | 8 |
| `daily_backward` | 2608 | 2016-01-04 | 0.22 MB | 20260924 | 8 |
| `index_daily` | 2608 | 2016-01-04 | <0.01 MB | 20260924 | 9 |

Each daily cross-section is **5,570 rows** (measured 2026-09-24, including 347 Beijing Stock Exchange `.BJ`
symbols). The whole quote directory for one year is ~115 MB — by far the cheapest place to start.

## Download

```bash
# One year of daily bars (pick one adjustment scheme; backward is the usual backtest choice)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "1_kline_data/daily_backward/dt=2026*/*" --local_dir ./quantdb

# Index bars only, for benchmarks
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "1_kline_data/index_daily/*" --local_dir ./quantdb
```

## Fields

<!-- kline_en: 17 行 | 译自 quantdb.cn §1 日线 K 线数据 -->
| Field | Type | Description | Example |
|---|---|---|---|
| trade_date | datetime | Trading date (Beijing time 00:00:00) | 2026-07-22 |
| open | float | Open price (CNY) | 1500.00 |
| high | float | High price (CNY) | 1520.50 |
| low | float | Low price (CNY) | 1495.00 |
| close | float | Close price (CNY) | 1510.00 |
| volume | int | Volume (shares; identical across the three adjustment schemes) | 3200000 |
| amount | float | Turnover value (10k CNY) | 48320.00 |
| trade_date (source column `time`) | datetime | Trading date |  |
| open / high / low / close | float | Index level OHLC |  |
| volume | int | Volume (shares, normalized) |  |
| amount | float | Turnover value (10k CNY) |  |
| IndexCode | string | Index code (e.g. 000300.SH) |  |
| Category | string | Index category (broad-based / style / industry / theme) |  |

> Excerpted from the [field spec](https://www.quantdb.cn/docs/fields.html); the shipped files use
> lowercase `symbol` / `time` throughout, as measured below.

Actual columns as shipped (measured):

```
symbol, time, open, high, low, close, volume, amount
```

`index_daily` adds one more column, `Category` (broad-based / style / industry / theme).

> All three daily tables ship `symbol, time, open, high, low, close, volume, amount`;
> `index_daily` has those 8 plus `Category`.

## Units

| Field | Unit | Notes |
|---|---|---|
| open / high / low / close | CNY | unadjusted = raw; forward anchored at the latest day; backward anchored at the first day |
| volume | shares | **indices are in shares too, not lots** |
| amount | 10k CNY | turnover value |

## Reading

```python
import pandas as pd
df = pd.read_parquet("quantdb/1_kline_data/daily_unadjusted/dt=20260924/data.parquet")
df.head()
```

A single partition file has no `dt` column — the date lives in both the directory name and the `time`
column. When concatenating a range, `time` is already sufficient; add `dt` only if you want to key off
the directory:

```python
import glob, pandas as pd
frames = []
for p in sorted(glob.glob("quantdb/1_kline_data/daily_backward/dt=*/data.parquet")):
    d = pd.read_parquet(p)
    d["dt"] = p.split("dt=")[1].split("/")[0]
    frames.append(d)
kline = pd.concat(frames, ignore_index=True)
```

## Caveats

- Forward adjustment is anchored to the **latest** trading day, so every weekly sync shifts the entire
  forward-adjusted history. Use `daily_backward` when you need stable historical series.
- `volume` is identical across all three schemes; only prices are rescaled.
- Suspended stocks simply do not appear in that day's cross-section, so row counts vary by day.

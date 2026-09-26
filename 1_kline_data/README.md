# `1_kline_data/` — 日线行情

K 线数据根目录。**按交易日分区**，每个分区一个 `data.parquet`，内容是当日全市场截面。

## 目录布局

```
1_kline_data/
├── daily_unadjusted/dt=YYYYMMDD/data.parquet   # 不复权日线
├── daily_forward/dt=YYYYMMDD/data.parquet      # 等比前复权日线
├── daily_backward/dt=YYYYMMDD/data.parquet     # 等比后复权日线
└── index_daily/dt=YYYYMMDD/data.parquet        # 指数日线（宽基/风格/行业/主题）
```

## 实测体积

| 子目录 | 分区数 | 时间跨度 | 单日大小 | 最新分区 | 列数 |
|---|---:|---|---:|---|---:|
| `daily_unadjusted` | 2608 | 2016-01-04 起 | 0.12 MB | 20260924 | 8 |
| `daily_forward` | 2608 | 2016-01-04 起 | 0.13 MB | 20260924 | 8 |
| `daily_backward` | 2608 | 2016-01-04 起 | 0.22 MB | 20260924 | 8 |
| `index_daily` | 2608 | 2016-01-04 起 | <0.01 MB | 20260924 | 9 |

三套日线单日均约 5200 行。日线全套（三套复权 + 指数）一年约 115 MB，是最值得优先下载的部分。

## 下载

```bash
# 一年日线（三套里挑一套，通常选后复权做回测）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "1_kline_data/daily_backward/dt=2026*/*" --local_dir ./quantdb

# 只要指数日线做基准
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "1_kline_data/index_daily/*" --local_dir ./quantdb
```

## 字段说明

<!-- kline: 17 行 -->
| 字段名称 | 数据类型 | 字段说明 | 示例值 |
|---|---|---|---|
| trade_date | datetime | 交易日日期（北京时间 00:00:00） | 2026-07-22 |
| open | float | 开盘价（元） | 1500.00 |
| high | float | 最高价（元） | 1520.50 |
| low | float | 最低价（元） | 1495.00 |
| close | float | 收盘价（元） | 1510.00 |
| volume | int | 成交量（股，前/后/不复权三套相同） | 3200000 |
| amount | float | 成交额（万元） | 48320.00 |
| trade_date（原始列 time） | datetime | 交易日 |  |
| open / high / low / close | float | 指数点位开高低收 |  |
| volume | int | 成交量（股，已归一化） |  |
| amount | float | 成交金额（万元） |  |
| IndexCode | string | 指数代码（如 000300.SH） |  |
| Category | string | 指数类别（宽基/风格/行业/主题） |  |

> 上表摘自[官网字段规范](https://www.quantdb.cn/docs/fields.html)。注意两点差异：
> 官网 K 线表用 `trade_date` 命名，**实际数据列名是 `time`**；官网 `Symbol` 在实际数据里是小写 `symbol`。

实际列清单（`daily_*`，实测）：

```
symbol, time, open, high, low, close, volume, amount
```

`index_daily` 额外多一列 `Category`（指数分类标签：宽基 / 风格 / 行业 / 主题）。

## 单位与口径

| 字段 | 单位 | 说明 |
|---|---|---|
| open / high / low / close | 元 | 不复权为原始价；前复权锚定最新日；后复权锚定首日 |
| volume | 股 | **指数同样是股，不是手** |
| amount | 万元 | 成交额 |

## 读取

```python
import pandas as pd
df = pd.read_parquet("quantdb/1_kline_data/daily_unadjusted/dt=20260924/data.parquet")
df.head()
```

拼接区间做基准收益时，注意 `dt=` 目录名不会被 pandas 自动识别为列（单文件内没有 `dt` 列），
需自己补上或用 DuckDB `hive_partitioning=true`：

```python
import glob, pandas as pd
frames = []
for p in sorted(glob.glob("quantdb/1_kline_data/daily_backward/dt=*/data.parquet")):
    d = pd.read_parquet(p)
    d["dt"] = p.split("dt=")[1].split("/")[0]   # 手动补分区列
    frames.append(d)
kline = pd.concat(frames, ignore_index=True)
```

## 注意

- 前复权锚定的是**数据最新交易日**，因此每次同步后全部历史的前复权价都会平移；需要稳定历史序列请用 `daily_backward`。
- 三套复权之间的 `volume` 完全一致，只有价格平移。
- 停牌日不会出现在当日截面里，因此各分区的行数会波动。

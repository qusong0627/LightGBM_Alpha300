# QuantDB — 中国 A 股量化数据集（300+ 因子）

[![Dataset on ModelScope](https://img.shields.io/badge/Dataset-ModelScope-blue)](https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300)
[![License Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-green)](LICENSE)
[![Fields spec](https://img.shields.io/badge/%E5%AD%97%E6%AE%B5%E8%A7%84%E8%8C%83-quantdb.cn-lightgrey)](https://www.quantdb.cn/docs/fields.html)

覆盖沪深全市场（主板 / 创业板 / 科创板）约 5500 只标的、2016 年至今的 A 股量化研究数据集：
日线三套复权行情、指数、财务三大报表、融资融券、技术指标、估值与盘口情绪，
以及 **L1 因子 110 维 / L2 高频因子 211 维 / 合并宽表 329 列**，可直接用于多因子建模与 LightGBM 训练。

> **数据本体托管在魔搭社区**：<https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300>
> 本仓库**不含数据文件**（56 GB / 约 7.4 万个 Parquet）。仓库目录与数据目录一一对应，
> 每个目录下的 `README.md` 说明该部分的数据布局、实测体积和字段口径。

---

## 数据集导航

| 目录 | 内容 | 分区方式 | 实测体积 | 文档 |
|---|---|---|---:|---|
| `1_kline_data/` | 不复权 / 前复权 / 后复权日线、指数日线 | 按交易日，2608 区 | 0.12–0.22 MB/日 | [README](1_kline_data/README.md) |
| `2_base_sector/` | 融资融券、合约资料、行业概念、交易日历、指数权重 | 混合 | 3.2 MB（合约表） | [README](2_base_sector/README.md) |
| `3_financial_data/` | 资产负债 / 利润 / 现金流 / 股本 / 股东户数 / 每股指标 / 分红 | 逐股文件 | 5559 只/表 | [README](3_financial_data/README.md) |
| `4_bond_etf/` | 可转债要素、ETF 申赎清单 PCF | 快照单文件 | ~1.1 MB | [README](4_bond_etf/README.md) |
| `5_technical_derived/` | 技术指标 35 列、估值 16 列、盘口情绪 17 列 | 按交易日 | 0.45–1.12 MB/日 | [README](5_technical_derived/README.md) |
| `6_ml_datasets/` | `features_daily` 78 / `l1_factors` 119 / `l2_factors` 219 / `l1_l2_factors` 329 | 按交易日 | 1.74–14.39 MB/日 | [README](6_ml_datasets/README.md) |

## 快速下载

前置：`pip install modelscope pandas pyarrow`（modelscope ≥ 1.40，命令已实测）

```bash
# 全量（约 56 GB，请先确认磁盘与带宽）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --local_dir ./quantdb --max-workers 8
```

**先看样本再决定下什么**（27 张表 × 300 行，共约 3.1 MB，几秒完成）：

```bash
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "preview/*" --local_dir ./quantdb-preview
```

**只下一张表 / 一段时间**（下载粒度 = 单个交易日分区）：

```bash
# 某一张因子表
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_factors/*" --local_dir ./quantdb

# 2026 年的 L1+L2 合并宽表
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/dt=2026*/*" --local_dir ./quantdb

# 单个具体文件（路径里含 dt= 也照常工作）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  "1_kline_data/daily_unadjusted/dt=20260924/data.parquet" --local_dir ./quantdb
```

Python 等价写法：

```python
from modelscope import snapshot_download

path = snapshot_download(
    repo_id="qusong0627/LightGBM_Alpha300",
    repo_type="dataset",
    local_dir="./quantdb",
    allow_patterns=["6_ml_datasets/l1_factors/dt=2026*/*"],
)
```

> ⚠️ 参数是 `--repo-type dataset`，**不是** `--dataset`；网上常见的 `--dataset` 写法在当前版本会直接报错。

## 该下多少：实测体积

取各表自身最新分区实测，一年按约 244 个交易日估算：

| 数据集 | 单日 | 一年（估） | 分区数 | 最新分区 |
|---|---:|---:|---:|---|
| `1_kline_data/daily_unadjusted` | 0.12 MB | ~29 MB | 2608 | 20260924 |
| `1_kline_data/daily_forward` | 0.13 MB | ~32 MB | 2608 | 20260924 |
| `1_kline_data/daily_backward` | 0.22 MB | ~54 MB | 2608 | 20260924 |
| `5_technical_derived/technical_indicators` | 1.12 MB | ~273 MB | 2608 | 20260924 |
| `5_technical_derived/valuation` | 0.45 MB | ~110 MB | **2604** | **20260918** |
| `5_technical_derived/market_sentiment` | 0.52 MB | ~127 MB | **2604** | **20260918** |
| `6_ml_datasets/features_daily` | 1.74 MB | ~0.42 GB | 2608 | 20260924 |
| `6_ml_datasets/l1_factors` | 3.69 MB | ~0.90 GB | 2608 | 20260924 |
| `6_ml_datasets/l2_factors` | 9.92 MB | ~2.42 GB | 2120 | 20260924 |
| `6_ml_datasets/l1_l2_factors` | 14.39 MB | ~3.51 GB | 2120 | 20260924 |
| `preview/*` | — | 合计 3.1 MB | — | — |
| **全仓库** | — | **~56 GB** | 2608 交易日 | 20260924 |

单日截面约 5200 只标的；`l1_l2_factors` 在 2026-09-24 为 5196 行 × 329 列 ≈ 14 MB。

## 全局约定

### 股票代码格式

<!-- code-format: 3 行 -->
| 交易市场 | 后缀 | 示例代码 |
|---|---|---|
| 上海证券交易所 | .SH | 600000.SH , 688000.SH |
| 深圳证券交易所 | .SZ | 000001.SZ , 300001.SZ |
| 北京证券交易所 | .BJ | 830000.BJ , 430000.BJ |

### 复权口径

<!-- adj-types: 3 行 -->
| 复权类型 | API 参数 | 算法说明 | 典型应用场景 |
|---|---|---|---|
| 不复权 | unadjusted | 原始真实成交价，包含除权除息跳空缺口 | 真实交易成本分析、分红收益回算 |
| 前复权 | forward | 以最新价格为基准，历史价格向下平移调整 | 量化回测、技术指标计算（最常用） |
| 后复权 | backward | 以 2016 年第一个交易日为基准（本地历史起点），最新价格向上平移调整 | 长期真实复利收益、累计历史涨幅评估 |

补充两点实现细节：

- `daily_forward` 为**等比**前复权（以最新交易日收盘定标，历史恒为正）；`daily_backward` 为**等比**后复权（锚定窗口首日，新除权事件不会使历史漂移）。
- **技术指标基于后复权序列计算，估值与情绪指标基于不复权真实价计算。** 因子表（`l1_factors` 及之后）内的 OHLCV 一律为**后复权**价——例如 2026-09-24 的 `000001.SZ`，不复权收盘 11.30 元，因子表里是 18.65 元。跨表拼接时务必确认口径。

### 单位口径

<!-- units: 12 行 -->
| 数据类型 | 单位 | 口径说明 |
|---|---|---|
| 价格（open/high/low/close） | 元（人民币） | 精确到分/厘 |
| 日线成交量（daily_*/features/L1/技术衍生） | 股 | volume 为股，三种复权量额相同 |
| 日线成交额（daily_*/features/L1） | 万元 | amount 为万元 |
| 分钟线（min1/min5）成交量 | 股 | volume 为股（已归一化） |
| 分钟线（min1/min5）成交额 | 万元 | amount 为万元（已归一化） |
| ETF 日线（etf_kline） | 股 / 万元 | volume 为股，amount 为万元（已归一化） |
| 指数日线（index_daily） | 股 / 万元 | volume 为股，amount 为万元，均价=amount*1e4/volume |
| Tick 累计成交量 / 盘口挂单量 | 股 | volume/askVol/bidVol/tickvol 为股（已归一化；历史 V1 分片由后端换算） |
| Tick 累计成交额 | 万元 | amount 为万元（已归一化；历史 V1 分片由后端换算） |
| 市值（total_mv / float_mv） | 元 | 市值 = 股本 × 收盘价 |
| 股本（total_capital） | 股 | 总股本 / 流通股本 |
| 财务报告数据 | 元 | 标准会计报表口径，除特别标注外 |

## 读取方式

目录名本身就是 `dt=` 分区，除 pandas 外可直接交给 DuckDB / pyarrow 做分区裁剪：

```python
import pandas as pd

kline = pd.read_parquet("quantdb/1_kline_data/daily_unadjusted/dt=20260924/data.parquet")
# 列: symbol, time, open, high, low, close, volume, amount
```

```python
import glob
frames = [pd.read_parquet(p) for p in sorted(glob.glob(
    "quantdb/6_ml_datasets/l1_factors/dt=2026*/data.parquet"))]
train = pd.concat(frames, ignore_index=True)   # 实测 4 个分区 = 20832 行 × 119 列
```

```python
import duckdb
df = duckdb.sql("""
  SELECT symbol, close, mom_ret_20d, micro_vpin_20
  FROM read_parquet('quantdb/6_ml_datasets/l1_l2_factors/dt=*/data.parquet',
                    hive_partitioning=true)
  WHERE dt >= 20260101
""").df()
```

## 已知数据状况

以下均为在真实分区上**实测**得到、官网字段文档未覆盖的信息：

1. **两个因子已断更**：`ind_netflow_rank_20` 与 `concept_flow_rank` 在 2026-09-24 全表 100% 为空（退化为常数列），但 2026-06-01 仍正常——上游行业/概念资金流在某个时点停止供给。**建议按「列空值率阈值」动态剔除，不要写死列名。**
2. **`valuation` 与 `market_sentiment` 滞后**：这两张表只同步到 `dt=20260918`（2604 分区），其余表到 `20260924`（2608 分区）。按「所有表都有最新日」写死日期会读到空结果。
3. **`features_daily` 的标签列名与官网文档不一致**：官网 `fields.html` 记作 `return_1d` ~ `return_60d`，**实际数据列名是 `future_return_1d` ~ `future_return_60d`**。
4. **命名大小写不一致**：官网文档部分表写 `Symbol` / `trade_date`，实际数据统一是 `symbol` / `time`（日线、估值、情绪、技术指标表）与 `date`（因子表）。
5. **L2 仅 2018 年起**（合并宽表同理，2120 个交易日）：训练窗口跨 2018 年时注意样本结构突变。
6. **高缺失 L2 因子**：`micro_jump_skew` 约 54% 缺失、`micro_jump_recovery_time` 约 48%、`micro_price_impact_large` 约 24%；其余 `micro_*`/`flow_*` 列中位缺失率接近 0。
7. **估值因子有极端值**：`fun_pe` 实测单日跨度 -1206 ~ +1069（含负 PE 与极小分母）。LightGBM 按秩分裂受影响较小，做中性化或线性模型时务必 winsorize / 分位数化。
8. **⚠️ 特征与标签分离**：`future_return_*` 是未来 N 日收益，属于 ML **标签**，不是特征。把它们留在 `feature_name` 里就是直接标签泄漏。

## 官网字段文档覆盖情况

字段口径以 <https://www.quantdb.cn/docs/fields.html> 为准，本仓库各目录 README 的字段表即摘录自该页。经与实际 Parquet 列逐一核对：

| 数据集 | 官网记录字段数 | 实际列数 | 覆盖情况 |
|---|---:|---:|---|
| `l1_factors` | 110 | 119 | **110 个因子全部对得上**，未记录的 9 列是 `symbol/date/time` + OHLCV 主键 |
| `l2_factors` | 211 | 219 | **211 个因子全部对得上**，未记录的 8 列同上 |
| `features_daily` | 48 | 78 | 命名有出入（`return_*` vs `future_return_*`），另有 37 列未记录（行业/地区/主营/涨跌停/动态估值等属性列） |
| `technical_indicators` | 15 | 35 | 文档用合并行（如 `ma5 / ma10 / ma20 / ma60`），20 列未单列；**且 `close` 被标成「前复权」，实测是后复权** |
| `valuation` | 7 | 16 | 9 列未记录 |
| `margin_trading` | — | 10 | 官网列了数据中不存在的 `slo_sell_amount`，实际有的 `finance_repay`/`slo_repay`/`slo_sell_volume` 未记 |
| `market_sentiment` | **0** | 17 | **官网完全没有这一节**，各合成指标无官方口径可查 |
| K 线 | 10 | 8 | 文档含 `trade_date`/`IndexCode` 等命名，实际为 `time`/`symbol` |

> **本项目文档的口径原则：以数据实测为准。** 官网 `quantdb.cn/docs/fields.html` 与魔搭 README
> 只要和真实 Parquet 冲突，一律按数据写，并在对应位置标注上游的错误（见各目录 README 的警告块），
> 而不是照抄文档。上面的复权口径判定就是用 300 只标的逐一比对三套日线得出的。

> 官网另描述了 `min1_kline` / `min5_kline` / `etf_kline` / Tick 分级行情等内容，**这些不在本魔搭数据集快照中**；本仓库的目录文档只写实际能下到的东西。

## 许可与引用

- 数据许可：**Apache License 2.0**（见 [LICENSE](LICENSE)）
- 仅供量化研究与学习使用，**不构成任何投资建议**
- 原始行情数据版权归交易所及上游数据服务商所有，请勿将数据用于商业再分发
- 使用者应自行核验数据准确性，本数据集不对因数据错误或遗漏造成的损失承担责任

数据获取与更新以魔搭数据集页为准：<https://www.modelscope.cn/datasets/qusong0627/LightGBM_Alpha300>

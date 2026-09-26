# `2_base_sector/` — 基础与板块数据

全数据集的**元数据基准**：合约资料、行业与概念成分、交易日历、指数权重、融资融券明细。
建议**最先下载**——后续所有分析都要靠它们做标的过滤、日期对齐和分组聚合。快照类文件总共只有约 3.6 MB。

## 目录布局与实测体积

```
2_base_sector/
├── margin_trading/                  # 融资融券，按交易日分区
│   └── dt=YYYYMMDD/data.parquet     #   2607 个分区，0.10 MB/日，最新 dt=20260923
├── instrument_detail/
│   └── instrument_list.parquet      #   3.2 MB — 5576 行 × 152 列
├── sector_concept/
│   └── sector_members.parquet       #   0.2 MB — 79447 行 × 4 列（长表）
├── trading_calendar/
│   └── trading_days.parquet         #   2 列：TradingDate + IsTradingDay
└── index_weights/                   #   按指数分文件，共 7 个
    000016.SH.parquet  000300.SH.parquet  000688.SH.parquet  000852.SH.parquet
    000905.SH.parquet  000906.SH.parquet  399006.SZ.parquet
```

## 各数据实测详情

### `index_weights/` — 指数成分权重

列：`Symbol, Weight, IndexCode`；**`Weight` 是百分数**（同指数合计 ≈ 100，不是 1）。经成分数逐一核对：

| 文件 | 成分数 | 权重合计 | 指数 |
|---|---:|---:|---|
| `000016.SH` | 50 | 100.00 | 上证50 |
| `000300.SH` | 300 | 100.00 | 沪深300 |
| `000688.SH` | 50 | 100.00 | 科创50 |
| `000905.SH` | 500 | 99.99 | 中证500 |
| `000906.SH` | 800 | 99.99 | 中证800 |
| `000852.SH` | 1000 | 99.98 | 中证1000 |
| `399006.SZ` | 100 | 99.98 | 创业板指 |

> 这是**当期快照**，不含历史权重序列，因此不能用它做严格的历史成分还原（回测时会有幸存者偏差）。

### `instrument_detail/instrument_list.parquet`

**5576 行 × 152 列**，是一张大宽快照，不是精简合约表。实测标记类字段计数：

| 字段 | 含义 | 命中数 |
|---|---|---:|
| `BelongHS300` | 属于沪深300 | 300 |
| `BelongRZRQ` | 融资融券标的 | 3881 |
| `BelongHSGT` | 沪深港通标的 | 3532 |
| `BelongHasKQZ` | 含可转债 | 314 |
| `IsSTGP` | ST 股 | 204 |
| `IsQuitGP` | 退市风险 | **0** |
| `IsHKGP` | 港股 | **0** |

> ⚠️ **这些标记列的 dtype 是 object，取值为字符串 `'0'` / `'1'`，不是整数。**
> 写 `inst.IsSTGP == 0` 会**一行都匹配不到**、得到空集；必须写 `== "0"`，或先
> `inst[cols] = inst[cols].astype(int)`。全表 5576 行。

其余 152 列还包含 `Name`、`MainBusiness`（主营）、`rs_hyname`（行业名）、`tdx_dyname`（地域名）、
`IPO_Price`、`J_start`（上市日期）、`ZTPrice`/`DTPrice`（涨/跌停价）、`DynaPE`/`PB_MRQ`/`StaticPE_TTM`（估值）、
`EverZTCount`/`YearZTDay`（涨停统计）、`StaffNum`（员工数）、`ReportDate`（最近报告期）等。
**字段名基本是通达信风格的原生命名，官网字段文档只覆盖了其中约 8 个。**

### `sector_concept/sector_members.parquet`

长表，79447 行 × 4 列（`SectorCode, SectorName, SectorType, Symbol`）。`SectorType` 实测 5 类：

| SectorType | 板块数 | 行数 | 板块名示例 |
|---|---:|---:|---|
| 概念板块 | 421 | 68290 | 5G概念、国防军工、稀缺资源、含H股 |
| 地区板块 | 32 | 5576 | 黑龙江、新疆板块、吉林板块 |
| 行业板块(二级) | 80 | 4143 | 一般零售、电子商务、贸易 |
| 行业板块(一级) | 48 | 1433 | 煤炭开采、油气开采、石油化工 |
| 风格板块 | 2 | 5 | 轮动趋势、板块趋势 |

一只股票会出现在多个板块，聚合时**必须按 `Symbol` 去重**，否则样本会被重复计数放大。

### `trading_calendar/trading_days.parquet`

`TradingDate`（`YYYYMMDD` **字符串**，非日期类型）+ `IsTradingDay`（int）。字符串格式与分区目录名 `dt=20260924` 可直接对齐。

### `margin_trading/` — 融资融券

按日分区，2607 个分区，T+1 披露，最新到 `dt=20260923`（**比日线晚一个交易日**）。实测 10 列：

```
symbol, time, finance_balance, slo_volume, finance_buy, slo_sell_volume,
finance_repay, slo_repay, finance_net, slo_net
```

| 字段组 | 单位 | 说明 |
|---|---|---|
| `finance_*` | 万元 | 融资余额 / 买入 / 偿还 / 净买入 |
| `slo_*` | 股 | 融券余量 / 卖出 / 偿还 / 净卖出 |

恒等式可用于数据自检：

```
finance_net      = finance_buy - finance_repay
slo_sell_volume  = slo_net + slo_repay
```

## 与官网字段文档的差异（实测）

字段规范见 <https://www.quantdb.cn/docs/fields.html>，官网 `base` 段摘录如下：

<!-- base: 21 行 -->
| 字段名称 | 数据类型 | 字段说明 |
|---|---|---|
| Symbol | string | 股票代码（如 600519.SH） |
| Name | string | 证券简称 |
| J_zgb | float | 总股本（万股） |
| ActiveCapital | float | 流通股本（万股） |
| ListDate | date | 上市日期（YYYY-MM-DD） |
| EstablishDate | date | 成立日期 |
| Province | string | 所属省份 |
| SecurityType | string | 证券类型 |
| BelongRZRQ | string | 融资融券标的标识（1=是） |
| SectorCode | string | 板块代码（如 880201.SH） |
| SectorName | string | 板块名称 |
| SectorType | string | 板块分类（行业/概念/风格/地区） |
| Symbol | string | 成分个股代码 |
| trade_date | date | 交易日期 |
| Symbol | string | 股票代码 |
| finance_balance | float | 融资余额 |
| slo_volume | float | 融券余量 |
| finance_buy | float | 融资买入额 |
| slo_sell_amount | float | 融券卖出额 |
| finance_net | float | 融资净买入 |
| slo_net | float | 融券净卖出 |

对照实际数据：

- 官网列了 `slo_sell_amount`，**实际数据没有这一列**；反过来实际有的 `finance_repay`、`slo_repay`、`slo_sell_volume` 官网未单列。
- 官网写 `Symbol` / `trade_date`，实际 `margin_trading` 里是小写 `symbol` / `time`。
- 官网把合约资料（`Name`/`ListDate`/`Province`/`EstablishDate`/`ActiveCapital`/`SecurityType`/`BelongRZRQ`/`J_zgb`）
  和板块成分（`SectorCode`/`SectorName`/`SectorType`）**写在同一张表里描述**，实际它们分属
  `instrument_detail` 与 `sector_concept` 两个不同文件。

## 下载

```bash
# 元数据快照（建议无条件先下，总共约 3.6 MB）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "2_base_sector/instrument_detail/*" \
  --include "2_base_sector/sector_concept/*" \
  --include "2_base_sector/trading_calendar/*" \
  --include "2_base_sector/index_weights/*" --local_dir ./quantdb

# 融资融券按年取
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "2_base_sector/margin_trading/dt=2026*/*" --local_dir ./quantdb
```

## 读取

```python
import pandas as pd

inst = pd.read_parquet("quantdb/2_base_sector/instrument_detail/instrument_list.parquet")
sect = pd.read_parquet("quantdb/2_base_sector/sector_concept/sector_members.parquet")
cal  = pd.read_parquet("quantdb/2_base_sector/trading_calendar/trading_days.parquet")
w300 = pd.read_parquet("quantdb/2_base_sector/index_weights/000300.SH.parquet")
marg = pd.read_parquet("quantdb/2_base_sector/margin_trading/dt=20260923/data.parquet")

# 可交易池：剔除 ST、退市风险，只留融资融券标的
# 注意标记列是字符串 '0'/'1'，不要用整数比较
pool = inst.loc[(inst.IsSTGP == "0") & (inst.IsQuitGP == "0") & (inst.BelongRZRQ == "1"), "Symbol"]

# 某概念板块成分
concept = sect.loc[sect.SectorName == "5G概念", "Symbol"]
```

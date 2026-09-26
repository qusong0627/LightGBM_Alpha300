# `3_financial_data/` — 财务数据

**逐股文件**布局：每只标的一个 `data.parquet`，文件名 `{Symbol}.parquet`，内含该标的自 2016 年以来的全部历史（按报告期）。

```
3_financial_data/
├── balance/          资产负债表
├── income/           利润表
├── cashflow/         现金流量表
├── capital/          股本结构
├── holder_num/       股东户数
├── per_share/        每股指标（TTM）
├── pershare_index/   每股财务指标与比率
└── dividend_factors/ 分红送配复权因子
```

## 实测：列数与体积

下表列数为逐一读取真实文件核对的结果（600519.SH、000001.SZ、300750.SZ 三只抽样与 `preview/` 样本完全一致）。
魔搭数据集 README 已于 2026-09-26 对齐，两边数字一致：

| 表 | 列数 | 单只样本体积 | 报告期数 | 文件数 |
|---|---:|---:|---:|---:|
| `balance` | 75 | 60.8 KB | 42 | 5559 |
| `income` | 41 | 36.7 KB | 42 | 5559 |
| `cashflow` | 68 | 59.0 KB | 42 | — |
| `capital` | 10 | 7.7 KB | 50 | — |
| `holder_num` | 38 | 28.4 KB | 42 | — |
| `per_share` | 10 | 9.6 KB | 42 | — |
| `pershare_index` | 99 | 86.3 KB | 42 | — |
| `dividend_factors` | 7 | 5.0 KB | 30 事件 | — |

单只标的 8 张表合计约 **290 KB**；全量 8 × 5559 ≈ 4.4 万个文件，粗估 1.5 GB 量级（按抽样体积外推，非实测总量）。

## 公共列

除 `dividend_factors` 外，各表开头都是：

| 列 | 类型 | 说明 |
|---|---|---|
| `m_timetag` | object（`YYYYMMDD` **字符串**） | 报告期，抽样范围 20160331 ~ 20260630 |
| `m_anntime` | object（`YYYYMMDD` 字符串） | **公告日**，抽样最晚 20260815 |
| `Symbol` | object | 带市场后缀代码，如 `600519.SH` |

> **`m_anntime` 是 point-in-time 回测的关键列。** 财务数据必须按公告日生效，不能用报告期直接对齐——
> 否则会在 4 月就用上 3 月末才披露的年报，造成未来函数。拼接行情时正确做法：
> `merge_asof(kline.sort_values('time'), fin.sort_values('m_anntime'), left_on='time', right_on='m_anntime')`。
> `m_timetag`/`m_anntime` 都是字符串，比较前需先转日期。

`holder_num` 额外有 `declareDate`、`endDate` 两列且列序不同（`declareDate, endDate, Symbol, m_timetag, ...`）。

## 字段说明

资产负债表、利润表与每股指标的官方口径（摘自 <https://www.quantdb.cn/docs/fields.html>）：

<!-- financial: 30 行 -->
| 字段名称 | 数据类型 | 会计科目说明 |
|---|---|---|
| m_timetag | date | 报告期截至日（YYYY-MM-DD） |
| m_anntime | date | 财报实际对外公告日 |
| cash_equivalents | float | 货币资金 |
| account_receivable | float | 应收账款 |
| total_current_assets | float | 流动资产合计 |
| fix_assets | float | 固定资产 |
| tot_assets | float | 资产总计 |
| shortterm_loan | float | 短期借款 |
| accounts_payable | float | 应付账款 |
| total_current_liability | float | 流动负债合计 |
| long_term_loans | float | 长期借款 |
| tot_liab | float | 负债合计 |
| undistributed_profit | float | 未分配利润 |
| tot_shrhldr_eqy_excl_min_int | float | 归属于母公司股东权益合计 |
| total_equity | float | 所有者权益合计 |
| m_timetag / m_anntime | date | 报告期 / 公告日 |
| revenue | float | 营业总收入 |
| total_operating_cost | float | 营业总成本 |
| cost_of_goods_sold | float | 营业成本 |
| research_expenses | float | 研发费用 |
| oper_profit | float | 营业利润 |
| tot_profit | float | 利润总额 |
| net_profit_excl_min_int_inc | float | 归属于母公司所有者的净利润 |
| s_fa_eps_basic | float | 基本每股收益（EPS） |
| s_fa_bps | float | 每股净资产（BPS） |
| s_fa_eps_basic | float | 基本每股收益（EPS） |
| du_return_on_equity | float | 净资产收益率 ROE（摊薄） |
| du_profit_rate | float | 销售净利率 |
| inc_revenue_rate | float | 营业收入同比增长率（%） |
| inc_net_profit_rate | float | 净利润同比增长率（%） |

其余各表的实际列名可直接从样本读出，例如 `income` 前几列为
`m_timetag, m_anntime, revenue, operating_revenue, total_operating_cost, cost_of_goods_sold, ...`；
`pershare_index` 用的是 `s_fa_*` 前缀的通达信风格原生字段名（`s_fa_ocfps`、`s_fa_bps`、`s_fa_eps_basic`、`s_fa_eps_diluted` …），
**这类字段名官网基本没有逐一解释，是使用时最大的口径盲区。**

### `dividend_factors`（7 列）

```
time, type, interest, allotPrice, stockBonus, allotment, Symbol
```

每 10 股口径的分红送配记录，用于复权因子合成。

| 列 | 说明 |
|---|---|
| `time` | 除权除息日 |
| `type` | 分配类型 |
| `interest` | 每股现金分红（每 10 股口径） |
| `allotPrice` | 配股价 |
| `stockBonus` | 送股比例 |
| `allotment` | 配股比例 |

## 下载

逐股文件数量大（每表 5559 个文件）。**按标的下**比全量划算得多：

```bash
# 只要某几只股票的全套财务
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  "3_financial_data/income/600519.SH.parquet" \
  "3_financial_data/balance/600519.SH.parquet" \
  --local_dir ./quantdb

# 只要一张表的全市场（注意是 5559 个文件）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "3_financial_data/income/*" --local_dir ./quantdb --max-workers 16

# 只要沪深300成分股的利润表：先用 glob 批量脚本按 Symbol 列表拉单文件
```

## 读取

```python
import pandas as pd, glob

# 单只标的
income = pd.read_parquet("quantdb/3_financial_data/income/600519.SH.parquet")
income["m_anntime"] = pd.to_datetime(income["m_anntime"], format="%Y%m%d")
income["m_timetag"] = pd.to_datetime(income["m_timetag"], format="%Y%m%d")

# 全市场拼一张利润表面板（5559 个文件，注意耗时）
frames = [pd.read_parquet(p) for p in glob.glob("quantdb/3_financial_data/income/*.parquet")]
panel = pd.concat(frames, ignore_index=True)
```

## 注意

- **数据按报告期披露更新**，最新到 2026 年中报（抽样 `m_timetag` 最大值 20260630）。
- 每只股票的报告期数不同（次新股、退市股、`capital` 有 50 期而 `dividend_factors` 只有 30 个事件），拼接后不要假设等长面板。
- `holder_num` 的披露频率与其他表不一致，做股东户数因子时注意对齐公告日而不是报告期。
- 金额字段单位为**元**，每股字段为**元/股**。

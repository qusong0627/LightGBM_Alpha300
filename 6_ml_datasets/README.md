# `6_ml_datasets/` — 机器学习数据集（300+ 因子）

本目录是**建模主表**：四张按交易日分区的宽表，每个分区是当日全市场截面（约 5200 行）。
L1/L2 因子共 321 个，官网对每一个都给了中文标签与口径说明——本目录文档已完整摘录。

```
6_ml_datasets/
├── features_daily/dt=YYYYMMDD/data.parquet      #   78 列，1.74 MB/日，2608 区
├── l1_factors/dt=YYYYMMDD/data.parquet          #  119 列，3.69 MB/日，2608 区
├── l2_factors/dt=YYYYMMDD/data.parquet          #  219 列，9.92 MB/日，2120 区（2018 起）
└── l1_l2_factors/dt=YYYYMMDD/data.parquet       #  329 列，14.39 MB/日，2120 区（2018 起）
```

## 实测体积

| 表 | 列数 | 分区数 | 单日 | 一年（估） | 全历史（估） |
|---|---:|---:|---:|---:|---:|
| `features_daily` | 78 | 2608 | 1.74 MB | ~0.42 GB | ~4.5 GB |
| `l1_factors` | 119 | 2608 | 3.69 MB | ~0.90 GB | ~9.6 GB |
| `l2_factors` | 219 | 2120 | 9.92 MB | ~2.42 GB | ~21 GB |
| `l1_l2_factors` | 329 | 2120 | 14.39 MB | ~3.51 GB | ~30.5 GB |

单日截面行数：`l1_factors` 5208 行，`l1_l2_factors` 5196 行（2026-09-24 实测，内连接后略少）。

## 该下哪一张

- **只要日线特征 + 标签做单因子测试** → `features_daily`
- **多因子建模（2016 年起）** → `l1_factors`
- **要微观结构 / 资金流因子（2018 年起）** → `l1_l2_factors`（已合并，不必分别下 l1 和 l2）
- **只要 L2 原始高频因子** → `l2_factors`

## 下载

```bash
# 先看样本（27 张表 × 300 行，3.1 MB）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "preview/*" --local_dir ./quantdb-preview

# 一年 L1 因子（约 0.9 GB）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_factors/dt=2026*/*" --local_dir ./quantdb

# 2018 年起的合并宽表（约 30 GB，建模主力）
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/*" --local_dir ./quantdb --max-workers 16

# 只要训练窗口
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "6_ml_datasets/l1_l2_factors/dt=202[3-6]*/*" --local_dir ./quantdb
```

Python 侧：

```python
from modelscope import snapshot_download
snapshot_download(
    repo_id="qusong0627/LightGBM_Alpha300", repo_type="dataset",
    local_dir="./quantdb",
    allow_patterns=["6_ml_datasets/l1_l2_factors/dt=2026*/*"],
)
```

## 主键与价格口径

| 表 | 主键列 | 价格列 |
|---|---|---|
| `features_daily` | `symbol, time` | `close` |
| `l1_factors` / `l2_factors` / `l1_l2_factors` | `symbol, date`（另有 `time`） | 后复权 `open/high/low/close/volume/amount` |

> **因子表里的 OHLCV 是后复权价**，不是真实成交价。实测 2026-09-24 的 `000001.SZ`：
> 不复权收盘 11.30 元，`l1_factors.close` 是 18.65 元。做价格类展示或涨跌停判断时不要用它。

## 1. `features_daily`（78 列）

日线特征宽表 = 技术指标 + 估值 + 情绪 + 标的属性 + 6 个未来收益标签。完整列名：

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

官方口径摘录（**注意命名差异，见下方警告**）：

<!-- factors_features: 45 行 -->
| 字段名称 | 中文标签 | 口径说明 |
|---|---|---|
| time / Symbol | 交易日 / 股票代码 | 行索引， {6位数字}.{SH/SZ/BJ} |
| close | 收盘价(元，后复权) | 后复权收盘，全部技术指标的基准序列 |
| ma5 | MA5 | 后复权收盘 N 日简单均线 |
| ma10 | MA10 |  |
| ma20 | MA20 |  |
| ma60 | MA60 |  |
| ma_gap_5 | MA乖离5(%) | (close-MA)/MA×100 |
| ma_gap_10 | MA乖离10(%) |  |
| ma_gap_20 | MA乖离20(%) |  |
| rsi_6 / rsi_14 | RSI6 / RSI14 | 主流 SMA 口径：avg_gain/avg_loss 用 ewm(alpha=1/N)；纯涨=100、平盘=50、纯跌=0、首行=NaN |
| kdj_k | KDJ-K | RSV=(C-LLV(L,9))/(HHV(H,9)-LLV(L,9))×100；K/D=SMA(RSV,3,1)=ewm(com=2)；J=3K-2D；无波动窗口 RSV=50 |
| kdj_d | KDJ-D |  |
| kdj_j | KDJ-J |  |
| macd_dif | MACD-DIF | DIF=EMA12-EMA26，DEA=DIF 的 EMA9，柱=2×(DIF-DEA) |
| macd_dea | MACD-DEA |  |
| macd_hist | MACD柱 |  |
| vol_std_5 | 5日波动率 | 日收益百分比序列 N 日滚动标准差 |
| vol_std_20 | 20日波动率 |  |
| vol_std_60 | 60日波动率 |  |
| vol_atr_14 | 14日ATR | 真实波幅 TR 的 Wilder RMA（ewm alpha=1/14） |
| vol_to_ma5 | 量比MA5 | 当日量/N日均量（含当日），非盘口量比 |
| vol_to_ma20 | 量比MA20 |  |
| volume_ma_3 | 3日均量(股) | 成交量 3 日滚动均值 |
| amount_ma_5 | 5日均额(万元) | 成交额 5 日滚动均值 |
| volume_trend_3d | 3日量能趋势 | 成交量环比变化率的 3 日均值 |
| return_1d | 未来1日收益（标签）(%) | (close.shift(-d)/close-1)×100，基于不复权真实收盘价（反映真实可交易收益，除权窗口不失真） |
| return_3d | 未来3日收益（标签）(%) |  |
| return_5d | 未来5日收益（标签）(%) |  |
| return_10d | 未来10日收益（标签）(%) |  |
| return_20d | 未来20日收益（标签）(%) |  |
| return_60d | 未来60日收益（标签）(%) |  |
| pct_change | 当日涨跌幅(%) | 当日收盘相对前收的涨跌幅 |
| beta_20 | 20日Beta | CAPM Beta，相对沪深 300（000300.SH），基于不复权收益 |
| total_capital | 总股本(股) | 公告日对齐的最新股本（市值口径用未复权真实价×股本） |
| circulating_capital | 流通股本(股) |  |
| total_mv / float_mv | 总市值 / 流通市值(元) | 总股本×收盘 / 流通股本×收盘 |
| net_profit_ttm | 归母净利TTM(元) | 归母净利润单季差分后 4 季滚动和 |
| revenue_ttm | 营收TTM(元) | 营业收入单季差分后 4 季滚动和 |
| equity | 归母权益(元) | 归属于母公司股东权益合计（公告日） |
| annual_net_profit | 年报归母净利(元) | 最近年报归母净利润（静态 PE 分母） |
| pe_ttm | PE(TTM) | total_mv / net_profit_ttm |
| pe_static | PE(静态) | total_mv / annual_net_profit |
| pb | PB | total_mv / equity |
| ps_ttm | PS(TTM) | total_mv / revenue_ttm |
| dividend_rate | 股息率 | 近 12 个月每股累计分红 / 收盘价 |

> ⚠️ **官网文档与实际数据不一致（实测）**：
> - 官网把标签写成 `return_1d` ~ `return_60d`，**实际列名是 `future_return_1d` ~ `future_return_60d`**。
> - 官网写 `Symbol`，实际是小写 `symbol`。
> - 官网只记了 48 个字段，实际 78 列——**37 列未记录**，主要是标的属性列
>   （`industry_code/name`、`region_area_code/name`、`main_business`、`list_date`、`is_st`、`is_hsgt`、`is_margin`、
>   `in_hs300`、`is_kcb_creatable`、`is_quit_risk`、`is_hk`）和通达信风格的行情衍生列
>   （`zt_price`、`dt_price`、`seal_strength`、`hs_turnover`、`zaf`、`beta_now`、`dyna_pe`、`static_pe_ttm`、
>   `div_yield`、`pb_mrq`、`ever_zt_count`、`year_zt_days`、`total_cap_yi`、`float_mv_yi`、`free_float_shares`、`ipo_price`）。

> **⚠️ `future_return_*` 是标签，不是特征。** 6 个未来 N 日收益列和特征同表，
> 自动生成特征列表（「除主键外全部」）会直接标签泄漏。

## 2. `l1_factors`（119 列 = 8 主键 + 110 L1 因子 + `time`）

主键与价格：`symbol, date, time, open, high, low, close, volume, amount`（后复权）。
110 个日频因子按前缀组织：

| 前缀 | 数量 | 类别 |
|---|---:|---|
| `turn_*` | 16 | 换手率 |
| `amt_*` | 16 | 成交额 |
| `mom_*` | 16 | 动量 |
| `ind_*` | 14 | 行业 |
| `fun_*` | 11 | 财务估值 |
| `concept_*` | 10 | 概念板块 |
| `vol_*` | 8 | 波动率 |
| `tech_*` | 6 | 技术形态 |
| `chip_*` | 6 | 筹码分布 |
| `style_*` | 5 | 风格 |
| `mfi_*` + `obv_*` | 2 | 经典资金流 |

### 110 个 L1 因子完整口径（摘自官网，与实际列逐一核对：110/110 全部对得上）

<!-- factors_l1: 110 行 -->
| 字段名称 | 中文标签 | 口径说明 |
|---|---|---|
| turn_1 | 1日换手率 | 当日换手率 |
| turn_3 | 3日换手率 | 换手率 N 日滚动均值 |
| turn_5 | 5日换手率 |  |
| turn_10 | 10日换手率 |  |
| turn_20 | 20日换手率 |  |
| turn_60 | 60日换手率 |  |
| turn_std_20 | 20日换手率标准差 | 换手率 20 日滚动标准差 |
| turn_z_20 | 20日换手率Z分数 | (当日-20日均)/20日标准差 |
| turn_ratio_1_5 | 1/5日换手率比 | 当日/5日均 |
| turn_ratio_1_20 | 1/20日换手率比 | 当日/20日均 |
| turn_trend_5_20 | 5/20日换手趋势 | 5日均/20日均 |
| turn_acc_5 | 5日换手加速度 | 换手率 5 日滚动求和 |
| turn_acc_20 | 20日换手加速度 | 换手率 20 日滚动求和 |
| turn_breakout_20 | 20日换手率突破 | 当日/20日最高换手率 |
| turn_hl_pos_20 | 20日换手率位置 | (当日-20日最低)/(20日最高-最低) |
| turn_high_days_10 | 10日换手高点天数 | 近10日（不含当日）换手率超20日均的天数 |
| amt_log | 对数成交额 | log(当日成交额) |
| amt_ma_5 | 5日均额对数 | log(成交额 N 日均值) |
| amt_ma_20 | 20日均额对数 |  |
| amt_ma_60 | 60日均额对数 |  |
| amt_z_20 | 20日成交额Z分数 | (对数额-20日均)/20日标准差 |
| amt_ratio_1_5 | 1/5日额比 | 当日/5日均 |
| amt_ratio_1_20 | 1/20日额比 | 当日/20日均 |
| amt_ratio_5_20 | 5/20日额比 | 5日均/20日均 |
| amt_skew_20 | 20日成交额偏度 | 成交额 20 日滚动偏度 |
| amt_close_pos | 成交额收盘位置 | (收盘-最低)/(最高-最低)，日内收盘相对位置 |
| amt_net_flow_5 | 5日资金净流入 | (OBV-OBV.shift(N))/N日累计量，负值=净流出 |
| amt_net_flow_20 | 20日资金净流入 |  |
| amt_up_ratio_5 | 5日放量上涨占比 | 上涨日成交额/N日总成交额 |
| amt_up_ratio_20 | 20日放量上涨占比 |  |
| amt_high_days_10 | 10日放量高点天数 | 近10日（不含当日）成交额超20日均的天数 |
| mfi_14 | 14日资金流量指标 | 典型价格×量的 14 日 MFI |
| obv_slope_20 | 20日能量潮斜率 | OBV 20 日最小二乘斜率/20日均量 |
| amt_vol_ratio_20 | 20日量额比 | 日收益与成交量 20 日滚动相关系数 |
| mom_ret_1d | 1日收益率 | close/close.shift(N)-1，过去 N 日收益（与 8.1 未来收益标签方向相反） |
| mom_ret_3d | 3日收益率 |  |
| mom_ret_5d | 5日收益率 |  |
| mom_ret_10d | 10日收益率 |  |
| mom_ret_20d | 20日收益率 |  |
| mom_ret_60d | 60日收益率 |  |
| mom_ma_gap_5 | MA5乖离率 | close/SMA(close,N)-1 |
| mom_ma_gap_20 | MA20乖离率 |  |
| mom_macd_dif | MACD快线 | 与 8.1 同口径（EMA12-26，DEA=EMA9，柱=2×差） |
| mom_macd_dea | MACD慢线 |  |
| mom_macd_hist | MACD柱体 |  |
| mom_rsi_6 | 6日RSI | 与 8.1 同口径（主流 ewm 平滑） |
| mom_rsi_14 | 14日RSI |  |
| mom_kdj_k | KDJ-K | 与 8.1 同口径（9 日 RSV，SMA(3,1)） |
| mom_kdj_d | KDJ-D |  |
| mom_kdj_j | KDJ-J |  |
| vol_std_5 | 5日收益标准差 | 后复权日收益 N 日滚动标准差 |
| vol_std_10 | 10日收益标准差 |  |
| vol_std_20 | 20日收益标准差 |  |
| vol_atr_14 | 14日平均真实波幅 | TR 的 ATR(14)，与 8.1 同口径 |
| vol_parkinson_20 | 20日Parkinson波动率 | 基于高低价的 Parkinson 估计，20 日窗口 |
| vol_gk_20 | 20日GarmanKlass波动率 | 基于 OHLC 的 Garman-Klass 估计，20 日窗口 |
| vol_amp_1 | 1日振幅 | (high-low)/close |
| vol_amp_20 | 20日平均振幅 | 振幅 20 日滚动均值 |
| tech_bb_width | 布林带宽度 | Bollinger(20,2σ) 上下轨宽度 |
| tech_bb_pos | 布林带位置 | 收盘在上下轨间的相对位置 |
| tech_cci_20 | 20日顺势指标 | CCI(20) |
| tech_adx_14 | 14日平均趋向指标 | ADX(14)，趋势强度 |
| tech_close_to_high_20 | 20日收盘距高点 | 1-close/rolling_max(high,20) |
| tech_max_drawdown_20 | 20日最大回撤 | 1-close/rolling_max(close,20) |
| fun_float_mv | 对数流通市值 | log(真实价×流通股本) |
| fun_total_mv | 对数总市值 | log(真实价×总股本) |
| fun_mv_rank | 市值截面百分位 | 当日流通市值截面 percentile rank |
| fun_pe | 市盈率 PE_TTM | 总市值/归母净利TTM |
| fun_pb | 市净率 PB | 总市值/归母权益 |
| fun_bp | 账面市值比 BP | 1/PB |
| fun_ep | 盈利收益率 EP_TTM | 归母净利TTM/总市值 |
| fun_value_zscore | 综合估值Z分数 | (zscore(EP)+zscore(BP))/2，每日截面 |
| fun_roe | 净资产收益率 ROE(%) | 归母净利TTM/归母权益×100 |
| fun_peg | PEG估值增长比 | PE/净利润同比增长率 |
| fun_np_growth | 净利润同比增长率 | TTM 净利润同比（income 自算） |
| chip_profit_ratio_20 | 20日筹码获利比例 | 成本低于现价的筹码占比（对应窗口） |
| chip_profit_ratio_60 | 60日筹码获利比例 |  |
| chip_concentration_20 | 20日筹码集中度 | (P95-P5)/现价，成本分布宽度相对值 |
| chip_floating_ratio | 5日浮动筹码占比 | 近5日筹码/总筹码 |
| chip_cost_90_width | 90%筹码成本宽度(元) | P95-P5 绝对宽度 |
| chip_profit_delta_5 | 5日筹码获利增量 | 获利比例20(t)-获利比例20(t-5) |
| style_beta_20 | 20日中证500Beta | 滚动 Cov(个股,基准)/Var(基准) |
| style_beta_60 | 60日中证500Beta |  |
| style_idio_vol_20 | 20日特质波动率 | CAPM 残差（r-beta×rb）N 日滚动标准差 |
| style_idio_vol_60 | 60日特质波动率 |  |
| style_residual_ret_20 | 20日残差动量 | CAPM 残差 20 日滚动求和 |
| ind_ret_5 | 行业5日收益率分位 | 行业内个股过去收益均值的跨行业 percentile rank |
| ind_ret_10 | 行业10日收益率分位 |  |
| ind_ret_20 | 行业20日收益率分位 |  |
| ind_rotation_speed_20 | 行业轮动速度 | \|短期排名-长期排名\|/行业数（5日 vs 20日） |
| ind_strength_20 | 20日行业相对强度 | 个股20日收益-所属行业20日均收益 |
| ind_dispersion_20 | 行业内部离散度 | 行业内个股20日收益标准差 |
| ind_breadth_up_20 | 行业上涨宽度 | 行业内20日收益&gt;0的个股占比 |
| ind_volume_ratio_20 | 行业成交量占比分位 | 行业总成交额的跨行业 percentile rank |
| ind_crowding_20 | 行业拥挤度分位 | 行业成交额/全市场成交额的跨行业 rank |
| ind_netflow_rank_20 | 行业资金流分位 | 行业内 L2 净流入合计（flow_net_amount）的跨行业 rank —— **当前已断更，见下方「已知数据状况」** |
| ind_relative_pe | 行业相对PE分位 | 行业PE中位/全市场PE中位的跨行业 rank |
| ind_concentration | 行业集中度 | 行业内 top5 流通市值/行业总流通市值 |
| ind_relative_momentum_20 | 行业相对超额动量 | 行业20日均收益-全市场20日均收益 |
| ind_momentum_decay | 行业动量衰减度 | 行业5日均收益-行业20日均收益 |
| concept_hot_score | 概念综合热度得分 | 归属概念中20日收益&gt;3%的个数 |
| concept_momentum_top3 | Top3概念动量得分 | 归属概念按20日收益取前3的均值 |
| concept_exposure_top1 | 主营最高概念敞口 | 个股在最强概念内的成交额排名分位 |
| concept_rotation_score | 概念板块轮动得分 | 归属概念5日排名与20日排名差绝对值的均值 |
| concept_crowding_max | 最高概念拥挤度 | 归属概念成交额占比分位的最大值 |
| concept_diversity | 所属概念多样性指数 | 归属概念个数 |
| concept_flow_rank | 关联概念资金流分位 | 归属概念 L2 净流入合计分位的均值 —— **当前已断更，见下方「已知数据状况」** |
| concept_leader_score | 概念龙头评分 | 个股在各归属概念内成交额分位的均值 |
| concept_cross_sector | 跨行业概念强度 | 归属概念是否覆盖≥2个一级行业（1/0） |
| concept_volume_ratio | 概念板块成交占比 | 归属概念成交额/全市场成交额的均值 |

## 3. `l2_factors`（219 列 = 8 主键 + 211 L2 因子）

基于日内**逐笔成交与逐笔委托**计算，仅 **2018 年起**（2120 个交易日）。

| 前缀 | 数量 | 类别 |
|---|---:|---|
| `micro_*` | 139 | 微观结构：报价/有效价差、VPIN（8/20/50/100 桶）、订单不平衡、十档深度、价格冲击（Amihud、Kyle's λ）、撤单行为、交易间隔与序列熵、跳跃、时段分解（T1–T8）、流动性族、逆向选择与信息份额 |
| `flow_*` | 44 | 资金流：净/买/卖金额与比率、超大单/大单/中单/小单结构、订单流毒性、一致性、大单占比 |
| `vol_*` | 28 | 已实现波动率：RV/RRV/偏度/峰度、5/10/15 分钟与上/下午分解、跳跃成分 |

### 211 个 L2 因子完整口径（与实际列核对：211/211 全部对得上）

<!-- factors_l2: 211 行 -->
| 字段名称 | 中文标签 | 口径说明 |
|---|---|---|
| micro_qsp_equal | 等权相对买卖价差 | qsp=(ask1-bid1)/mid×100 等权均值，百分比 |
| micro_qsp_time | 时间加权相对价差 | qsp 按快照时间间隔加权 |
| micro_qsp_volume | 量加权相对价差 | qsp 按快照成交量加权 |
| micro_esp_equal | 等权有效价差 | esp=2×\|成交价-成交时刻mid\|/mid×100 等权均值 |
| micro_esp_time | 时间加权有效价差 | esp 按成交时间间隔加权 |
| micro_esp_volume | 量加权有效价差 | esp 按成交量加权 |
| micro_aqsp | 绝对买卖价差(元) | ask1-bid1 等权均值，元 |
| micro_aesp | 绝对有效价差(元) | 2×\|成交价-mid\| 等权均值，元 |
| micro_qsp_vol_20 | 相对价差波动 | 全天 qsp 标准差 |
| micro_esp_vol_20 | 有效价差波动 | 全天 esp 标准差 |
| micro_spread_slope_1_5 | 1-5档价差深度斜率 | 1~5档价格~深度量加权回归斜率，日内均值 |
| micro_spread_slope_5_10 | 5-10档价差深度斜率 | 5~10档同上 |
| micro_spread_slope_1_10 | 1-10档价差深度斜率 | 1~10档同上 |
| micro_bid_ask_vol_ratio | 买卖一档量比 | mean(bid_vol1)/mean(ask_vol1) |
| micro_spread_recovery | 价差恢复速度 | qsp 冲高超均值+1σ后回到均值1.1倍内的平均快照数 |
| micro_spread_open | 开盘相对价差 | 开盘时段 qsp 均值 |
| micro_spread_mid | 盘中相对价差 | 盘中时段 qsp 均值 |
| micro_spread_close | 收盘相对价差 | 收盘时段 qsp 均值 |
| micro_spread_vol_ratio | 价差波动比 | 等权 qsp / 逐笔收益波动代理 |
| micro_spread_U_shape | 价差U型形态 | (open+close)/2/mid-1，clip 到 [-5,5] |
| vol_realized_rv | 已实现方差 | 5分钟桶VWAP对数收益平方和，r=100×Δlog(VWAP) |
| vol_realized_rrv | 已实现相对方差 | 5分钟桶高低区间平方和 |
| vol_realized_rskew | 已实现偏度 | √m×Σr³/RV^1.5 |
| vol_realized_rkurt | 已实现峰度 | m×Σr⁴/RV² |
| vol_realized_5min | 5分钟已实现波动 | 300s桶 RV |
| vol_realized_10min | 10分钟已实现波动 | 600s桶 RV |
| vol_realized_15min | 15分钟已实现波动 | 900s桶 RV |
| vol_realized_am | 上午已实现波动 | 上午时段 5分钟桶 RV |
| vol_realized_pm | 下午已实现波动 | 下午时段 5分钟桶 RV |
| vol_realized_opening | 开盘已实现波动 | 开盘时段 5分钟桶 RV |
| vol_realized_closing | 收盘已实现波动 | 收盘时段 5分钟桶 RV |
| vol_realized_jump | 已实现跳跃 | max(RV-BV,0)，BV 为双幂变差 |
| vol_realized_sjv | 已实现跳跃变差 | (RV-BV)/RV×√m 标准化跳跃统计量 |
| flow_net_amount | 净流入金额(元) | 主动买入金额-主动卖出金额 |
| flow_buy_amount | 主动买入金额 | side=0 成交金额合计 |
| flow_sell_amount | 主动卖出金额 | side=1 成交金额合计 |
| flow_net_ratio | 净流入占比 | (买-卖)/(买+卖)，金额口径 |
| flow_buy_ratio | 买入占比 | 买笔数/总笔数 |
| flow_sell_ratio | 卖出占比 | 卖笔数/总笔数 |
| flow_super_net | 超大单净流入 | 单笔≥100万的买-卖金额 |
| flow_large_net | 大单净流入 | 单笔20~100万的买-卖金额 |
| flow_medium_net | 中单净流入 | 单笔5~20万的买-卖金额 |
| flow_small_net | 小单净流入 | 单笔&lt;5万的买-卖金额 |
| flow_large_ratio | 大单占比 | 大单(买-卖)/(买+卖) |
| flow_medium_ratio | 中单占比 | 中单(买-卖)/(买+卖) |
| flow_small_ratio | 小单占比 | 小单(买-卖)/(买+卖) |
| flow_large_pct | 大单成交占比 | (超大+大单)金额/全天金额 |
| flow_imbalance_volume | 成交量失衡 | (买量-卖量)/(买量+卖量) |
| flow_imbalance_open | 开盘失衡 | 开盘时段金额失衡 |
| flow_imbalance_mid_am | 上午盘中失衡 | 上午盘中时段金额失衡 |
| flow_imbalance_mid_pm | 下午盘中失衡 | 下午盘中时段金额失衡 |
| flow_imbalance_close | 收盘失衡 | 收盘时段金额失衡 |
| flow_imbalance_autocorr | 失衡自相关 | 5分钟桶金额失衡序列 lag1 自相关 |
| flow_imbalance_revert_speed | 失衡恢复速度 | 失衡&gt;0.3后回落到&lt;0.1的平均桶数 |
| flow_consistency | 资金流一致性 | 相邻两笔同向占比 |
| flow_money_flow_index | 资金流量指标 | (上涨笔金额-下跌笔金额)/(两者之和) |
| flow_big_trade_ratio | 大单交易占比 | 单笔≥50万笔数/总笔数 |
| micro_vpin_8 | 8桶VPIN | 等量分桶(N=8)买卖量不平衡均值 |
| micro_vpin_20 | 20桶VPIN | 等量分桶(N=20)同上 |
| micro_vpin_50 | 50桶VPIN | 等量分桶(N=50)同上，桶大小=总量/50 |
| micro_vpin_ma_5 | 5期VPIN均值 | N=50桶VPIN序列尾部5桶均值 |
| micro_vpin_ma_20 | 20期VPIN均值 | N=50桶VPIN序列尾部20桶均值 |
| micro_vpin_delta_5 | 5期VPIN变化 | 末桶-前5桶 VPIN 差 |
| micro_vpin_zscore_20 | VPIN Z分数 | (末桶-20桶均值)/20桶标准差 |
| micro_pin | PIN知情交易概率 | \|买量-卖量\|/(买量+卖量)，全天总量口径 |
| micro_order_flow_toxicity | 订单流毒性 | VPIN50×全天价格方向(±1) |
| micro_volume_sync | 量价同步性 | corr(\|逐笔收益\|,逐笔量) |
| micro_vpin_surge | VPIN激增次数 | 桶VPIN&gt;均值+2σ的桶数 |
| micro_vpin_trend | VPIN趋势 | 桶VPIN序列一元回归斜率 |
| micro_vpin_vol_ratio | VPIN波动比 | VPIN50/对数收益平方和代理波动 |
| micro_vpin_amount_ratio | VPIN金额比 | VPIN50/log(全天成交额) |
| micro_depth_bid | 买盘深度 | log1p(10档买量合计均值) |
| micro_depth_ask | 卖盘深度 | log1p(10档卖量合计均值) |
| micro_depth_ratio_1 | 1档买卖深度比 | mean(买1量/卖1量) |
| micro_depth_ratio_5 | 5档买卖深度比 | mean(买1~5量/卖1~5量) |
| micro_depth_ratio_10 | 10档买卖深度比 | mean(买1~10量/卖1~10量) |
| micro_depth_weighted_bid | 加权买盘深度价 | 买盘量加权平均价，日内均值 |
| micro_depth_weighted_ask | 加权卖盘深度价 | 卖盘量加权平均价，日内均值 |
| micro_depth_concentration_bid | 买盘集中度 | 前5档买量 Herfindahl 指数均值 |
| micro_depth_concentration_ask | 卖盘集中度 | 前5档卖量 Herfindahl 指数均值 |
| micro_depth_slope | 深度斜率 | 买盘价格~档位量加权回归斜率(1~99% winsorize) |
| micro_depth_effective | 有效深度 | mean(min(买1量,卖1量)) |
| micro_depth_recovery | 深度恢复速度 | 买盘深度跌破均值-0.5σ后回到0.9倍均值的平均快照数 |
| micro_depth_open | 开盘深度比 | 开盘时段 mean(买总量/卖总量) |
| micro_depth_mid | 盘中深度比 | 盘中时段同上 |
| micro_depth_close | 收盘深度比 | 收盘时段同上 |
| micro_depth_volatility | 深度波动 | 买卖总量比的 std/mean(变异系数) |
| micro_depth_extreme | 深度极端值 | (末快照买1量-均值)/标准差 |
| micro_depth_imbalance_1 | 1档深度失衡 | mean((买1-卖1)/(买1+卖1)) |
| micro_depth_imbalance_5 | 5档深度失衡 | 前5档总量同上 |
| micro_depth_imbalance_10 | 10档深度失衡 | 前10档总量同上 |
| flow_cancel_rate | 撤单率 | 撤单笔数/新增笔数 |
| flow_cancel_buy_rate | 买撤单率 | 撤买/新增买 |
| flow_cancel_sell_rate | 卖撤单率 | 撤卖/新增卖 |
| flow_cancel_imbalance | 撤单失衡 | (撤买-撤卖)/(撤买+撤卖) |
| flow_cancel_depth_dist | 撤单深度分布 | 撤单价距新增中位价的平均相对距离 |
| flow_cancel_lifetime | 撤单存活期(秒) | 相邻委托时间间隔均值近似 |
| flow_cancel_hft_ratio | 高频撤单比 | 间隔&lt;1秒的撤单/总撤单 |
| flow_cancel_spoof | 幌骗撤单(笔) | 撤单量&gt;5倍新增均量的笔数 |
| flow_order_arrival_rate | 委托到达率(笔/分) | 总委托/240分钟 |
| flow_order_imbalance | 委托失衡 | (新增买量-新增卖量)/(两者之和) |
| flow_order_book_rebuild | 订单簿重建 | 有撤单的分钟内新增笔数/总撤单 |
| flow_order_size_skew | 委托规模偏度 | 新增委托量的偏度 |
| flow_order_large_ratio | 大单委托占比 | 委托量&gt;均值+3σ的占比 |
| flow_cancel_price_deviation | 撤单价格偏离 | 撤单价距新增均价的距离/0.1% tick |
| flow_cancel_post_price_change | 撤单后价格变化(元) | 撤单后5秒窗口成交价变化均值(searchsorted 向量化) |
| flow_order_type_diversity | 委托类型多样性 | 委托类型分布香农熵 |
| flow_order_quality | 委托质量 | (新增-撤单)/总委托 |
| flow_order_duration_p50 | 委托间隔中位数(秒) | 相邻委托间隔 P50 |
| flow_order_duration_p90 | 委托间隔P90(秒) | 相邻委托间隔 P90 |
| flow_cancel_cluster | 撤单聚集度 | 按分钟分桶撤单数的 std/mean |
| micro_zone_vol_ratio_T1 | 集合竞价量占比 | 09:15-09:25成交量/全天成交量，快照+逐笔合计 |
| micro_zone_vol_ratio_T2 | 开盘量占比 | 09:30-10:00成交量/全天成交量 |
| micro_zone_vol_ratio_T3 | 上午前段量占比 | 10:00-11:00成交量/全天成交量 |
| micro_zone_vol_ratio_T4 | 上午尾段量占比 | 11:00-11:30成交量/全天成交量 |
| micro_zone_vol_ratio_T5 | 下午首段量占比 | 13:00-13:30成交量/全天成交量 |
| micro_zone_vol_ratio_T6 | 下午中段量占比 | 13:30-14:30成交量/全天成交量 |
| micro_zone_vol_ratio_T7 | 尾盘量占比 | 14:30-14:57成交量/全天成交量 |
| micro_zone_vol_ratio_T8 | 收盘集合量占比 | 14:57-15:00成交量/全天成交量 |
| micro_zone_rv_ratio_AM | 上午波动占比 | 上午5分钟桶RV/全天RV |
| micro_zone_rv_ratio_PM | 下午波动占比 | 下午5分钟桶RV/全天RV |
| micro_zone_rv_ratio_open | 开盘波动占比 | 开盘时段RV/全天RV |
| micro_zone_rv_ratio_close | 收盘波动占比 | 收盘时段RV/全天RV |
| micro_zone_spread_ratio_open | 开盘价差占比 | 开盘qsp均值/全天qsp均值 |
| micro_zone_spread_ratio_mid | 盘中价差占比 | 盘中qsp均值/全天qsp均值 |
| micro_zone_spread_ratio_close | 收盘价差占比 | 收盘qsp均值/全天qsp均值 |
| micro_open_gap | 开盘跳空 | (首快照价-昨收)/昨收 |
| micro_call_auction_vol_ratio | 集合竞价量比 | 09:15-09:25(+结果快照)量/全天量 |
| micro_close_auction_impact | 收盘竞价冲击 | (最终价-倒数第2快照价)/倒数第2价 |
| micro_close_squeeze | 收盘挤压 | (尾10笔收益-尾30笔收益)/尾30笔收益 |
| micro_lunch_break_gap | 午间跳空 | (下午首价-上午末价)/上午末价 |
| micro_zone_vpin_open | 开盘VPIN | 开盘时段8桶VPIN |
| micro_zone_imbalance_am | 上午失衡 | 上午金额(买-卖)/(买+卖) |
| micro_zone_imbalance_pm | 下午失衡 | 下午金额(买-卖)/(买+卖) |
| micro_zone_morning_evening_ratio | 上下午比值 | 上午金额/下午金额 |
| micro_zone_distribution | 时段分布集中度 | 8时段成交量分布香农熵 |
| micro_jump_count_1pct | 1%跳跃次数 | \|逐笔收益\|&gt;1%的笔数 |
| micro_jump_count_05pct | 0.5%跳跃次数 | \|逐笔收益\|&gt;0.5%的笔数 |
| micro_jump_size_mean | 跳跃平均幅度 | &gt;0.5%跳跃的平均\|幅度\|，无跳跃=0 |
| micro_jump_size_max | 跳跃最大幅度 | &gt;0.5%跳跃的最大\|幅度\|，无跳跃=0 |
| micro_jump_skew | 跳跃偏度 | (上跳数-下跳数)/(总数)，阈值0.5% |
| micro_jump_recovery_time | 跳跃恢复时间(秒) | 跳跃后500笔内回到±0.1%的平均秒数 |
| micro_jump_cluster | 跳跃聚集度 | 含≥2跳的5分钟桶占比 |
| micro_amihud_illiquidity | Amihud非流动性 | mean(\|log收益\|/成交额)×1e6 |
| micro_kyle_lambda | Kyle Lambda | Δprice对带符号量回归斜率取绝对值 |
| micro_price_impact_large | 大单价格冲击 | 单笔&gt;50万成交后5笔平均收益 |
| micro_impact_elasticity | 价格冲击弹性 | \|Δprice\|对log(量)回归斜率 |
| micro_impact_cost | 冲击成本(元) | 平均价差+平均\|逐笔价差\| |
| micro_impact_permanent_ratio | 永久冲击占比 | 大额成交10笔后\|价差\|/(10笔后+当笔)，金额&gt;3倍均值 |
| micro_impact_decay_half_life | 冲击衰减半衰期(笔) | 冲击衰到一半的平均笔数 |
| micro_jump_flag | 跳跃标记 | 有&gt;0.5%跳跃=1，否则0 |
| micro_trade_interval_mean | 成交间隔均值(秒) | 相邻逐笔秒差均值 |
| micro_trade_interval_median | 成交间隔中位数 | 相邻逐笔秒差中位数 |
| micro_trade_interval_cv | 成交间隔变异系数 | 间隔std/mean |
| micro_trade_zero_interval_ratio | 零间隔占比 | 间隔=0的笔数/总间隔数 |
| micro_trade_arrival_rate | 成交到达率(笔/分) | 总笔数/实际交易分钟数 |
| micro_trade_sign_autocorr | 成交方向自相关 | 买卖方向(±1) lag1 相关 |
| micro_trade_sign_transition_BB | 买买连续转化率 | 买后仍买的占比 |
| micro_trade_sign_transition_EB | 卖买转化率 | 卖后转买的占比 |
| micro_trade_continuation | 成交持续性(笔) | 同向连续段平均长度 |
| micro_trade_seq_entropy | 成交序列熵 | BB/BE/EB/EE四状态香农熵 |
| micro_trade_run_length | 同向连击(笔) | 最长同向连续笔数 |
| micro_trade_buy_pressure | 买盘压力(元) | 买单金额×向上越mid幅度的和 |
| micro_trade_sell_pressure | 卖盘压力(元) | 卖单金额×向下越mid幅度的和 |
| micro_liquidity_amihud_5 | Amihud非流动性(近5桶) | 近5个5分钟桶Amihud均值×1e6 |
| micro_liquidity_amihud_20 | Amihud非流动性(近20桶) | 近20(不足则全)桶同上 |
| micro_liquidity_ratio | 流动性比率 | 收益波动×一档深度/平均价差 |
| micro_liquidity_turnover_decay | 换手率衰减 | (09:30-10:00量占比)/12.5% |
| micro_liquidity_volume_depth_ratio | 量深比 | 全天成交量/一档平均挂单量 |
| micro_liquidity_volatility | 流动性波动 | 分桶Amihud标准差 |
| micro_liquidity_extreme | 流动性极端值(个) | Amihud&gt;均值+3σ的桶数 |
| micro_liquidity_close | 收盘流动性 | 收盘时段分桶Amihud均值 |
| micro_liquidity_roll | Roll价差估计 | 2×√max(0,-一阶自协方差)，逐笔价差口径 |
| micro_liquidity_cost | 流动性成本(元) | 平均买卖价差+平均\|逐笔价差\| |
| micro_liquidity_twa | 时间加权流动性 | 分桶Amihud全天均值 |
| micro_liquidity_concentration | 流动性集中度 | 最大5分钟桶量/全天量 |
| micro_liquidity_daily_pattern | 日内流动性模式 | (首2桶+尾2桶)/2/全天均值，U型代理 |
| micro_liquidity_relative_spread | 相对买卖价差(%) | (ask1-bid1)/mid×100均值 |
| micro_liquidity_holden | Holden价差估计 | Amihud中位与价差均值的可用项均值 |
| micro_vpin_100 | 100桶VPIN | 按笔数等分100桶的买卖不平衡均值 |
| micro_vpin_kyle | Kyle VPIN | 20桶带符号量不平衡均值 |
| micro_vpin_quantile_ratio | VPIN分位比 | 50桶中VPIN&gt;0.5的占比 |
| micro_vpin_tail_risk | VPIN尾部风险 | 50桶VPIN的95分位 |
| micro_vpin_entropy | VPIN熵 | 50桶VPIN归一化香农熵 |
| micro_adverse_selection | 逆选择成本 | 前500笔\|成交价-5秒后mid\|/成交价均值 |
| micro_realized_spread | 已实现价差 | 方向×(成交价-5秒后mid)/成交价均值 |
| micro_adverse_selection_cost | 逆选择成本率 | 逆选择成本/平均相对价差 |
| micro_spread_decomposition | 价差分解 | \|已实现价差\|/平均有效价差，逆向选择占比 |
| micro_information_share | 信息份额 | corr(带符号量,5笔后价格变化) |
| micro_toxicity_persistence | 毒性持续性 | 桶VPIN序列lag1自相关 |
| micro_vpin_hurst | VPIN Hurst指数 | 桶VPIN序列R/S法Hurst |
| micro_vpin_conviction | VPIN信心度 | 连续高于中位的VPIN段/总桶数 |
| micro_toxicity_asymmetry | 毒性不对称 | 分桶买占比均值-卖占比均值 |
| micro_informed_ratio | 知情交易比例 | 90分位大单占比×方向集中度 |
| micro_liquidity_mining | 流动性挖掘因子 | 90分位大单后5笔平均冲击/大单占比 |
| micro_vpin_bayesian | 贝叶斯VPIN | 先验0.5收缩后的VPIN后验均值 |
| vol_turnover_total | 总成交量(股) | 全天逐笔量合计，join流通市值即换手率 |
| vol_turnover_open | 开盘成交占比 | 09:30-10:00量/全天量 |
| vol_turnover_close | 尾盘成交占比 | 14:30-15:00量/全天量 |
| vol_turnover_am_pm | 上下午量比 | 上午量/下午量 |
| vol_turnover_concentration | 换手集中度 | 按笔数10等分最大桶占比/均值比 |
| vol_price_corr | 量价相关性 | 分钟桶量与分钟收益corr |
| vol_price_divergence | 量价背离 | 分钟量变与价变方向不一致占比 |
| vol_weighted_price | 量加权价格偏离 | (收盘价-VWAP)/VWAP |
| vol_up_down_ratio | 上涨下跌量比 | 上涨笔量/下跌笔量 |
| vol_large_trade_intensity | 大单成交强度 | 90分位大单金额占比/笔数占比 |
| vol_tick_density | 分笔密度(笔/秒) | 总笔数/实际交易秒数 |
| vol_skew | 成交量偏度 | 分钟桶量的偏度 |
| vol_kurtosis | 成交量峰度 | 分钟桶量的峰度 |
| vol_gini | 成交量基尼系数 | 分钟桶量的不均匀度，0~1 |
| vol_persistence | 成交量持续性 | 分钟桶量lag1自相关 |

## 4. `l1_l2_factors`（329 列）

`symbol, date` + 后复权 OHLCV + 110 个 L1 + 211 个 L2，按 `(symbol, date)` **内连接**，OHLCV 取 L1 侧。
**多因子建模首选这张表**——不必分别下载 `l1_factors` 和 `l2_factors`。

<details>
<summary>展开查看全部 329 个列名</summary>

**共 329 列**

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

## 已知数据状况（实测）

1. **两个因子已断更**：`ind_netflow_rank_20`、`concept_flow_rank` 在 2026-09-24 全表 100% 为空且退化为常数列，
   但 2026-06-01 仍正常。**建议按列空值率动态剔除，不要写死列名。**
2. **高缺失 L2 因子**：`micro_jump_skew` ~54%、`micro_jump_recovery_time` ~48%、`micro_price_impact_large` ~24%；
   其余 `micro_*`/`flow_*` 列中位缺失率接近 0。
3. **`fun_pe` 极端值**：实测单日跨度 -1206 ~ +1069。LightGBM 按秩分裂影响较小，做中性化/线性模型需 winsorize 或改用 `fun_ep`/`fun_bp`。
4. **`micro_*` 与 `flow_*` 的金额列单位是万元**，其余比率类无量纲。
5. **2018 年边界**：L2 与合并表在 2018 年前无数据，跨期训练注意样本结构突变。

## 一个最小可用建模骨架

```python
import glob, pandas as pd

paths = sorted(glob.glob("quantdb/6_ml_datasets/l1_l2_factors/dt=202[3-6]*/data.parquet"))
df = pd.concat([pd.read_parquet(p) for p in paths], ignore_index=True)

LABEL = "future_return_5d"          # 需自行按 5 日收益对齐，本表不含标签列
drop = [LABEL] if LABEL in df.columns else []

# 动态剔除高空值因子（应对上面第 1、2 条）
null_rate = df.isna().mean()
dead = null_rate[null_rate > 0.30].index.tolist()

feat = [c for c in df.columns
        if c not in {"symbol","date","time","open","high","low","close","volume","amount"} | set(drop) | set(dead)
        and not c.startswith("future_return_")]
print(f"特征 {len(feat)} 列，剔除 {len(dead)} 个高空值列：{dead[:5]}")
```

> 注意：`l1_l2_factors` **本身不含标签列**。标签要从同目录 `features_daily` 或
> `5_technical_derived/technical_indicators` 里取 `future_return_*`，按 `(symbol, time/date)` 对齐后拼接。

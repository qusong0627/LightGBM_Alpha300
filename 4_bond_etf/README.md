# `4_bond_etf/` — 可转债与 ETF

**快照类单文件**，只含当期截面，没有历史序列。整个目录实测约 1.2 MB。

```
4_bond_etf/
├── convertible_bond/
│   └── bond_detail.parquet          # 0.05 MB — 316 只 × 25 列
└── etf_pcf/
    ├── etf_pcf.parquet              # 0.10 MB — 1604 只 × 19 列（申赎清单主表）
    ├── etf_components.parquet       # 1.00 MB — 190366 行 × 6 列（成分明细，长表）
    └── trackzs_etf_info.parquet     # 0.01 MB — 111 只 × 8 列（跟踪指数的 ETF）
```

## 下载

```bash
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "4_bond_etf/*" --local_dir ./quantdb
```

## `convertible_bond/bond_detail.parquet`（316 × 25）

| 列 | 说明 |
|---|---|
| `BondCode` / `bondCode` | 转债代码（带后缀 / 纯数字，**同一信息两种命名各存一列**） |
| `stockCode` / `bondName` | 正股代码 / 转债名称 |
| `bondConvPrice` | 转股价 |
| `bondCouponRate` | 票面利率 |
| `bondRestScope` | 剩余规模 |
| `putBackPrice` / `putDate` / `putPrice` | 回售价格 / 回售起始日 / 回售价 |
| `forceRedeemPrice` / `forceRedeemDate` / `forceRedeemRedeemPrice` | 强赎触发价 / 强赎登记日 / 强赎价 |
| `convStartDate` / `bondMaturityDate` | 转股起始日 / 到期日 |
| `expireRedeemPrice` | 到期赎回价 |
| `conversionPremium` / `conversionValue` | 转股溢价率 / 转股价值 |
| `bondValue` / `expireYield` | 纯债价值 / 到期收益率 |
| `bondRating` / `stockRating` | 债项评级 / 主体评级 |
| `stockPrice` / `bondPrice` / `bondPremium` | 正股价 / 转股价 / 溢价 |

官方口径摘录：

<!-- bond: 11 行 -->
| 字段名称 | 数据类型 | 指标说明 |
|---|---|---|
| BondCode / bondName | string | 可转债代码与名称 |
| stockCode | string | 正股代码 |
| bondConvPrice | float | 最新转股价（元） |
| conversionPremium | float | 转股溢价率（%） |
| bondValue | float | 纯债价值（元） |
| expireYield | float | 到期收益率（YTM %） |
| bondMaturityDate | date | 债券到期日 |
| trade_date（原始列 time） | datetime | 交易日 |
| open / high / low / close | float | 开高低收（元） |
| volume | int | 成交量（股，已归一化） |
| amount | float | 成交额（万元，已归一化） |

> **实测坑**：抽样里 `bondValue`、`expireYield` 全为 `0.0`，`forceRedeemDate`、`putDate` 为 `NaT`。
> 这几个字段在当前快照中**未填充**，不要直接用于策略过滤；用前请按列检查空值/零值率。

## `etf_pcf/` — ETF 申赎清单

### `etf_pcf.parquet`（1604 × 19）—— 主表

| 列 | 说明 |
|---|---|
| `EtfCode` / `etfCode` / `etfExchID` | ETF 代码（带后缀 / 纯数字 / 交易所） |
| `prCode` | 产品代码（部分为空） |
| `name` | ETF 名称 |
| `nav` / `navPerCU` | 净值 / 单位赎回明细净值 |
| `cashBalance` / `maxCashRatio` | 现金替代额度 / 最大现金替代比例 |
| `reportUnit` | 报告单位 |
| `ecc` | 预估现金差额 |
| `needPublish` / `enableCreation` / `enableRedemption` | 是否公布 / 可申购 / 可赎回标记 |
| `creationLimit` / `redemptionLimit` | 申购 / 赎回上限 |
| `type` | 类型标记 |
| `tradingDay` / `preTradingDay` | 清单交易日 / 前一交易日（**货基类 ETF 这两列是空字符串**） |

### `etf_components.parquet`（190366 × 6）—— 成分明细

```
EtfCode, ComponentCode, componentExchID, componentCode, componentVolume, ReplaceFlag
```

`ReplaceFlag` 标记现金替代类型；`componentVolume` 为最小申赎单位下的成分数量（可为 0）。
与主表按 `EtfCode` 关联。

### `trackzs_etf_info.parquet`（111 × 8）—— 跟踪指数的 ETF

```
TrackIndex, Code, Name, NowPrice, PreClose, IOPV, Zgb, Sz
```

`TrackIndex` 就是指数代码（如 `000300.SH`），可直接和 [`2_base_sector/index_weights/`](../2_base_sector/README.md) 关联，
用 ETF 与其跟踪指数的成分做对照。`Zgb` / `Sz` 为通达信风格的规模/份额字段名。

## 注意

- 这三个文件都是**当期快照**，无历史版本，不能做跨期 ETF 成分回测。
- 官网描述的 `etf_kline`（ETF 日 K 线）**不在本魔搭快照中**；ETF 行情请用 [`1_kline_data/`](../1_kline_data/README.md) 里对应的代码。
- 字段名大小写混用（`EtfCode` 与 `etfCode`、`BondCode` 与 `bondCode` 成对出现），取值时注意别拿错列。

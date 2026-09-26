# `4_bond_etf/` — Convertible Bonds and ETFs

**Snapshot files only** — current cross-sections, no history. The whole directory is ~1.2 MB.

```
4_bond_etf/
├── convertible_bond/
│   └── bond_detail.parquet          # 0.05 MB — 316 × 25
└── etf_pcf/
    ├── etf_pcf.parquet              # 0.10 MB — 1604 × 19 (PCF header)
    ├── etf_components.parquet       # 1.00 MB — 190366 × 6 (constituent detail, long table)
    └── trackzs_etf_info.parquet     # 0.01 MB — 111 × 8 (index-tracking ETFs)
```

Layout: [中文](README.md) · [back to top](../README.en.md)

## Download

```bash
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "4_bond_etf/*" --local_dir ./quantdb
```

## `convertible_bond/bond_detail.parquet` (316 × 25)

| Column | Meaning |
|---|---|
| `BondCode` / `bondCode` | bond code (with suffix / bare) — **the same value stored twice under different names** |
| `stockCode` / `bondName` | underlying stock / bond name |
| `bondConvPrice` | conversion price |
| `bondCouponRate` | coupon rate |
| `bondRestScope` | remaining outstanding size |
| `putBackPrice` / `putDate` / `putPrice` | put-back trigger / date / price |
| `forceRedeemPrice` / `forceRedeemDate` / `forceRedeemRedeemPrice` | forced-redemption trigger / date / price |
| `convStartDate` / `bondMaturityDate` | conversion start / maturity date |
| `expireRedeemPrice` | redemption price at maturity |
| `conversionPremium` / `conversionValue` | conversion premium / conversion value |
| `bondValue` / `expireYield` | straight bond value / yield to maturity |
| `bondRating` / `stockRating` | issue rating / issuer rating |
| `stockPrice` / `bondPrice` / `bondPremium` | underlying price / bond price / premium |

Upstream definitions, translated:

<!-- bond_en: 11 行 | 译自 quantdb.cn §5 ETF / 可转债 -->
| Field | Type | Description |
|---|---|---|
| BondCode / bondName | string | Convertible bond code and name |
| stockCode | string | Underlying stock code |
| bondConvPrice | float | Latest conversion price (CNY) |
| conversionPremium | float | Conversion premium (%) |
| bondValue | float | Straight bond value (CNY) |
| expireYield | float | Yield to maturity (YTM %) |
| bondMaturityDate | date | Maturity date |
| trade_date (source column `time`) | datetime | Trade date |
| open / high / low / close | float | OHLC (CNY) |
| volume | int | Volume (shares, normalized) |
| amount | float | Turnover value (10k CNY, normalized) |

> The last four rows describe `etf_kline`, which is **not in this snapshot**.
> And in the shipped `bond_detail.parquet`, `bondValue` and `expireYield` are **all zero**,
> while `forceRedeemDate` / `putDate` are `NaT` — the fields exist but are unpopulated.

> **Measured data-quality problem:** in the shipped file `bondValue` and `expireYield` are **all `0.0`**, and
> `forceRedeemDate` / `putDate` are `NaT`. The columns exist but are unpopulated — do not use them as
> strategy filters without checking fill rates per column first.

## `etf_pcf/` — ETF creation and redemption lists

### `etf_pcf.parquet` (1604 × 19) — header

| Column | Meaning |
|---|---|
| `EtfCode` / `etfCode` / `etfExchID` | ETF code (with suffix / bare / exchange) |
| `prCode` | product code (empty for some rows) |
| `name` | ETF name |
| `nav` / `navPerCU` | net asset value / NAV per creation unit |
| `cashBalance` / `maxCashRatio` | cash substitute balance / max cash substitute ratio |
| `reportUnit` | reporting unit |
| `ecc` | estimated cash component |
| `needPublish` / `enableCreation` / `enableRedemption` | publication / creation / redemption flags |
| `creationLimit` / `redemptionLimit` | creation / redemption caps |
| `type` | type marker |
| `tradingDay` / `preTradingDay` | list trade date / prior trade date — **empty strings for money-market ETFs** |

### `etf_components.parquet` (190366 × 6) — constituent detail

```
EtfCode, ComponentCode, componentExchID, componentCode, componentVolume, ReplaceFlag
```

`ReplaceFlag` marks the cash-substitute type; `componentVolume` is the constituent quantity per creation
unit (may be 0). Join to the header on `EtfCode`.

### `trackzs_etf_info.parquet` (111 × 8) — index-tracking ETFs

```
TrackIndex, Code, Name, NowPrice, PreClose, IOPV, Zgb, Sz
```

`TrackIndex` is an index code (e.g. `000300.SH`) and joins directly to
[`2_base_sector/index_weights/`](../2_base_sector/README.en.md), letting you compare an ETF against its
index constituents. `Zgb` / `Sz` are native TDX-style size/share fields.

## Caveats

- All three are **current snapshots** with no historical versions — no cross-period ETF constituent backtests.
- The spec's `etf_kline` (ETF daily bars) is **not in this snapshot**; use the ETF codes against
  [`1_kline_data/`](../1_kline_data/README.en.md) instead.
- Column naming mixes cases (`EtfCode` vs `etfCode`, `BondCode` vs `bondCode` appear as pairs) —
  check you are reading the intended one.

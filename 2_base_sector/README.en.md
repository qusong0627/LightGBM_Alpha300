# `2_base_sector/` — Reference and Sector Data

The **metadata baseline** for the whole dataset: instrument master, industry/concept constituents,
trading calendar, index weights, and margin-trading detail. Download this first — every other analysis
needs it for universe filtering, date alignment and grouping. The snapshot files total ~3.6 MB.

Layout: [中文](README.md) · [back to top](../README.en.md)

## Structure and measured size

```
2_base_sector/
├── margin_trading/                  # day-partitioned, 2607 partitions, 0.10 MB/day, latest dt=20260923
│   └── dt=YYYYMMDD/data.parquet
├── instrument_detail/
│   └── instrument_list.parquet      #   3.2 MB — 5576 rows × 152 columns
├── sector_concept/
│   └── sector_members.parquet       #   0.2 MB — 79447 rows × 4 columns (long table)
├── trading_calendar/
│   └── trading_days.parquet         #   2 columns: TradingDate + IsTradingDay
└── index_weights/                   #   one file per index, 7 total
```

## Details

### `index_weights/` — index constituent weights

Columns `Symbol, Weight, IndexCode`. **`Weight` is a percentage** (sums to ≈100 per index, not 1).
Index identities confirmed by constituent count:

| File | Constituents | Weight sum | Index |
|---|---:|---:|---|
| `000016.SH` | 50 | 100.00 | SSE 50 |
| `000300.SH` | 300 | 100.00 | CSI 300 |
| `000688.SH` | 50 | 100.00 | STAR 50 |
| `000905.SH` | 500 | 99.99 | CSI 500 |
| `000906.SH` | 800 | 99.99 | CSI 800 |
| `000852.SH` | 1000 | 99.98 | CSI 1000 |
| `399006.SZ` | 100 | 99.98 | ChiNext Index |

> These are **current snapshots**, not historical weight series — using them for backtests over
> past periods introduces survivorship bias.

### `instrument_detail/instrument_list.parquet`

**5,576 rows × 152 columns** — a wide snapshot, not a slim master table. Measured flag counts:

| Field | Meaning | Count |
|---|---|---:|
| `BelongHS300` | CSI 300 constituent | 300 |
| `BelongRZRQ` | Margin-trading eligible | 3881 |
| `BelongHSGT` | Stock-Connect eligible | 3532 |
| `BelongHasKQZ` | Has an outstanding convertible bond | 314 |
| `IsSTGP` | ST-flagged | 204 |
| `IsQuitGP` | Delisting risk | **0** |
| `IsHKGP` | HK-listed | **0** |

> ⚠️ **These flag columns have dtype `object` and hold the strings `'0'` / `'1'`, not integers.**
> `inst.IsSTGP == 0` matches **nothing** and silently yields an empty frame; compare against `"0"`,
> or cast first: `inst[cols] = inst[cols].astype(int)`. The table has 5,576 rows in total.

The remaining columns include `Name`, `MainBusiness`, `rs_hyname` (industry), `tdx_dyname` (region),
`IPO_Price`, `J_start` (listing date), `ZTPrice`/`DTPrice` (limit up/down), `DynaPE`/`PB_MRQ`/`StaticPE_TTM`,
`EverZTCount`/`YearZTDay` (limit-up counts), `StaffNum`, `ReportDate` and more.
**They are largely native TDX-style names — read the actual Parquet column list before using them.**

### `sector_concept/sector_members.parquet`

Long table, 79,447 rows × 4 columns (`SectorCode, SectorName, SectorType, Symbol`). Measured `SectorType` breakdown:

| SectorType | Sectors | Rows | Example names |
|---|---:|---:|---|
| 概念板块 (concept) | 421 | 68290 | 5G概念, 国防军工, 稀缺资源, 含H股 |
| 地区板块 (region) | 32 | 5576 | 黑龙江, 新疆板块, 吉林板块 |
| 行业板块(二级) (industry L2) | 80 | 4143 | 一般零售, 电子商务, 贸易 |
| 行业板块(一级) (industry L1) | 48 | 1433 | 煤炭开采, 油气开采, 石油化工 |
| 风格板块 (style) | 2 | 5 | 轮动趋势, 板块趋势 |

A stock appears in many sectors, so **deduplicate on `Symbol`** before aggregating or you will double-count.

### `trading_calendar/trading_days.parquet`

`TradingDate` (a `YYYYMMDD` **string**, not a date type) + `IsTradingDay` (int).
The string form aligns directly with partition directory names like `dt=20260924`.

### `margin_trading/` — margin trading and securities lending

Day-partitioned, 2607 partitions, disclosed T+1, latest `dt=20260923` (**one day behind the quote tables**).
The latest partition has 3,869 rows, 341 of them Beijing Stock Exchange `.BJ` symbols; 10 columns:

```
symbol, time, finance_balance, slo_volume, finance_buy, slo_sell_volume,
finance_repay, slo_repay, finance_net, slo_net
```

| Group | Unit | Meaning |
|---|---|---|
| `finance_*` | 10k CNY | financing balance / buy / repay / net buy |
| `slo_*` | shares | securities-lending balance / sold / repaid / net sold |

Identities that hold in the data and are useful as a self-check:

```
finance_net      = finance_buy - finance_repay
slo_sell_volume  = slo_net + slo_repay
```

## Field definitions (excerpt)

Field definitions, translated from the spec page and kept verbatim; **these names do not all match the
shipped files (which are always lowercase) — trust the measured column lists in the sections above**:

<!-- base_en: 21 行 | 译自 quantdb.cn §3 基础数据与索引 -->
| Field | Type | Description |
|---|---|---|
| Symbol | string | Stock code (e.g. 600519.SH) |
| Name | string | Security short name |
| J_zgb | float | Total share capital (10k shares) |
| ActiveCapital | float | Circulating share capital (10k shares) |
| ListDate | date | Listing date (YYYY-MM-DD) |
| EstablishDate | date | Company establishment date |
| Province | string | Province |
| SecurityType | string | Security type |
| BelongRZRQ | string | Margin-trading eligible flag (1 = yes) |
| SectorCode | string | Sector code (e.g. 880201.SH) |
| SectorName | string | Sector name |
| SectorType | string | Sector class (industry / concept / style / region) |
| Symbol | string | Constituent stock code |
| trade_date | date | Trade date |
| Symbol | string | Stock code |
| finance_balance | float | Financing balance |
| slo_volume | float | Securities-lending remaining volume |
| finance_buy | float | Financing buy amount |
| slo_sell_amount | float | Securities-lending sell amount |
| finance_net | float | Net financing buy |
| slo_net | float | Net securities-lending sell |

> The table above groups instrument-master fields together with sector-constituent fields; in the data
> they live in two separate files (`instrument_detail` and `sector_concept`).

- Sector names in `sector_members.parquet` are Chinese only — there is no English alias column.

## Download

```bash
# Metadata snapshots (~3.6 MB — download unconditionally)
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "2_base_sector/instrument_detail/*" \
  --include "2_base_sector/sector_concept/*" \
  --include "2_base_sector/trading_calendar/*" \
  --include "2_base_sector/index_weights/*" --local_dir ./quantdb

# Margin trading by year
modelscope download --repo-type dataset qusong0627/LightGBM_Alpha300 \
  --include "2_base_sector/margin_trading/dt=2026*/*" --local_dir ./quantdb
```

## Reading

```python
import pandas as pd

inst = pd.read_parquet("quantdb/2_base_sector/instrument_detail/instrument_list.parquet")
sect = pd.read_parquet("quantdb/2_base_sector/sector_concept/sector_members.parquet")
cal  = pd.read_parquet("quantdb/2_base_sector/trading_calendar/trading_days.parquet")
w300 = pd.read_parquet("quantdb/2_base_sector/index_weights/000300.SH.parquet")
marg = pd.read_parquet("quantdb/2_base_sector/margin_trading/dt=20260923/data.parquet")

# Tradable universe: drop ST / delisting risk, keep margin-eligible names
# Flag columns are the strings '0'/'1' — never compare them to integers
pool = inst.loc[(inst.IsSTGP == "0") & (inst.IsQuitGP == "0") & (inst.BelongRZRQ == "1"), "Symbol"]
```

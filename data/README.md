# Data

Raw inputs for the corporate default–prediction project. All three files are pulled from
[WRDS (Wharton Research Data Services)](https://wrds-www.wharton.upenn.edu/) using **CRSP Version 2
(CIZ format)** and the **CRSP/Compustat Merged (CCM)** linking table. Nothing in `raw/` is
transformed — every cleaning, merging, and feature-construction step happens downstream in
[`clean_data.ipynb`](../notebooks/clean_data.ipynb), which writes the analysis panel to
`intermediate/panel_clean.csv`.

```
data/
├── raw/                 # WRDS downloads — NOT committed (see "Access & licensing")
│   ├── annual.csv       # Compustat annual fundamentals (via CCM)
│   ├── stock.csv        # CRSP monthly stock file
│   └── delisting.csv    # CRSP delisting events
└── intermediate/
    └── panel_clean.csv  # cleaned firm-year panel produced by clean_data.ipynb
```

## Access & licensing

The raw WRDS extracts are **not redistributed in this repository** — CRSP and Compustat are
licensed data. To reproduce the panel you need a WRDS subscription. Download the three files
using the specifications below, place them in `data/raw/`, and run `notebooks/clean_data.ipynb`.
The derived panel `intermediate/panel_clean.csv` is not committed either; it is reproduced by
running that notebook.

## Data sources

| File | WRDS source | Download specification |
|---|---|---|
| `annual.csv` | Compustat Fundamentals Annual via CCM | CCM link types `LC` and `LU`; primary link markers `P` and `C`; fiscal period-end within the link date range. Compustat filters: `indfmt=INDL`, `datafmt=STD`, `popsrc=D`, `consol=C`. The link table supplies `lpermno` directly, so no separate CCM merge is needed downstream. |
| `stock.csv` | CRSP Monthly Stock File (CIZ) | Full CRSP universe — **no** share-code or exchange-code filters, either at download or downstream. `shrcd` and `exchcd` are not available in the CIZ extract, so the conventional ordinary-common-shares screen is not applied explicitly. The effective universe restriction comes from the CCM link: only securities linked to a Compustat firm survive the merge. |
| `delisting.csv` | CRSP Delisting File (CIZ) | One row per security per delisting event: the delisting date, the reason code, and the delisting return. |

## Raw files

> The tables below list the columns the cleaning pipeline actually consumes. CIZ column names
> are shown as downloaded; `clean_data.ipynb` lowercases all headers on load.

### `annual.csv` — Compustat annual fundamentals (one row per firm-fiscal year)

`lpermno` is the link to the CRSP files (`lpermno` = CRSP `Permno`).

| Variable | Type | Description |
|---|---|---|
| `gvkey` | string | Compustat permanent firm identifier |
| `lpermno` | integer | CRSP permanent security number (from the CCM link) |
| `fyear` | integer | Fiscal year |
| `fyr` | integer | Fiscal year-end month (1–12) |
| `sich` | integer | Historical SIC code (used for the financial/utility exclusions) |
| `at` | float | Total assets ($MM) |
| `sale` | float | Net sales / revenue ($MM) |
| `dltt` | float | Long-term debt — total ($MM) |
| `dlc` | float | Debt in current liabilities ($MM) |
| `lt` | float | Total liabilities ($MM) |
| `ni` | float | Net income (loss) ($MM) |
| `oibdp` | float | Operating income before depreciation — EBITDA ($MM) |
| `xint` | float | Interest and related expense — total ($MM) |
| `act` | float | Current assets — total ($MM) |
| `lct` | float | Current liabilities — total ($MM) |
| `che` | float | Cash and short-term investments ($MM) |

`conm` (company name) is convenient to keep in the extract for eyeballing, but the pipeline does
not use it.

The fiscal year-end date (`datadate`) is **reconstructed in code** from `fyear` and `fyr`, so it
is not required in the raw extract. Book equity is computed as `at − lt`.

> **Changed from earlier versions:** the download no longer relies on `ceq`, `revt`, `oancf`,
> `ppent`, or `csho` — book equity is derived from `at − lt` and market capitalization comes
> from CRSP (`MthPrc × ShrOut`), so those Compustat fields are not used. Drop them from the
> extract if you want a leaner file.

### `stock.csv` — CRSP monthly stock file (one row per security-month)

| Variable | Type | Description |
|---|---|---|
| `Permno` | integer | CRSP permanent security number |
| `MthCalDt` | date | Calendar month-end date |
| `MthRet` | float | Monthly holding-period return (incl. distributions) |
| `MthPrc` | float | Month-end closing price (negative = bid-ask average when no close) |
| `ShrOut` | float | Shares outstanding at month-end (thousands) |

Market cap, the trailing 12-month return, and equity volatility are all derived from these
columns downstream (`MthRet` drives the return and volatility windows; `MthPrc × ShrOut`
gives market cap). `MthRetx` (return excluding dividends) is no longer used and can be omitted.

### `delisting.csv` — CRSP delisting events (one row per security delisting)

| Variable | Type | Description |
|---|---|---|
| `Permno` | integer | CRSP permanent security number |
| `DelistingDt` | date | Delisting date |
| `DelReasonType` | string | Abbreviated CIZ reason code (not the legacy numeric codes) |
| `DlRet` | float | Delisting return (available but not currently used) |

Observed `DelReasonType` values include `BKPY` (bankruptcy), `INSC` (insolvency / insufficient
capital), `DELQ` (delinquency — failed exchange requirements), `VIO` (rule violation), `INSF`
(insufficient float), `OFFRE` (tender offer), `PUBI` (going private), `SHLD` (shell company),
the `MV*` family (moved to another exchange), and a long tail of administrative codes.

## Downstream output: `panel_clean.csv` and the default flag

`clean_data.ipynb` merges the three files on `lpermno = Permno`, builds the predictor set,
winsorizes annually (cross-sectional, by fiscal year), and writes the firm-year analysis panel.
The default outcome is constructed there, not in the raw files:

- **`default_flag`** (the single analysis outcome — renamed from the earlier `event_strict`) is
  set to 1 from `DelReasonType ∈ {BKPY, INSC, DELQ}`, assigned to a firm's **last observed
  fiscal year**, and retained only when that fiscal year precedes the delisting year by **one or
  two years**. This keeps predictors strictly before the event (point-in-time) and gives each
  defaulting firm at most one event. The earlier `default_strict` (BKPY only) / `default_broad`
  split has been removed.

## Notes

- **Sample exclusions** (financials, SIC 6000–6999; utilities, SIC 4900–4999) are applied in
  code, not in the raw files.
- **Sample period:** 1980–2022. This describes the extract used for the published results; the
  code does not impose a year filter, so a wider download will produce a larger panel and
  different numbers.
- **Interest coverage** is `oibdp / xint`, set to a high capped value (maximal coverage) when
  interest expense is zero *or missing*, on the reasoning that a firm with no recorded interest
  expense carries no debt-servicing burden. Because interest coverage is one of the required
  non-missing predictors, this keeps those firm-years in the panel rather than dropping them.
- **Equity volatility** is computed from trailing 12-month *monthly* returns (annualized), so the
  CRSP **daily** file is not required. No Merton distance-to-default model is estimated — the
  distance-to-default components (equity volatility, market leverage, log price) enter as separate
  predictors, so `nimtaavg`/geometric-average features are intentionally not built.
- After merging, duplicate `gvkey × fyear` rows (one Compustat firm linked to multiple CRSP
  securities) are resolved by keeping the highest-market-cap share class.
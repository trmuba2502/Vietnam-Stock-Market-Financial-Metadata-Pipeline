# Vietnam Stock Market — Financial Metadata Pipeline

A Python data pipeline that automatically downloads, cleans, and standardizes **10 years of financial statements** for all Vietnamese listed companies (HOSE · HNX · UPCOM).

---

## Overview

| Item | Detail |
|------|--------|
| **Coverage** | ~1,549 listed companies across HOSE, HNX, UPCOM |
| **Period** | 10 fiscal years (annual) |
| **Statements** | Income Statement · Balance Sheet · Cash Flow |
| **Ratios computed** | ROA · ROE · Debt Ratio · Gross Margin · Net Margin · Current Ratio |
| **Data source** | [vnstock](https://github.com/thinh-vu/vnstock) API (VCI / TCBS) |

---

## Project Structure

```
Vietnam-Stock-Market-Financial-Metadata-Pipeline/
│
├── source_code/
│   ├── download_financials.py   # Downloads raw IS, BS, CF statements for every ticker
│   ├── transform_financials.py  # Maps Vietnamese VAS accounts → 46-column English schema + ratios
│   ├── fetch_exchange.py        # Fetches exchange listing (HOSE/HNX/UPCOM) from vnstock
│   ├── build_database.py        # Builds normalized CSV tables
│   ├── update_companies.py      # Refreshes the company master list (tickers & names)
│   ├── update_industries2.py    # Refreshes industry sector classifications
│   └── quality_check.py         # Validates data integrity (FK checks, null rates)
│
├── requirements.txt
└── README.md
```

> `financial_data/` (raw data, CSVs, outputs) and `logs/` are excluded from version control via `.gitignore`.

---

## How It Works

### Step 1 — `download_financials.py`

Iterates over every ticker in `Companies.csv` and downloads 3 financial statements (Income Statement, Balance Sheet, Cash Flow) for each company using the vnstock API.

**Key behaviors:**
- **Resume-safe** — each completed ticker is cached as a separate CSV in `financial_data/temp/`. If the process is interrupted, re-running it automatically skips already-downloaded tickers.
- **Anti-rate-limit** — built-in exponential backoff, randomized delays, and HTTP 429 auto-retry to handle strict API throttling.
- **Flags:**
  - `--retry` — re-attempt previously failed tickers
  - `--test` — run on 10 major tickers only (smoke-test mode)

```bash
python source_code/download_financials.py
python source_code/download_financials.py --retry   # retry failures
python source_code/download_financials.py --test    # quick test
```

**Output:** `financial_data/temp/<ticker>_*.csv` (one file per company per statement type)

---

### Step 2 — `transform_financials.py`

Aggregates all per-ticker CSVs from `temp/` into a single clean master file.

**Key behaviors:**
- Maps raw Vietnamese account names to a standardized 46-column English schema using a regex + substring keyword dictionary.
- Handles both **general VAS** (manufacturing, services) and **banking/securities VAS** account block differences automatically.
- Computes financial ratios: ROA, ROE, Debt Ratio, Gross Margin, Net Profit Margin, Current Ratio.
- Attaches company metadata (CompanyID, IndustryID) via join with `Companies.csv`.

```bash
python source_code/transform_financials.py
```

**Output:** `financial_data/Financials_Clean.csv` — 46 columns, ~14,500 rows (company × fiscal year)

---

### Step 3 — `fetch_exchange.py` *(run once)*

Fetches the real exchange listing for each stock (HOSE / HNX / UPCOM) directly from the vnstock API and writes the `exchange_id` column into `Companies.csv`.

```bash
python source_code/fetch_exchange.py
```

**Output:** `exchange_id` column added to `financial_data/Companies.csv`

---

### Step 4 — `build_database.py`

Reads `Financials_Clean.csv`, `Companies.csv`, and `Industries.csv` and produces a fully normalized relational database output.

```bash
python source_code/build_database.py
```

**Output** in `financial_data/final/`:

| File | Description |
|------|-------------|
| `01_industries.csv` | 24 industry sectors |
| `02_companies.csv` | 1,549 companies with exchange |
| `03_stock_exchanges.csv` | HOSE / HNX / UPCOM |
| `04_account_dictionary.csv` | VAS account code lookup (IS/BS/CF) |
| `05_income_statements.csv` | Annual P&L per company |
| `06_balance_sheets.csv` | Annual balance sheet per company |
| `07_cash_flows.csv` | Annual cash flow per company |
| `08_financial_ratios.csv` | Pre-computed ratios per company |
| `vietnam_financial_db.sql` | PostgreSQL DDL + `\COPY` load script |
| `schema_details.md` | Full schema documentation |

---

### Step 5 — `quality_check.py`

Validates the integrity of the output:
- Checks that every ticker in `Financials_Clean.csv` has a matching entry in `Companies.csv` (FK integrity).
- Reports missing value rates per column.

```bash
python source_code/quality_check.py
```

---

## Metadata Utilities

| Script | Purpose |
|--------|---------|
| `update_companies.py` | Re-fetches the full ticker list from vnstock and updates `Companies.csv` |
| `update_industries2.py` | Re-fetches GICS-style industry classifications and updates `Industries.csv` |

Run these before `download_financials.py` if you want the latest company/sector data.

```bash
python source_code/update_companies.py
python source_code/update_industries2.py
```

---

## Pipeline Flow

```
vnstock API
    │
    ├─► update_companies.py    ──► Companies.csv    (ticker master)
    ├─► update_industries2.py  ──► Industries.csv   (sector master)
    └─► fetch_exchange.py      ──► Companies.csv    (+ exchange_id)
    │
    ▼
download_financials.py  ──►  financial_data/temp/   (raw per-ticker CSVs)
    │
    ▼
transform_financials.py ──►  Financials_Clean.csv   (46-col clean master)
    │
    ▼
build_database.py       ──►  financial_data/final/  (normalized tables + SQL)
    │
    ▼
quality_check.py        ──►  console report          (integrity validation)
```

---

## Requirements

| Package | Version |
|---------|---------|
| `vnstock` | ≥ 0.4.0 |
| `pandas` | ≥ 1.5.0 |
| `tqdm` | ≥ 4.60.0 |
| `requests` | ≥ 2.28.0 |

```bash
pip install -r requirements.txt
```

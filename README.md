# Vietnam Stock Market — Financial Metadata Pipeline

A complete end-to-end data engineering pipeline that downloads, cleans, and transforms **10 years of financial statements** for all Vietnamese listed companies (HOSE · HNX · UPCOM) into a **3NF-normalized PostgreSQL database** ready for analysis and modeling.

---

## Overview

| Item | Detail |
|------|--------|
| **Coverage** | ~1,549 listed companies across HOSE, HNX, UPCOM |
| **Period** | 10 fiscal years (annual) |
| **Statements** | Income Statement · Balance Sheet · Cash Flow |
| **Ratios** | ROA · ROE · Debt Ratio · Gross Margin · Net Margin · Current Ratio |
| **Output** | 8-table 3NF PostgreSQL schema + normalized CSV files |
| **Data Source** | [vnstock](https://github.com/thinh-vu/vnstock) API (VCI / TCBS) |

---

## Project Structure

```
Vietnam-Stock-Market-Financial-Metadata-Pipeline/
│
├── source_code/
│   ├── download_financials.py   # Downloads raw IS, BS, CF statements for every ticker
│   ├── transform_financials.py  # Maps Vietnamese VAS accounts → 46-column English schema + ratios
│   ├── fetch_exchange.py        # Fetches exchange listing (HOSE/HNX/UPCOM) from vnstock
│   ├── build_database.py        # Builds 8 normalized CSV tables + PostgreSQL DDL script + schema docs
│   ├── update_companies.py      # Refreshes Companies.csv (ticker list)
│   ├── update_industries2.py    # Refreshes Industries.csv (sector classification)
│   └── quality_check.py         # Validates FK integrity and missing-value rates
│
├── requirements.txt
└── README.md
```

> `financial_data/` (raw data, CSVs, database output) and `logs/` are excluded from version control via `.gitignore`. Create the `financial_data/` folder locally before running the pipeline.

---

## Database Schema (3NF — 8 Tables)

```
stock_exchanges ──< companies >── industries
                         │
              ┌──────────┴──────────┐
              │                     │
    income_statements          balance_sheets
    cash_flows                 financial_ratios

account_dictionary  (standalone VAS account lookup)
```

| # | Table | Rows | Key |
|---|-------|-----:|-----|
| 1 | `industries` | 24 | `industry_id` |
| 2 | `stock_exchanges` | 3 | `exchange_id` |
| 3 | `companies` | 1,549 | `company_id` |
| 4 | `account_dictionary` | 31 | `account_id` (e.g. `IS-01`) |
| 5 | `income_statements` | ~14,500 | `(company_id, fiscal_year)` |
| 6 | `balance_sheets` | ~14,500 | `(company_id, fiscal_year)` |
| 7 | `cash_flows` | ~14,500 | `(company_id, fiscal_year)` |
| 8 | `financial_ratios` | ~14,500 | `(company_id, fiscal_year)` |

> Full column-level documentation: [`financial_data/final/schema_details.md`](financial_data/final/schema_details.md)

---

## Quickstart

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. (First time only) Download raw financial statements

```bash
python source_code/download_financials.py
```

- Resume-safe: already-downloaded tickers are cached in `financial_data/temp/` and skipped automatically.
- Use `--retry` to re-attempt previously failed tickers.
- Use `--test` to run on 10 major tickers for a quick smoke-test.

### 3. Transform raw data → clean master file

```bash
python source_code/transform_financials.py
```

Produces `financial_data/Financials_Clean.csv` — a 46-column, English-schema master file.

### 4. Fetch real exchange listings (HOSE / HNX / UPCOM)

```bash
python source_code/fetch_exchange.py
```

Downloads accurate exchange data from vnstock and updates `Companies.csv` with `exchange_id`. This only needs to be run once (or when the listing changes).

### 5. Build the normalized database

```bash
python source_code/build_database.py
```

Generates all 8 CSV tables, the PostgreSQL script, and schema documentation in `financial_data/final/`.

### 6. Load into PostgreSQL

```bash
cd financial_data/final
psql -U your_user -d your_database -f vietnam_financial_db.sql
```

The SQL script uses `\COPY` (psql client-side) to load data directly from the CSV files — no INSERT bloat.

### 7. Validate data quality

```bash
python source_code/quality_check.py
```

---

## Pipeline Summary

```
vnstock API
    │
    ▼
download_financials.py  ──►  financial_data/temp/  (raw per-ticker CSVs)
    │
    ▼
transform_financials.py ──►  Financials_Clean.csv  (46-col clean master)
    │
    ├── fetch_exchange.py ──►  Companies.csv  (+ exchange_id column)
    │
    ▼
build_database.py       ──►  financial_data/final/
                               ├── 01–08 *.csv  (normalized tables)
                               ├── vietnam_financial_db.sql
                               └── schema_details.md
```

---

## Key Features

- **Resume-safe downloads** — Cached per-ticker CSVs let the pipeline continue from exactly where it left off after a crash or rate-limit block.
- **VAS account mapping** — A regex + substring dictionary automatically reconciles Vietnam Accounting Standards (both general and banking/securities blocks) into the unified English schema.
- **Accurate exchange data** — Exchange listing (HOSE/HNX/UPCOM) is fetched directly from the vnstock API, not guessed from ticker patterns.
- **3NF normalized database** — Redundancy-free schema with proper FK constraints, ready to load into any PostgreSQL instance.
- **Anti-rate-limit handling** — Exponential backoff, random delays, and HTTP 429 auto-retry keep the download stable against strict API limits.

---

## Example Queries

```sql
-- Top 10 companies by ROE in 2024
SELECT c.ticker, c.company_name, fr.roe_pct
FROM financial_ratios fr
JOIN companies c ON c.company_id = fr.company_id
WHERE fr.fiscal_year = 2024
ORDER BY fr.roe_pct DESC NULLS LAST
LIMIT 10;

-- Revenue and net income trend for VCB
SELECT fiscal_year, revenue, net_income
FROM income_statements
WHERE company_id = (SELECT company_id FROM companies WHERE ticker = 'VCB')
ORDER BY fiscal_year;

-- Average debt ratio by industry in 2023
SELECT i.industry_name, ROUND(AVG(fr.debt_ratio)::NUMERIC, 4) AS avg_debt_ratio
FROM financial_ratios fr
JOIN companies c ON c.company_id = fr.company_id
JOIN industries i ON i.industry_id = c.industry_id
WHERE fr.fiscal_year = 2023
GROUP BY i.industry_name
ORDER BY avg_debt_ratio DESC;
```

---

## Requirements

| Package | Version |
|---------|---------|
| `vnstock` | ≥ 0.4.0 |
| `pandas` | ≥ 1.5.0 |
| `tqdm` | ≥ 4.60.0 |
| `requests` | ≥ 2.28.0 |

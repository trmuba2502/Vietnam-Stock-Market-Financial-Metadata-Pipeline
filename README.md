# Vietnam Stock Market Financial Metadata Pipeline

This project is a complete Data Pipeline designed to scrape, clean, and transform the financial statements of all listed companies across Vietnam's three major stock exchanges (HOSE, HNX, UPCOM) covering the most recent 10-year period.

The primary objective is to extract raw financial data in Vietnamese and standardize it into a **46-component English Schema** to facilitate data analysis, quantitative research, and Machine Learning modeling.

---

## Project Structure

The repository is modularized into independently functioning components as follows:

```text
Database-Management/
│
├── source_code/                 # Contains all automated source code scripts
│   ├── download_financials.py      # Core script: Downloads Financial Statements (Income, Balance, Cashflow) with Resume/Retry built-in to prevent API blocks.
│   ├── transform_financials.py     # Core script: Processes raw data, maps Vietnamese keywords to the English Schema, and calculates financial ratios.
│   ├── update_industries2.py       # Utility: Updates the industry classification categories using a reliable Github source (English Sectors).
│   ├── update_companies.py         # Utility: Updates the list of stock tickers and company names.
│   ├── quality_check.py            # Utility: Validates final data quality (checks for invalid IDs and missing values).
│   ├── read_pdf.py                 # Utility: Script used to read the initial schema definition from the PDF.
│   └── ...                         # Other testing, patching, and backup scripts.
│
├── financial_data/              # Contains the final structured CSV database
│   ├── Companies.csv               # Company Metadata Lookup Table (CompanyID, Ticker, Name, IndustryID).
│   ├── Industries.csv              # Standardized Industry Classification Table (IndustryID, IndustryName).
│   ├── Financials_Clean.csv        # The MASTER File - Standardized 46-column financial data along with computed ratios.
│   └── temp/                    # Directory for caching raw individual financial statement CSVs during the download process.
│
├── logs/                        # Execution logs to track downloading progress and catch errors.
│
├── requirements.txt                # List of required Python dependencies (vnstock, pandas, tqdm).
└── README.md                       # Project overview and instructions.
```

## Workflow / How to Use

To update and extract data, follow the pipeline execution order below:

### 1. Install Dependencies
Ensure you have your virtual environment activated (if applicable) and install the necessary packages:
```bash
pip install -r requirements.txt
```

### 2. Update Metadata (Company Names, Industry Sectors)
Update the lists of companies and industry groups. This will establish the lookup tables required for ID mapping.
```bash
python source_code/update_industries2.py
```
*(If you need to update the basic ticker lists first, you can use `update_companies.py`)*

### 3. Start the Download Pipeline
The download process supports a persistent caching mechanism. Successfully downloaded tickers are automatically saved into the `temp/` folder. If you encounter a network issue or the process crashes, simply run this script again—it will intelligently skip the companies it has already downloaded.
```bash
python source_code/download_financials.py
```
*Tip: Provide the `--retry` flag to re-attempt failed downloads, or `--test` to scrape a sample of 10 major tickers.*

### 4. Transform Data into Master File
Once the download is complete, run the transform file to aggregate the data, map the Vietnamese terms to the English schema, compute metrics, and generate `Financials_Clean.csv`.
```bash
python source_code/transform_financials.py
```

### 5. Quality Assurance Check
Run the quality check script to verify that every ticker present in the clean data table has a valid foreign key reference within the Companies lookup table.
```bash
python source_code/quality_check.py
```

## Key Backend Features
- **Zero Data Loss**: Automatically detects partial caches and supports continuing from exactly where it left off, circumventing crashes.
- **Robust Anti-Bot Handling**: Integrated exponential backoffs, random delays, and HTTP 429 scale-retries to deal with strict API rate limits.
- **Advanced Automated Mapping**: The pipeline relies on a powerful Regex and substring dictionary approach to seamlessly reconcile Vietnam Accounting Standards (both Non-Financial and Banking/Securities Blocks) mapping them consistently into the unified 46-column structure.

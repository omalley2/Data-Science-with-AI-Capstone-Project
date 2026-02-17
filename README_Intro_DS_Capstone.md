# Intro to Data Science Capstone

## Project Title
**Bank Health Segmentation and ROA Forecasting with FDIC + FRED Data**

This README documents the capstone analysis in `Intro_DS_Capstone.ipynb` and is written for instructors/reviewers who need to evaluate, rerun, or replicate results.

---

## 1) Capstone Questions and Scope

### Primary Questions
1. Can active U.S. banks be segmented into meaningful operating profiles using FDIC financial metrics?
2. How accurately can next-period bank ROA be forecast using lagged bank metrics plus macro indicators?

### Scope
- Unit of analysis: active banks in the project dataset
- Time alignment: bank performance joined with macro variables at month level
- Target for prediction: next-period ROA

---

## 2) Project Requirements (and How This Project Meets Them)

These are the capstone expectations from the course starter notebook/rubric, mapped to this project:

1. **Clear, answerable question**
  - Defined above with measurable outcomes (forecast error + interpretable clusters).

2. **Appropriate data and documented quality checks**
  - Uses public, reproducible sources (FDIC + FRED).
  - Notebook includes missingness checks, duplicates, dtype validation, and feature definitions.

3. **Methods matched to question, including baseline**
  - Baseline: linear regression.
  - Iterations: winsorized linear regression, random forest (untuned/tuned), XGBoost, Prophet, and KMeans clustering.

4. **Evaluation with honest limitations**
  - Holdout metrics: RMSE, MAE, R².
  - Overfitting diagnostics via train-vs-test gap.
  - Explicit limitations and caveats included in notebook conclusions.

5. **Clear communication**
  - Notebook narrative is organized from question → data → methods → results → recommendations.

6. **Reproducibility**
  - Ordered notebook execution.
  - Environment + dependencies documented.
  - CSV exports provided for re-run without live API pulls.

### Course Deliverables Covered
- Capstone notebook with saved outputs
- Written summary (this README + notebook conclusion sections)
- Data source documentation and citations

---

## 3) APIs Used

### A) FDIC BankFind API (Bank Financials)
- Base docs: https://banks.data.fdic.gov/docs/
- Purpose in project: bank institution metadata + quarterly financial statement metrics
- Integration file: `fdic_bank_api.py`
- Endpoints used via helper:
  - `/api/institutions` (bank lookup/search by CERT/name/state)
  - `/api/financials` (historical bank financial records)

**Examples of FDIC-derived fields used in analysis**
- `total_assets`, `total_deposits`, `total_loans`
- `roa`, `roe`, `nim`
- `efficiency_ratio`, `tier1_capital_ratio`

### B) FRED API (Macroeconomic Series)
- Base docs: https://fred.stlouisfed.org/docs/api/fred/
- Python client used: `fredapi`
- Purpose in project: macro context joined to bank panel

**Core FRED series family used in project docs/code**
- Rates: `DFF`, `DGS10`, `DGS2`, `T10Y2Y`
- Macro: `UNRATE`, `GDPC1`, `CPIAUCSL`, `UMCSENT`
- Delinquency examples: `DRCCLACBS`, `DRSFRMACBS`, `DRCLACBS`

---

## 4) Data Artifacts

### Source/Working Data
- `data/bank_data.csv` (FDIC pull output)
- `data/fred_data.csv` (FRED pull output)

### Database Tables Used in Notebook
- `bank_performance`
- `economic_data`

### Exported Reproducibility Files
The export script writes timestamped files to `data/exports/`:
- `bank_performance_YYYYMMDD_HHMMSS.csv`
- `economic_data_YYYYMMDD_HHMMSS.csv`
- `capstone_joined_active_YYYYMMDD_HHMMSS.csv`

---

## 5) Notebook Workflow (High Level)

1. Setup and data load (from PostgreSQL)
2. Data audit and cleaning
3. Feature engineering (lags, changes, next-period target)
4. EDA and correlation inspection
5. Baseline model (linear regression)
6. Robustness step (train-only winsorization)
7. Model comparison (RF, tuned RF, XGBoost, Prophet)
8. Clustering (StandardScaler + KMeans + silhouette + PCA)
9. Evaluation, limitations, recommendations

---

## 6) Key Results Snapshot

### Forecasting
- Best pooled holdout model: **winsorized linear regression**
  - RMSE: **0.3531**
  - MAE: **0.2429**
  - R²: **0.0357**
- Random forest variants were slightly worse on holdout.
- XGBoost and Prophet showed larger train-test gaps (overfitting in this setup).

### Clustering
- Best `k` by silhouette: **3**
- Cluster membership:
  - Cluster 0: Bank of America, Citibank, JPMorgan Chase, Wells Fargo
  - Cluster 1: Capital One, TD Bank
  - Cluster 2: Goldman Sachs Bank, Huntington National Bank, U.S. Bank

---

## 7) Instructor Replication Guide (CSV-Based)

This section is designed so your instructor can replicate analysis **without rerunning API pulls**.

### Option A — Full DB + Notebook Reproduction (closest to author run)

1. Install Python dependencies:

```bash
pip install -r requirements.txt
```

2. Ensure PostgreSQL is running and `.env` is configured (`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`).

3. Build and seed DB:

```bash
psql -d bank_lending_db -f db/schema.sql
python3 db/seed_database.py
```

4. Open and run `Intro_DS_Capstone.ipynb` top-to-bottom.

### Option B — CSV-Only Reproduction (no live DB required)

Use the latest joined export in `data/exports/` and run modeling/clustering sections from that dataframe.

1. Install Python dependencies:

```bash
pip install -r requirements.txt
```

2. In notebook setup, load the latest joined export:

```python
from pathlib import Path
import pandas as pd

export_dir = Path('data/exports')
latest = sorted(export_dir.glob('capstone_joined_active_*.csv'))[-1]
df = pd.read_csv(latest, parse_dates=['bank_date', 'econ_date'])

# optional alignment for downstream cells expecting `date`
if 'date' not in df.columns:
   df['date'] = df['bank_date']
```

3. Continue with the notebook’s feature engineering, modeling, and clustering logic.

4. If a cell references SQL-specific load variables, point it to `df` from step 2.

### If You Need Fresh Exports

```bash
python3 scripts/export_postgres_csv.py
```

This regenerates all three reproducibility CSVs in `data/exports/`.

---

## 8) Reproducibility and Evaluation Notes

- Train-only preprocessing is used for winsorization to reduce leakage.
- Time-aware splitting is used where relevant.
- Random seeds are set where model randomness applies.
- Final recommendations prioritize holdout behavior over train performance.

---

## 9) Limitations

- Month-level FDIC/FRED alignment may smooth short-lived economic shocks.
- Bank-specific events (policy, litigation, mergers) are not fully captured in features.
- Prophet run is on aggregated series and does not preserve full bank-level panel detail.

---

## 10) File Map for Reviewers

- `Intro_DS_Capstone.ipynb` — primary analysis deliverable
- `README_Intro_DS_Capstone.md` — capstone summary and replication instructions
- `fdic_bank_api.py` — FDIC API helper and major-bank cert references
- `db/fetch_major_bank_data.py` — FDIC data extraction to CSV
- `scripts/export_postgres_csv.py` — PostgreSQL-to-CSV reproducibility export
- `data/exports/` — timestamped CSV artifacts for independent reruns

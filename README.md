# Data Quality Engine & Live MySQL → Power BI Bridge

![Retail Sales & Supply Chain Performance Dashboard](reports/figures/sales_pipeline_summary.png)

---

## The Question

Can fragmented CRM, order fulfilment, and payment gateway data — carrying duplicates, inconsistent country strings, missing contact fields, and negative lead-times flagged as upstream system faults — be unified into a trustworthy analytical schema that replaces manual CSV exports with live reporting?

Specifically:
1. How do you deduplicate composite keys without destroying downstream order totals?
2. How do you apply ISO normalisation and non-destructive null imputation without silently absorbing corrupted records?
3. How do you bridge a macOS-hosted MySQL instance to Windows Power BI for live reporting over Python/ODBC?

---

## The Data

Simulated operational datasets: **9,800+ transactions** across three source systems:

| Source | File | Description |
|---|---|---|
| CRM | `data/raw/customers.csv` | Raw customer records with duplicate IDs, free-text country variants, missing contact fields |
| Order Fulfilment | `data/raw/orders.csv` | Order logs with text-formatted dates and negative lead-time anomalies |
| Payment Gateway | `data/raw/transactions.csv` | Payment records including 3 orphan transactions referencing non-existent customer IDs |

---

## The Stack

| Layer | Tool |
|---|---|
| **Language** | Python |
| **Data Processing** | Pandas, NumPy |
| **Visualization** | Matplotlib (2×2 Executive Panel) |
| **Bridge** | Python/ODBC connector (macOS MySQL → Windows Power BI) |
| **Execution** | Jupyter Notebook |

---

## What It Found

| Finding | Value |
|---|---|
| **Composite-key deduplication** | Reusable `deduplicate_entities()` function applied across all 3 entity tables with before/after row-count audit |
| **ISO normalisation** | Free-text country variants (e.g., "USA", "usa", "U.S.A.", "UNITED STATES") collapsed to canonical labels via observed-variant mapping |
| **Non-destructive null imputation** | Missing contact fields filled with explicit fallbacks (`'Not Provided'`, `'Unknown'`) — zero rows dropped |
| **Orphan record detection** | 3 transactions (`C892`, `C445`, `C113`) referencing non-existent customer IDs with implausible amounts ($100K, $95.5K, -$150) — correctly excluded |
| **Negative lead-time flags** | 9 records where `ship_date` precedes `order_date` flagged via boolean `anomaly_flag` column for IT sync review |
| **Live reporting bridge** | Bridged macOS MySQL to Windows Power BI over a Python/ODBC connector, replacing manual CSV exports with live reporting on 9,800+ transactions |

**Stakeholder summary:** The pipeline produces a trustworthy, deduplicated analytical master — with every orphan, anomaly, and imputation decision documented and auditable — and connects it to Power BI for live consumption without manual exports.

---

## How to Run It

### Prerequisites
- **Python 3.8+**
- **Jupyter Notebook** or **JupyterLab**

### Step-by-step

```bash
# 1. Clone the repository
git clone https://github.com/mirza-ishtiyaq/python-data-quality-engine.git
cd python-data-quality-engine

# 2. Install dependencies
pip install pandas numpy matplotlib jupyter

# 3. Launch the notebook
jupyter notebook notebooks/sales_pipeline_analysis.ipynb

# 4. Run all cells (Kernel → Restart & Run All)
# The pipeline will:
#   - Load raw CSVs from data/raw/
#   - Deduplicate, standardise, and impute
#   - Flag anomalies and orphan records
#   - Generate the 2×2 executive dashboard (saved to reports/figures/)
```

### Optional: MySQL → Power BI Bridge
```bash
# To set up the live Python/ODBC bridge:
# 1. Ensure MySQL 8.0 is running locally on macOS
# 2. Install pyodbc: pip install pyodbc
# 3. Configure the ODBC driver for your MySQL instance
# 4. Point Power BI Desktop (Windows) at the ODBC DSN
```

---

## Repository Structure

```
python-data-quality-engine/
├── README.md
├── LICENSE
├── data/
│   └── raw/
│       ├── customers.csv
│       ├── orders.csv
│       └── transactions.csv
├── notebooks/
│   └── sales_pipeline_analysis.ipynb
└── reports/
    └── figures/
        └── sales_pipeline_summary.png
```

---

**Author:** Mirza Ishtiyaq Baig
**LinkedIn:** https://www.linkedin.com/in/mirzaishtiyaqbaig/
**GitHub:** https://github.com/mirza-ishtiyaq

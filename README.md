# Retail Pipeline Data Cleaning & Exploratory Analytics (Python & Pandas)

## Executive Summary & Business Context
In multi-channel retail systems, operational data is fragmented across customer relationship management (CRM) software, e-commerce order fulfillment databases, and payment processing gateways. Inconsistent data entry, system synchronization lag, and missing attributes frequently obscure true financial performance.

In this project, I built an end-to-end Python and Pandas data cleansing, integration, and exploratory analytics pipeline. By transforming raw, uncleaned transactional extracts into a unified analytical schema, I surfaced core revenue metrics, customer spend behaviors, and fulfillment SLA performance.

---

## DA / BA Core Competencies & Technical Skills

`Python` `Pandas` `NumPy` `Matplotlib` `Exploratory Data Analysis (EDA)` `Data Cleansing & Sanitization` `Primary Key Deduplication` `Categorical Standardization` `Missing Value Imputation` `Relational Joins & Schema Merging` `Time-Series Analysis` `Operational Anomaly Detection` `Business Metrics (AOV, ATV, LTV)`

---

## Technical Stack & Repository Architecture
* **Language:** Python
* **Data Processing:** Pandas, NumPy
* **Data Visualization:** Matplotlib (2×2 Executive Panel)
* **Execution Environment:** Jupyter Notebook (`sales_pipeline_analysis.ipynb`)

```
python-data-quality-engine/
├── README.md                                          # Project documentation & business insights
├── LICENSE                                            # MIT Open Source License
├── data/
│   └── raw/
│       ├── customers.csv                              # Raw CRM customer records
│       ├── orders.csv                                 # Raw order fulfillment logs
│       └── transactions.csv                           # Raw payment gateway transactions
├── notebooks/
│   └── sales_pipeline_analysis.ipynb                  # Main Python/Pandas data pipeline notebook
└── reports/
    └── figures/
        └── sales_pipeline_summary.png                 # Saved 2×2 executive dashboard export
```

---

## Data Cleaning & Engineering Deep-Dive

### 1. Primary Key Deduplication (Modular Function)
Asynchronous database writes generated duplicate records for customer entities (e.g., `C001`, `C002`, `C010`).
* **Methodology:** Wrapped `drop_duplicates(subset=[id_col], keep='first')` in a reusable `deduplicate_entities()` function applied across all three entity tables, with a before/after row-count audit printed on every call.

```python
def deduplicate_entities(df: pd.DataFrame, id_col: str) -> pd.DataFrame:
    """Deduplicate entity records based on primary key constraints."""
    initial_count = len(df)
    cleaned_df = df.drop_duplicates(subset=[id_col], keep='first').copy()
    dropped_count = initial_count - len(cleaned_df)
    print(f'[{id_col}] Deduplication: {initial_count} -> {len(cleaned_df)} rows ({dropped_count} duplicates removed)')
    return cleaned_df

customers_clean = deduplicate_entities(customers_raw, 'customer_id')
orders_clean = deduplicate_entities(orders_raw, 'order_id')
transactions_clean = deduplicate_entities(transactions_raw, 'transaction_id')
```

### 2. Categorical String Normalization (Country Attributes)
Country attributes contained free-text string variations (e.g., `"USA"`, `"usa"`, `"United States"`, `"US"`, `"U.S.A."`, `"UNITED STATES"`).
* **Methodology:** Constructed a standardization dictionary map (`country_map`) covering only the raw variants actually observed in `customers.csv` (not a full ISO-3166 table) and applied it through a reusable `standardize_countries()` function.

```python
def standardize_countries(df: pd.DataFrame, column: str, mapping: dict) -> pd.DataFrame:
    """Collapse free-text country variants onto a single canonical label per country."""
    df = df.copy()
    df[column] = df[column].replace(mapping)
    return df

country_map = {
    'United States': 'USA', 'usa': 'USA', 'US': 'USA', 'U.S.A.': 'USA', 'UNITED STATES': 'USA',
    'United Kingdom': 'UK', 'United kingdom': 'UK', 'united kingdom': 'UK', 'Great Britain': 'UK',
    'United Arab Emirates': 'UAE'
}
customers_clean = standardize_countries(customers_clean, 'country', country_map)
```

### 3. Non-Destructive Missing Value Imputation (Modular Function)
Optional contact attributes (`email`, `phone_primary`, `phone_secondary`, `city`) contained missing (`NaN`) values.
* **Methodology:** Rather than dropping rows (which would corrupt downstream order totals), missing fields were safely imputed via a reusable `impute_missing()` function using explicit string fallbacks (`fillna('Not Provided')` / `fillna('Unknown')`), maintaining row count integrity.

```python
def impute_missing(df: pd.DataFrame, fill_values: dict) -> pd.DataFrame:
    """Fill missing (NaN) values column-by-column with explicit fallbacks."""
    df = df.copy()
    for col, fallback in fill_values.items():
        if col in df.columns:
            df[col] = df[col].fillna(fallback)
    return df

customers_clean = impute_missing(customers_clean, {
    'email': 'Not Provided', 'phone_primary': 'Not Provided',
    'phone_secondary': 'Not Provided', 'city': 'Unknown',
})
orders_clean = impute_missing(orders_clean, {'email': 'Not Provided'})
```

### 4. Relational Data Merging & Referential Integrity Validation
Combined the cleaned `customers` and `orders` tables into a unified order-level master, and separately validated `transactions` against the customer master.
* **Methodology:** `transactions.csv` carries only `customer_id` (no `order_id`), so joining orders and transactions on `customer_id` alone would fan out into a full cross-product per customer — every order paired with every transaction for that customer — silently inflating summed/averaged revenue. Instead, `full_data` is built from a single `customers` ⋈ `orders` left join, and transactions are validated independently via `filter_to_known_customers()`, which also surfaced 3 orphan transaction rows (`C892`, `C445`, `C113`) referencing customer IDs that don't exist anywhere in `customers.csv`, carrying implausible amounts (`$100,000`, `$95,500`, `-$150`) — corrupted gateway records, correctly excluded rather than joined in.

```python
def filter_to_known_customers(df, valid_ids, id_col='customer_id'):
    """Split a table into (matched, orphaned) rows based on a foreign-key column."""
    is_known = df[id_col].isin(valid_ids)
    return df[is_known].copy(), df[~is_known].copy()

transactions_clean = transactions_clean.rename(columns={'purchase_amount': 'transaction_amount'})

# Order-level master: one row per order, no fan-out risk
full_data = pd.merge(customers_clean, orders_clean, on='customer_id', how='left')

# Transactions validated against the customer master, not merged into full_data
transactions_matched, transactions_orphaned = filter_to_known_customers(
    transactions_clean, set(customers_clean['customer_id'])
)
```

### 5. Temporal Lead-Time Calculation & Anomaly Detection
Calculated fulfillment lead-times (`ship_date - order_date`) to audit logistics SLAs.
* **Methodology:** Converted string date fields to `datetime64[ns]` objects, calculated transit duration via `.dt.days`, and wrote an explicit boolean `anomaly_flag` column onto the master frame (rather than only filtering into separate frames) so every negative-lead-time record — where `ship_date` precedes `order_date`, a sync/entry error — stays flagged for audit without being dropped from the financial data. The notebook prints a dedicated anomaly audit table listing every flagged `customer_id` / `order_id` pair.

```python
# Datetime conversion and lead-time calculation
full_data['ship_days'] = (full_data['ship_date'] - full_data['order_date']).dt.days
full_data['anomaly_flag'] = full_data['ship_days'] < 0

valid_shipments = full_data[~full_data['anomaly_flag']]
anomalous_shipments = full_data[full_data['anomaly_flag']]
```

---

## Business Analytics & Executive Insights

*All figures below are pulled directly from a headless execution of `sales_pipeline_analysis.ipynb` against the raw CSVs in `data/raw/` — not estimated.*

| Metric Category | Business Indicator | Value / Finding |
| :--- | :--- | :--- |
| **Financial Performance** | Average Order Value (AOV) | **$78.08** per order (49 order-level records) |
| **Financial Performance** | Average Transaction Value (ATV) | **$107.77** per transaction (20 validated records; 3 orphan transactions with unknown customer IDs excluded) |
| **Financial Performance** | Total Gross Revenue | **$3,826.12** across the cleaned order master |
| **Geographic Distribution** | Top Revenue Country | **USA** ($1,140.57), followed by **UK** ($1,023.55) & **Canada** ($670.00) |
| **Logistics SLA** | Clean Shipping Duration | **2.92 Days** average transit lead-time (40 valid records) |
| **Logistics SLA** | Shipping Speed Range | **2 Days** (fastest) to **3 Days** (slowest) among valid shipments |
| **Data Quality Audit** | System Anomaly Flags | **9** negative-lead-time records isolated via `anomaly_flag` and printed in a dedicated audit table for IT sync review |

### Executive Dashboard
![Retail Sales & Supply Chain Performance Dashboard](reports/figures/sales_pipeline_summary.png)

*2×2 panel generated by the notebook's final cell: top 5 customers by lifetime spend, revenue by country, monthly revenue trajectory, and the fulfillment lead-time distribution for valid (non-anomalous) shipments.*

---

## Author & Project Info
**Author:** Mirza Ishtiyaq Baig — Data Analyst / Analytics Engineer
**LinkedIn:** https://www.linkedin.com/in/mirzaishtiyaqbaig/
**Email:** mirzaishtiyaqbaig1@gmail.com
**GitHub:** https://github.com/mirza-ishtiyaq

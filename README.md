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
data-cleaning-analysis-pandas/
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
    └── figures/                                       # Saved dashboard visual exports
```

---

## Data Cleaning & Engineering Deep-Dive

### 1. Primary Key Deduplication & Coalescing
Asynchronous database writes generated duplicate records for customer entities (e.g., `C001`, `C002`, `C010`).
* **Methodology:** Applied `drop_duplicates(subset=['customer_id'], keep='first')` across entity tables to establish clean primary key constraints.
* **Custom Aggregation:** Implemented a `first_valid()` custom fallback function over grouped customer records to preserve non-null attributes across redundant rows.

```python
def first_valid(series):
    """Retrieve first non-null entry within a grouped attribute series."""
    for val in series:
        if pd.notna(val):
            return val
    return None

# Deduplicate customer entities via custom aggregation
customers_clean = customers.groupby('customer_id', as_index=False).agg(first_valid)
```

### 2. Categorical String Normalization (Country Attributes)
Country attributes contained free-text string variations (e.g., `"USA"`, `"usa"`, `"United States"`, `"US"`, `"U.S.A."`, `"UNITED STATES"`).
* **Methodology:** Constructed a standardization dictionary map (`country_map`) and executed vectorized string replacements (`replace()`) to enforce clean ISO country naming across all records.

```python
country_map = {
    'United States': 'USA', 'usa': 'USA', 'US': 'USA', 'U.S.A.': 'USA', 'UNITED STATES': 'USA',
    'United Kingdom': 'UK', 'United kingdom': 'UK', 'united kingdom': 'UK', 'Great Britain': 'UK',
    'United Arab Emirates': 'UAE'
}

# Apply dictionary mapping for categorical standardization
customers_cleaned['country'] = customers_cleaned['country'].replace(country_map)
```

### 3. Non-Destructive Missing Value Imputation
Optional contact attributes (`email`, `phone_primary`, `phone_secondary`, `city`) contained missing (`NaN`) values.
* **Methodology:** Rather than dropping rows (which would corrupt downstream order totals), missing fields were safely imputed using explicit string fallbacks (`fillna('Not Provided')` and `fillna('Unknown')`), maintaining row count integrity.

```python
# Non-destructive imputation for missing attributes
customers_cleaned['email'] = customers_cleaned['email'].fillna('Not Provided')
customers_cleaned['phone_primary'] = customers_cleaned['phone_primary'].fillna('Not Provided')
customers_cleaned['city'] = customers_cleaned['city'].fillna('Unknown')
```

### 4. Relational Data Merging & Schema Alignment
Combined 3 cleaned entity tables (`customers`, `orders`, `transactions`) into a unified master dataset.
* **Methodology:** Performed sequential left joins (`pd.merge(..., on='customer_id', how='left')`) and resolved column name collisions (renaming `purchase_amount` in transactions to `transaction_amount`).

```python
# Resolve column collision prior to merge
transactions_cleaned = transactions_cleaned.rename(columns={'purchase_amount': 'transaction_amount'})

# Left join customer master with order logs and transaction details
customers_orders = pd.merge(customers_cleaned, orders_cleaned, on='customer_id', how='left')
full_data = pd.merge(customers_orders, transactions_cleaned, on='customer_id', how='left')
```

### 5. Temporal Lead-Time Calculation & Anomaly Detection
Calculated fulfillment lead-times (`shipping_date - order_date`) to audit logistics SLAs.
* **Methodology:** Converted string date fields to `datetime64[ns]` objects, calculated transit duration via `.dt.days`, and isolated negative transit values (shipping date preceding order date) into an explicit `anomaly_flag` column for IT database audit without dropping financial data.

```python
# Datetime conversion and lead-time calculation
full_data['ship_days'] = (full_data['ship_date'] - full_data['order_date']).dt.days

# Isolate valid transit durations and flag negative anomalies
valid_shipments = full_data[full_data['ship_days'] >= 0]
bad_shipments = full_data[full_data['ship_days'] < 0]
```

---

## Business Analytics & Executive Insights

| Metric Category | Business Indicator | Value / Finding |
| :--- | :--- | :--- |
| **Financial Performance** | Average Order Value (AOV) | **$104.38** per order |
| **Financial Performance** | Average Transaction Value (ATV) | **$104.38** per transaction |
| **Geographic Distribution** | Top Revenue Country | **United States (USA)**, followed by **UK** & **Canada** |
| **Logistics SLA** | Clean Shipping Duration | **3.2 Days** average transit lead-time |
| **Logistics SLA** | Shipping Speed Range | **0 Days** (fastest) to **7 Days** (slowest) |
| **Data Quality Audit** | System Anomaly Flags | Isolated negative lead-time records flagged for IT sync review |

---

## Author & Project Info
* **Author:** Mirza Ishtiyaq Baig *(Data Analyst / Analytics Engineer)*
* **Repository:** `data-cleaning-analysis-pandas`

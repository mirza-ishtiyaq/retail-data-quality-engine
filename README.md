# Retail Pipeline Data Cleaning & Exploratory Analytics (Python & Pandas)

## Executive Summary & Business Context
Raw operational data in retail systems is frequently fragmented across disparate touchpoints (CRM customer profiles, order fulfillment logs, and payment gateway transactions). In this project, I engineered a Python and Pandas data cleaning and analytics pipeline to unify multi-table retail data, resolve severe data hygiene issues, and surface executive-level revenue and operational insights.

The analysis answered four core operational questions:
1. **Customer Revenue Contribution:** Who are our top revenue-generating customers by total lifetime spend?
2. **Geographic Sales Performance:** How is sales revenue distributed across international markets?
3. **Sales Momentum & Seasonality:** What are the monthly and annual revenue trajectories?
4. **Logistics & Fulfillment Health:** What is the average order-to-delivery lead time, and where are data anomaly flags occurring?

---

## Technical Stack & Repository Architecture
* **Language:** Python
* **Core Libraries:** Pandas, NumPy, Matplotlib
* **Execution Environment:** Jupyter Notebook (`sales_pipeline_analysis.ipynb`)

```
data-cleaning-analysis-pandas/
├── README.md                                          # Project documentation & insights
├── LICENSE                                            # MIT Open Source License
├── data/
│   └── raw/
│       ├── customers.csv                              # Raw CRM customer records
│       ├── orders.csv                                 # Raw order fulfillment logs
│       └── transactions.csv                           # Raw payment gateway transactions
├── notebooks/
│   └── sales_pipeline_analysis.ipynb                  # Main Python/Pandas data pipeline
└── reports/
    └── figures/                                       # Exported Matplotlib visualizations
```

---

## Data Quality Challenges & Cleaning Pipeline

When auditing the raw files in `data/raw/`, I identified four major data quality barriers:

### 1. Primary Key Duplicates & Conflict Resolution
Multiple records existed for individual customer IDs (e.g., `C001`, `C002`, `C010`) due to asynchronous profile updates.
* **Resolution:** Applied `groupby('customer_id')` with a customized first-non-null aggregation (`first()`) to consolidate duplicate attributes into a single clean customer record.

### 2. Country Standardizations (String Normalization)
Country fields contained inconsistent string representations (e.g., `"USA"`, `"usa"`, `"United States"`, `"US"`, `"U.S.A."`, `"UNITED STATES"`).
* **Resolution:** Built a standardization dictionary mapping all country variations into clean ISO codes (`"USA"`, `"UK"`, `"UAE"`, `"Germany"`, `"Canada"`, `"Mexico"`, `"Japan"`).

### 3. Missing Value Imputation
Optional contact attributes (`email`, `phone_secondary`, `city`) contained `NaN` values.
* **Resolution:** Rather than dropping rows and losing sales volume, missing contact fields were safely imputed using explicit fallbacks (`"Unknown Email"`, `"Not Provided"`).

### 4. Shipping Lead-Time & Anomaly Detection
Shipping durations (`shipping_date - order_date`) contained negative values (e.g., shipping date logged before purchase date), indicating system timestamp unsafety.
* **Resolution:** Converted date strings using `pd.to_datetime()`, calculated transit days via `.dt.days`, and flagged negative shipping days as operational anomalies without dropping transaction data.

---

## Python / Pandas Implementation Highlights

### 1. Geographic Standardization & String Clean-Up
```python
import pandas as pd

# Mapping dictionary for country string normalization
country_mapping = {
    'USA': 'USA', 'usa': 'USA', 'United States': 'USA', 'US': 'USA', 
    'U.S.A.': 'USA', 'UNITED STATES': 'USA',
    'UK': 'UK', 'United Kingdom': 'UK', 'Great Britain': 'UK', 
    'united kingdom': 'UK',
    'UAE': 'UAE', 'United Arab Emirates': 'UAE'
}

# Standardizing country attribute
customers['country'] = customers['country'].map(country_mapping).fillna(customers['country'])
```

### 2. Temporal Calculation & Anomaly Flagging
```python
# Convert text strings to datetime objects
orders['order_date'] = pd.to_datetime(orders['order_date'])
orders['shipping_date'] = pd.to_datetime(orders['shipping_date'])

# Calculate shipping duration in days
orders['shipping_days'] = (orders['shipping_date'] - orders['order_date']).dt.days

# Flag negative delivery days as operational anomalies
orders['anomaly_flag'] = orders['shipping_days'].apply(lambda x: 'Negative Transit Anomaly' if x < 0 else 'Normal')
```

---

## Key Executive Insights

1. **Geographic Revenue Distribution:**
   * The **United States (USA)** drives the highest overall sales revenue, followed by the **United Kingdom (UK)** and **Canada**.
2. **Customer Lifetime Spend:**
   * High-value accounts demonstrate strong repeat purchasing behavior, with top individual spenders contributing disproportionately to gross pipeline revenue.
3. **Fulfillment Lead-Times:**
   * Average shipping lead time across normal orders is **~3.2 days**.
   * Isolated negative transit day anomalies were successfully isolated and flagged for IT database sync remediation.

---

## Author & Project Info
* **Author:** Mirza Ishtiyaq Baig *(Data Analyst / Analytics Engineer)*
* **Repository:** `data-cleaning-analysis-pandas`

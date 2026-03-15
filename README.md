# Retail Customer Behaviour Analysis
### A Data Analytics Project using Python · PostgreSQL · Power BI

---

## About This Project

This project analyses retail shopping behaviour data to uncover patterns in customer spending, loyalty, product preferences, and the impact of discounts. It follows an end-to-end workflow — from raw data cleaning in Python, to structured querying in PostgreSQL, to an interactive Power BI dashboard.

The dataset contains **3,900 transaction records** spanning 18 attributes including purchase amount, age, gender, location, shipping type, subscription status, and payment method.

---

## Repository Structure

├── customer_behaviour_analysis.ipynb   	# Data cleaning, EDA & feature engineering
├── customer_behvaiour_analysis.sql              # 10 business questions answered in SQL
├── customer_behvaiour_analysis_dashboard.pbix   # Power BI dashboard
├── customer_shopping_behavioor.csv              # Source dataset
├── requirements.txt                            # Python dependencies
└── README.md

---

## Project Workflow

CSV Dataset (3,900 rows)
       │
       ▼
┌─────────────────────────┐
│  Python (Jupyter)        │
│  • Null handling         │
│  • Column normalisation  │
│  • Feature engineering   │
│    - age_group           │
│    - purchase_freq_days  │
└──────────┬──────────────┘
           │  SQLAlchemy → to_sql()
           ▼
┌─────────────────────────┐
│  PostgreSQL              │
│  • 10 business queries   │
│  • CTEs, Window Fns      │
│  • Customer segmentation │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Power BI Dashboard      │
│  • KPI cards             │
│  • Category breakdown    │
│  • Demographic analysis  │
└─────────────────────────┘

---

## Python — What the Notebook Does

The notebook (`Customer_Shopping_Behaviour_Analysis.ipynb`) performs the following steps:

**1. Data Loading & Inspection**
```python
df = pd.read_csv("customer_shopping_behaviour.csv")
df.info()          # column types and null counts
df.describe()      # statistical summary
```

**2. Null Value Treatment**
Only `Review Rating` had 37 nulls (~0.95%). These were imputed using the **category-level median** — a more precise approach than a global mean:
```python
df["Review Rating"] = df.groupby("Category")["Review Rating"] \
                        .transform(lambda x: x.fillna(x.median()))
```

**3. Column Normalisation**
All column names were lowercased and spaces replaced with underscores for SQL compatibility:
```python
df.columns = df.columns.str.lower().str.replace(' ', '_')
df = df.rename(columns={"purchase_amount_(usd)": "purchase_amount"})
```

**4. Feature Engineering — `age_group`**
Customers were binned into four equal-frequency groups using `pd.qcut`:
```python
labels = ["Young Adult", "Adult", "Middle-aged", "Senior"]
df["age_group"] = pd.qcut(df["age"], q=4, labels=labels)
```

**5. Feature Engineering — `purchase_frequency_days`**
The text-based frequency column was mapped to numeric day intervals:
```python
frequency_mapping = {
    "Weekly": 7, "Fortnightly": 14, "Bi-Weekly": 14,
    "Monthly": 30, "Quarterly": 90,
    "Every 3 Months": 90, "Annually": 365
}
df["purchase_frequency_days"] = df["frequency_of_purchases"].map(frequency_mapping)
```

**6. Redundant Column Removal**
`discount_applied` and `promo_code_used` were found to be 100% identical, so `promo_code_used` was dropped:
```python
(df['discount_applied'] == df['promo_code_used']).all()  # True
df = df.drop('promo_code_used', axis=1)
```

**7. Load to PostgreSQL**
```python
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://user:password@localhost:5432/customer_behavior")
df.to_sql("customer", engine, if_exists="replace", index=False)

---

## SQL — Business Questions Answered

All 10 queries run against the `customer` table in PostgreSQL.

| # | Question | Technique |
|---|----------|-----------|
| Q1 | Total revenue by gender | `GROUP BY` + `SUM` |
| Q2 | Discounted purchases above average spend | Subquery + `WHERE` |
| Q3 | Top 5 products by average review rating | `GROUP BY` + `AVG` + `LIMIT` |
| Q4 | Avg purchase: Standard vs Express shipping | Filtered `GROUP BY` |
| Q5 | Subscriber vs non-subscriber spend | `GROUP BY` + `AVG` + `SUM` |
| Q6 | Products with highest discount rate | `CASE WHEN` + percentage calc |
| Q7 | Customer segmentation (New/Returning/Loyal) | `CTE` + `CASE WHEN` |
| Q8 | Top 3 products per category | `CTE` + `ROW_NUMBER()` window function |
| Q9 | Repeat buyers vs subscription rate | `WHERE previous_purchases > 5` |
| Q10 | Revenue contribution by age group | Engineered `age_group` column |

**Sample — Customer Segmentation (Q7):**
```sql
WITH customer_type AS (
    SELECT customer_id, previous_purchases,
        CASE
            WHEN previous_purchases = 1               THEN 'New'
            WHEN previous_purchases BETWEEN 2 AND 10  THEN 'Returning'
            ELSE 'Loyal'
        END AS customer_segment
    FROM customer
)
SELECT customer_segment, COUNT(*) AS "Number of Customers"
FROM customer_type
GROUP BY customer_segment;
```

**Sample — Top 3 Products Per Category (Q8):**
```sql
WITH item_counts AS (
    SELECT category, item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (
               PARTITION BY category ORDER BY COUNT(customer_id) DESC
           ) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```

---

## Power BI Dashboard

The `.pbix` file connects directly to the PostgreSQL database and visualises:

- **KPI Cards** — Total Revenue, Total Orders, Avg Order Value, Avg Review Rating
- **Revenue by Category** — Breakdown by Clothing, Footwear, Accessories, Outerwear
- **Gender & Age Group Split** — Donut chart + clustered bar
- **Subscriber vs Non-Subscriber** — Revenue and avg spend comparison
- **Discount Impact** — Side-by-side avg purchase with/without discount
- **Top Products by Rating** — Ranked horizontal bar chart

### Connecting to your database
1. Open `customer_behviour_dashboard.pbix` in Power BI Desktop
2. Go to **Home → Transform Data → Data Source Settings**
3. Update the server/database to match your local PostgreSQL setup
4. Click **Apply & Refresh**

---

## Getting Started

### Prerequisites
- Python 3.10+
- PostgreSQL (any recent version)
- Power BI Desktop (Windows, free)

### Setup

```bash
# 1. Clone the repo
git clone https://github.com/<your-username>/retail-customer-behaviour-analysis.git
cd retail-customer-behaviour-analysis

# 2. Install dependencies
pip install -r requirements.txt

# 3. Create the PostgreSQL database
# In psql or pgAdmin: CREATE DATABASE customer_behaviour;

# 4. Run the notebook
jupyter notebook Customer_Shopping_Behaviour_Analysis.ipynb
# Update the connection string in the last cell, then run all cells

# 5. Run SQL queries
# Open customer_behaviour.sql in pgAdmin / DBeaver / VS Code SQL extension

# 6. Open the Power BI dashboard
# Open customer_behvaiour_dashboard.pbix in Power BI Desktop
# Update the data source connection string
```

---

## requirements.txt

```
pandas>=2.0
numpy>=1.26
sqlalchemy>=2.0
psycopg2-binary>=2.9
jupyter
ipykernel
```

---

## Key Findings

- **Subscribers** consistently show higher average spend and purchase more frequently
- **`discount_applied` and `promo_code_used` are perfectly correlated** in this dataset — one was dropped as redundant
- **Clothing** is the highest-volume category; **Accessories** tends to have the highest review ratings
- **Repeat buyers** (previous purchases > 5) are significantly more likely to be subscribers
- The engineered `age_group` column reveals **Middle-aged and Senior** segments drive the most total revenue

---

## Skills Demonstrated

- Category-aware null imputation using `groupby` + `transform`
- Feature engineering from text data (`pd.qcut`, dictionary mapping)
- Detecting and programmatically removing redundant columns
- Writing PostgreSQL CTEs and window functions (`ROW_NUMBER`, `PARTITION BY`)
- Building a full pipeline: CSV → Python → PostgreSQL → Power BI

---

## License

This project is licensed under the MIT License.

It was independently developed using a public retail dataset, taking inspiration
from [Amlan Mohanty's data analytics portfolio project](https://github.com/amlanmohanty1/customer-trends-data-analysis-SQL-Python-PowerBI).
Original work © 2025 Amlan Mohanty. Modifications and extensions © 2026 Your Name.

See the [LICENSE](LICENSE) file for full details.
---


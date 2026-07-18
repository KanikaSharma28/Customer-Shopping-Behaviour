# 🛍️ Customer Shopping Behaviour Analysis

> An end-to-end data analytics project analyzing customer purchasing patterns, 
> revenue trends, and shopping behaviour using **Python**, **MySQL**, and **Power BI**.

## 📌 Project Overview

This project performs a complete analysis of **3,900 customer shopping records** 
to uncover key business insights such as revenue trends, customer segmentation, 
seasonal patterns, and product performance.

The project follows a full **ETL pipeline**:
- 🐍 **Python** — Data cleaning, EDA, and feature engineering
- 🗄️ **MySQL** — Structured storage and 10 analytical SQL queries  
- 📊 **Power BI** — Interactive dashboard for business visualization

## 📂 Project Structure


customer-shopping-behaviour-analysis/
│
├── 📓 CustomerShoppingBehaviour.ipynb   # Python EDA & data cleaning notebook
├── 🗄️ s3.sql                            # 10 SQL queries for analysis
├── 📊 CustomerShoppingBehaviour.pbix    # Power BI dashboard file
├── 📁 dataset/
│   └── customer_shopping_behavior.csv   # Raw dataset (3,900 rows × 18 columns)
├── 📁 screenshots/
│   └── dashboard.png                    # Power BI dashboard screenshot
└── 📄 README.md

## 📊 Dataset Information

| Attribute | Details |
|---|---|
| **Source** | Kaggle |
| **Rows** | 3,900 |
| **Columns** | 18 |
| **Total Revenue** | $233,081 |
| **Missing Values** | 37 (Review Rating — handled) |

### 📋 Columns in Dataset

| Column | Description |
|---|---|
| Customer ID | Unique identifier for each customer |
| Age | Customer age |
| Gender | Male / Female |
| Item Purchased | Product name |
| Category | Clothing / Accessories / Footwear / Outerwear |
| Purchase Amount (USD) | Transaction value |
| Location | US state |
| Size | S / M / L / XL |
| Color | Product color |
| Season | Spring / Summer / Fall / Winter |
| Review Rating | 1.0 – 5.0 (37 nulls handled) |
| Subscription Status | Yes / No |
| Shipping Type | Standard / Express / Free / etc. |
| Discount Applied | Yes / No |
| Promo Code Used | Yes / No (dropped — identical to Discount Applied) |
| Previous Purchases | Count of past purchases |
| Payment Method | PayPal / Credit Card / Cash / etc. |
| Frequency of Purchases | Weekly / Monthly / Annually / etc. |

---

## 🐍 Python — Data Cleaning & Feature Engineering

**File:** `CustomerShoppingBehaviour.ipynb`  
**Libraries:** `pandas` `numpy` `sqlalchemy` `pymysql`

### ✅ Steps Performed

**1. Data Loading**
```python
df = pd.read_csv("customer_shopping_behavior.csv")
# Shape: 3,900 rows × 18 columns
```

**2. Missing Value Treatment**
```python
# Found 37 null values in Review Rating
# Filled using group-wise median per category
df["Review Rating"] = df.groupby("Category")["Review Rating"]\
    .transform(lambda x: x.fillna(x.median()))
```

**3. Column Standardisation**
```python
df.columns = df.columns.str.lower().str.replace(" ", "_")
df = df.rename(columns={"purchase_amount_(usd)": "purchase_amount"})
```

**4. Feature Engineering**
```python
# Age group using quartile-based segmentation
labels = ["Young Adult", "Adult", "Middle-aged", "Senior"]
df["age_group"] = pd.qcut(df["age"], q=4, labels=labels)

# Purchase frequency mapped to days
frequency_mapping = {
    "Weekly": 7, "Fortnightly": 14, "Monthly": 30,
    "Quarterly": 90, "Bi-Weekly": 14, "Annually": 365
}
df["purchase_frequency_days"] = df["frequency_of_purchases"].map(frequency_mapping)
```

**5. Duplicate Column Removal**
```python
# promo_code_used == discount_applied (100% identical)
(df["discount_applied"] == df["promo_code_used"]).all()  # True
df = df.drop("promo_code_used", axis=1)
```

**6. Export to MySQL**
```python
from sqlalchemy import create_engine
engine = create_engine("mysql+pymysql://root:***@localhost:3306/customer_behaviour")
df.to_sql("customer", engine, if_exists="replace", index=False)
# ✅ 3,900 rows loaded successfully
```

---

## 🗄️ MySQL — SQL Analysis

**File:** `s3.sql`  
**Database:** `customer_behaviour`  
**Table:** `customer`

### 📋 Queries Overview

| # | Question | Concepts Used |
|---|---|---|
| Q1 | Revenue by Gender | GROUP BY, SUM |
| Q2 | Discount users who spent above average | Subquery, WHERE |
| Q3 | Top 5 products by review rating | ORDER BY, AVG, LIMIT |
| Q4 | Standard vs Express shipping spend | WHERE IN, AVG |
| Q5 | Subscriber vs non-subscriber revenue | Multi-Aggregate |
| Q6 | Products with highest discount rate | CASE WHEN, ROUND |
| Q7 | Customer segmentation (New/Returning/Loyal) | CTE, CASE WHEN |
| Q8 | Top 3 products per category | CTE, ROW_NUMBER(), Window Function |
| Q9 | Repeat buyers and subscription status | Subquery, WHERE |
| Q10 | Revenue by age group | GROUP BY, ORDER BY |

### 🔍 Sample Query — Customer Segmentation (CTE)

```sql
WITH customer_type AS (
  SELECT customer_id, previous_purchases,
    CASE 
      WHEN previous_purchases = 1 THEN 'New'
      WHEN previous_purchases BETWEEN 2 AND 10 THEN 'Returning'
      ELSE 'Loyal'
    END AS customer_segment
  FROM customer
)
SELECT customer_segment, COUNT(*) AS "Number of Customers"
FROM customer_type
GROUP BY customer_segment;
```

**Result:**
| Segment | Count |
|---|---|
| New | 79 |
| Returning | 573 |
| Loyal | 3,248 |

### 🔍 Sample Query — Top 3 Products per Category (Window Function)

```sql
WITH item_counts AS (
  SELECT category, item_purchased,
    COUNT(customer_id) AS total_orders,
    ROW_NUMBER() OVER (
      PARTITION BY category 
      ORDER BY COUNT(customer_id) DESC
    ) AS item_rank
  FROM customer
  GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```

---

## 📊 Power BI — Interactive Dashboard

**File:** `CustomerShoppingBehaviour.pbix`

### 📌 Dashboard Components

| Visual | Description |
|---|---|
| KPI Cards | Total Customers, Avg Rating, Avg Purchase, Total Orders |
| Line Chart | Revenue by Age Group |
| Bar Chart | Revenue by Category |
| Line Chart | Sales by Season |
| Donut Chart | Subscription Status (%) |
| Pie Chart | Sales by Payment Method |
| Map Visual | Customer Location Distribution |
| Slicers | Gender, Location, Shipping Type, Purchase Frequency |

### 🖼️ Dashboard Preview

![Power BI Dashboard](screenshots/dashboard.png)

---

## 📈 Key Insights

```
✅ 68% of customers are Male — but Female customers spend slightly more on average
✅ Senior customers (55+) generate the highest revenue — $63,200
✅ Clothing is #1 category at 44.5% of all orders
✅ Fall season drives peak spending — avg $61.56 per purchase
✅ 83% of customers (3,248) are Loyal buyers with 10+ previous purchases
✅ 73% of repeat buyers are NOT subscribed — major loyalty programme opportunity
✅ Payment methods are equally distributed — PayPal leads at 17.4%
✅ Standard shipping generates higher avg spend ($61.13) vs Express ($58.38)
```

---

## 💡 Business Recommendations

| # | Recommendation | Based On |
|---|---|---|
| 1 | Target Senior customers with premium product lines | SQL Q10 |
| 2 | Run summer discount campaigns (lowest spend season) | EDA + SQL Q1 |
| 3 | Launch loyalty programme to convert 73% non-subscribed repeat buyers | SQL Q5 + Q9 |
| 4 | Promote Free Shipping as conversion tool | Power BI |
| 5 | Improve Clothing quality — lowest rating category (3.72) | SQL Q3 |
| 6 | Apply selective discounting — discounted products still sell well | SQL Q6 |

---

## 🛠️ Tools & Technologies

| Tool | Purpose | Version |
|---|---|---|
| Python | Data cleaning & feature engineering | 3.x |
| Pandas | Data manipulation | 2.x |
| NumPy | Numerical operations | 1.x |
| SQLAlchemy | Python–MySQL connection | 2.x |
| MySQL | Data storage & SQL analysis | 8.x |
| Power BI | Interactive dashboard | Desktop |
| Jupyter Notebook | Development environment | Latest |

---

## ⚙️ How to Run This Project

### Step 1 — Clone the Repository
```bash
git clone https://github.com/your-username/customer-shopping-behaviour-analysis.git
cd customer-shopping-behaviour-analysis
```

### Step 2 — Install Python Libraries
```bash
pip install pandas numpy sqlalchemy pymysql jupyter
```

### Step 3 — Run the Jupyter Notebook
```bash
jupyter notebook CustomerShoppingBehaviour.ipynb
```

### Step 4 — Set Up MySQL Database
```sql
CREATE DATABASE customer_behaviour;
```
Then update the connection string in the notebook:
```python
engine = create_engine("mysql+pymysql://your_username:your_password@localhost:3306/customer_behaviour")
```

### Step 5 — Run SQL Queries
Open `s3.sql` in MySQL Workbench and run all 10 queries.

### Step 6 — Open Power BI Dashboard
Open `CustomerShoppingBehaviour.pbix` in Power BI Desktop.

---

## 📦 Requirements

```
pandas>=2.0.0
numpy>=1.24.0
sqlalchemy>=2.0.0
pymysql>=1.1.0
jupyter>=1.0.0
```

---

## 👩‍💻 Author

**Kanika Sharma**  
B.Tech Computer Science & Engineering  
Global Institute of Technology & Management, Farrukhnagar  
Gurugram University | 2026

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?logo=github)](https://github.com/your-username)

---

## 📄 License

This project is open source and available under the 
[MIT License](LICENSE).

---

⭐ **If you found this project helpful, please give it a star!** ⭐

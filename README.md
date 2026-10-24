# Data Ingestion, Cleaning & Preprocessing with Pandas

**Retail / E-commerce Orders — end-to-end data-cleaning project (internship assignment)**

This project ingests a realistic **retail orders dataset (10,400 rows × 22 columns)** containing real-world data-quality problems, systematically cleans and preprocesses it with **Pandas**, engineers analytical features, visualizes insights, and exports a verified, analysis-ready dataset.

---

## 📁 Repository Contents

| File | Description |
|---|---|
| `Data_Cleaning_Preprocessing.ipynb` | **Main deliverable.** Fully executed Jupyter Notebook: audit → clean → engineer → visualize → export (uses relative paths; runs after cloning) |
| `retail_orders_raw.csv` | **Raw dataset, preserved untouched** — 10,400 rows × 22 cols with intentional quality issues |
| `clean_dataset.csv` | **Cleaned output** — 9,378 rows × 30 cols, zero missing, zero duplicates, verified by reload |
| `requirements.txt` | Python dependencies |
| `README.md` | This documentation |

---

## 🎯 Project Objective

To demonstrate a complete, reproducible data-cleaning workflow:

1. Load and inspect a messy retail dataset (rows, columns, dtypes, missing values).
2. Audit data quality **before** cleaning and store baseline metrics.
3. Handle missing values with appropriate, documented methods.
4. Detect and remove duplicate records.
5. Standardize inconsistent text and category names.
6. Correct data types and invalid dates.
7. Detect and **investigate** outliers without blindly deleting valid records.
8. Engineer features (month/year, ship delay, profit margin).
9. Compare quality metrics **before vs after** with actual data.
10. Create meaningful visualizations and export the cleaned data.

---

## 📊 Dataset Source

`retail_orders_raw.csv` is a **synthetically generated but realistic retail/e-commerce orders dataset** (seeded random generation, `seed=42`) modeled on classic retail superstore data, covering **2021-01-01 → 2024-12-31** (48 months):

- **10,400 rows** (10,000 base orders + 400 exact duplicates) × **22 columns**
- **Columns:** `Order_ID, Order_Date, Ship_Date, Customer_ID, Customer_Name, Segment, Country, Region, City, State, Postal_Code, Product_ID, Category, Sub_Category, Product_Name, Sales, Quantity, Discount, Profit, Shipping_Cost, Order_Priority, Payment_Mode`
- **Grain:** one row = one ordered product. ~2,500 customers, 3 categories, 16 sub-categories, 4 countries, 4 regions.
- **Why synthetic?** It guarantees a rich, controlled set of *realistic* defects (below) while the raw file is preserved so every cleaning step is auditable. All reported statistics are computed from the actual data at runtime — nothing is hard-coded.

---

## 🧹 Data-Quality Issues Found (Before Cleaning)

Audit results from the notebook (computed on the raw file):

| Issue | Evidence (actual) |
|---|---|
| **Missing values** | **2,982 cells (1.30%)** across 11 columns: Postal_Code 843, Discount 412, Ship_Date 394, Payment_Mode 302, Profit 295, Shipping_Cost 213, Customer_Name 191, Product_Name 143, Order_Date 132, Customer_ID 53, Quantity 4 |
| **Duplicate records** | **400 exact duplicates** (3.85% of rows) |
| **Inconsistent categories** | Category 26 spellings (→3), Sub-Category 94 (→16), Segment 23 (→3), Country 28 (→4), Region 28 (→4), Priority 28 (→4), Payment 41 (→6). E.g. `Furniture/furniture/FURNITURES /Furniturre`, `Tech/TECH/Technology`, `COD/C.O.D/Cash on Delivery` |
| **Inconsistent data types** | `Sales`/`Profit` mix floats with `"$1,234.56"` strings; `Discount` mixes `0.1` with `"10%"`; `Quantity` has padded strings and word-numbers (`"two"`); numerics stored as `object` |
| **Invalid dates** | 6+ formats (`YYYY-MM-DD`, `DD/MM/YYYY`, `MM-DD-YYYY`, `YYYY/MM/DD`, `DD-Mon-YYYY`, datetimes) plus invalid values (`2023-13-45`, `31-02-2022`, `not available`, blanks) → **401 bad Order_Date (3.86%)**, **533 bad Ship_Date (5.12%)** |
| **Illogical dates** | 432 ship-before-order rows + 320 rows with >30-day delays (up to **325 days**) from day/month-swap ambiguity |
| **Numerical outliers** | Sales: 64 negative (min −$899.34), 18 zero, 80 > $20k (max $119,551.37); Quantity 0/negative/word-errors + bulk up to 950; Discount: 77 negative, 81 > 100% (max 150%); Profit extremes ±$30k |

---

## 🔧 Cleaning Procedures (in notebook order, all executed)

1. **Missing values** — standardized placeholders to `NaN`; filled `Customer_Name→"Unknown"`, `Product_Name→"Unknown Product"`, `Customer_ID→"UNKNOWN"`, invalid postals (`ABCDE`, `No Zip`, …)→`"Unknown"`, `Payment_Mode`→mode; `Discount→0.0` (business default), `Profit`→median per Category, `Shipping_Cost`→median, `Ship_Date`→`Order_Date + 4-day median delay`; **dropped** only rows with unparseable `Order_Date` (essential time key).
2. **Duplicates** — `duplicated().sum()` → **400 found, inspected, removed** with `drop_duplicates()`; verified 0 remain.
3. **Standardization** — explicit mapping functions (case/spacing/typo-proof) for Category, Sub-Category, Segment, Country, Region, Order_Priority, Payment_Mode + title-casing for City/State/Customer_Name. Before→after unique counts verified (e.g. 94→16 sub-categories).
4. **Types & dates** — stripped `$`/commas/`%`, mapped word-numbers, coerced to `float/int/datetime64`; dropped 379 invalid-`Order_Date` rows; **fixed 1,252 ship dates** (500 NaT + 432 early + 320 long-delay → `Order_Date + 4 days`), flagged via new `Ship_Date_Fixed` column. Final delay range: **0–30 days, mean 3.93, median 4**.
5. **Outliers (investigated, not blindly deleted)** — IQR bounds + boxplots; inspected top-10 sales/quantity/discount cases; **dropped only impossible values** (Sales≤0: 75 rows, Quantity≤0/NaN: 26 rows, Discount<0 or >100%: 142 rows); **capped** borderline discounts (0.8, 1.0]→0.80 (20 rows); **kept but flagged** genuine VIP/bulk orders (`Is_Sales_Outlier`: 71 rows >$20k; `Is_Qty_Outlier`: 39 rows >20 units); kept all profits incl. losses (real outcomes).
6. **Feature engineering** — `Order_Year`, `Order_Month`, `Order_Month_Name` from dates; `Ship_Delay_Days`; **`Profit_Margin_Pct = Profit / Sales × 100`** (guarded for Sales>0); `Ship_Date_Fixed`, `Is_Sales_Outlier`, `Is_Qty_Outlier` flags.
7. **Verification** — before/after metric table + bar charts; final `info()`, missing audit, duplicate check; exported and **reloaded `clean_dataset.csv` to confirm 9,378 × 30 with matching columns**.

---

## 📈 Before → After Results (actual, from notebook outputs)

| Metric | Before (raw) | After (clean) |
|---|---|---|
| Rows × Columns | 10,400 × 22 | **9,378 × 30** (1,022 removed = 400 dups + 379 bad dates + 75 bad sales + 26 bad qty + 142 bad discounts) |
| Duplicate rows | 400 | **0** |
| Missing cells | 2,982 (1.30%) | **0 (0.00%)** |
| Columns with missing | 11 | **0** |
| Invalid Order_Date / Ship_Date | 401 / 533 | **0 / 0** (1,252 ship dates repaired & flagged) |
| Category / Sub-Category values | 26 / 94 | **3 / 16** |
| Segment / Country / Region / Payment values | 23 / 28 / 28 / 41 | **3 / 4 / 4 / 6** |
| Sales validity | 64 neg, 18 zero, 80 >$20k mixed types | **all > 0, float; 71 VIP kept & flagged** |
| Discount range | −0.30 … 1.50 (mixed `0.1`/`"10%"`) | **0.00 … 0.80, float** |
| Date dtype | mixed strings | **datetime64 (2021-01-01 → 2024-12-31)** |
| Ship delay | up to 325 days, mean 8.0 | **0–30 days, mean 3.93, median 4** |

**Cleaned-data highlights:** total sales **$8,412,524.72**, total profit **$689,420.16**, 9,378 orders; median profit margin **8.10%** (IQR 0.78–14.92%), 2,118 loss-making orders (22.58%); top sub-categories by sales: Machines ($1.19M), Copiers ($866K), Tables ($833K); profit correlates most with shipping cost (0.39) and sales (0.38), negatively with discount (−0.04).

**Visualizations in the notebook:** missing-value audit bars, IQR boxplots + post-cleaning distributions, categorical-consistency before/after bars, monthly sales/profit trends (48 months), sales by category/region, profit by segment, discount-vs-margin scatter, numeric correlation heatmap, payment/priority mixes, margin & yearly-order plots.

---

## ⚙️ Installation

```bash
# 1. Clone the repository
git clone https://github.com/appureddy1102-cloud/Pandas_Data_Cleaning_Project.git
cd Pandas_Data_Cleaning_Project

# 2. (Recommended) create a virtual environment
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt
```

Requirements: `pandas>=2.0`, `numpy>=1.24`, `matplotlib>=3.7`, `seaborn>=0.12`, `jupyter`, `nbconvert`, `ipykernel` (tested on Python 3.13 with pandas 2.2.3 / numpy 2.3.5 / matplotlib 3.10.9 / seaborn 0.13.2).

## ▶️ How to Run

```bash
# Option A — interactive (recommended for review)
jupyter notebook Data_Cleaning_Preprocessing.ipynb
# then: Kernel → Restart & Run All

# Option B — headless re-execution (regenerates clean_dataset.csv + outputs)
jupyter nbconvert --execute Data_Cleaning_Preprocessing.ipynb --to notebook --output Data_Cleaning_Preprocessing.ipynb --ExecutePreprocessor.timeout=600
```

The notebook uses **relative paths** (`retail_orders_raw.csv` → `clean_dataset.csv`), so it runs correctly right after cloning — no path edits needed. The committed notebook already contains all executed outputs and plots.

---

## ✅ Submission Checklist

- [x] Realistic 10,000+ row dataset with missing values, duplicates, dtype issues, invalid dates, messy categories, outliers
- [x] Raw dataset preserved separately (`retail_orders_raw.csv`)
- [x] All 5 required files present
- [x] Notebook covers every required step and **executes end-to-end without errors**
- [x] All statistics computed from actual data; cleaned CSV exported and reload-verified
- [x] Relative paths — runs after `git clone` + `pip install -r requirements.txt`

**Author:** appureddy1102-cloud · **Stack:** Python, Pandas, NumPy, Matplotlib, Seaborn, Jupyter

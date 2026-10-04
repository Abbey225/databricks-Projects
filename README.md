# 🧱 Databricks Medallion Lakehouse: Retail Star Schema

A Bronze → Silver → Gold data pipeline built on Databricks and Delta Lake. It ingests raw customer, product, and order data, cleans and enriches it in the Silver layer, and models it into a star schema in the Gold layer for analytics. Every Gold load is an idempotent `MERGE` upsert, so reruns insert new records and update changed ones without creating duplicates.

---

## 🏗️ Architecture

```
Raw files  →  Bronze (raw ingest)  →  Silver (cleaned & enriched)  →  Gold (star schema)
                                         customers_enr                  dimcustomer
                                         product_enr                    dimproducts
                                         orders_enr                     dimdate
                                                                        factorders
```

| Layer | Purpose |
| :--- | :--- |
| **Bronze** | Raw data landed as-is from the source files |
| **Silver** | Cleaned, typed, and enriched tables (`customers_enr`, `product_enr`, `orders_enr`) |
| **Gold** | Business-ready star schema with surrogate keys, built for reporting |

**Catalog:** `databricks_cata` (Unity Catalog), with schemas `silver` and `gold`.

---

## ⭐ Gold Layer: Star Schema

```
            dimcustomer
                 │
dimdate ─── factorders ─── dimproducts
```

### Dimension tables

| Table | Key | Business Key | Columns |
| :--- | :--- | :--- | :--- |
| `dimcustomer` | `customer_key` (identity) | `customer_id` | email, city, state, domain, full_name |
| `dimproducts` | `product_key` (identity) | `product_id` | product_name, category, brand, price |
| `dimdate` | `date_key` (yyyyMMdd) | `full_date` | day, week, month, quarter, year, is_weekend, and more |

### Fact table

| Table | Grain | Keys | Measures |
| :--- | :--- | :--- | :--- |
| `factorders` | One row per order | `customer_key`, `product_key`, `date_key` | quantity, total_amount, total_amt_discount |

---

## 🔑 Key Design Decisions

- **Surrogate keys:** `dimcustomer` and `dimproducts` use Delta `GENERATED ALWAYS AS IDENTITY` columns. Keys are generated once and stay stable across upserts.
- **Date key:** `dimdate` uses an integer `yyyyMMdd` key, so the fact table can compute it directly from the order date without a lookup.
- **Idempotent loads:** Every table loads with a Delta `MERGE`. Matched rows are updated, and new rows are inserted. Rerunning the pipeline is safe.
- **Key lookups in the fact:** `factorders` resolves `customer_key` and `product_key` by joining the Silver orders to the Gold dimensions on the business keys.
- **Audit columns:** `create_date` is set on insert, and `update_date` is refreshed on update.

---

## ▶️ Load Order

Dimensions must be loaded before the fact table so the key lookups find every row.

1. `dimcustomer`
2. `dimproducts`
3. `dimdate`
4. `factorders`

---

## 📁 Repository Structure

```
├── README.md
├── notebooks/        # Bronze, Silver, and Gold notebooks
├── sql/              # CREATE TABLE and MERGE scripts, one file per table
└── docs/             # Architecture diagram
```

---

## 🚀 How to Run

1. Import the notebooks into your Databricks workspace.
2. Make sure the catalog `databricks_cata` and the schemas `silver` and `gold` exist.
3. Run the Bronze and Silver notebooks to build `customers_enr`, `product_enr`, and `orders_enr`.
4. Run the SQL scripts in `sql/` in the load order above.

---

## 🔍 Sample Query

Sales by month and product category:

```sql
SELECT
  d.year,
  d.month_name,
  p.category,
  SUM(f.quantity)           AS units_sold,
  SUM(f.total_amount)       AS total_sales,
  SUM(f.total_amt_discount) AS total_after_discount
FROM databricks_cata.gold.factorders f
JOIN databricks_cata.gold.dimdate     d ON f.date_key    = d.date_key
JOIN databricks_cata.gold.dimproducts p ON f.product_key = p.product_key
GROUP BY d.year, d.month_number, d.month_name, p.category
ORDER BY d.year, d.month_number, total_sales DESC;
```

---

## ✅ Data Quality Checks

```sql
-- Orders with no matching customer, product, or date
SELECT
  COUNT(*)                                       AS total_rows,
  COUNT(DISTINCT order_id)                       AS distinct_orders,
  SUM(CASE WHEN customer_key IS NULL THEN 1 END) AS missing_customer,
  SUM(CASE WHEN product_key  IS NULL THEN 1 END) AS missing_product
FROM databricks_cata.gold.factorders;
```

---

## 🛠️ Tech Stack

Databricks · Delta Lake · Spark SQL · PySpark · Unity Catalog · Medallion Architecture · Dimensional Modeling (Star Schema) · Git

---

## 📫 Author

**Abbey** · [GitHub](https://github.com/Abbey225) · obembeabiodunrotimi225@gmail.com

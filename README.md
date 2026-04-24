# Olist Customers — Medallion Architecture on Databricks

> A full Bronze → Silver → Gold data pipeline built on Databricks,
> using Delta Lake and PySpark, with Power BI as the consumption layer.

---

## Project Structure

```
olist-medallion/
├── notebooks/
│   ├── 01_bronze_ingest.py      # Raw ingestion from CSV → Delta
│   ├── 02_silver_clean.py       # Cleaning, typing, deduplication
│   ├── 03_gold_agg.py           # Business aggregations for Power BI
│   └── 04_validate.py           # Cross-layer row count sanity checks
├── data/
│   └── olist_customers_dataset.csv   # Source file (upload to DBFS manually)
└── README.md
```

---

## Dataset

| Field | Type | Description |
|---|---|---|
| `customer_id` | string | Unique order-level customer ID |
| `customer_unique_id` | string | Unique person-level customer ID |
| `customer_zip_code_prefix` | integer | First 5 digits of ZIP code |
| `customer_city` | string | Customer city name |
| `customer_state` | string | Brazilian state abbreviation (e.g. SP, RJ) |

**Source:** [Olist Brazilian E-Commerce Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
**Size:** ~99,441 rows

---

## Architecture Overview

```
CSV Upload (DBFS)
      │
      ▼
┌─────────────┐
│   BRONZE    │  Raw data, no transformations. Audit columns added.
│             │  Table: olist.bronze_customers
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SILVER    │  Cleaned, typed, normalised, deduplicated.
│             │  Table: olist.silver_customers
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    GOLD     │  Business aggregations ready for reporting.
│             │  Tables: gold_customers_by_state
│             │           gold_customers_by_city
│             │           gold_customers_by_zip
└──────┬──────┘
       │
       ▼
  Power BI (Databricks Connector)
```

---

## Layer Details

### Bronze — Raw Ingestion (`01_bronze_ingest.py`)

**Goal:** Land the source data into Delta Lake exactly as-is. Never modify source columns at this layer.

**What it does:**
- Reads the CSV from DBFS (`/FileStore/tables/olist_customers_dataset.csv`)
- Adds two audit columns: `_ingested_at` (timestamp) and `_source_file` (filename)
- Writes to `olist.bronze_customers` as a managed Delta table

**Nothing is cleaned here.** If the source has nulls, bad types, or duplicate rows — Bronze keeps them. This is intentional: Bronze is your audit trail.

---

### Silver — Cleaning & Typing (`02_silver_clean.py`)

**Goal:** Make the data trustworthy and queryable. Silver is the single source of truth for analysts.

**What was handled:**

| Problem | How it was fixed |
|---|---|
| `customer_zip_code_prefix` stored as string | Cast to `IntegerType()` |
| City names with inconsistent casing (`"São Paulo"`, `"sao paulo"`, `"SAO PAULO"`) | `F.lower(F.trim(...))` applied to `customer_city` |
| State names with inconsistent casing | `F.upper(F.trim(...))` applied to `customer_state` |
| Leading/trailing whitespace in text fields | `F.trim()` applied to city and state |
| Rows missing `customer_id` or `customer_unique_id` | Dropped via `dropna(subset=[...])` |
| Duplicate rows on `customer_id` | Removed via `dropDuplicates(["customer_id"])` |

A `_cleaned_at` audit timestamp is added. The difference in row count between Bronze and Silver tells you the data quality loss rate.

---

### Gold — Business Aggregations (`03_gold_agg.py`)

**Goal:** Pre-aggregate data into reporting-ready tables. Power BI reads these directly — no heavy computation at query time.

**What was handled / built:**

**Table 1 — `gold_customers_by_state`**
Answers: *How many customers does each state have? What's the repeat purchase rate?*

| Column | Description |
|---|---|
| `customer_state` | State abbreviation |
| `total_customers` | Count of all customer records |
| `unique_customers` | Count of distinct `customer_unique_id` (person-level) |
| `distinct_cities` | How many cities are represented in that state |
| `repeat_customer_rate` | `1 - (unique / total)` — proportion of repeat buyers |

**Table 2 — `gold_customers_by_city`**
Answers: *Which cities drive the most customers, broken down by state?*

| Column | Description |
|---|---|
| `customer_state` | State abbreviation |
| `customer_city` | Normalised city name |
| `total_customers` | Customer count in that city |
| `unique_customers` | Distinct person-level count |

**Table 3 — `gold_customers_by_zip`**
Answers: *Where are customers geographically concentrated? (for map visuals)*

| Column | Description |
|---|---|
| `customer_zip_code_prefix` | 5-digit ZIP prefix |
| `customer_state` | State |
| `total_customers` | Customer count in that ZIP zone |

---

## How to Run

### Prerequisites
- Databricks account (free tier works)
- Cluster running (Runtime 12+ / Spark 3.3+)
- CSV uploaded to DBFS: `Data > Add Data > Upload File`

### Execution Order

```bash
# Run notebooks in this exact order — each depends on the previous Delta table
01_bronze_ingest.py   →   02_silver_clean.py   →   03_gold_agg.py   →   04_validate.py
```

### Database Setup (run once before anything)

```python
spark.sql("CREATE DATABASE IF NOT EXISTS olist")
```

---

## Power BI Connection

1. Open **Power BI Desktop** → Get Data → **Databricks**
2. Enter your **Server hostname** (Compute → your cluster → Advanced Options → JDBC/ODBC)
3. Enter your **HTTP path** (same panel)
4. Authentication: **Personal Access Token**
   - Databricks → Settings → Developer → Access Tokens → Generate New Token
5. In the Navigator, select the three Gold tables under `olist`
6. Click **Load**

### Recommended Visuals

| Visual Type | Table | Fields |
|---|---|---|
| Bar chart | `gold_customers_by_state` | state vs total_customers |
| KPI card | `gold_customers_by_state` | repeat_customer_rate |
| Map | `gold_customers_by_zip` | zip_prefix (location) + total_customers |
| Treemap | `gold_customers_by_city` | city + total_customers, sliced by state |

---

## Git Integration (Databricks → GitHub)

See **"Pushing to GitHub"** section below for full step-by-step instructions.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks (free tier) | Compute, notebook authoring, cluster |
| Apache Spark / PySpark | Distributed data processing |
| Delta Lake | ACID-compliant storage format |
| DBFS | Distributed file system for CSV landing |
| Power BI Desktop | Business intelligence and reporting |
| GitHub | Version control for notebooks |

---

## Author

**Jimmy**

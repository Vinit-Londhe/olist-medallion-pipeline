# Olist Medallion Pipeline

Data & AI Engineer take-home assessment — built on Databricks Free Edition using the Olist Brazilian 
E-Commerce dataset.

## Dataset choice

I went with **Option 1: Olist Brazilian E-Commerce Public Dataset** (from Kaggle) rather than the built-in 
retail-org sample. Olist has multiple linked tables, a real geographic dimension (customer/seller state and 
city), delivery timing data, and review scores — which gave me more to work with for the commercial analysis 
in Part B, particularly around freight cost and regional demand patterns.

## Databricks setup

- **Workspace/edition:** Databricks Free Edition (serverless-only)
- **Compute:** Serverless — no cluster/runtime version to select; Free Edition attaches serverless compute 
  automatically. Check the notebook's environment selector (top right of any notebook) for the exact 
  serverless environment version in use at run time.
- **Catalog:** `olist`
- **Schemas:** `olist.bronze`, `olist.silver`, `olist.gold`
- **Raw file storage:** Unity Catalog managed volume at `olist.bronze.raw_files` 
  (`/Volumes/olist/bronze/raw_files/`)

## Repo structure

```
olist-medallion-pipeline/
├── README.md                    # this file
├── analysis_answers.md          # Part B — commercial analysis, recommendation, caveats
├── write_up.md                  # short non-technical summary for a stakeholder
├── ingest/
│   ├── load_bronze              # A1 — loads all 9 raw CSVs into bronze Delta tables
│   └── requirements.txt
├── pipeline/
│   ├── 01_create_tables.sql     # A2 — silver layer DDL, typed schema, key strategy
│   ├── 02_bronze_to_silver      # A2/A4 — cleaning, typing, dedup, silver population
│   ├── 03_silver_to_gold        # A4 — star schema: dims + fact with hashed surrogate keys
│   └── data_model.md            # A3 — 3NF entity model, ERD (Mermaid), normalisation notes
└── sql/
    └── analytics_queries.sql    # A5 — five analytics queries against the gold layer
```

## How to reproduce this end to end

### 1. Set up the catalog and volume

Run in a SQL editor or notebook:

```sql
CREATE CATALOG IF NOT EXISTS olist;
CREATE SCHEMA IF NOT EXISTS olist.bronze;
CREATE SCHEMA IF NOT EXISTS olist.silver;
CREATE SCHEMA IF NOT EXISTS olist.gold;
CREATE VOLUME IF NOT EXISTS olist.bronze.raw_files;
```

### 2. Get the raw data in

Download the dataset from Kaggle 
([olistbr/brazilian-ecommerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)) and unzip it 
locally — this gives you 9 CSV files. Free Edition restricts outbound internet access, so the Kaggle API 
cannot be called from inside a notebook; download locally and upload instead.

In Databricks: **New → Add or upload data → Upload files to a volume**, target `olist.bronze.raw_files`, 
and upload all 9 CSVs.

### 3. Run the ingest step (bronze)

Open `ingest/load_bronze` and run all cells. This reads each CSV as-is (no cleaning), adds `_loaded_at` and 
`_source` metadata columns, and writes each one to a bronze Delta table under `olist.bronze`.

Verify with:
```sql
SHOW TABLES IN olist.bronze;
```
You should see 9 tables.

### 4. Build the silver layer

Run `pipeline/01_create_tables.sql` first — this creates the 10 empty, properly typed silver tables (real 
datatypes, not everything STRING; NOT NULL on key columns).

Then run `pipeline/02_bronze_to_silver` — this reads from bronze, casts types, deduplicates (using window 
functions where a simple `dropDuplicates` isn't sufficient, e.g. reviews), splits the customer entity into 
`customers` (unique people) and `customer_orders` (the order-to-person bridge), and writes into the silver 
tables.

Verify with:
```sql
SHOW TABLES IN olist.silver;
```
You should see 10 tables.

### 5. Build the gold layer

Run `pipeline/03_silver_to_gold`. This builds four dimension tables (`dim_customer`, `dim_product`, 
`dim_seller`, `dim_date`) with hashed surrogate keys (`xxhash64` on the natural key, chosen for 
reproducibility — the same input always produces the same key, which keeps re-runs idempotent), and one 
fact table (`fact_order_items`) at order-item grain, joined to all four dimensions.

Verify with:
```sql
SHOW TABLES IN olist.gold;
```

### 6. Run the analytics queries

Open `sql/analytics_queries.sql` in the SQL editor and run each query in turn. These answer: top 10 
categories by revenue, monthly revenue trend, an outlier query on freight ratio (z-score based), a revenue 
breakdown by customer state, and a realistic ad-hoc stakeholder question (top sellers by recent item volume).

### 7. Read the analysis

`analysis_answers.md` contains the commercial insights, the recommended campaign, assumptions/caveats, the 
CFO memo, and a bonus LLM use case. `write_up.md` is a shorter, non-technical version of the same findings.

## How this connects to Git

This repo was developed inside a Databricks Git folder linked to GitHub via a personal access token. Work 
was done on a `feature/pipeline` branch and merged into `main` via pull request. Notebooks are stored in 
source `.py`/`.sql` format (not `.ipynb` or `.dbc`) so they diff cleanly in GitHub.

## Assumptions and shortcuts taken given the time-box

- Bronze schema is auto-inferred by Spark; all explicit typing happens in silver, not bronze.
- `customer_unique_id` (the real person) is kept separate from `customer_id` (per-order identifier) via a 
  dedicated `customer_orders` bridge table in silver — getting this backwards would silently break any 
  repeat-customer analysis.
- 9 rows in the raw reviews file had a NULL `order_id` due to a CSV parsing artifact (unescaped characters 
  in review comments shifting subsequent columns) — these were dropped. A further 57 rows had only a 
  malformed timestamp from the same cause — these were kept, with the timestamp set to NULL via `try_cast`, 
  since the rest of the row was still usable.
- The fact table is built at **order-item grain** (not order grain), which avoids fan-out from the 
  payments and reviews tables (which can have multiple rows per order) by simply not joining them into this 
  fact — a separate fact table would be needed for payment- or review-level analysis.
- Part C (Delta Live Tables / dbt, GitHub Actions CI, Databricks Asset Bundle) was treated as bonus and not 
  attempted, per the brief's own guidance that a clean, well-versioned Part A and B is preferred over a 
  rushed Part C. Happy to discuss how I'd approach it live.
- The freight-subsidy revenue estimate in the recommendation (Part B2) is a stated assumption, not a 
  modelled elasticity figure — flagged explicitly in `analysis_answers.md`.
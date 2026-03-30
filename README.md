# 🚕 Automated Cloud-Native Pipeline Architecture Using Azure Stack

> An automated, cloud-native data engineering pipeline that ingests, transforms, and serves NYC Green Taxi trip data using the Azure Modern Data Stack — built on a **Bronze → Silver → Gold** Medallion Architecture.

---

## 📐 Architecture Overview

```
NYC TLC Web (HTTP)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              Azure Data Factory (ADF)                   │
│                                                         │
│  Ingestion_pipeline                                     │
│    └─► Get Metadata (year exists check)                 │
│         └─► If Year NOT exists                          │
│              └─► copy_data pipeline                     │
│                   └─► ForEach Month (1–12)              │
│                        └─► Copy Activity                │
│                             Source : nyc_web (HTTP)     │
│                             Sink   : bronze container   │
└───────────────────────┬─────────────────────────────────┘
                        │ Parquet (Snappy)
                        ▼
┌───────────────────────────────────────────────────────────────┐
│              Azure Data Lake Storage Gen2 (ADLS)              │
│                                                               │
│  bronze/trip-data/{year}/{month}/green_tripdata_*.parquet     │
│  bronze/trip_zone/                                            │
│  bronze/trip_type/                                            │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────────┐
│              Azure Databricks (PySpark + Delta Lake)          │
│                                                               │
│  01_silver_notebook                                           │
│    ├─ Auth via Azure Key Vault (Service Principal OAuth2)     │
│    ├─ silver.tripzone  (Delta, from CSV)                      │
│    ├─ silver.triptype  (Delta, from CSV)                      │
│    └─ silver.tripdata  (Delta, incremental batch)             │
│         • Column rename & type casting                        │
│         • Derived columns: trip_year, trip_month              │
│         • Filters: total_amount > 0, distance > 0, pax > 0   │
│         • Processed-files tracking (Delta)                    │
│         • Partitioned by trip_year / trip_month               │
│                                                               │
│  02_gold_notebook                                             │
│    └─ Aggregated / analytics-ready Gold tables                │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
              silver / gold containers
              (ADLS Gen2 — Delta format)
```

---

## 🛠️ Tech Stack

| Layer | Service / Tool |
|---|---|
| Orchestration | Azure Data Factory (ADF) |
| Storage | Azure Data Lake Storage Gen2 (ADLS) |
| Compute | Azure Databricks (PySpark) |
| Table Format | Delta Lake |
| Secrets | Azure Key Vault |
| Source Data | [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) |
| File Format | Parquet (Snappy compressed) |

---

## 📁 Repository Structure

```
Automated Cloud-Native Pipeline Architecture Using Azure Stack/
│
├── adf/
│   ├── pipelines/
│   │   ├── Ingestion_pipeline.json      # Master ingestion pipeline
│   │   └── copy_data.json               # ForEach month copy pipeline
│   ├── datasets/
│   │   ├── dynamic_nyc_dataset.json     # HTTP source dataset
│   │   ├── bronze_container.json        # ADLS sink dataset
│   │   └── metadata.json                # Year existence check dataset
│   └── linkedservices/
│       ├── nyc_web.json                 # HTTP linked service (TLC)
│       └── mylakeplace31.json           # ADLS Gen2 linked service
│
├── databricks/
│   ├── 01_silver_notebook.py            # Bronze → Silver transformation
│   └── 02_gold_notebook.py              # Silver → Gold aggregation
│
└── README.md
```

---

## ⚙️ Pipeline Details

### 1. Ingestion Pipeline (`Ingestion_pipeline`)

The master pipeline runs with a `year` parameter and performs a smart idempotency check before triggering any copy operations.

- **Get Metadata** — checks if the year folder already exists in the Bronze container on ADLS.
- **If Year Exists** — skips ingestion entirely (prevents duplicate loads).
- **If Year Does NOT Exist** — triggers the `copy_data` child pipeline.

### 2. Copy Pipeline (`copy_data`)

Loops over months 1–12 sequentially and copies each monthly Parquet file from the NYC TLC CDN to the Bronze layer.

- Source: `https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_{year}-{month}.parquet`
- Sink: `abfss://bronze@mylakeplace31.dfs.core.windows.net/trip-data/{year}/{month}/green_tripdata_{year}-{month}.parquet`
- Skipped months (e.g., file not yet published) are tracked in a `skipped_month` array variable.

### 3. Silver Notebook (`01_silver_notebook`)

Reads raw Parquet files from the Bronze layer and produces clean, typed Delta tables in the Silver layer.

**Transformations applied:**
- Column renames to snake_case (`VendorID` → `vendor_id`, `PULocationID` → `pickup_location_id`, etc.)
- Type casting (timestamps, integers, doubles)
- Derived columns: `trip_year`, `trip_month`
- Row filters: `total_amount > 0`, `trip_distance > 0`, `passenger_count > 0`
- Incremental load via processed-files tracking (Delta table)
- Partitioned by `trip_year` / `trip_month`

**Reference tables:**
- `silver.tripzone` — pickup/dropoff zone lookup (CSV → Delta)
- `silver.triptype` — trip type lookup (CSV → Delta)

### 4. Gold Notebook (`02_gold_notebook`)

Produces analytics-ready, aggregated Gold tables from the Silver layer for BI and reporting.

---

## 🔐 Security

Authentication to ADLS Gen2 uses **Azure Active Directory Service Principal** with OAuth2. Credentials (`client_id`, `client_secret`, `tenant_id`) are stored in **Azure Key Vault** and retrieved at runtime via Databricks secret scope — no credentials are hardcoded.

```python
client_id     = dbutils.secrets.get(scope="keyvault3145", key="sp-client-id")
client_secret = dbutils.secrets.get(scope="keyvault3145", key="sp-client-secret")
tenant_id     = dbutils.secrets.get(scope="keyvault3145", key="sp-tenant-id")
```

---

## 📊 Data Source

**NYC TLC Green Taxi Trip Records**  
Source: [https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)  

Key fields: `vendor_id`, `pickup_datetime`, `dropoff_datetime`, `pickup_location_id`, `dropoff_location_id`, `passenger_count`, `trip_distance`, `fare_amount`, `total_amount`, `payment_type`, `trip_type`

---

## 🗂️ Medallion Layer Reference

| Layer | Container | Format | Description |
|---|---|---|---|
| Bronze | `bronze/trip-data/` | Parquet (Snappy) | Raw data as-is from source |
| Silver | `silver/` | Delta Lake | Cleaned, typed, partitioned |
| Gold | `gold/` | Delta Lake | Aggregated, analytics-ready |

---

## 📝 License

This project is for educational and portfolio purposes.

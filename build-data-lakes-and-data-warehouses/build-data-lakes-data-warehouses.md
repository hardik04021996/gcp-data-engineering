# Build Data Lakes and Data Warehouses on Google Cloud

## Overview
- **Data Lake**: Central repository to store all types of data (structured, semi-structured, unstructured) with **schema-on-read**.
- **Data Warehouse**: Analytical system optimized for structured, curated data using **schema-on-write**.
- **Data Lakehouse**: Unifies the flexibility of lakes and performance of warehouses — stores data in open formats with governance and transactional consistency.

**Goal** → Enable unified analytics and ML across all data (operational + analytical) using open storage and powerful compute on Google Cloud.

---

## Core Concepts

### Data Lakes
- Built on **Cloud Storage** for cost-efficient, scalable data retention.
- Store raw and historical data in formats like **Parquet**, **ORC**, **Avro**.
- Support schema-on-read via processing engines (e.g., Spark, Dataflow, BigQuery external tables).

### Data Warehouses
- Implemented using **BigQuery** for fast, SQL-based analytics.
- Data is pre-cleaned, transformed, and optimized for query performance.
- Support for **partitioning, clustering, materialized views**, and **BI integrations** (Looker, Data Studio).

### Data Lakehouse Architecture
- Combines **object storage** (Cloud Storage) and **analytics engine** (BigQuery) through **BigLake**.
- Provides **governance, schema evolution, performance optimization, and unified access** across data sources.
- Uses open table formats such as **Apache Iceberg** for metadata, schema versioning, and ACID compliance.

---

## Apache Iceberg Fundamentals

| Feature | Description |
|----------|--------------|
| **Schema Evolution** | Columns can be added, renamed, dropped, or reordered without rewriting data. Column IDs ensure compatibility. |
| **Hidden Partitioning** | Partition logic is stored in metadata, enabling automatic partition pruning during queries. |
| **Time Travel** | Query data as it existed at a specific snapshot or timestamp. |
| **Atomic Transactions** | ACID guarantees through atomic metadata commits and conflict resolution. |

**Value:** Iceberg decouples metadata management from storage, enabling reliable data versioning and efficient file pruning.

**Q: How does a data lakehouse architecture combine the best features of data lakes and data warehouses?**  
**A:** By implementing a **metadata and governance layer** on top of open-format files in low-cost object storage.

---

## BigLake and BigQuery Integration

### BigLake
- Acts as a **storage abstraction layer** that connects **BigQuery** with **Cloud Storage** data in open formats (Iceberg, Parquet, ORC).
- Provides consistent **security, access control, and governance** across lake and warehouse.
- Allows **UPDATE**, **DELETE**, and **MERGE** on Iceberg tables directly via BigQuery.

### BigQuery
- **Serverless, distributed query engine** (Dremel) that uses:
  - **Slots** → Independent compute units executing query fragments in parallel.
  - **Shuffle** → Redistribution of intermediate data across slots for joins and aggregations.
- Supports **federated queries** to query across multiple systems (e.g., Cloud Storage, AlloyDB, Spanner).

**Q: How does BigQuery achieve its incredible speed?**  
**A:** BigQuery uses a **massively parallel processing (MPP)** engine where thousands of **slots** process data chunks simultaneously. Intermediate results are **shuffled** via Google’s high-speed **Jupiter network** for efficient joins and aggregations.

**Q: How does BigQuery enable a unified analytics platform in the Cymbal company’s lakehouse?**  
**A:** Through **federated queries** that analyze data in external sources like **AlloyDB** and **Iceberg tables in Cloud Storage**, without moving data.

---

## AlloyDB — High-Performance OLTP Engine
- **Fully managed PostgreSQL-compatible** database for operational workloads.
- Optimized for **high-throughput, low-latency transactions**.
- Ideal for real-time apps (orders, inventory, user data) feeding analytical systems.
- Supports analytical acceleration through **columnar engine** and **vectorized execution**.

| Use Case | Service |
|-----------|----------|
| Real-time order management | **AlloyDB** |
| Historical trend analysis | **BigQuery** |
| Raw data storage | **Cloud Storage** (Iceberg) |

---

## Federated Analytics with BigQuery
- BigQuery can query data across **AlloyDB**, **Cloud Storage (Iceberg)**, and **BigQuery native tables** — without data duplication.
- Uses the **EXTERNAL_QUERY** function to execute subqueries in remote databases.

```sql
WITH log AS (
  SELECT customer_id, log_id, timestamp, url
  FROM EXTERNAL_QUERY(
    'project.region.AlloyDB-weblog',
    'SELECT customer_id, CAST(log_id AS VARCHAR(200)) AS log_id, timestamp, url FROM web_log LIMIT 100')
)
SELECT log.customer_id, log.timestamp, log.url, C.*
FROM customers.customer_details AS C
JOIN log ON C.id = log.customer_id
ORDER BY C.id
LIMIT 100;
```

**Advantage:** Unified SQL query execution across operational and analytical systems.

---

## Partitioning and Clustering in BigQuery vs. Iceberg

| Feature | BigQuery Native Tables | Iceberg Tables in GCS |
|----------|------------------------|------------------------|
| **Partitioning** | Physical division of data (date/integer) for faster scans | Logical partitions defined in metadata for automatic pruning |
| **Hidden Partitioning** | Not supported | Supported via partition transforms |
| **Clustering** | Managed dynamically by BigQuery (auto reclustering) | Controlled during data write (by Spark, Dataflow) |
| **Performance Optimization** | Done by BigQuery engine automatically | Relies on manifest metadata and file layout |

**Q: Which statement is true about BigQuery and Iceberg integration?**  
**A:** BigQuery offers **first-class native support** for **Apache Iceberg** through **BigLake**, understanding its metadata for partitioning and clustering, and supporting direct **UPDATE**, **DELETE**, and **MERGE** statements on Iceberg tables.

---

## Dataplex — Governance and Metadata Management
- Centralized management layer for **cataloging, monitoring, and governance**.
- Auto-discovers and classifies datasets in **Cloud Storage** and **BigQuery**.
- Enforces **data quality and security policies** across domains.

### Sensitive Data Protection
- Stages: **Discovery → Classification → Protection**.
- Automatically identifies sensitive information and applies masking/encryption.

---

## Cost Optimization Practices
1. **Storage Class Selection** – Use Nearline/Coldline for infrequently accessed raw data (Bronze zone).
2. **Query Optimization** – Use partitioning & clustering to minimize data scanned.
3. **Flat-Rate Pricing** – Use dedicated query slots for predictable workloads.
4. **Budgets & Alerts** – Configure spending alerts in Cloud Billing.

---

## Machine Learning and Vector Search in BigQuery

### BigQuery ML
- Enables **ML model creation using SQL** — supports regression, classification, and forecasting.
- No data movement required; models run directly on warehouse data.

**Example:** Logistic Regression for transaction prediction
```sql
CREATE OR REPLACE MODEL `bqml_lab.sample_model`
OPTIONS(model_type='logistic_reg') AS
SELECT
  IF(totals.transactions IS NULL, 0, 1) AS label,
  IFNULL(device.operatingSystem, '') AS os,
  device.isMobile AS is_mobile,
  IFNULL(geoNetwork.country, '') AS country,
  IFNULL(totals.pageviews, 0) AS pageviews
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
WHERE _TABLE_SUFFIX BETWEEN '20160801' AND '20170631';
```

**Q: What is the goal of BigQuery ML?**  
**A:** To **democratize ML** by allowing SQL users to create and deploy ML models directly in BigQuery without moving data.

### Vector Search
- Finds semantically similar items using **embeddings** instead of keywords.
- Steps:
  1. Generate embeddings with `ML.GENERATE_EMBEDDING()`.
  2. Create a **VECTOR INDEX** for efficient nearest-neighbor search.
  3. Use `VECTOR_SEARCH()` to retrieve similar records.

---

## Medallion Architecture on GCP

| Layer | Description | GCP Implementation |
|--------|--------------|--------------------|
| **Bronze** | Raw data landing zone | Cloud Storage (JSON, Avro, Parquet) |
| **Silver** | Cleaned and enriched data | Iceberg tables on Cloud Storage or BigQuery external tables |
| **Gold** | Curated, aggregated data for BI | BigQuery managed tables + Looker dashboards |

---

## End-to-End Lakehouse Flow on GCP
1. **Ingestion** → Datastream / Pub/Sub / Dataflow into Cloud Storage (Bronze).
2. **Processing** → Spark on Dataproc or Dataflow transforms → Iceberg (Silver).
3. **Curation** → BigQuery transformations → Business tables (Gold).
4. **Analysis** → BI/ML/Vector Search in BigQuery.
5. **Governance** → Dataplex manages metadata, policies, and data quality.

---

## Summary
- **Cloud Storage**: Foundation of the data lake.
- **Apache Iceberg**: Open table format with schema evolution and ACID support.
- **BigLake**: Bridge between BigQuery and Cloud Storage.
- **BigQuery**: Unified analytical engine for structured, unstructured, and external data.
- **AlloyDB**: High-performance OLTP database with analytical extensions.
- **Dataplex**: Metadata and governance fabric for the entire lakehouse.
- **BigQuery ML & Vector Search**: Native ML and semantic search directly in SQL.

This integrated ecosystem creates a **unified Lakehouse** on GCP — enabling reliable, governed, and scalable analytics across all data sources.


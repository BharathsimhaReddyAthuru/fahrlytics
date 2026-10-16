# Fahrlytics

An end-to-end Lakehouse platform on Databricks Unity Catalog for a simulated German ride-hailing business operating in Berlin, Munich, Hamburg, Frankfurt, and Cologne. Raw transactional and telemetry data flows through a medallion architecture into anti-leakage feature stores and governed machine learning models.

## Architecture

```
Bronze (raw ingestion)  ->  Silver (conformance & DQ)  ->  Gold (features & datamarts)  ->  ML (training & registry)
                        Orchestrated by Databricks Workflows (Serverless DAG)
```

## Layers

### Bronze: raw ingestion
Append-only Delta tables with metadata columns (`_ingested_at`, `_source_file`) for lineage and watermarking.

| Notebook | Description | Outputs |
|---|---|---|
| `bronze_csv` | Batch ingestion of `customers.csv`, `drivers.csv`, `payments.csv` | `bronze.customers_raw`, `drivers_raw`, `payments_raw` |
| `bronze_events` | JSON ingestion of trip telemetry and app clickstream with schema evolution | `bronze.trips_raw`, `clickstream_raw` |

### Silver: conformance and data quality
Incremental, idempotent `MERGE INTO` upserts keyed on natural business keys, driven by `_ingested_at > max(_processed_at)` watermarks. City name variants (e.g. München, FFM, HH) are standardized to canonical city keys via regex.

| Notebook | Description |
|---|---|
| `silver_customers` | Cleaned, conformed customers |
| `silver_drivers` | Cleaned, conformed drivers |
| `silver_trips` | Status normalization, distance/duration validation, impossibility flags |
| `silver_payments` | Trip reconciliation, currency checks, orphan detection, surge fare audits |
| `silver_clickstream` | Sessionization, dwell times, bounce flags |

### Gold: feature stores and datamarts

| Notebook | Description |
|---|---|
| `gold_customer_features` | Churn features with a strict cutoff (2026-09-01) and 30-day forward label window to prevent leakage; velocity, conversion, tenure, payment friction |
| `gold_trip_features` | Hourly demand grid (city x date x hour) with sin/cos time encodings and lags at t-1h, t-24h, t-168h |
| `gold_payment_features` | Fraud signals: card-linked account counts, payment velocity, late-night flags, fare-per-km deviation |
| `gold_funnel_metrics` | Search -> selection -> checkout conversion by A/B variant |

### ML: PySpark MLlib, MLflow, Unity Catalog

| Notebook | Problem | Algorithm | Key design point |
|---|---|---|---|
| `ml_customer_churn` | Binary classification | GBTClassifier | Temporal window split to avoid target leakage; ROC-AUC and PR-AUC |
| `ml_demand_forecasting` | Time-series regression | GBTRegressor | Chronological holdout (September 2026) to avoid lookahead; RMSE and MAE |
| `ml_fraud_detection` | Imbalanced classification (~1.8% positives) | RandomForestClassifier | Cost-sensitive `weightCol`; PR-AUC and weighted recall |

Models are registered to Unity Catalog under `fahrlytics.models.*`.

**Serverless note:** SparkML model persistence on Shared/Serverless compute uses a Unity Catalog volume for artifact staging:

```python
os.environ["MLFLOW_DFS_TMP"] = "/Volumes/fahrlytics/models/mlflow_staging"
```

## Orchestration

- A multi-task Databricks Workflow DAG with explicit `depends_on` rules: downstream tasks start as soon as parents succeed and halt on upstream failure.
- Runs on Databricks Serverless compute (no cold-start VM provisioning).
- Project is mounted via Databricks Git Folders linked to GitHub.


## Tech Stack

Databricks, Unity Catalog, Delta Lake, SQL, PySpark, MLlib, MLflow, Databricks Jobs, GitHub

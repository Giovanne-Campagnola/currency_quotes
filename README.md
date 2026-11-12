# Currency Quotes Pipeline

Currency quotes ingestion from the Banco Central do Brasil (PTAX) public API using the medallion architecture (bronze → silver → gold) on Databricks.

## Layers (Unity Catalog catalog `currency`)

| Layer | Location |
|---|---|
| Landing (raw Parquet) | `/Volumes/currency/landing/currency_analysis/Quotes/New` and `/Quotes/Processed` |
| Silver (cleaned, deduplicated) | `currency.02_silver_currency.quotes` (partitioned by `dc_currency`) |
| Gold (curated dimension & fact) | `currency.03_gold_currency.dim_currencies`, `currency.03_gold_currency.fct_quotes` |

## Notebooks

1. `bronze/ingest_currencies.ipynb` — loads the currency reference dimension from PTAX into `dim_currencies`
2. `bronze/ingest_quotes.ipynb` — pulls PTAX quotes for all currencies into the `New` volume folder
3. `silver/bronze_to_silver_quotes.ipynb` — reads `New`, merges into `quotes`, moves files to `Processed`
4. `gold/silver_to_gold_quotes.ipynb` — rebuilds `fct_quotes` filling date gaps with the last known quote

Run them in order — or let the scheduled job do it.

## Scheduled job

The pipeline runs automatically via the Databricks Job **`currency_quotes_daily`** (`job_id: 1036646122285599`).

| Setting | Value |
|---|---|
| Schedule | Monday–Friday, 15:00 (`America/Sao_Paulo`), `quartz_cron_expression: 0 0 15 ? * MON-FRI` |
| Compute | Serverless (`performance_target: STANDARD`), no cluster |
| Tasks (DAG) | `ingest_currencies` → `ingest_quotes` → `bronze_to_silver_quotes` → `silver_to_gold_quotes` |
| Environment | `default` (serverless environment version `4`, dependency `requests==2.32.3`) |
| Concurrency | `max_concurrent_runs: 1`, queue enabled |
| Notifications | Email to `giovannecampagnola1@gmail.com` on failure |
| Status | `UNPAUSED` |

Each task path points to the workspace Git folder `/Users/giovannecampagnola1@gmail.com/currency_quotes/...`.
The `ingest_quotes` task republishes the full 2022→present load for every currency on each run,
so scheduled runs are idempotent (the silver layer deduplicates on `dc_currency` + `dt_period`).
A manual run can be triggered with:

```bash
databricks jobs run-now 1036646122285599
```

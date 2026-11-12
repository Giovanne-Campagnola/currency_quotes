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

Run them in order. Scheduled runs require a Databricks Job (not yet configured).

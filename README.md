# Retail Orders Lakehouse on Azure

I built this project to get hands-on with Azure Data Factory and Databricks using a retail dataset (customers and orders). Raw CSV files land in ADLS Gen2, Data Factory copies them into a bronze layer using one config-driven pipeline, and Databricks notebooks clean the data and build the silver and gold layers as Delta tables.

## Flow

```
CSV files (landing)  ->  ADF  ->  Bronze  ->  Silver (Databricks)  ->  Gold (Databricks)
```

## Data

| File | What it is | Rows |
|---|---|---|
| customers.csv | Customer details (name, address, city, state, pincode) | 12,435 |
| orders_new.csv | Historical orders | 68,883 |
| orders.csv | A small new batch of orders | 4 |

I have not uploaded the raw files here because the customers file has usernames and passwords in it.

## What I used

- Azure Data Factory
- ADLS Gen2 with separate containers for landing, bronze, silver, gold and config
- Azure Databricks (PySpark, Delta Lake)
- Azure Key Vault and managed identity for access

## How it works

**Ingestion.** Instead of making one pipeline per file, I kept a small `pipeline_config.json` that lists each source and where it should go. The pipeline `pl_ingest_bronze` reads that file with a Lookup activity, loops over it with ForEach, and runs a Copy activity for each entry using parameterized datasets. To add a new file, I only add a row to the config.

**Access.** Data Factory reads and writes storage through its managed identity, so no storage keys are used anywhere. The Databricks token is kept in Key Vault and the linked service reads it from there.

**Silver.** The notebooks clean the bronze data in PySpark and save it as Delta tables. The customers table is handled as SCD Type 2 using a Delta MERGE, so when a customer's details change, the old row is closed and a new current row is added.

**Gold.** From silver I built a few tables meant for reporting: daily orders by status, an order status breakdown with percentages, orders by state, a customer segmentation based on how recently and how often each customer ordered, and a list of the top customers.

## Repo layout

```
adf/pipelines/         exported pipeline JSON
adf/datasets/          exported dataset JSON
databricks/notebooks/  silver, SCD2 and gold notebooks
config/                pipeline_config.json
docs/                  screenshots
```

## Notes

- The dataset has no price or amount column, so the customer segmentation uses recency and frequency only, with no revenue.
- I also set up a master pipeline in ADF to run the notebooks after ingestion. I am still sorting out the Databricks compute side of that, so the notebooks are currently run from Databricks directly.

# ServiceNow IT Ticket Pipeline

Medallion Architecture (Bronze → Silver → Gold) pipeline processing ~10,000 mock ServiceNow incident records using **PySpark** and **Delta Lake** on **Azure Databricks serverless compute**.

The solution is implemented as a single Databricks notebook `it_ticket_pipeline` that is fully self-contained, reproducible, and uses **Unity Catalog managed volumes** for all layers.

## Repository Contents
- `servicenow_incidents_10k.csv` – Raw dataset (committed to repo)
- `it_ticket_pipeline` – Databricks notebook with the complete pipeline

## Prerequisites
- Azure Databricks Premium workspace with serverless compute enabled.
- The notebook assumes the raw CSV (`servicenow_incidents_10k.csv`) is uploaded to the same Workspace path as in the code (e.g., `/Workspace/Users/your.email@domain.com/servicenow-pipeline/`).
- No additional setup outside Databricks is needed — the notebook handles all Unity Catalog creation internally.

## Pipeline Overview
- **Bronze**: Raw ingestion of the CSV (preserved as-is) into a Delta volume
- **Silver**: Cleaning & transformation
  - Column names normalized to lowercase + underscores
  - Records with missing `number` (ticket ID) removed
  - Date fields converted to proper timestamps
  - Numeric fields cast to integers
- **Gold**: Aggregated analytics view
  - Ticket counts grouped by `state` (status) and `priority`
  - Saved as Delta volume (queryable)
  - Exported as a single downloadable CSV

All layers use **Unity Catalog managed volumes** (`/Volumes/...`) – no legacy DBFS paths.

### Before Running
1. Upload `servicenow_incidents_10k.csv` to your Databricks Workspace.
   - Recommended: Place it in the same Repos folder as the notebook (e.g., next to `it_ticket_pipeline`).
   - Or any accessible Workspace path.
2. In Cell 2, update the `csv_path` variable to point to your uploaded file location.
   - Example: `/Workspace/Users/your.email@domain.com/servicenow-pipeline/servicenow_incidents_10k.csv`

## How to Run the Pipeline

1. Open the notebook `it_ticket_pipeline` in your Azure Databricks workspace.
2. Attach **serverless compute** (default – no cluster configuration needed).
3. Run the cells **in order** from top to bottom:

   | Cell | Purpose | Notes |
   |------|--------|-------|
   | 1    | Complete cleanup + Unity Catalog setup | Drops and recreates catalog, schemas, and volumes. Run this first (or whenever you want a fresh start). Safe to run multiple times. |
   | 2    | Load raw CSV | Uses Pandas to read the Workspace file and convert to Spark DataFrame. |
   | 3    | Bronze layer | Writes raw data as Delta to `/Volumes/it_ticket_medallion/bronze/raw_delta` (repartitioned to ~4 files). |
   | 4    | Silver layer | Reads Bronze → cleans → writes to `/Volumes/it_ticket_medallion/silver/clean_delta`. |
   | 5    | Gold layer | Reads Silver → aggregates → writes Delta to `/Volumes/it_ticket_medallion/gold/aggregated_delta` and CSV to `/Volumes/it_ticket_medallion/gold/ticket_aggregates_csv`. |

4. After running the final cell:
   - View the aggregation with `display(gold_df)`
   - Download the CSV:
     - Go to **Catalog** (sidebar)
     - Navigate: `it_ticket_medallion` → `gold` → `ticket_aggregates_csv`
     - Click the `part-00000-*.csv` file → **Download**

The pipeline is **idempotent** – every write uses `mode("overwrite")`, so you can re-run any cell (or the whole notebook) multiple times with identical results and no duplicates.

## Design Decisions
- **Pandas for CSV ingest**: Workspace/Repos files are not directly readable by Spark on serverless; Pandas provides reliable access.
- **Unity Catalog managed volumes**: Preferred modern approach (governed, no DBFS paths, works seamlessly on serverless).
- **External Delta tables in volumes**: Demonstrates Delta Lake ACID, schema enforcement, and time travel capabilities.
- **Repartition(4) on Bronze**: Reduces number of small files for cleaner storage (raw CSV write would otherwise create many tiny Parquet files).
- **Overwrite mode**: Suitable for static demo dataset; ensures reproducibility.
- **Pure PySpark DataFrames**: No SQL queries in pipeline logic (as requested).
- **Serverless compute**: Minimal cost and instant startup.

## Production Considerations
- Incremental processing with `MERGE` for ongoing data
- Data quality checks (e.g., Great Expectations or Delta Live Tables expectations)
- Schema evolution handling
- Job orchestration via Databricks Workflows
- Monitoring and alerting on pipeline runs

The solution directly aligns with the requested Medallion Architecture while using modern Azure Databricks best practices (Unity Catalog, volumes, serverless).

Thank you for the opportunity!
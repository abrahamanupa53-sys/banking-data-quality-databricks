# Banking Data Quality on Azure Databricks

This project checks the quality of banking data. A bank copies its daily transactions from the core banking system (the **source**) into its analytics platform (the **target**). This project finds records that went missing, got changed, or got copied twice along the way, and saves a data quality report as Delta tables.

## What the project does

- Loads 4 CSV files into Azure Databricks using **schemas written as simple text strings** (no `inferSchema`).
- Cleans the data in layers: **Bronze → Silver → Gold** (the medallion architecture).
- Runs **8 data quality checks** (assignment questions 2 to 9).
- Saves every bad record and a summary of the results as **Delta tables**.

## Architecture

```
 CSV files            Bronze               Silver                DQ checks           Gold
 (Volume)      ->   raw tables     ->   cleaned tables    ->   checks 2-9    ->   gold_dq_exceptions
 /Volumes/          all columns         trimmed, blanks                          gold_dq_summary
 banking/dq/raw/    as text             -> NULL, typed
```

- **Bronze** keeps the data exactly as it arrived. Every column is read as text (`STRING`), so nothing is lost or changed by mistake.
- **Silver** cleans it: removes extra spaces, turns blank values into NULL, and converts columns to the right types with `try_cast` (Amount and Balance to `decimal(18,2)`, Transaction_Date to `date`). Duplicates are kept on purpose, so check 6 can find them.
- **Gold** holds the final results: one table with every exception, and one summary table.

The full diagram is in `architecture/banking_dq_architecture.drawio`.

## Where things live in Databricks

| Item | Name |
|---|---|
| Cloud | Microsoft Azure |
| Catalog (Unity Catalog) | `banking` |
| Schema | `banking.dq` |
| Raw files (Volume) | `/Volumes/banking/dq/raw/` |
| Storage | Azure Data Lake Storage Gen2, through external location `banking_loc` |
| Gold tables | `banking.dq.gold_dq_exceptions`, `banking.dq.gold_dq_summary` |

## Datasets

| File | Rows | Columns |
|---|---|---|
| `customers.csv` | 1,000 | Customer_ID, Customer_Name, City, Mobile_Number, KYC_Status |
| `accounts.csv` | 1,500 | Account_ID, Customer_ID, Account_Type, Balance, Status |
| `source_transactions.csv` | 10,000 | Transaction_ID, Account_ID, Transaction_Date, Amount, Type, Status |
| `target_transactions.csv` | 9,760 | Transaction_ID, Account_ID, Transaction_Date, Amount, Type, Status |

The data is synthetic (made up for practice). It contains no real customer information.

## Folder structure

```
banking-data-quality/
├── README.md                          <- this file
├── notebooks/
│   ├── 01_setup                       <- creates the schema and volume
│   ├── 02_bronze                      <- loads the CSVs as raw tables (Q1)
│   ├── 03_silver                      <- cleans the data and sets types
│   └── 04_gold                        <- runs checks 2-9 and saves the Gold tables (Q10)
├── data/
│   ├── customers.csv
│   ├── accounts.csv
│   ├── source_transactions.csv
│   └── target_transactions.csv
├── docs/
│   └── Banking_data_quality_architecture.docx <- step-by-step guide with screenshots
└── architecture/
    └── banking_dq_architecture.drawio <- architecture diagram
```

## How to run it

1. **Set up storage and the catalog:**
   - Create an ADLS Gen2 storage account with a container called `banking`.
   - Give the Databricks access connector the **Storage Blob Data Contributor** role on it.
   - In Catalog Explorer, create the external location `banking_loc` for that container, then create the catalog `banking` on it.
   - Create the schema and volume (this is what `01_setup` does):
   ```sql
   CREATE SCHEMA IF NOT EXISTS banking.dq;
   CREATE VOLUME IF NOT EXISTS banking.dq.raw;
   ```
2. **Upload the 4 CSV files** from `data/` to the Volume `/Volumes/banking/dq/raw/`
   (Catalog Explorer → banking → dq → raw → Upload to this volume).
3. **Import the notebooks** from `notebooks/` into your workspace
   (Workspace → Import).
4. **Run them in order** (01_setup, 02_bronze, 03_silver, 04_gold) on Serverless or any cluster.
5. **Check the results** in the Gold tables:
   ```sql
   SELECT * FROM banking.dq.gold_dq_summary;
   SELECT * FROM banking.dq.gold_dq_exceptions;
   ```

## The checks and results

| # | Check | What it looks for | Result |
|---|---|---|---|
| 1 | Load data | Load the 4 files, show schema and row count | 4 tables loaded |
| 2 | Missing customer details | Customers with no name, mobile number, or KYC status | **34** customers (name 1, mobile 21, KYC 12) |
| 3 | Orphan accounts | Accounts whose Customer_ID does not exist in customers | **10** accounts |
| 4 | Missing in target | Transactions in source but not in target | **260** transactions |
| 5 | Extra in target | Transactions in target but not in source (test IDs starting with `TX`) | **5** transactions |
| 6 | Duplicates in target | Transaction IDs that appear more than once in target | **15** IDs (T000419 to T000433), each appears twice |
| 7 | Bad amounts | Target amount is missing, zero, or negative | **34** transactions (missing 20, zero 9, negative 5) |
| 8 | Daily count mismatch | Dates where source and target row counts differ | **31** dates |
| 9 | Amount mismatch | Same Transaction_ID but different amount (including missing) | **48** transactions |
| 10 | Summary | Save all exceptions and the summary as Delta tables | 2 Gold tables saved |

**Why the target has 9,760 rows:**
10,000 (source) − 260 (missing) + 5 (extra test rows) + 15 (duplicate copies) = **9,760**

## Design choices

- **No `inferSchema`.** Every file is read with a schema written by hand as a simple text string, for example:
  ```python
  customers_schema = "Customer_ID STRING, Customer_Name STRING, City STRING, Mobile_Number STRING, KYC_Status STRING"
  df_customers = spark.read.csv("/Volumes/banking/dq/raw/customers.csv", header=True, schema=customers_schema)
  ```
  This is faster, and Spark can't guess a wrong type (for example, reading a mobile number as a number and losing a leading zero).
- **Bronze is all text.** Converting types happens later, in Silver, so bad values show up as NULL that we can count instead of making the load fail.
- **Trim before checking.** Values that are only spaces count as missing.
- **Delta tables for output.** They are reliable, can be queried with SQL, and keep a history of changes.

## Tech stack

- Azure Databricks (Serverless compute)
- Unity Catalog (catalog, schema, volume, external location)
- Azure Data Lake Storage Gen2
- PySpark and Spark SQL
- Delta Lake
- draw.io for the architecture diagram

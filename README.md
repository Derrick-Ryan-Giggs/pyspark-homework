# Module 6 — Batch Processing with Apache Spark

Part of the [DataTalksClub Data Engineering Zoomcamp 2026](https://github.com/DataTalksClub/data-engineering-zoomcamp)

---

## Setup

### Requirements
- Ubuntu 24.04 (or WSL)
- Java 17+
- Python 3.12+
- [uv](https://docs.astral.sh/uv/) package manager

### Install PySpark

```bash
uv init
uv add pyspark jupyter ipykernel

# Register the kernel so Jupyter uses the correct environment
uv run python -m ipykernel install --user --name=pyspark-env --display-name "PySpark (uv)"

# Launch Jupyter
uv run jupyter notebook
```

> When opening any notebook, set the kernel to **PySpark (uv)** via Kernel > Change Kernel.

### Verify installation

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("test") \
    .getOrCreate()

print(spark.version)
```

---

## Project Structure

```
PySpark/
│
├── 03_test.ipynb             # Verify Spark works, create first SparkSession
├── 04_pyspark.ipynb          # DataFrames, schemas, UDFs using FHVHV taxi data
├── 05_taxi_schema.ipynb      # Define schemas for Yellow & Green taxi, save to Parquet
├── 06_spark_sql.ipynb        # SparkSQL, temp views, union Yellow + Green data
├── 07_groupby_join.ipynb     # GroupBy internals, Sort-Merge vs Broadcast joins
├── 08_rdds.ipynb             # RDD operations, mapPartitions
├── 09_spark_gcs.ipynb        # Connect Spark to Google Cloud Storage
├── HomeWork.ipynb            # 2026 Homework solutions
│
├── data/
│   ├── raw/                  # Downloaded CSV source files
│   ├── pq/                   # Processed Parquet files
│   └── report/               # Query output / reports
│
└── taxi_zone_lookup.csv      # NYC taxi zone lookup table
```

---

## Data Sources

All data comes from the [DataTalksClub NYC TLC mirror](https://github.com/DataTalksClub/nyc-tlc-data) and the official [NYC TLC website](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

```bash
# FHVHV (For-Hire High Volume) - used in 04_pyspark.ipynb
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/fhvhv/fhvhv_tripdata_2021-01.csv.gz

# Yellow taxi
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz

# Green taxi
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2021-01.csv.gz

# Zone lookup table
wget https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv

# Homework dataset - Yellow November 2025
wget https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2025-11.parquet
```

---

## Notebooks Overview

### 03_test.ipynb
Verify PySpark is installed and working. Creates a basic SparkSession and runs `spark.range(10)`.

### 04_pyspark.ipynb
First real look at PySpark DataFrames using FHVHV taxi data. Covers reading CSV with and without explicit schemas, using pandas to sample and infer types, repartitioning, saving to Parquet, and writing User-Defined Functions (UDFs).

### 05_taxi_schema.ipynb
Defines explicit `StructType` schemas for both Yellow and Green taxi datasets. Loops over multiple months converting CSV files to Parquet. Explains the difference between Yellow (`tpep_`) and Green (`lpep_`) datetime column naming.

### 06_spark_sql.ipynb
Combines Yellow and Green taxi data using SparkSQL. Renames columns to align both datasets, unions them into a single view, and runs aggregation queries using `spark.sql()`. Saves a monthly revenue report to Parquet.

### 07_groupby_join.ipynb
Explores Spark internals. Covers the two-phase GroupBy shuffle (map-side aggregation, network shuffle, reduce-side aggregation), Sort-Merge joins between large tables, and Broadcast joins using the small zone lookup table. Uses `df.explain()` to inspect execution plans and see where shuffles occur.

### 08_rdds.ipynb
Works with the low-level RDD API. Covers `map`, `filter`, `flatMap`, `reduceByKey`, DataFrame to RDD conversion, and `mapPartitions` for partition-level processing such as loading a model once per partition rather than once per row.

### 09_spark_gcs.ipynb
Configures Spark with the GCS connector JAR to read and write directly to Google Cloud Storage using `gs://` paths. Requires a GCP account and `gcloud auth application-default login`.

### HomeWork.ipynb
Solutions to the Module 6 homework using Yellow taxi November 2025 data.

---

## Key Concepts

### Why Parquet over CSV

| | CSV | Parquet |
|--|-----|---------|
| Size on disk | Large | ~5x smaller |
| Schema | None (all strings) | Typed |
| Read speed | Slow | Fast (columnar reads) |
| Spark preference | No | Yes |

### GroupBy internals

Every GroupBy triggers a shuffle — data moves across the network so rows with matching keys land on the same partition. This is the most expensive operation in Spark and is visible as an `Exchange` node in `df.explain()`.

### Broadcast Join vs Sort-Merge Join

```python
# Sort-Merge (default) — shuffles both sides across the network
df_large.join(df_medium, on="key")

# Broadcast — copies small table to every executor, no shuffle needed
from pyspark.sql import functions as F
df_large.join(F.broadcast(df_small), on="key")
```

Use broadcast joins whenever one table is small (under a few hundred MB). The execution plan will show `BroadcastHashJoin` instead of `Exchange`.

### Spark UI

Available at `http://localhost:4040` while a Spark session is running. Shows jobs, stages, tasks, DAG visualisation, and shuffle read/write sizes.

---

## Resources

- [DataTalksClub DE Zoomcamp](https://github.com/DataTalksClub/data-engineering-zoomcamp)
- [PySpark Documentation](https://spark.apache.org/docs/latest/api/python/)
- [Spark SQL Guide](https://spark.apache.org/docs/latest/sql-programming-guide.html)
- [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

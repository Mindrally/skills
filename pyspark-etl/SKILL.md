---
name: pyspark-etl
description: "Best practices for building performant, testable PySpark ETL pipelines with Spark SQL and Apache Iceberg. Use when writing or reviewing PySpark jobs, designing joins and window functions, working with map/array higher-order functions, or building idempotent cumulative/snapshot table merges."
---

# PySpark ETL

This skill covers patterns for building production-grade, testable ETL pipelines with PySpark, Spark SQL, and Apache Iceberg, including project structure, join and window-function idioms, and safe cumulative-table merge patterns.

## Workflow for Building a PySpark ETL Job

1. **Scaffold the job class** — Create a class that manages the `SparkSession` lifecycle, accepts an injectable session for testing, and exposes an abstract `run_job` method.
2. **Define config via a factory function** — Keep config as a plain dataclass; parse CLI args in a separate factory function so tests can construct configs without touching `sys.argv`.
3. **Read source data with a shared, partition-aware reader** — Use a generic reader utility for date filters, hour ranges, and latest-partition lookups; keep business filters in the ETL class.
4. **Compose the pipeline with `.transform()`** — Chain named methods (`read_source().transform(self.enrich).transform(self.merge_with_existing)`) so `run_job` stays pure orchestration.
5. **Apply transformations idiomatically** — Use `select` over `withColumn` chains, explicit join types, explicit window frames, and native functions instead of UDFs.
6. **Write with schema-evolution safety** — Use `.byName()` when writing to Iceberg tables so column order doesn't matter.
7. **Validate output** — Check primary-key uniqueness and null counts on key columns after every write.
8. **Test locally** — Unit test transformation methods against a local `SparkSession` with small, hand-built DataFrames.

## Project Structure

### ETL class scaffold

```python
from abc import ABC, abstractmethod
import logging
from pyspark.sql import SparkSession

class BaseETL(ABC):
    def __init__(self, config, app_name="ETL Job", spark_session=None):
        self.spark = spark_session or SparkSession.builder.appName(app_name).getOrCreate()
        self.config = config
        self.logger = logging.getLogger(self.__class__.__name__)

    @abstractmethod
    def run_job(self): ...

    def stop(self):
        self.spark.stop()
```

### Config as a factory function

Keep the dataclass as pure data; put CLI parsing in a standalone factory so configs are easy to build in tests.

```python
import argparse
from dataclasses import dataclass

@dataclass
class MyConfig:
    read_date: int = 20260101

def create_config() -> MyConfig:
    parser = argparse.ArgumentParser()
    parser.add_argument("--read_date", type=int, default=20260101)
    args = parser.parse_args()
    return MyConfig(read_date=args.read_date)
```

### Partition-aware shared reader

Build one generic reader for partition mechanics; keep domain-specific filters visible in the ETL, not buried in a one-off reader class.

```python
import pyspark.sql.functions as F

class PartitionedReader:
    @staticmethod
    def read_latest(spark, table_name, partition_col):
        row = spark.read.table(table_name).agg(F.max(partition_col)).first()
        if row is None or row[0] is None:
            return spark.createDataFrame([], spark.read.table(table_name).schema)
        return spark.read.table(table_name).filter(F.col(partition_col) == row[0])

    @staticmethod
    def read_by_date(spark, table_name, partition_col, date_value):
        return spark.read.table(table_name).filter(F.col(partition_col) == date_value)

events = PartitionedReader.read_by_date(spark, "catalog.my_table", "event_date", 20260319)
events = events.filter(F.col("event_type").isin("login", "purchase"))
```

## Code Style

- Import functions as `import pyspark.sql.functions as F` and always use `F.col()` instead of `df.colA` attribute access — attribute access binds a column to a specific DataFrame variable and breaks after joins or reassignment.
- Extract complex boolean logic inside `.filter()` or `F.when()` into named variables once it exceeds 3 expressions.
- Prefer `select` over chains of `withColumn` — `select` states the output schema in one pass, while each `withColumn` call adds a projection to the query plan.
- Use `.alias()` instead of `withColumnRenamed`.
- Limit chained method calls to 5 per statement; separate `select`/`filter` chains from `withColumn` chains from `join` chains by operation type.

```python
# BAD — 3 intermediate DataFrames, one projection per call
df = df.withColumn("a", F.col("a").cast("double"))
df = df.withColumn("b", F.upper(F.col("b")))
df = df.withColumn("c", F.lit(1))

# GOOD — one DataFrame, explicit schema contract
df = df.select(
    F.col("a").cast("double"),
    F.upper(F.col("b")).alias("b"),
    F.lit(1).alias("c"),
)
```

## Joins

- Always specify `how=` explicitly — never rely on the default.
- Prefer left joins over right joins for readability; flip DataFrame order instead of using `how="right"`.
- Alias whole DataFrames for disambiguation after joins rather than renaming every column with `withColumnRenamed`.
- Wrap small dimension/lookup tables in `F.broadcast()` when joining against a large fact table, especially after filters or transformations that prevent Spark from inferring the size automatically (`spark.sql.autoBroadcastJoinThreshold` only auto-broadcasts tables Spark can size, typically under 10MB). Confirm with `df.explain()` — look for `BroadcastHashJoin` vs `SortMergeJoin`.
- Never reach for `.dropDuplicates()` to mask unexpected duplicate rows — find the root cause; it also adds shuffle overhead.

```python
flights = flights.alias("f")
parking = parking.alias("p")
result = flights.join(F.broadcast(parking), "code", how="left").select(
    F.col("f.start_time").alias("flight_start"),
    F.col("p.total_time").alias("parking_total"),
)
```

## Window Functions

Use `from pyspark.sql import Window as W`.

- Always specify an explicit frame — without one, `F.sum().over(w)` behaves differently depending on whether `orderBy` is present (running sum vs. total).
- Know the difference between `row_number()` + filter (drops rows, keeps the best one) and `first()` over a window (overwrites a column, keeps all rows).
- Pass `ignorenulls=True` to `F.first()`/`F.last()` — otherwise a null in the first row of a partition nulls the entire partition's result.
- Avoid empty `partitionBy()`; it forces all data into a single partition. Use `.agg()` for global aggregations instead.

```python
w = W.partitionBy("key").orderBy("num").rowsBetween(W.unboundedPreceding, W.unboundedFollowing)
df = df.withColumn("version", F.first("version", ignorenulls=True).over(w))
```

## Map & Array Higher-Order Functions

- Use `map_zip_with` instead of `map_concat` when merging maps needs per-key conflict resolution (e.g., keep the entry with the later timestamp) rather than one side blindly winning.
- Use `transform` + `array_max`/`array_min` to extract values out of nested structs without a UDF.
- Avoid UDFs — check for a built-in Spark function or higher-order function first. UDFs break Catalyst optimization and add serialization overhead.

```python
merged = F.map_zip_with(
    new_map, existing_map,
    lambda key, v1, v2: (
        F.when(v1.isNull(), v2)
        .when(v2.isNull(), v1)
        .otherwise(F.when(v1.event_ts >= v2.event_ts, v1).otherwise(v2))
    ),
)
```

## Cumulative / Snapshot Table Patterns

- Merges must be idempotent — re-running with the same input data must not create duplicates.
- Merges must be order-independent — backfilling old data must never overwrite newer data. Resolve conflicts with an explicit criterion (event timestamp, version number, partition date), never positional precedence like `coalesce` argument order.
- Validate primary-key uniqueness and null counts on key columns as an audit step after every write.

## Data Quality & Performance

- Use `F.lit(None)` for empty columns — never empty strings or sentinel values like `"NA"`.
- Avoid `.otherwise()` as a catch-all in `F.when()` chains for categorical mappings; an unmapped value should surface as null, not silently collapse into "Other".
- Never leave `.show()`, `.collect()`, or `.printSchema()` in production code — they force full materialization or add driver overhead. `.count()` is fine when used intentionally for row-count logging or to force materialization before a DAG fork.
- Use `.persist()` only when a DataFrame is referenced by multiple subsequent actions. Choose the storage level deliberately: `MEMORY_AND_DISK` (safe default), `MEMORY_ONLY` (fastest, risks recompute on eviction), `DISK_ONLY` (for DataFrames too large for memory).

## Iceberg Write Patterns

- Use `.byName()` when writing so Spark matches columns by name, not position — this keeps writes safe across schema evolution.

```python
df.write.byName().mode("overwrite").insertInto("catalog.my_table")
```

- Use the `__partitions` Iceberg metadata table to find the latest snapshot instead of scanning the full table:

```python
partition_df = spark.read.table("catalog.my_table__partitions").select(
    "partition.partition_date", "partition.partition_hour"
)
max_partition = partition_df.orderBy(
    F.col("partition_date").desc(), F.col("partition_hour").desc()
).first()
if max_partition is None:
    raise ValueError("No partitions found in catalog.my_table")
```

- Choose `write.distribution-mode` deliberately: `"none"` (fastest, no re-shuffle, file sizes depend on upstream partitioning), `"hash"` (shuffles by partition key for evenly sized files), `"range"` (sorts before writing, best scan performance but most expensive).

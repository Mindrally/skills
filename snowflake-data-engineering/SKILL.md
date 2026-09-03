---
name: snowflake-data-engineering
description: "Best practices for Snowflake SQL, semi-structured data, and data pipelines built with Dynamic Tables, Streams, Tasks, and Snowpipe. Use when writing Snowflake SQL, designing ingestion or transformation pipelines, tuning warehouse performance and cost, or working with Time Travel, cloning, RBAC, or Iceberg tables on Snowflake."
---

# Snowflake Data Engineering

This skill covers SQL conventions, pipeline architecture (Dynamic Tables, Streams, Tasks, Snowpipe), performance tuning, and cost/access management on Snowflake.

## Workflow for Building a Snowflake Pipeline

1. **Land raw data** — Use Snowpipe (`AUTO_INGEST = TRUE`) for continuous file loads from an external stage, or Snowpipe Streaming for low-latency row-level ingestion via SDK.
2. **Choose a transformation approach** — Prefer Dynamic Tables for declarative, most pipelines; fall back to Streams + Tasks only when you need procedural logic or stored-procedure calls.
3. **Model semi-structured data** — Land raw JSON/Avro/Parquet as `VARIANT`, then flatten into typed relational columns as early as practical.
4. **Chain pipeline stages** — Build Dynamic Tables on top of each other (or Streams feeding Tasks) so each stage narrows scope from raw to cleaned to aggregated.
5. **Tune for performance** — Add clustering keys or Search Optimization only where query patterns justify them; tag queries for cost attribution.
6. **Set access controls** — Apply least-privilege RBAC with functional roles (loader, transformer, analyst) and masking/row-access policies for sensitive data.
7. **Monitor cost and freshness** — Track `WAREHOUSE_METERING_HISTORY` and `QUERY_HISTORY`, set Resource Monitors, and validate `TARGET_LAG` matches actual freshness requirements.

## SQL and Semi-Structured Data

- Use `VARIANT`, `OBJECT`, and `ARRAY` types for JSON, Avro, Parquet, and ORC data.
- Access nested fields with colon notation and cast explicitly: `src:customer.name::STRING`, `src:price::NUMBER(10,2)`, `src:created_at::TIMESTAMP_NTZ`.
- Flatten arrays with `LATERAL FLATTEN`:

```sql
SELECT f.value:name::STRING AS name
FROM my_table, LATERAL FLATTEN(input => src:items) f;
```

- Flatten semi-structured data into relational columns whenever it contains dates, numbers stored as strings, or arrays — keeping data inside `VARIANT` prevents Snowflake's automatic subcolumnarization from paying off.
- Avoid mixing types within the same `VARIANT` field for the same reason.
- Remember that a JSON `null` is stored as the string `"null"`, distinct from a SQL `NULL`. Use `STRIP_NULL_VALUES => TRUE` on load when you want them treated the same.

### SQL coding standards

- Use `snake_case` for all identifiers; avoid quoted identifiers.
- Prefer CTEs over deeply nested subqueries for readability.
- Use `CREATE OR REPLACE` for idempotent DDL.
- Use `COPY INTO` for bulk loading, never row-by-row `INSERT`.
- Use `MERGE` for upserts:

```sql
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.name = s.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name);
```

- In stored procedures, prefix session/local variables with `:` when referencing them inside SQL statements:

```sql
CREATE PROCEDURE my_proc(p_id INT) RETURNS STRING LANGUAGE SQL AS
BEGIN
  LET result STRING;
  SELECT name INTO :result FROM users WHERE id = :p_id;
  RETURN result;
END;
```

## Performance Optimization

- Add cluster keys only for very large tables (multi-TB) on columns frequently used in `WHERE`/`JOIN`/`GROUP BY`:

```sql
ALTER TABLE large_events CLUSTER BY (event_date, region);
```

- Use the Search Optimization Service for point lookups on high-cardinality columns or substring/regex search:

```sql
ALTER TABLE logs ADD SEARCH OPTIMIZATION ON EQUALITY(sender_ip), SUBSTRING(error_message);
```

- Use Materialized Views to pre-compute expensive single-table aggregations.
- Reuse prior results with `RESULT_SCAN(LAST_QUERY_ID())` instead of re-running an identical query.
- Tag queries for cost attribution: `ALTER SESSION SET QUERY_TAG = 'etl_daily_load';`
- Never run `SELECT *` on wide tables — it defeats columnar pruning benefits.

## Data Pipelines

Choose the right primitive:

| Approach | When to use |
|---|---|
| Dynamic Tables | Declarative: define the query, Snowflake manages refresh. Default choice for most pipelines. |
| Streams + Tasks | Imperative CDC + scheduling; needed for procedural logic or stored-procedure calls. |
| Snowpipe | Continuous file loading from S3/GCS/Azure. |
| Snowpipe Streaming | Low-latency row-level ingestion via SDK (Java, Python). |

### Dynamic Tables

```sql
CREATE OR REPLACE DYNAMIC TABLE cleaned_events
  TARGET_LAG = '5 minutes'
  WAREHOUSE = transform_wh
  AS
  SELECT event_id, event_type, user_id, event_data:page::STRING AS page, event_timestamp
  FROM raw_events
  WHERE event_type IS NOT NULL;

-- Chain for multi-step pipelines
CREATE OR REPLACE DYNAMIC TABLE user_sessions
  TARGET_LAG = '10 minutes'
  WAREHOUSE = transform_wh
  AS
  SELECT user_id, MIN(event_timestamp) AS session_start, MAX(event_timestamp) AS session_end,
         COUNT(*) AS event_count
  FROM cleaned_events GROUP BY user_id;
```

`TARGET_LAG` sets the freshness target. `REFRESH_MODE` can be `AUTO`, `FULL`, or `INCREMENTAL`. Manage lifecycle with `ALTER DYNAMIC TABLE ... SET TARGET_LAG / REFRESH / SUSPEND / RESUME`.

### Streams (CDC)

```sql
CREATE OR REPLACE STREAM raw_events_stream ON TABLE raw_events;
```

Streams add `METADATA$ACTION`, `METADATA$ISUPDATE`, and `METADATA$ROW_ID` columns. Set `APPEND_ONLY = TRUE` for insert-only sources to lower overhead.

### Tasks (scheduled/triggered)

```sql
CREATE OR REPLACE TASK process_events
  WAREHOUSE = transform_wh
  SCHEDULE = 'USING CRON 0 */1 * * * America/Los_Angeles'
  WHEN SYSTEM$STREAM_HAS_DATA('raw_events_stream')
  AS
  INSERT INTO cleaned_events
  SELECT event_id, event_type, user_id, event_timestamp
  FROM raw_events_stream WHERE event_type IS NOT NULL;
```

Build Task DAGs with `CREATE TASK child_task ... AFTER parent_task ...`. Tasks are created `SUSPENDED` by default — remember `ALTER TASK ... RESUME` or nothing will run.

### Snowpipe

```sql
CREATE OR REPLACE PIPE my_pipe AUTO_INGEST = TRUE AS
  COPY INTO raw_events FROM @my_external_stage FILE_FORMAT = (TYPE = 'JSON');
```

A common end-to-end pattern is Snowpipe landing raw data, feeding a chain of Dynamic Tables.

## Time Travel and Data Protection

- Query historical data with Time Travel (1 day by default, up to 90 on Enterprise+):

```sql
SELECT * FROM my_table AT(TIMESTAMP => '2026-01-15 10:00:00'::TIMESTAMP);
SELECT * FROM my_table BEFORE(STATEMENT => '<query_id>');
```

- Recover dropped objects with `UNDROP TABLE/SCHEMA/DATABASE`.
- Use zero-copy cloning for dev/test environments or backups without duplicating storage: `CREATE TABLE clone CLONE source;`, `CREATE SCHEMA dev CLONE prod;`.

## Snowflake Postgres

- Snowflake offers managed PostgreSQL (v16/17/18) with full wire compatibility: `CREATE POSTGRES INSTANCE my_instance COMPUTE_FAMILY='STANDARD_S' STORAGE_SIZE_GB=50;`
- Bridge OLTP to analytics with the `pg_lake` extension, which exposes Iceberg tables readable from both Postgres and Snowflake.
- Use `FORK` for point-in-time recovery and `HIGH_AVAILABILITY = TRUE` for production instances.

## Warehouse and Cost Management

- Size warehouses by query complexity, not raw data volume — start at X-Small and scale up only when needed.
- Set `AUTO_SUSPEND = 60` and `AUTO_RESUME = TRUE`; use separate warehouses per workload so a heavy job doesn't starve interactive queries.
- Use multi-cluster warehouses for concurrency scaling, not for single-query speed.
- Use transient tables for staging data to avoid Fail-safe storage cost.
- Monitor spend via `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` and `WAREHOUSE_METERING_HISTORY`, and set Resource Monitors to cap credit consumption.

## Access Control

- Apply least-privilege RBAC using database roles for object grants.
- Use masking policies for PII and row access policies for multi-tenant isolation.
- Structure functional roles around pipeline stages: loader (write raw), transformer (read raw, write analytics), analyst (read analytics only).

## Data Sharing and Iceberg

- Use `CREATE SHARE` for zero-copy cross-account data sharing, or the Snowflake Marketplace for external exchange.
- Create Iceberg tables with `CREATE ICEBERG TABLE ... CATALOG='SNOWFLAKE' EXTERNAL_VOLUME='vol' BASE_LOCATION='path/';` for interoperability with Spark, Flink, and Trino.

## Anti-Patterns

- Do not use Streams + Tasks for simple transformations that a Dynamic Table can express declaratively.
- Do not set `TARGET_LAG` shorter than the actual freshness requirement — it directly drives compute cost.
- Do not forget to `RESUME` tasks after creation; they start `SUSPENDED`.
- Do not run `SELECT *` on wide tables, and do not skip clustering analysis on multi-TB tables before adding cluster keys.
- Do not hardcode database/schema names in reusable pipeline code.

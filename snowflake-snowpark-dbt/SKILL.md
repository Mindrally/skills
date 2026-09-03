---
name: snowflake-snowpark-dbt
description: "Best practices for Snowpark Python (DataFrames, UDFs, UDTFs, stored procedures) and dbt with the dbt-snowflake adapter. Use when writing server-side Snowpark pipelines, registering UDFs or stored procedures, choosing dbt materializations, configuring incremental models, or setting up sources and tests for a Snowflake-backed dbt project."
---

# Snowflake Snowpark Python & dbt

This skill covers building production data transformation pipelines with Snowpark Python (Snowflake's server-side Python API) and with dbt using the `dbt-snowflake` adapter.

## Workflow for a Snowpark or dbt Transformation

1. **Snowpark: open a session** — Build a `Session` from environment-scoped credentials, specifying role, warehouse, database, and schema explicitly.
2. **Snowpark: express transforms with the DataFrame API** — Prefer `.filter()`, `.select()`, `.group_by().agg()`, and `.join()` over raw SQL strings for reusable pipeline code; DataFrames are lazily evaluated and only execute on `.collect()`/`.show()`/a write action.
3. **Snowpark: push compute server-side** — Use scalar UDFs for row-wise logic, vectorized (pandas) UDFs for ML inference, UDTFs when one input row produces multiple output rows, and stored procedures for multi-step server-side orchestration.
4. **dbt: model in layers** — Staging models (`stg_*`) rename and type-cast; mart models express business logic on top of staging.
5. **dbt: choose a materialization** — `view` for cheap logic, `table` only when reads are frequent, `incremental` for large fact tables, `dynamic_table` for near-real-time freshness needs.
6. **dbt: define sources and tests** — Declare sources in `_sources.yml` with freshness thresholds; add `unique`/`not_null` tests on key columns.
7. **dbt: run selectively** — Use `dbt run --select model+` (model and downstream) or `+model` (model and upstream) instead of full-project runs during iteration.
8. **dbt: build and validate** — Run `dbt build` (run + test in dependency order) before merging, and `dbt docs generate` to keep documentation current.

## Snowpark Python

Snowpark runs Python server-side inside a Snowflake warehouse — data never leaves Snowflake. Core abstractions: `Session`, `DataFrame`, UDF, UDTF, UDAF, and Stored Procedure.

### Session

```python
import os
from snowflake.snowpark import Session

session = Session.builder.configs({
    "account": os.environ["SNOWFLAKE_ACCOUNT"],
    "user": os.environ["SNOWFLAKE_USER"],
    "password": os.environ["SNOWFLAKE_PASSWORD"],
    "role": "my_role", "warehouse": "my_wh", "database": "my_db", "schema": "my_schema",
}).create()
```

Never hardcode credentials — always read them from environment variables or a secrets manager.

### DataFrame API

DataFrames are lazily evaluated: Snowpark builds a query plan and only executes it on `collect()`/`show()` or a write action.

```python
df = session.table("customers")
df_filtered = df.filter(df["region"] == "US").select("name", "email", "revenue")
df_agg = df.group_by("region").agg(sum("revenue").alias("total_revenue"))
df_agg.show()
```

Key operations: `.filter()`, `.select()`, `.group_by().agg()`, `.join()`, `.sort()`, `.with_column()`, `.drop()`, `.distinct()`, `.limit()`, `.union_all()`, `.flatten()`, `.write.save_as_table()`.

### Scalar UDFs

```python
from snowflake.snowpark.functions import udf

@udf(name="normalize_email", replace=True)
def normalize_email(email: str) -> str:
    return email.strip().lower() if email else None
```

### Vectorized UDFs

Vectorized (pandas) UDFs are 10-100x faster than scalar UDFs for ML inference because they batch rows instead of invoking Python per row.

```python
import pandas as pd
from snowflake.snowpark.functions import udf

@udf(name="predict_score", packages=["scikit-learn", "pandas"], replace=True)
def predict_score(features: pd.Series) -> pd.Series:
    import pickle, sys
    model = pickle.load(open(sys.path[0] + "/model.pkl", "rb"))
    return pd.Series(model.predict(features.values.reshape(-1, 1)))
```

### UDTFs (return multiple rows per input)

```python
from snowflake.snowpark.types import StructType, StructField, StringType

class Tokenizer:
    def process(self, text: str):
        for token in text.split():
            yield (token,)

tokenize = session.udtf.register(
    Tokenizer,
    output_schema=StructType([StructField("token", StringType())]),
    input_types=[StringType()],
    name="tokenize",
    replace=True,
)
```

### Stored procedures

```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import sproc

@sproc(name="daily_etl", replace=True, packages=["snowflake-snowpark-python"])
def daily_etl(session: Session) -> str:
    raw = session.table("raw_events")
    cleaned = raw.filter(raw["event_type"].is_not_null())
    cleaned.write.mode("overwrite").save_as_table("cleaned_events")
    return f"Processed {cleaned.count()} rows"
```

### Packages and file access

- Add third-party packages with `session.add_packages("pandas", "scikit-learn==1.3.0", "xgboost")` — pin versions for production UDFs and stored procedures.
- Attach static files (e.g., a pickled model) with `session.add_import("@my_stage/model.pkl")`.
- For pandas-on-Snowflake with no data movement to the client, use `modin.pandas` with the Snowpark plugin: `import modin.pandas as pd; import snowflake.snowpark.modin.plugin; df = pd.read_snowflake("my_table")`.

## dbt with the Snowflake Adapter

Install with `pip install dbt-snowflake`.

### profiles.yml

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: myaccount
      user: myuser
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: transformer
      database: analytics
      warehouse: transforming
      schema: public
      threads: 4
```

Never commit real credentials into `profiles.yml` — always source secrets from `env_var()`.

### Materializations

Available materializations: `view`, `table`, `incremental`, `ephemeral`, `dynamic_table`. Default to `view` for cheap logic; reserve `table` for models read frequently enough to justify storage cost; use `incremental` for large fact tables and `dynamic_table` when near-real-time freshness matters.

### Dynamic Tables in dbt

```sql
{{ config(materialized='dynamic_table', snowflake_warehouse='transforming', target_lag='1 hour') }}
SELECT customer_id, SUM(amount) AS lifetime_value FROM {{ ref('stg_orders') }} GROUP BY 1
```

### Incremental models

```sql
{{
  config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    on_schema_change='sync_all_columns'
  )
}}
SELECT * FROM {{ ref('stg_events') }}
{% if is_incremental() %}
  WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

Always guard `{{ this }}` with `{% if is_incremental() %}` — referencing it unconditionally breaks the first (full) run, when the target table doesn't exist yet.

### Snowflake-specific configs

- `cluster_by=['col1', 'col2']` — clustering, large tables only (generally >1TB).
- `transient=true` — no Fail-safe, lower storage cost; use for staging models.
- `query_tag='finance_daily'` — workload attribution for cost tracking.
- `copy_grants=true` — preserve access grants across a `CREATE OR REPLACE`.
- `snowflake_warehouse='lg_wh'` — per-model warehouse override for heavy transforms.
- `secure=true` — secure views, for models exposing sensitive columns.

### Sources with freshness checks

```yaml
sources:
  - name: raw
    database: raw_db
    schema: jaffle_shop
    tables:
      - name: customers
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
```

### Testing

```yaml
models:
  - name: stg_customers
    columns:
      - name: customer_id
        tests: [unique, not_null]
```

### Key commands

- `dbt run`, `dbt test`, `dbt build` (run + test in dependency order), `dbt compile`
- `dbt run --select my_model+` — model and everything downstream
- `dbt run --select +my_model` — model and everything upstream
- `dbt source freshness` — check source staleness against configured thresholds
- `dbt docs generate && dbt docs serve` — build and preview documentation

### Custom schema naming

```jinja
{% macro generate_schema_name(custom_schema_name, node) %}
  {% if custom_schema_name %}{{ custom_schema_name | trim }}{% else %}{{ target.schema }}{% endif %}
{% endmacro %}
```

## Best Practices

- Prefer vectorized (pandas) UDFs over scalar UDFs for ML inference.
- Pin package versions in production UDFs and stored procedures.
- Use the Snowpark DataFrame API over raw SQL strings in reusable Python pipelines.
- Use staging models (`stg_*`) to rename and type-cast; keep business logic in mart models.
- Use `incremental` materialization for fact tables and `dynamic_table` for near-real-time needs.
- Set `on_schema_change='sync_all_columns'` on incremental models to handle upstream schema drift safely.
- Use `copy_grants=true` to avoid permission churn on rebuild, and tag models for selective execution.
- Use separate warehouses for dbt runs versus interactive analyst queries.

## Anti-Patterns

- Do not `.collect()` large Snowpark DataFrames to the client — keep processing server-side.
- Do not use Python loops over rows in Snowpark — use DataFrame operations or vectorized UDFs.
- Do not reference `{{ this }}` in an incremental model without an `{% if is_incremental() %}` guard.
- Do not set `cluster_by` on small tables (under roughly 1TB) — the overhead outweighs the benefit.
- Do not default every model to `materialized='table'` — views are free until queried.
- Do not hardcode database/schema names in dbt models — use `{{ ref() }}` and `{{ source() }}`.

---
chapter: 12
title: "The Transformation Layer: dbt"
estimated_pages: 35-40
status: draft
last_updated: 2025-01-22
---

# The Transformation Layer: dbt

The previous chapters covered where data lives (storage layer) and how to move it (orchestration). This chapter addresses a critical question: once data lands in your warehouse or lakehouse, how do you transform it into something useful?

For years, this transformation work happened in two places: ETL tools that transformed data before loading, or sprawling SQL scripts maintained by hand. Neither approach scaled well. ETL tools created black boxes. SQL scripts became unmaintainable tangles of dependencies.

dbt (data build tool) changed this landscape by applying software engineering principles to SQL transformations. Version control. Testing. Documentation. Modularity. Dependency management. These weren't new ideas—they'd been standard in software development for decades—but dbt brought them to analytics engineering.

This chapter explores dbt as a transformation layer: its mental model, core concepts, and patterns for building maintainable data transformations at scale.

## The Problem dbt Solves

### Before dbt: The SQL Script Jungle

Consider a typical pre-dbt analytics workflow:

```
/sql_scripts/
├── 01_create_staging_tables.sql
├── 02_load_daily_trades.sql
├── 03_join_with_symbols.sql
├── 04_compute_metrics.sql
├── 05_update_dashboard_tables.sql
├── fix_for_btc_issue.sql
├── johns_metrics_v2.sql
├── metrics_final_FINAL.sql
└── run_everything.sh
```

Problems multiply:
- **Dependencies are implicit:** Script 04 must run after 03, but nothing enforces this
- **No testing:** How do you know metrics are correct?
- **No documentation:** What does `johns_metrics_v2.sql` do?
- **No modularity:** Common logic copied between scripts
- **Credentials in code:** Connection strings scattered everywhere
- **No version history:** Who changed what, when?

### The ETL Tool Approach

Traditional ETL tools (Informatica, Talend, SSIS) solved some problems but created others:
- **Visual workflows:** Drag-and-drop interfaces that became incomprehensible at scale
- **Proprietary logic:** Transformations locked in vendor-specific formats
- **Limited testing:** GUI-based tools didn't integrate with testing frameworks
- **Version control friction:** Binary or XML formats that don't diff well

### dbt's Innovation

dbt's insight: treat SQL transformations like software. Models are code. Dependencies are explicit. Tests are first-class. Documentation lives with the code.

```sql
-- models/marts/trading/daily_metrics.sql
-- This is a dbt model: SQL with metadata

{{ config(
    materialized='incremental',
    unique_key='date',
    partition_by={'field': 'date', 'data_type': 'date'}
) }}

SELECT
    date,
    symbol,
    COUNT(*) as trade_count,
    SUM(quantity) as total_volume,
    AVG(price) as avg_price
FROM {{ ref('stg_trades') }}
{% if is_incremental() %}
WHERE date > (SELECT MAX(date) FROM {{ this }})
{% endif %}
GROUP BY date, symbol
```

Key innovations visible here:
- **`ref()` function:** Explicit dependency on `stg_trades`
- **Configuration in code:** Materialization, partitioning defined in the model
- **Incremental logic:** Built-in patterns for processing only new data
- **Jinja templating:** Programmable SQL generation

## How dbt Works

### The Execution Model

dbt operates on a simple principle: compile SQL templates, then execute them against your warehouse.

```
┌─────────────────────────────────────────────────────────────────┐
│                      dbt Execution Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. PARSE                                                        │
│  ┌──────────────┐                                                │
│  │ models/*.sql │──▶ Parse SQL + Jinja                          │
│  │ schema.yml   │──▶ Extract refs, configs                      │
│  └──────────────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  2. BUILD DAG                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  stg_trades ──▶ int_trades_enriched ──▶ daily_metrics   │   │
│  │  stg_symbols ─────────┘                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│         │                                                        │
│         ▼                                                        │
│  3. COMPILE                                                      │
│  ┌──────────────┐                                                │
│  │ Jinja ──▶ SQL │  {{ ref('stg_trades') }} ──▶ "schema.stg_trades"│
│  └──────────────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  4. EXECUTE                                                      │
│  ┌──────────────┐                                                │
│  │  Warehouse   │  CREATE TABLE daily_metrics AS (SELECT ...)   │
│  │  (Snowflake, │                                                │
│  │   BigQuery,  │                                                │
│  │   etc.)      │                                                │
│  └──────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:** dbt doesn't move data. It generates and executes SQL. Your warehouse does all the heavy lifting.

### Project Structure

A typical dbt project follows this structure:

```
trading_analytics/
├── dbt_project.yml           # Project configuration
├── profiles.yml              # Connection profiles (usually in ~/.dbt/)
├── models/
│   ├── staging/              # Raw data cleanup
│   │   ├── stg_trades.sql
│   │   ├── stg_symbols.sql
│   │   └── staging.yml       # Tests and documentation
│   ├── intermediate/         # Business logic building blocks
│   │   ├── int_trades_enriched.sql
│   │   └── intermediate.yml
│   └── marts/                # Final, consumer-facing models
│       ├── trading/
│       │   ├── daily_metrics.sql
│       │   └── trading.yml
│       └── finance/
│           └── revenue.sql
├── seeds/                    # Static reference data (CSV)
│   └── symbol_metadata.csv
├── macros/                   # Reusable SQL snippets
│   └── calculate_vwap.sql
├── tests/                    # Custom data tests
│   └── assert_positive_volume.sql
└── snapshots/                # Slowly changing dimension tracking
    └── symbol_history.sql
```

### The DAG

dbt builds a Directed Acyclic Graph from `ref()` calls:

```sql
-- stg_trades.sql
SELECT * FROM {{ source('raw', 'trades') }}

-- stg_symbols.sql
SELECT * FROM {{ source('raw', 'symbols') }}

-- int_trades_enriched.sql
SELECT t.*, s.name as symbol_name
FROM {{ ref('stg_trades') }} t
JOIN {{ ref('stg_symbols') }} s ON t.symbol = s.symbol

-- daily_metrics.sql
SELECT date, symbol, COUNT(*) as trades
FROM {{ ref('int_trades_enriched') }}
GROUP BY date, symbol
```

dbt infers:
```
source.raw.trades ──▶ stg_trades ──┐
                                   ├──▶ int_trades_enriched ──▶ daily_metrics
source.raw.symbols ──▶ stg_symbols ┘
```

This DAG enables:
- **Correct execution order:** Dependencies run first
- **Selective execution:** Run just one model and its upstream dependencies
- **Lineage tracking:** Understand where data comes from

## Core Concepts

### Models

Models are SQL SELECT statements that define transformations. Each model becomes a table or view in your warehouse.

```sql
-- models/marts/trading/trader_performance.sql
{{ config(
    materialized='table',
    tags=['daily', 'trading']
) }}

WITH trades AS (
    SELECT * FROM {{ ref('stg_trades') }}
),

aggregated AS (
    SELECT
        trader_id,
        COUNT(*) as trade_count,
        SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) as winning_trades,
        SUM(profit) as total_profit,
        AVG(profit) as avg_profit
    FROM trades
    GROUP BY trader_id
)

SELECT
    trader_id,
    trade_count,
    winning_trades,
    ROUND(winning_trades * 100.0 / trade_count, 2) as win_rate,
    total_profit,
    avg_profit
FROM aggregated
```

### Materializations

How models are persisted in the warehouse:

**View:** A database view (no data storage, computed on query)
```sql
{{ config(materialized='view') }}
SELECT * FROM {{ ref('raw_trades') }} WHERE status = 'filled'
```

**Table:** A physical table (data copied, faster queries)
```sql
{{ config(materialized='table') }}
SELECT * FROM {{ ref('stg_trades') }}
```

**Incremental:** Append or merge new data (efficient for large tables)
```sql
{{ config(
    materialized='incremental',
    unique_key='trade_id'
) }}

SELECT * FROM {{ ref('stg_trades') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Ephemeral:** Not persisted; inlined as CTE in downstream models
```sql
{{ config(materialized='ephemeral') }}
SELECT symbol, AVG(price) as avg_price FROM {{ ref('stg_trades') }}
GROUP BY symbol
```

### Incremental Strategies

For large datasets, incremental models are essential. dbt 1.9 introduced microbatch, adding to existing strategies:

**Append:** Add new rows without touching existing data
```sql
{{ config(
    materialized='incremental'
) }}

SELECT * FROM {{ source('raw', 'events') }}
{% if is_incremental() %}
WHERE event_time > (SELECT MAX(event_time) FROM {{ this }})
{% endif %}
```

**Merge (Upsert):** Update existing rows, insert new ones
```sql
{{ config(
    materialized='incremental',
    unique_key='trade_id',
    incremental_strategy='merge'
) }}

SELECT * FROM {{ ref('stg_trades') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Delete+Insert:** Delete matching rows, then insert
```sql
{{ config(
    materialized='incremental',
    unique_key='date',
    incremental_strategy='delete+insert'
) }}

-- Delete and reinsert entire days
SELECT * FROM {{ ref('stg_daily_data') }}
{% if is_incremental() %}
WHERE date >= DATEADD(day, -3, CURRENT_DATE)
{% endif %}
```

**Microbatch (dbt 1.9+):** Process time-series data in configurable batches
```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_timestamp',
    begin='2024-01-01',
    batch_size='day'
) }}

SELECT
    event_timestamp,
    user_id,
    event_type,
    -- dbt automatically filters to current batch
    COUNT(*) OVER (PARTITION BY user_id) as user_event_count
FROM {{ source('raw', 'events') }}
```

Microbatch benefits:
- **Independent batches:** Failed batches can be retried individually
- **Parallel execution:** Multiple batches process simultaneously
- **Targeted backfills:** Reprocess specific date ranges with `--event-time-start` and `--event-time-end`
- **Simplified SQL:** Write for one batch, dbt handles the rest

### Sources

Sources declare external tables that dbt reads from but doesn't manage:

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: raw
    database: raw_db
    schema: trading
    tables:
      - name: trades
        description: Raw trade events from exchange API
        columns:
          - name: trade_id
            description: Unique trade identifier
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
        loaded_at_field: _loaded_at

      - name: symbols
        description: Symbol reference data
```

Reference in models:
```sql
SELECT * FROM {{ source('raw', 'trades') }}
```

**Freshness checks:**
```bash
dbt source freshness
# Warns if trades table hasn't been updated in 12 hours
# Errors if not updated in 24 hours
```

### Tests

dbt includes built-in tests and supports custom tests.

**Schema tests (YAML):**
```yaml
# models/staging/staging.yml
version: 2

models:
  - name: stg_trades
    description: Cleaned trading data
    columns:
      - name: trade_id
        description: Unique identifier
        tests:
          - unique
          - not_null

      - name: symbol
        tests:
          - not_null
          - relationships:
              to: ref('stg_symbols')
              field: symbol

      - name: quantity
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
```

**Singular tests (SQL):**
```sql
-- tests/assert_trade_value_positive.sql
-- Returns rows that fail the test (should return 0 rows)

SELECT *
FROM {{ ref('stg_trades') }}
WHERE price * quantity <= 0
```

**Run tests:**
```bash
dbt test                    # Run all tests
dbt test --select stg_*     # Test staging models only
dbt test --select tag:critical  # Test critical models
```

### Documentation

Documentation lives alongside code:

```yaml
# models/marts/trading/trading.yml
version: 2

models:
  - name: daily_metrics
    description: |
      Daily aggregated trading metrics per symbol.

      This model is the primary source for trading dashboards.
      Updated incrementally each day.

      **Grain:** One row per date per symbol
      **Update frequency:** Daily at 6 AM UTC

    columns:
      - name: date
        description: Trading date (UTC)

      - name: symbol
        description: Trading pair symbol (e.g., BTC-USD)

      - name: trade_count
        description: Number of filled trades

      - name: total_volume
        description: Sum of trade quantities

      - name: vwap
        description: |
          Volume-weighted average price.
          Calculated as: SUM(price * quantity) / SUM(quantity)
```

Generate documentation site:
```bash
dbt docs generate
dbt docs serve  # Opens browser to documentation site
```

The documentation site includes:
- Model descriptions
- Column definitions
- Lineage graphs
- Source freshness
- Test results

### Macros

Macros are reusable SQL snippets using Jinja:

```sql
-- macros/calculate_vwap.sql
{% macro calculate_vwap(price_col, quantity_col) %}
    SUM({{ price_col }} * {{ quantity_col }}) / NULLIF(SUM({{ quantity_col }}), 0)
{% endmacro %}
```

Use in models:
```sql
SELECT
    symbol,
    date,
    {{ calculate_vwap('price', 'quantity') }} as vwap
FROM {{ ref('stg_trades') }}
GROUP BY symbol, date
```

**Common macro patterns:**

Generate surrogate keys:
```sql
{% macro generate_surrogate_key(field_list) %}
    {{ dbt_utils.generate_surrogate_key(field_list) }}
{% endmacro %}
```

Date spine generation:
```sql
{% macro date_spine(start_date, end_date) %}
    {{ dbt_utils.date_spine(
        datepart="day",
        start_date="cast('" ~ start_date ~ "' as date)",
        end_date="cast('" ~ end_date ~ "' as date)"
    ) }}
{% endmacro %}
```

### Snapshots

Snapshots track slowly changing dimensions (SCD Type 2):

```sql
-- snapshots/symbol_snapshot.sql
{% snapshot symbol_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='symbol',
        strategy='timestamp',
        updated_at='updated_at'
    )
}}

SELECT * FROM {{ source('raw', 'symbols') }}

{% endsnapshot %}
```

Result: Historical record of every symbol change with `dbt_valid_from` and `dbt_valid_to` columns.

**dbt 1.9 enhancement:** Snapshots can now be defined in YAML:
```yaml
# snapshots/symbols.yml
snapshots:
  - name: symbol_snapshot
    config:
      strategy: timestamp
      unique_key: symbol
      updated_at: updated_at
    columns:
      - name: symbol
        description: Symbol identifier
```

## Project Organization Patterns

### The Staging/Intermediate/Marts Pattern

The most common pattern organizes models into three layers:

```
models/
├── staging/        # 1:1 with source tables, cleaning only
├── intermediate/   # Business logic building blocks
└── marts/          # Final consumer-facing models
```

**Staging:** Clean raw data. One staging model per source table.
- Rename columns to consistent conventions
- Cast data types
- Handle nulls
- Filter obviously bad records
- No joins, no business logic

```sql
-- models/staging/stg_trades.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'trades') }}
),

cleaned AS (
    SELECT
        -- Rename to consistent convention
        trade_id,
        UPPER(symbol) as symbol,
        CAST(price AS DECIMAL(18, 8)) as price,
        CAST(quantity AS DECIMAL(18, 8)) as quantity,
        CAST(trade_timestamp AS TIMESTAMP) as traded_at,
        status,
        -- Add metadata
        _loaded_at as loaded_at
    FROM source
    WHERE trade_id IS NOT NULL  -- Filter invalid records
)

SELECT * FROM cleaned
```

**Intermediate:** Build reusable business logic. Join staging tables, apply transformations.
- Named with `int_` prefix
- May not be exposed to end users
- Think of these as building blocks

```sql
-- models/intermediate/int_trades_enriched.sql
WITH trades AS (
    SELECT * FROM {{ ref('stg_trades') }}
),

symbols AS (
    SELECT * FROM {{ ref('stg_symbols') }}
),

enriched AS (
    SELECT
        t.*,
        s.name as symbol_name,
        s.base_currency,
        s.quote_currency,
        t.price * t.quantity as trade_value
    FROM trades t
    LEFT JOIN symbols s ON t.symbol = s.symbol
)

SELECT * FROM enriched
```

**Marts:** Final, consumer-facing models organized by business domain.
- Wide tables optimized for analysis
- Clear business meaning
- Documented thoroughly

```sql
-- models/marts/trading/fct_daily_trading.sql
{{ config(
    materialized='incremental',
    unique_key=['date', 'symbol']
) }}

WITH trades AS (
    SELECT * FROM {{ ref('int_trades_enriched') }}
    {% if is_incremental() %}
    WHERE DATE(traded_at) > (SELECT MAX(date) FROM {{ this }})
    {% endif %}
),

daily AS (
    SELECT
        DATE(traded_at) as date,
        symbol,
        symbol_name,
        COUNT(*) as trade_count,
        SUM(quantity) as total_volume,
        SUM(trade_value) as total_value,
        {{ calculate_vwap('price', 'quantity') }} as vwap,
        MIN(price) as low_price,
        MAX(price) as high_price
    FROM trades
    GROUP BY DATE(traded_at), symbol, symbol_name
)

SELECT * FROM daily
```

### Naming Conventions

Consistent naming improves maintainability:

| Layer | Prefix | Example |
|-------|--------|---------|
| Staging | `stg_` | `stg_trades`, `stg_symbols` |
| Intermediate | `int_` | `int_trades_enriched` |
| Fact tables | `fct_` | `fct_daily_trading` |
| Dimension tables | `dim_` | `dim_symbols` |
| Metrics | `metrics_` | `metrics_trading_summary` |

### Modular Marts

Organize marts by business domain:

```
models/marts/
├── trading/
│   ├── fct_trades.sql
│   ├── fct_daily_trading.sql
│   ├── dim_symbols.sql
│   └── trading.yml
├── finance/
│   ├── fct_revenue.sql
│   ├── dim_accounts.sql
│   └── finance.yml
└── analytics/
    ├── user_engagement.sql
    └── analytics.yml
```

Each domain can have different:
- Materialization strategies
- Refresh schedules
- Access permissions
- Testing requirements

## Advanced Patterns

### Packages

Extend dbt with community packages:

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1

  - package: dbt-labs/codegen
    version: 0.12.1

  - package: calogica/dbt_expectations
    version: 0.10.3
```

Install with:
```bash
dbt deps
```

**dbt_utils:** Common utilities (surrogate keys, date spines, pivots)
**codegen:** Generate model and schema YAML from existing tables
**dbt_expectations:** Great Expectations-style data quality tests

### Dynamic SQL with Jinja

Jinja enables powerful SQL generation:

**Conditional logic:**
```sql
SELECT
    date,
    symbol,
    {% if target.name == 'prod' %}
        -- Full aggregation in production
        SUM(quantity) as volume,
        COUNT(*) as trades
    {% else %}
        -- Simplified in development
        0 as volume,
        0 as trades
    {% endif %}
FROM {{ ref('stg_trades') }}
GROUP BY date, symbol
```

**Looping:**
```sql
{% set metrics = ['volume', 'trades', 'value'] %}
{% set periods = [7, 30, 90] %}

SELECT
    symbol,
    {% for metric in metrics %}
        {% for period in periods %}
            SUM({{ metric }}) OVER (
                PARTITION BY symbol
                ORDER BY date
                ROWS BETWEEN {{ period - 1 }} PRECEDING AND CURRENT ROW
            ) as {{ metric }}_{{ period }}d
            {% if not loop.last %},{% endif %}
        {% endfor %}
        {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('fct_daily_trading') }}
```

**Programmatic model generation:**
```sql
-- macros/generate_metric_model.sql
{% macro generate_metric_model(metric_name, aggregation, source_model) %}

SELECT
    date,
    symbol,
    {{ aggregation }}({{ metric_name }}) as {{ metric_name }}_{{ aggregation }}
FROM {{ ref(source_model) }}
GROUP BY date, symbol

{% endmacro %}
```

### Cross-Database Patterns

dbt supports multiple adapters. Write portable SQL:

```sql
-- Use dbt's cross-database macros
SELECT
    {{ dbt.date_trunc('day', 'traded_at') }} as trade_date,
    {{ dbt.safe_divide('total_value', 'total_volume') }} as avg_price,
    {{ dbt.concat(['symbol', "'-'", 'exchange']) }} as symbol_exchange
FROM {{ ref('stg_trades') }}
```

### Hooks

Execute SQL before or after model runs:

```sql
{{ config(
    post_hook=[
        "GRANT SELECT ON {{ this }} TO ROLE analyst",
        "ANALYZE {{ this }}"
    ]
) }}

SELECT * FROM {{ ref('stg_trades') }}
```

Project-wide hooks in `dbt_project.yml`:
```yaml
on-run-end:
  - "{{ dbt_utils.log_info('dbt run completed') }}"
  - "CALL refresh_dashboards()"
```

### Custom Materializations

For advanced use cases, create custom materializations:

```sql
-- macros/materializations/table_with_clustering.sql
{% materialization clustered_table, adapter='snowflake' %}

    {% set target_relation = this %}
    {% set cluster_by = config.get('cluster_by') %}

    {% call statement('main') %}
        CREATE OR REPLACE TABLE {{ target_relation }}
        CLUSTER BY ({{ cluster_by | join(', ') }})
        AS (
            {{ sql }}
        )
    {% endcall %}

    {{ return({'relations': [target_relation]}) }}

{% endmaterialization %}
```

Use in models:
```sql
{{ config(
    materialized='clustered_table',
    cluster_by=['date', 'symbol']
) }}

SELECT * FROM {{ ref('stg_trades') }}
```

## Testing Strategies

### Test Pyramid

Apply the testing pyramid to dbt:

```
        /\
       /  \    E2E Tests
      /    \   (Full pipeline validation)
     /──────\
    /        \  Integration Tests
   /          \ (Cross-model consistency)
  /────────────\
 /              \ Unit Tests
/                \ (Column-level: unique, not_null, relationships)
```

### Schema Tests (Unit Level)

```yaml
models:
  - name: fct_daily_trading
    columns:
      - name: date
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: date

      - name: symbol
        tests:
          - not_null
          - relationships:
              to: ref('dim_symbols')
              field: symbol

      - name: trade_count
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0

      - name: vwap
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true
```

### Custom Tests (Integration Level)

```sql
-- tests/assert_daily_volume_matches_trades.sql
-- Volume in daily table should match sum of trades

WITH daily AS (
    SELECT date, symbol, total_volume
    FROM {{ ref('fct_daily_trading') }}
),

expected AS (
    SELECT
        DATE(traded_at) as date,
        symbol,
        SUM(quantity) as expected_volume
    FROM {{ ref('stg_trades') }}
    GROUP BY DATE(traded_at), symbol
)

SELECT
    d.date,
    d.symbol,
    d.total_volume,
    e.expected_volume,
    ABS(d.total_volume - e.expected_volume) as difference
FROM daily d
JOIN expected e ON d.date = e.date AND d.symbol = e.symbol
WHERE ABS(d.total_volume - e.expected_volume) > 0.01
```

### Data Quality Tests

Use dbt_expectations for comprehensive testing:

```yaml
models:
  - name: fct_daily_trading
    tests:
      # Table-level tests
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          max_value: 1000000

      - dbt_expectations.expect_table_columns_to_match_ordered_list:
          column_list:
            - date
            - symbol
            - trade_count
            - total_volume
            - vwap

    columns:
      - name: date
        tests:
          - dbt_expectations.expect_column_values_to_be_unique:
              row_condition: "symbol = 'BTC-USD'"  # Unique per symbol

          - dbt_expectations.expect_column_distinct_count_to_be_greater_than:
              value: 30  # At least 30 days of data
```

## dbt in Production

### CI/CD Pipeline

A typical dbt CI/CD pipeline:

```yaml
# .github/workflows/dbt.yml
name: dbt CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  dbt-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: Install packages
        run: dbt deps

      - name: Check formatting
        run: sqlfmt --check models/

      - name: Run dbt build (CI environment)
        run: dbt build --target ci --full-refresh
        env:
          DBT_PROFILES_DIR: ./

      - name: Test
        run: dbt test --target ci

  dbt-deploy:
    needs: dbt-ci
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to production
        run: dbt run --target prod
        env:
          DBT_PROFILES_DIR: ./
```

### Orchestration Integration

**With Airflow (via Cosmos):**
```python
from cosmos import DbtTaskGroup, ProfileConfig, ProjectConfig

profile_config = ProfileConfig(
    profile_name="trading_analytics",
    target_name="prod",
)

dbt_tasks = DbtTaskGroup(
    project_config=ProjectConfig(dbt_project_path="/path/to/dbt"),
    profile_config=profile_config,
    operator_args={"install_deps": True},
)
```

**With Dagster (native integration):**
```python
from dagster_dbt import DbtCliResource, dbt_assets

@dbt_assets(manifest=dbt_project.manifest_path)
def my_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()
```

**With Prefect:**
```python
from prefect import flow
from prefect_dbt import DbtCoreOperation

@flow
def dbt_flow():
    dbt_run = DbtCoreOperation(
        commands=["dbt run --select marts.*"],
        project_dir="path/to/dbt",
        profiles_dir="~/.dbt"
    )
    dbt_run.run()
```

### Slim CI

Run only models affected by changes:

```bash
# In CI, run only modified models and their downstream dependents
dbt run --select state:modified+ --defer --state ./prod-manifest
```

This requires:
1. Production manifest artifact (`manifest.json`)
2. Comparison of current state to production

### Environment Strategy

```yaml
# profiles.yml
trading_analytics:
  target: dev

  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      schema: dev_{{ env_var('USER') }}  # Per-developer schema
      # ...

    ci:
      type: snowflake
      schema: ci_{{ env_var('GITHUB_RUN_ID') }}  # Unique per CI run
      # ...

    prod:
      type: snowflake
      schema: analytics
      # ...
```

## Case Study: Building a Trading Analytics Platform

Let's trace a complete dbt implementation for trading data.

### Requirements

- Daily and hourly trading metrics
- Symbol performance rankings
- Trader leaderboards
- Real-time (ish) dashboards

### Project Structure

```
trading_analytics/
├── models/
│   ├── staging/
│   │   ├── stg_trades.sql
│   │   ├── stg_symbols.sql
│   │   ├── stg_traders.sql
│   │   └── staging.yml
│   ├── intermediate/
│   │   ├── int_trades_enriched.sql
│   │   └── int_trader_stats.sql
│   └── marts/
│       ├── trading/
│       │   ├── fct_hourly_trading.sql
│       │   ├── fct_daily_trading.sql
│       │   ├── dim_symbols.sql
│       │   └── trading.yml
│       └── analytics/
│           ├── symbol_rankings.sql
│           ├── trader_leaderboard.sql
│           └── analytics.yml
├── macros/
│   ├── calculate_vwap.sql
│   └── generate_date_spine.sql
└── tests/
    └── assert_volume_consistency.sql
```

### Key Models

**Hourly metrics with microbatch:**
```sql
-- models/marts/trading/fct_hourly_trading.sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='traded_at',
    begin='2024-01-01',
    batch_size='hour',
    unique_key=['hour', 'symbol']
) }}

WITH trades AS (
    SELECT * FROM {{ ref('int_trades_enriched') }}
),

hourly AS (
    SELECT
        DATE_TRUNC('hour', traded_at) as hour,
        symbol,
        symbol_name,
        COUNT(*) as trade_count,
        SUM(quantity) as volume,
        SUM(trade_value) as value,
        {{ calculate_vwap('price', 'quantity') }} as vwap,
        MIN(price) as low,
        MAX(price) as high,
        FIRST_VALUE(price) OVER (
            PARTITION BY DATE_TRUNC('hour', traded_at), symbol
            ORDER BY traded_at
        ) as open,
        LAST_VALUE(price) OVER (
            PARTITION BY DATE_TRUNC('hour', traded_at), symbol
            ORDER BY traded_at
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) as close
    FROM trades
    GROUP BY DATE_TRUNC('hour', traded_at), symbol, symbol_name
)

SELECT * FROM hourly
```

**Symbol rankings:**
```sql
-- models/marts/analytics/symbol_rankings.sql
{{ config(materialized='table') }}

WITH daily AS (
    SELECT * FROM {{ ref('fct_daily_trading') }}
    WHERE date >= DATEADD(day, -30, CURRENT_DATE)
),

metrics AS (
    SELECT
        symbol,
        symbol_name,
        SUM(volume) as total_volume_30d,
        SUM(value) as total_value_30d,
        COUNT(DISTINCT date) as active_days,
        AVG(trade_count) as avg_daily_trades,
        -- Price change
        (LAST_VALUE(vwap) OVER (PARTITION BY symbol ORDER BY date) /
         FIRST_VALUE(vwap) OVER (PARTITION BY symbol ORDER BY date) - 1) * 100 as price_change_pct
    FROM daily
    GROUP BY symbol, symbol_name
),

ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (ORDER BY total_value_30d DESC) as volume_rank,
        ROW_NUMBER() OVER (ORDER BY avg_daily_trades DESC) as activity_rank,
        ROW_NUMBER() OVER (ORDER BY price_change_pct DESC) as performance_rank
    FROM metrics
)

SELECT * FROM ranked
```

### Testing Configuration

```yaml
# models/marts/trading/trading.yml
version: 2

models:
  - name: fct_hourly_trading
    description: Hourly OHLCV data per symbol
    config:
      tags: ['hourly', 'trading', 'critical']
    columns:
      - name: hour
        tests:
          - not_null
      - name: symbol
        tests:
          - not_null
          - relationships:
              to: ref('dim_symbols')
              field: symbol
      - name: volume
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
      - name: vwap
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true

  - name: fct_daily_trading
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - date
            - symbol
```

### Result

Running `dbt build`:
```
Running with dbt=1.9.0
Found 12 models, 8 tests, 2 snapshots, 3 sources

Concurrency: 4 threads (target='prod')

1 of 12 START sql view model staging.stg_trades ........................ [RUN]
2 of 12 START sql view model staging.stg_symbols ....................... [RUN]
...
12 of 12 START sql incremental model marts.fct_hourly_trading ........... [RUN]
12 of 12 OK created sql incremental model marts.fct_hourly_trading ...... [OK in 45.2s]

Finished running 12 models, 8 tests in 0 hours 2 minutes 15 seconds.

Completed successfully
Done. PASS=20 WARN=0 ERROR=0 SKIP=0 TOTAL=20
```

## Summary

dbt transformed how we approach data transformations:

**Core concepts:**
- **Models:** SQL SELECT statements that define transformations
- **Materializations:** How models persist (view, table, incremental)
- **Sources:** External tables dbt reads from
- **Tests:** Data quality validation
- **Documentation:** Lives with code

**Project organization:**
- **Staging:** Clean raw data, 1:1 with sources
- **Intermediate:** Business logic building blocks
- **Marts:** Consumer-facing models by domain

**Modern features (dbt 1.9):**
- **Microbatch:** Process time-series in parallel batches
- **YAML snapshots:** Cleaner SCD configuration
- **Concurrent cloning:** Faster environment setup

**Integration patterns:**
- Orchestrate with Airflow, Dagster, or Prefect
- CI/CD with slim builds (state:modified+)
- Environment strategy (dev/ci/prod)

**Key insight:** dbt succeeds because it applies proven software engineering principles—version control, testing, documentation, modularity—to SQL transformations. The tool is powerful, but the discipline it brings is transformative.

**Looking ahead:**
- Chapter 13 covers data modeling patterns in depth
- Chapter 14 explores data quality and observability
- Chapter 11 showed how orchestrators integrate with dbt

## Further Reading

- [dbt Documentation](https://docs.getdbt.com/) — Official documentation
- [dbt Core 1.9 Release](https://www.getdbt.com/blog/dbt-core-v1-9-is-ga) — Microbatch and new features
- [About Microbatch Incremental Models](https://docs.getdbt.com/docs/build/incremental-microbatch) — Deep dive on microbatch
- Tristan Handy. "The Analytics Engineering Guide" — dbt Labs perspective
- [dbt-utils Package](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) — Essential utilities
- [dbt Expectations](https://hub.getdbt.com/calogica/dbt_expectations/latest/) — Great Expectations-style tests
- [Astronomer Cosmos](https://astronomer.github.io/astronomer-cosmos/) — dbt + Airflow integration
- [Dagster dbt Integration](https://docs.dagster.io/integrations/dbt) — Native Dagster support

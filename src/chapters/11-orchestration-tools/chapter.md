---
chapter: 11
title: "Orchestration Tools: Airflow, Prefect, and Dagster"
estimated_pages: 45-50
status: draft
last_updated: 2025-01-22
---

# Orchestration Tools: Airflow, Prefect, and Dagster

Chapter 10 established the principles of orchestration: DAGs, scheduling, idempotency, backfills, and failure handling. These concepts apply regardless of which tool you use. This chapter examines the three dominant orchestration tools—Airflow, Prefect, and Dagster—through that principled lens.

Each tool embodies different philosophies about how orchestration should work. Airflow, the incumbent, has evolved from a simple scheduler into a service-oriented platform. Prefect reimagined orchestration as a Python-native experience with hybrid cloud architecture. Dagster introduced asset-centric thinking, reshaping pipelines around data products rather than tasks.

Understanding these philosophical differences—not just syntax variations—helps you choose the right tool and use it effectively.

## The Evolution of Orchestration Tools

### First Generation: Cron and Scripts

Before orchestration tools, data pipelines were cron jobs calling shell scripts. Dependencies were implicit (run at 2 AM because upstream "should" be done by then). Failures required manual intervention. Backfills meant editing scripts and running them by hand.

### Second Generation: Airflow (2014-2015)

Airflow, created at Airbnb and open-sourced in 2015, introduced:
- **Explicit DAGs:** Dependencies as code, not timing assumptions
- **Web UI:** Visual monitoring, logs, manual triggers
- **Operators:** Reusable task templates (BashOperator, PythonOperator, etc.)
- **Backfills:** Native support for processing historical data
- **Connections:** Centralized credential management

Airflow became the de facto standard because it solved real problems at scale. But it carried legacy decisions: heavyweight architecture, configuration-as-code (DSL-like), and coupling between scheduler and execution.

### Third Generation: Prefect and Dagster (2018-2019)

Both Prefect and Dagster emerged from frustration with Airflow's limitations:

**Prefect (2018):** "Make orchestration feel like regular Python." Native Python functions as tasks, dynamic workflows, hybrid cloud architecture.

**Dagster (2019):** "Pipelines should be about data, not tasks." Asset-centric design, strong typing, integrated testing.

These tools represent different philosophies:
- Airflow: Task scheduling platform
- Prefect: Python-native workflow orchestration
- Dagster: Data asset orchestration

### Current State (2025)

All three tools have matured significantly:
- **Airflow 3.0:** Service-oriented architecture, Task SDK, asset-based scheduling
- **Prefect 3:** Improved cloud integration, work pools, artifacts
- **Dagster:** Software-defined assets, embedded ELT, observability focus

The tools are converging in capabilities while maintaining distinct philosophies.

## Apache Airflow

Airflow remains the most widely deployed orchestrator, with a massive ecosystem of providers and integrations. Airflow 3.0, released in 2025, represents the most significant architectural change since its creation.

### Core Concepts

**DAG (Directed Acyclic Graph):**
The container for your workflow. Defines tasks and their dependencies.

```python
from airflow.sdk import DAG, task
from datetime import datetime

with DAG(
    dag_id="daily_trading_etl",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
) as dag:
    # Tasks defined here
    pass
```

**Tasks:**
Units of work within a DAG. Can be defined using operators or decorators.

```python
from airflow.sdk import task

@task
def extract():
    # Extract data from source
    return {"records": 1000}

@task
def transform(data):
    # Transform data
    return {"records": data["records"], "transformed": True}

@task
def load(data):
    # Load to destination
    print(f"Loaded {data['records']} records")
```

**Operators:**
Pre-built task templates for common operations.

```python
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator

# Copy S3 object
copy_task = S3CopyObjectOperator(
    task_id="copy_raw_data",
    source_bucket_key="raw/trades/",
    dest_bucket_key="staging/trades/",
)

# Submit Spark job
spark_task = SparkSubmitOperator(
    task_id="run_aggregation",
    application="s3://code/aggregate_trades.py",
    conf={"spark.executor.memory": "4g"},
)
```

**XComs:**
Mechanism for passing data between tasks.

```python
@task
def extract():
    return {"count": 100}  # Automatically pushed to XCom

@task
def process(data):  # Automatically pulled from XCom
    print(f"Processing {data['count']} records")

# Data flows through function arguments
data = extract()
process(data)
```

### Airflow 3.0 Architecture

Airflow 3.0 introduces a service-oriented architecture that fundamentally changes how Airflow operates.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Airflow 3.0 Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │   Web UI     │   │  Scheduler   │   │  API Server  │        │
│  │   (React)    │   │              │   │   (FastAPI)  │        │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘        │
│         │                  │                  │                 │
│         └──────────────────┼──────────────────┘                 │
│                            │                                    │
│                            ▼                                    │
│                   ┌────────────────┐                            │
│                   │   Metadata DB  │                            │
│                   │   (Postgres)   │                            │
│                   └────────┬───────┘                            │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                 │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │  Executor   │   │  Executor   │   │    Edge     │           │
│  │  (Celery)   │   │    (K8s)    │   │  Executor   │           │
│  └─────────────┘   └─────────────┘   └─────────────┘           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Task Execution API                       │ │
│  │   - Stable contract for remote execution                    │ │
│  │   - Task SDK for external runtimes                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Key Airflow 3.0 Changes:**

**Task SDK:**
A lightweight runtime for executing Airflow tasks in external systems—containers, edge environments, serverless functions.

```python
# New SDK namespace
from airflow.sdk import DAG, task, Asset

# Clean, typed task definitions
@task
def compute_metrics(date: str) -> dict:
    """Process trading data for a given date."""
    df = spark.read.parquet(f"s3://data/trades/date={date}/")
    metrics = df.groupBy("symbol").agg(...)
    return {"records": metrics.count()}
```

**Asset-Based Scheduling:**
Beyond datasets, Airflow 3 introduces assets—explicit declarations of data artifacts.

```python
from airflow.sdk import Asset

# Define assets
trades_raw = Asset("s3://data/raw/trades/")
trades_processed = Asset("s3://data/processed/trades/")

@task(inlets=[trades_raw], outlets=[trades_processed])
def process_trades():
    """Transform raw trades to processed."""
    # Task declares its data dependencies
    ...

# DAG triggers when asset is updated
@dag(schedule=[trades_raw])
def downstream_analysis():
    """Runs when trades_raw is updated."""
    ...
```

**DAG Versioning:**
Track changes to DAGs natively, without external version control workarounds.

```python
# Airflow tracks DAG versions automatically
# Historical runs retain their DAG version
# UI shows version history and diffs
```

**Scheduler-Managed Backfills:**
Backfills are now managed by the scheduler, following the same execution model as regular runs.

```python
# CLI backfill (now integrated with scheduler)
airflow dags backfill daily_etl \
    --start-date 2024-01-01 \
    --end-date 2024-01-31

# UI-based backfill also available
```

**Improved UI:**
Modern React-based interface with Chakra UI components, better navigation, and enhanced visualizations.

### TaskFlow API Patterns

The TaskFlow API (introduced in Airflow 2, enhanced in Airflow 3) enables writing DAGs as native Python:

```python
from airflow.sdk import DAG, task
from datetime import datetime

@dag(
    dag_id="trading_analytics",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
    catchup=False,
)
def trading_analytics():

    @task
    def extract_trades(date: str) -> list[dict]:
        """Extract trades from source system."""
        trades = fetch_from_api(date)
        return trades

    @task
    def enrich_trades(trades: list[dict]) -> list[dict]:
        """Add reference data to trades."""
        enriched = []
        for trade in trades:
            trade["symbol_name"] = lookup_symbol(trade["symbol"])
            enriched.append(trade)
        return enriched

    @task
    def compute_metrics(trades: list[dict]) -> dict:
        """Compute daily trading metrics."""
        return {
            "total_volume": sum(t["quantity"] for t in trades),
            "trade_count": len(trades),
            "unique_symbols": len(set(t["symbol"] for t in trades)),
        }

    @task
    def store_metrics(metrics: dict, date: str):
        """Store metrics to warehouse."""
        insert_to_db("daily_metrics", {**metrics, "date": date})

    # Define flow
    date = "{{ ds }}"  # Jinja template for execution date
    raw_trades = extract_trades(date)
    enriched = enrich_trades(raw_trades)
    metrics = compute_metrics(enriched)
    store_metrics(metrics, date)

# Instantiate the DAG
dag = trading_analytics()
```

**Conditional execution (Airflow 3):**

```python
from airflow.sdk import task

@task.skip_if(lambda context: context["params"].get("skip_export"))
def export_data():
    """Only runs if skip_export param is not set."""
    ...

@task.run_if(lambda context: context["dag_run"].run_type == "backfill")
def validate_backfill():
    """Only runs during backfills."""
    ...
```

### Operators and Providers

Airflow's strength is its ecosystem. Providers package operators for external systems:

```python
# AWS Provider
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator

# GCP Provider
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import GCSToBigQueryOperator

# Spark Provider
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator

# dbt Provider
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
```

**Provider ecosystem:**
- 80+ official providers
- AWS, GCP, Azure fully supported
- Spark, Databricks, Snowflake, dbt
- Kafka, Elasticsearch, MongoDB
- HTTP, SMTP, Slack for notifications

### Executors

Executors determine how tasks run:

**LocalExecutor:** Tasks run as processes on the scheduler node.
- Good for development and small workloads
- No additional infrastructure required

**CeleryExecutor:** Tasks distributed via Celery message queue.
- Horizontal scaling with worker nodes
- Requires Redis or RabbitMQ

**KubernetesExecutor:** Each task runs in a Kubernetes pod.
- Full isolation between tasks
- Dynamic resource allocation
- Native cloud-native deployments

**Edge Executor (Airflow 3):** Tasks run on edge devices or remote locations.
- Distributed, event-driven workloads
- IoT and edge computing scenarios

```python
# Task-specific executor override
@task(executor_config={
    "KubernetesExecutor": {
        "request_memory": "2Gi",
        "request_cpu": "1",
        "image": "custom-image:latest",
    }
})
def heavy_computation():
    ...
```

### When to Choose Airflow

**Choose Airflow when:**
- Large, complex pipeline ecosystems requiring integrations
- Team already has Airflow expertise
- Need for extensive operator ecosystem
- Enterprise requirements (audit logging, RBAC, etc.)
- Managed service availability matters (MWAA, Cloud Composer, Astronomer)

**Airflow challenges:**
- Steeper learning curve than alternatives
- More infrastructure to manage
- DAGs must be in a specific location (not easily dynamic)
- XCom limitations for large data (use external storage)

## Prefect

Prefect takes a Python-first approach: if you can write Python functions, you can write workflows. The framework emphasizes minimal ceremony and hybrid cloud architecture.

### Core Philosophy

Prefect's guiding principles:
1. **Workflows should be native Python:** No DSL, no special syntax
2. **Orchestration should be optional:** Run flows locally or orchestrated
3. **Hybrid architecture:** Control plane in cloud, execution in your environment

### Core Concepts

**Flow:**
The top-level container for a workflow. Any Python function decorated with `@flow`.

```python
from prefect import flow, task

@flow(name="daily-trading-etl")
def trading_etl(date: str):
    """Main ETL workflow for trading data."""
    raw = extract_trades(date)
    processed = transform_trades(raw)
    load_trades(processed, date)
    return {"status": "success", "date": date}

# Run locally—no orchestration required
if __name__ == "__main__":
    trading_etl("2024-01-15")
```

**Task:**
A discrete unit of work within a flow. Provides caching, retries, and observability.

```python
from prefect import task
from datetime import timedelta

@task(retries=3, retry_delay_seconds=60)
def extract_trades(date: str) -> list[dict]:
    """Fetch trades from API with automatic retry."""
    response = requests.get(f"{API_URL}/trades?date={date}")
    response.raise_for_status()
    return response.json()

@task(cache_key_fn=lambda ctx, params: f"transform-{params['date']}")
def transform_trades(trades: list[dict]) -> list[dict]:
    """Transform trades—cached by date."""
    return [enrich(trade) for trade in trades]

@task(timeout_seconds=300)
def load_trades(trades: list[dict], date: str):
    """Load trades with timeout."""
    db.insert_batch("trades", trades)
```

**Subflows:**
Flows can call other flows, enabling composition.

```python
@flow
def extract_flow(date: str) -> dict:
    trades = extract_trades(date)
    prices = extract_prices(date)
    return {"trades": trades, "prices": prices}

@flow
def transform_flow(data: dict) -> dict:
    enriched_trades = enrich_with_prices(data["trades"], data["prices"])
    return {"trades": enriched_trades}

@flow
def daily_etl(date: str):
    raw_data = extract_flow(date)
    transformed = transform_flow(raw_data)
    load_flow(transformed, date)
```

### Deployment and Infrastructure

Prefect separates orchestration (scheduling, monitoring) from execution (running code).

**Work Pools:**
Define where work runs—different infrastructure for different needs.

```python
# Create work pool via CLI
prefect work-pool create "kubernetes-pool" --type kubernetes

# Or via Python
from prefect.infrastructure import KubernetesJob

kubernetes_infra = KubernetesJob(
    namespace="data-pipelines",
    image="my-registry/trading-etl:latest",
    job_watch_timeout_seconds=600,
)
```

**Workers:**
Poll work pools and execute flows in your infrastructure.

```bash
# Start a worker for a specific pool
prefect worker start --pool "kubernetes-pool"

# Worker runs in your environment—code never leaves your infrastructure
```

**Deployments:**
Package flows for scheduled or triggered execution.

```python
from prefect import flow
from prefect.deployments import Deployment
from prefect.server.schemas.schedules import CronSchedule

@flow
def trading_etl(date: str):
    ...

# Create deployment
deployment = Deployment.build_from_flow(
    flow=trading_etl,
    name="daily-trading-etl",
    work_pool_name="kubernetes-pool",
    schedule=CronSchedule(cron="0 6 * * *"),  # 6 AM daily
    parameters={"date": "{{ now | strftime('%Y-%m-%d') }}"},
)
deployment.apply()
```

### Prefect Cloud Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Prefect Cloud (Control Plane)                │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │   Scheduler   │  │      UI       │  │  Automations  │       │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘       │
│          │                  │                  │                │
│          └──────────────────┼──────────────────┘                │
│                             │ (outbound-only connection)        │
└─────────────────────────────┼───────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│   Your Infrastructure    │     │   Your Infrastructure    │
│                          │     │                          │
│  ┌────────────────────┐ │     │  ┌────────────────────┐ │
│  │  Prefect Worker    │ │     │  │  Prefect Worker    │ │
│  │  (polls work pool) │ │     │  │  (K8s executor)    │ │
│  └─────────┬──────────┘ │     │  └─────────┬──────────┘ │
│            │            │     │            │            │
│            ▼            │     │            ▼            │
│  ┌────────────────────┐ │     │  ┌────────────────────┐ │
│  │   Flow Execution   │ │     │  │   K8s Jobs         │ │
│  │   (your data stays)│ │     │  │   (your VPC)       │ │
│  └────────────────────┘ │     │  └────────────────────┘ │
└─────────────────────────┘     └─────────────────────────┘
```

**Key architectural benefit:** Your code and data never leave your infrastructure. Prefect Cloud only manages scheduling and metadata.

### Advanced Patterns

**Dynamic workflows:**
Generate tasks at runtime based on data.

```python
from prefect import flow, task

@task
def process_symbol(symbol: str, date: str) -> dict:
    trades = fetch_trades(symbol, date)
    return compute_metrics(trades)

@flow
def process_all_symbols(date: str):
    # Dynamically create tasks based on data
    symbols = get_active_symbols()  # Returns ["BTC", "ETH", "SOL", ...]

    # Map over symbols—creates tasks dynamically
    results = process_symbol.map(symbols, date=[date] * len(symbols))

    # Aggregate results
    aggregate_metrics(results)
```

**Caching:**
Avoid redundant computation.

```python
from prefect import task
from prefect.tasks import task_input_hash

@task(cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=24))
def expensive_computation(data: dict) -> dict:
    """Result cached for 24 hours based on input hash."""
    return heavy_processing(data)
```

**Artifacts:**
Store and visualize outputs.

```python
from prefect import task
from prefect.artifacts import create_markdown_artifact

@task
def generate_report(metrics: dict):
    markdown = f"""
    # Daily Trading Report

    | Metric | Value |
    |--------|-------|
    | Total Volume | {metrics['volume']:,} |
    | Trade Count | {metrics['count']:,} |
    | Avg Price | ${metrics['avg_price']:.2f} |
    """

    create_markdown_artifact(
        key="daily-report",
        markdown=markdown,
        description="Daily trading summary"
    )
```

### When to Choose Prefect

**Choose Prefect when:**
- Python-first team wanting minimal learning curve
- Need to run workflows locally during development
- Hybrid architecture is important (keep data in your environment)
- Building dynamic, data-driven workflows
- Small to mid-size team wanting quick adoption

**Prefect challenges:**
- Smaller ecosystem than Airflow
- Asset/data lineage is less native than Dagster
- Self-hosted Prefect server requires more setup than Prefect Cloud

## Dagster

Dagster reimagines orchestration around data assets rather than tasks. Instead of "run task A, then task B," you declare "I want this dataset to exist, and here's how to produce it."

### Core Philosophy

Dagster's key insight: pipelines exist to produce data assets. Tasks are implementation details; assets are what matters.

**Task-centric view (Airflow/Prefect):**
```
extract_task >> transform_task >> load_task
"First extract, then transform, then load"
```

**Asset-centric view (Dagster):**
```
raw_trades → processed_trades → daily_metrics
"I need daily_metrics, which depends on processed_trades, which depends on raw_trades"
```

The shift in perspective enables:
- Automatic dependency inference from data lineage
- Clear understanding of what data exists and its freshness
- Better integration with data-centric tools like dbt

### Core Concepts

**Assets:**
The primary abstraction—a data artifact and the code that produces it.

```python
from dagster import asset, AssetExecutionContext
import pandas as pd

@asset
def raw_trades(context: AssetExecutionContext) -> pd.DataFrame:
    """Extract raw trading data from source."""
    context.log.info("Extracting trades...")
    df = fetch_from_api()
    return df

@asset
def processed_trades(raw_trades: pd.DataFrame) -> pd.DataFrame:
    """Clean and enrich trading data."""
    # Dependency on raw_trades is explicit in function signature
    df = raw_trades.copy()
    df["symbol_name"] = df["symbol"].map(symbol_lookup)
    df["value"] = df["price"] * df["quantity"]
    return df

@asset
def daily_metrics(processed_trades: pd.DataFrame) -> pd.DataFrame:
    """Compute daily aggregated metrics."""
    return processed_trades.groupby("date").agg({
        "value": "sum",
        "quantity": "sum",
        "trade_id": "count"
    }).rename(columns={"trade_id": "trade_count"})
```

**Key insight:** Dependencies are inferred from function signatures. Dagster builds the DAG automatically.

**Ops and Jobs (legacy/advanced):**
For complex orchestration logic, Dagster also supports ops (tasks) and jobs (DAGs).

```python
from dagster import op, job

@op
def extract_op(context):
    return fetch_data()

@op
def transform_op(context, data):
    return process(data)

@job
def etl_job():
    transform_op(extract_op())
```

**Resources:**
External services and connections, injected as dependencies.

```python
from dagster import asset, ConfigurableResource
import boto3

class S3Resource(ConfigurableResource):
    bucket: str
    region: str = "us-east-1"

    def get_client(self):
        return boto3.client("s3", region_name=self.region)

    def read_parquet(self, key: str) -> pd.DataFrame:
        # Implementation
        ...

@asset
def trades_data(s3: S3Resource) -> pd.DataFrame:
    """Read trades from S3."""
    return s3.read_parquet("trades/latest/")
```

**Definitions:**
The entry point that collects assets, resources, and schedules.

```python
from dagster import Definitions, define_asset_job, ScheduleDefinition

defs = Definitions(
    assets=[raw_trades, processed_trades, daily_metrics],
    resources={
        "s3": S3Resource(bucket="trading-data"),
        "warehouse": SnowflakeResource(account="..."),
    },
    schedules=[
        ScheduleDefinition(
            job=define_asset_job("daily_refresh", selection="*"),
            cron_schedule="0 6 * * *",
        )
    ],
)
```

### Software-Defined Assets

The asset graph provides powerful capabilities:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Dagster Asset Graph                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     ┌──────────────┐         ┌──────────────┐                   │
│     │  raw_trades  │────────▶│  processed_  │                   │
│     │              │         │    trades    │                   │
│     │  [External]  │         │              │──────┐            │
│     └──────────────┘         └──────────────┘      │            │
│                                                     │            │
│     ┌──────────────┐                               │            │
│     │ raw_prices   │─────────────────────────┐     │            │
│     │              │                         │     │            │
│     │  [External]  │                         ▼     ▼            │
│     └──────────────┘                    ┌──────────────┐        │
│                                         │ daily_metrics│        │
│                                         │              │        │
│                                         │  [Internal]  │        │
│                                         └──────────────┘        │
│                                                                  │
│  Legend: [External] = data from outside systems                 │
│          [Internal] = data Dagster produces                     │
│          Arrows = data dependencies                             │
└─────────────────────────────────────────────────────────────────┘
```

**Materialization:**
Producing an asset is called "materializing" it.

```python
# Materialize specific assets
dagster asset materialize --select daily_metrics

# Materialize asset and all upstream dependencies
dagster asset materialize --select daily_metrics --upstream

# Materialize everything
dagster asset materialize --select "*"
```

**Freshness policies:**
Declare how fresh assets should be.

```python
from dagster import asset, FreshnessPolicy

@asset(freshness_policy=FreshnessPolicy(maximum_lag_minutes=60))
def daily_metrics(processed_trades: pd.DataFrame) -> pd.DataFrame:
    """Must be refreshed within 60 minutes of upstream changes."""
    ...
```

Dagster alerts when assets violate their freshness policies.

### Partitioned Assets

Dagster excels at partitioned data—common in time-series workloads.

```python
from dagster import asset, DailyPartitionsDefinition, AssetExecutionContext

daily_partitions = DailyPartitionsDefinition(start_date="2024-01-01")

@asset(partitions_def=daily_partitions)
def daily_trades(context: AssetExecutionContext) -> pd.DataFrame:
    """Partitioned by day."""
    partition_date = context.partition_key  # "2024-01-15"
    return fetch_trades(partition_date)

@asset(partitions_def=daily_partitions)
def daily_metrics(context: AssetExecutionContext, daily_trades: pd.DataFrame) -> pd.DataFrame:
    """Aggregates the same day's trades."""
    return daily_trades.groupby("symbol").agg(...)
```

**Backfilling partitions:**
```bash
# Backfill January 2024
dagster asset materialize \
    --select daily_metrics \
    --partition 2024-01-01...2024-01-31
```

The UI shows partition status visually:
```
daily_metrics partitions:
[✓][✓][✓][✓][✓][✓][✓][✗][✗][✗][✓][✓][✓][✓][✓]...
Jan-1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
                        └─ Missing partitions
```

### dbt Integration

Dagster's asset model aligns naturally with dbt's model concept.

```python
from dagster_dbt import DbtCliResource, dbt_assets, DbtProject

dbt_project = DbtProject(project_dir="path/to/dbt_project")

@dbt_assets(manifest=dbt_project.manifest_path)
def dbt_models(context: AssetExecutionContext, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()

# Now dbt models appear in Dagster's asset graph
# with their dependencies correctly mapped
```

Each dbt model becomes a Dagster asset with:
- Automatic lineage tracking
- Freshness monitoring
- Unified observability

### Testing in Dagster

Dagster emphasizes testability:

```python
from dagster import materialize

def test_daily_metrics():
    # Create test data
    test_trades = pd.DataFrame({
        "symbol": ["BTC", "BTC", "ETH"],
        "price": [50000, 51000, 3000],
        "quantity": [1, 2, 10],
    })

    # Materialize with test input
    result = materialize(
        [daily_metrics],
        input_values={"processed_trades": test_trades},
    )

    # Assert on output
    output_df = result.output_for_node("daily_metrics")
    assert len(output_df) == 2  # BTC and ETH
    assert output_df.loc["BTC", "quantity"] == 3
```

### When to Choose Dagster

**Choose Dagster when:**
- Data assets and lineage are central to your work
- Heavy dbt usage (alignment is excellent)
- Strong emphasis on data quality and observability
- Team values developer experience and testing
- Building a modern data platform from scratch

**Dagster challenges:**
- Smaller ecosystem than Airflow
- Asset-centric model requires mental shift
- Less suitable for non-data orchestration (general workflows)
- Younger community and fewer managed options

## Comparison Framework

Rather than a feature checklist, here's how to think about the trade-offs:

### Architectural Philosophy

| Aspect | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| Core abstraction | Tasks in DAGs | Python functions | Data assets |
| Dependency model | Explicit task dependencies | Function arguments | Asset lineage |
| Scheduling trigger | Time or external | Time, events, or ad-hoc | Time, freshness, or ad-hoc |
| Data passing | XCom (limited) or external storage | Native Python objects | Native Python objects |

### Operational Characteristics

| Aspect | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| Infrastructure | Heavy (scheduler, DB, workers) | Lightweight (workers only) | Medium (daemon, DB) |
| Managed offerings | Many (MWAA, Composer, Astronomer) | Prefect Cloud | Dagster Cloud |
| Local development | Requires setup | Works immediately | Works immediately |
| Scaling | Celery/K8s executors | Work pools | Runs, assets, partitions |

### Team and Ecosystem

| Aspect | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| Learning curve | Steeper (concepts, config) | Gentle (Python-native) | Medium (asset model) |
| Community size | Largest | Growing | Growing |
| Provider ecosystem | Extensive (80+) | Moderate | Moderate |
| Enterprise features | Strong (RBAC, audit) | Prefect Cloud | Dagster Cloud |

### Decision Framework

**Choose Airflow if:**
- You have complex, heterogeneous pipelines with many integrations
- Enterprise requirements (compliance, RBAC, audit trails)
- Team already knows Airflow or has dedicated platform engineers
- You want managed service options from multiple vendors
- You need the extensive operator ecosystem

**Choose Prefect if:**
- Your team values Python-native development
- You need hybrid architecture (cloud control, on-prem execution)
- Workflows are dynamic and data-driven
- You want quick adoption without heavy infrastructure
- Local development experience matters

**Choose Dagster if:**
- Your pipelines are fundamentally about producing data assets
- You use dbt heavily and want unified lineage
- Data freshness and quality are primary concerns
- You're building a modern data platform greenfield
- Developer experience and testing are priorities

### Migration Considerations

**From Airflow to Prefect:**
- Rewrite DAGs as flows (straightforward mapping)
- Replace operators with native Python or Prefect tasks
- Move connections to Prefect blocks
- Consider gradual migration (run both in parallel)

**From Airflow to Dagster:**
- Reconceptualize pipelines as asset graphs
- Replace operators with asset definitions
- Leverage dbt integration if applicable
- Asset model may reveal implicit dependencies

**Hybrid approaches:**
Some organizations run multiple orchestrators:
- Airflow for legacy pipelines
- Dagster/Prefect for new development
- Triggers between systems via events/APIs

## Case Study: Migrating from Airflow to Dagster

Let's trace a concrete migration scenario.

### Original Airflow DAG

```python
# Airflow DAG: daily trading pipeline
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from datetime import datetime

def extract_trades(**context):
    date = context["ds"]
    trades = fetch_from_api(date)
    # Write to S3 (passed via file, not XCom)
    write_to_s3(f"s3://raw/trades/{date}/data.parquet", trades)

def transform_trades(**context):
    date = context["ds"]
    # Read from S3
    trades = read_from_s3(f"s3://raw/trades/{date}/data.parquet")
    processed = enrich(trades)
    write_to_s3(f"s3://processed/trades/{date}/data.parquet", processed)

def compute_metrics(**context):
    date = context["ds"]
    trades = read_from_s3(f"s3://processed/trades/{date}/data.parquet")
    metrics = aggregate(trades)
    write_to_warehouse("daily_metrics", metrics)

with DAG(
    dag_id="trading_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",
) as dag:
    extract = PythonOperator(task_id="extract", python_callable=extract_trades)
    transform = PythonOperator(task_id="transform", python_callable=transform_trades)
    metrics = PythonOperator(task_id="metrics", python_callable=compute_metrics)

    extract >> transform >> metrics
```

### Migrated Dagster Assets

```python
# Dagster: same pipeline as assets
from dagster import asset, DailyPartitionsDefinition, AssetExecutionContext
import pandas as pd

daily_partitions = DailyPartitionsDefinition(start_date="2024-01-01")

@asset(partitions_def=daily_partitions)
def raw_trades(context: AssetExecutionContext) -> pd.DataFrame:
    """Extract raw trading data from API."""
    date = context.partition_key
    context.log.info(f"Extracting trades for {date}")
    return fetch_from_api(date)

@asset(partitions_def=daily_partitions)
def processed_trades(context: AssetExecutionContext, raw_trades: pd.DataFrame) -> pd.DataFrame:
    """Clean and enrich trading data."""
    context.log.info(f"Processing {len(raw_trades)} trades")
    df = raw_trades.copy()
    df["symbol_name"] = df["symbol"].map(symbol_lookup)
    df["value"] = df["price"] * df["quantity"]
    return df

@asset(partitions_def=daily_partitions)
def daily_metrics(context: AssetExecutionContext, processed_trades: pd.DataFrame) -> pd.DataFrame:
    """Compute daily aggregated metrics."""
    metrics = processed_trades.groupby("symbol").agg({
        "value": "sum",
        "quantity": "sum",
        "trade_id": "count"
    })
    context.log.info(f"Computed metrics for {len(metrics)} symbols")
    return metrics
```

### What Changed

**Explicit → Implicit dependencies:**
Airflow required `extract >> transform >> metrics`. Dagster infers this from function signatures.

**File-based → In-memory:**
Airflow tasks communicated via S3 files. Dagster assets pass DataFrames directly (or use I/O managers for persistence).

**Task-centric → Asset-centric:**
The mental model shifted from "what tasks run" to "what data exists."

**Scheduling → Freshness:**
Instead of "run at 6 AM," Dagster can enforce "daily_metrics must be < 24 hours old."

### Benefits Observed

1. **Clearer lineage:** UI shows asset dependencies visually
2. **Easier testing:** Assets can be tested in isolation with mock inputs
3. **Partition visibility:** See which dates are missing at a glance
4. **dbt integration:** Existing dbt models joined the asset graph seamlessly

## Summary

This chapter examined three orchestration tools through their philosophical differences:

**Airflow 3.0:**
- Task scheduling platform with extensive ecosystem
- New service-oriented architecture (Task SDK, API server)
- Asset-based scheduling for event-driven workflows
- Best for complex, integration-heavy environments

**Prefect:**
- Python-native orchestration with minimal ceremony
- Hybrid architecture (cloud control, local execution)
- Dynamic workflows and flexible deployment
- Best for Python teams wanting quick adoption

**Dagster:**
- Asset-centric orchestration
- Strong data lineage and freshness tracking
- Excellent dbt integration
- Best for data platform teams prioritizing observability

**The underlying principles remain constant:**
- DAGs model dependencies (Chapter 10)
- Idempotency enables reliability
- Backfills reprocess history
- Monitoring provides observability

**Choose based on:**
- Philosophy alignment (tasks vs. assets)
- Team expertise and learning capacity
- Ecosystem requirements
- Operational constraints

**Key insight:** The tool matters less than the principles. A well-designed pipeline in any of these tools—with clear dependencies, idempotent operations, and proper error handling—will serve you well. A poorly designed pipeline will fail regardless of the tool.

**Looking ahead:**
- Chapter 12 covers dbt, which has its own DAG-based orchestration
- Chapter 13 explores data modeling patterns
- Chapter 15 shows orchestration in complete architectures

## Further Reading

- [Apache Airflow 3.0 Release Notes](https://airflow.apache.org/docs/apache-airflow/3.0.0/release_notes.html) — Official documentation
- [Airflow 3.0 Blog Announcement](https://airflow.apache.org/blog/airflow-three-point-oh-is-here/) — Overview of major changes
- [Prefect Documentation](https://docs.prefect.io/) — Official Prefect 3 docs
- [Dagster Documentation](https://docs.dagster.io/) — Official Dagster docs
- [Dagster vs Prefect Comparison](https://dagster.io/vs/dagster-vs-prefect) — Dagster's comparison
- [Orchestrating dbt with Airflow, Dagster & Prefect](https://medium.com/tech-with-abhishek/orchestrating-dbt-with-airflow-dagster-prefect-advanced-patterns-and-best-practices-2025) — Integration patterns
- Reis, J. & Housley, M. (2022). *Fundamentals of Data Engineering*, Chapter 8 — Orchestration overview

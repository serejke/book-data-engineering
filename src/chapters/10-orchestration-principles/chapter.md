---
chapter: 10
title: "Pipeline Orchestration Principles"
estimated_pages: 25-30
status: draft
last_updated: 2025-01-22
---

# Pipeline Orchestration Principles

Data processing doesn't happen in isolation. Your Spark job that computes daily metrics depends on upstream data landing. Your ML model training waits for feature engineering. Your dashboard refresh triggers after aggregations complete. These dependencies create a directed graph of tasks that must execute in the right order, at the right time, with proper handling of failures.

This is orchestration: the coordination of computational tasks across time and systems. Before diving into specific tools (Chapter 11), this chapter establishes the mental model for thinking about orchestration. The concepts here—DAGs, idempotency, backfills, dependencies—apply regardless of whether you use Airflow, Prefect, Dagster, or any future tool.

## Why Orchestration Matters

The simplest data pipeline might be a cron job running a Python script:

```bash
0 6 * * * /usr/bin/python /scripts/daily_etl.py
```

This works until it doesn't:

**What happens when the job fails?** Cron doesn't know it failed. No one gets notified. Tomorrow's job runs on incomplete data.

**What if yesterday's job took 8 hours and today's job takes 10?** Jobs overlap, competing for resources, potentially corrupting data.

**What if your job depends on an upstream system?** The source data might not be ready at 6 AM. Your job runs on partial data.

**How do you reprocess historical data?** Manually running scripts for 365 days is error-prone and tedious.

**How do you understand what ran?** No centralized view of successes, failures, durations.

Orchestration tools solve these problems by providing:
- **Dependency management:** Jobs run when their dependencies are satisfied
- **Scheduling:** Time-based and event-based triggers
- **Retries and alerting:** Automatic recovery and notification
- **Backfills:** Reprocess historical time ranges
- **Observability:** Centralized logging, metrics, lineage

## Directed Acyclic Graphs (DAGs)

The fundamental abstraction in orchestration is the Directed Acyclic Graph (DAG): a collection of tasks with directed dependencies and no cycles.

### DAG Structure

```
       ┌─────────────┐
       │  Extract    │
       │  (Task A)   │
       └──────┬──────┘
              │
              ▼
       ┌─────────────┐
       │  Transform  │
       │  (Task B)   │
       └──────┬──────┘
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
┌─────────────┐ ┌─────────────┐
│  Aggregate  │ │   Export    │
│  (Task C)   │ │  (Task D)   │
└──────┬──────┘ └──────┬──────┘
       │               │
       └───────┬───────┘
               │
               ▼
        ┌─────────────┐
        │   Notify    │
        │  (Task E)   │
        └─────────────┘
```

**Directed:** Edges have direction (A → B means A must complete before B starts).

**Acyclic:** No cycles allowed (you can't have A → B → C → A).

**Why no cycles?** Cycles create infinite loops. If A depends on C and C depends on A, neither can ever start. The acyclic constraint ensures the graph can always be resolved.

### Task Dependencies

Dependencies come in several forms:

**Sequential:** One task after another
```python
extract >> transform >> load
```

**Fan-out:** One task triggers multiple downstream tasks
```python
transform >> [aggregate, export, archive]
```

**Fan-in:** Multiple tasks must complete before one starts
```python
[source_a, source_b, source_c] >> merge
```

**Complex:** Arbitrary combinations
```python
# Task A and B run in parallel
# Task C waits for A
# Task D waits for A and B
# Task E waits for C and D
```

### DAG vs Workflow vs Pipeline

These terms are often used interchangeably, but distinctions exist:

**DAG:** The graph structure itself—tasks and their dependencies.

**Pipeline:** A broader concept encompassing data flow, often including the data movement between tasks.

**Workflow:** The runtime execution of a DAG, including scheduling, retries, and state management.

In practice, most tools conflate these. An Airflow "DAG" includes scheduling configuration. A Prefect "flow" includes task definitions and dependencies. Don't get hung up on terminology—focus on the concepts.

## Scheduling Paradigms

When does a DAG run? Two fundamental approaches exist.

### Time-Based Scheduling

Run at fixed intervals:

```python
# Run daily at midnight
schedule = "0 0 * * *"

# Run hourly
schedule = "0 * * * *"

# Run every Monday at 6 AM
schedule = "0 6 * * 1"
```

Time-based scheduling assumes:
- Data arrives on a predictable schedule
- Processing time is bounded
- Missing runs should create backlog

**Cron expressions** (the `0 0 * * *` syntax) are the standard way to express schedules. The five fields represent: minute, hour, day of month, month, day of week.

### Event-Based Scheduling

Run when something happens:

```python
# Run when new files appear in S3
trigger = S3KeySensor(bucket="raw-data", prefix="trades/")

# Run when upstream DAG completes
trigger = ExternalTaskSensor(external_dag_id="upstream_etl")

# Run when message arrives in Kafka
trigger = KafkaTrigger(topic="data-ready")
```

Event-based scheduling assumes:
- Data arrival is unpredictable
- Processing should happen as soon as possible
- Missing triggers mean no work to do

### Hybrid Approaches

Real systems often combine both:

```python
# Run daily, but wait for data to be ready
schedule = "0 6 * * *"  # Trigger at 6 AM

@task
def wait_for_data():
    sensor = S3KeySensor(
        bucket="raw-data",
        prefix=f"trades/{date}/",
        timeout=3600  # Wait up to 1 hour
    )
    return sensor.poke()

@task
def process_data():
    # Actual processing
    ...

wait_for_data >> process_data
```

This DAG runs at 6 AM daily but waits (up to an hour) for upstream data before processing.

## Logical Time vs Physical Time

A critical concept in orchestration: **logical time** (the data being processed) vs **physical time** (when processing occurs).

### The Execution Date Concept

When a daily DAG runs at 2024-01-16 00:00:00, what data should it process?

**Option 1:** Data from 2024-01-16 (the current day)
**Problem:** The day isn't over yet—data is incomplete.

**Option 2:** Data from 2024-01-15 (the previous day)
**This is the convention.** The "execution date" or "logical date" is the start of the interval being processed, not when processing occurs.

```
┌─────────────────────────────────────────────────────────┐
│  Interval being processed: 2024-01-15 00:00 to 23:59   │
│  execution_date = 2024-01-15                            │
│  Physical run time = 2024-01-16 00:00                   │
└─────────────────────────────────────────────────────────┘
```

This convention ensures:
- The interval is complete when processing starts
- Backfills process historical intervals with correct dates
- Date-partitioned outputs use the logical date, not the run date

### Working with Dates in Code

```python
# Airflow example
def process_daily_trades(execution_date, **context):
    # execution_date is the START of the interval
    date_str = execution_date.strftime("%Y-%m-%d")

    # Read yesterday's data
    input_path = f"s3://raw/trades/date={date_str}/"

    # Write to yesterday's partition
    output_path = f"s3://processed/trades/date={date_str}/"

    process(input_path, output_path)

# When run at 2024-01-16 00:00:00
# execution_date = 2024-01-15 00:00:00
# Processes data from 2024-01-15
```

Different tools have different naming conventions:
- **Airflow:** `execution_date` (legacy), `data_interval_start`, `data_interval_end`
- **Prefect:** `scheduled_start_time`
- **Dagster:** `partition_key`

Regardless of naming, the concept is the same: separate the logical period from the physical execution time.

## Idempotency: The Foundation of Reliability

An idempotent operation produces the same result whether executed once or multiple times. In orchestration, idempotency is essential for:
- **Retry safety:** Failed tasks can be retried without side effects
- **Backfill correctness:** Rerunning historical dates produces consistent results
- **Operational simplicity:** "Just run it again" is a valid recovery strategy

### Achieving Idempotency

**Pattern 1: Overwrite outputs**
```python
def process_daily(date):
    data = read_source(date)
    result = transform(data)
    # Overwrite, don't append
    result.write.mode("overwrite").parquet(f"output/date={date}/")
```

Running this twice for the same date produces identical output.

**Pattern 2: Delete-then-insert**
```python
def process_daily(date):
    # Delete existing records for this date
    db.execute(f"DELETE FROM metrics WHERE date = '{date}'")

    # Insert fresh results
    data = compute_metrics(date)
    db.insert("metrics", data)
```

**Pattern 3: Use upsert/merge**
```python
def process_daily(date):
    data = compute_metrics(date)

    # MERGE handles both insert and update
    spark.sql(f"""
        MERGE INTO metrics AS target
        USING updates AS source
        ON target.id = source.id AND target.date = '{date}'
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)
```

### Non-Idempotent Anti-Patterns

**Append without deduplication:**
```python
# BAD: Running twice doubles the data
def process_daily(date):
    data = read_source(date)
    result = transform(data)
    result.write.mode("append").parquet("output/")  # No date partition!
```

**Increment counters:**
```python
# BAD: Running twice doubles the increment
def process_daily(date):
    count = get_daily_count(date)
    db.execute(f"UPDATE totals SET count = count + {count}")
```

**Generate timestamps in output:**
```python
# BAD: Different timestamp each run
def process_daily(date):
    data = read_source(date)
    data["processed_at"] = datetime.now()  # Different each time!
    write_output(data)
```

**Fix: Use logical time**
```python
def process_daily(date):
    data = read_source(date)
    data["processed_at"] = date  # Deterministic!
    write_output(data, date)
```

## Backfills and Catchup

Backfills—reprocessing historical data—are a fundamental orchestration operation. Common scenarios:

- **Bug fixes:** Logic error discovered, need to recompute last 90 days
- **New pipelines:** Deploy new DAG that needs to process historical data
- **Schema changes:** Add new columns, reprocess to populate them
- **Disaster recovery:** Restore from backup, reprocess since backup point

### Backfill Mechanics

Most orchestrators support backfills natively:

```bash
# Airflow: backfill a date range
airflow dags backfill \
    --start-date 2024-01-01 \
    --end-date 2024-01-31 \
    daily_etl

# Prefect: create runs for each date
for date in date_range("2024-01-01", "2024-01-31"):
    create_flow_run(flow_name="daily_etl", parameters={"date": date})

# Dagster: materialize a partition range
dagster job execute \
    --job daily_etl \
    --partition-range 2024-01-01...2024-01-31
```

### Backfill Strategies

**Sequential:** Process one date at a time, in order.
- Pro: Lower resource usage
- Pro: Easier debugging
- Con: Slow for large ranges

**Parallel:** Process multiple dates concurrently.
- Pro: Much faster
- Con: Higher resource usage
- Con: May overwhelm upstream systems

**Priority-based:** Process recent dates first, then historical.
- Pro: Restores recent data quickly
- Con: Historical data remains inconsistent longer

### Catchup Configuration

When deploying a new DAG with a start date in the past, should it automatically run for all missed intervals?

```python
# Airflow
DAG(
    dag_id="daily_etl",
    start_date=datetime(2024, 1, 1),
    catchup=True,  # Run for all dates since start_date
    # catchup=False,  # Only run for current/future dates
)
```

**catchup=True:**
- Useful for new pipelines that need historical data
- Can overwhelm systems if start_date is far in the past
- Respects concurrency limits (usually)

**catchup=False:**
- Safer for operational DAGs
- Manual backfill required for historical data
- Prevents accidental resource exhaustion

## Failure Handling

Failures are inevitable. Good orchestration handles them gracefully.

### Retry Strategies

```python
# Simple retry
@task(retries=3, retry_delay=timedelta(minutes=5))
def fetch_data():
    response = requests.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()

# Exponential backoff
@task(
    retries=5,
    retry_delay=timedelta(seconds=30),
    retry_exponential_backoff=True,  # 30s, 60s, 120s, 240s, 480s
    max_retry_delay=timedelta(minutes=15)
)
def fetch_data():
    ...
```

### Failure Modes

**Transient failures:** Temporary issues that resolve themselves.
- Network timeouts
- API rate limiting
- Resource contention

**Strategy:** Retry with backoff.

**Permanent failures:** Issues requiring human intervention.
- Invalid credentials
- Schema changes
- Data corruption

**Strategy:** Alert and stop. Don't waste resources retrying indefinitely.

**Partial failures:** Some partitions succeed, others fail.
- One hour of 24 fails
- Some files process, others error

**Strategy:** Task-level granularity allows retrying only failed portions.

### Circuit Breaker Pattern

When upstream systems are degraded, stop hammering them:

```python
failures = 0
MAX_FAILURES = 5
COOLDOWN = 300  # seconds

def call_api_with_circuit_breaker():
    global failures

    if failures >= MAX_FAILURES:
        if time_since_last_failure() < COOLDOWN:
            raise CircuitBreakerOpen("Too many failures, waiting")
        failures = 0  # Reset after cooldown

    try:
        result = call_api()
        failures = 0
        return result
    except Exception as e:
        failures += 1
        raise
```

### Alerting

Configure alerts for:
- **Task failures:** After retries exhausted
- **SLA misses:** Daily job not complete by 8 AM
- **Long-running tasks:** Task exceeds expected duration
- **Resource issues:** Out of memory, disk full

```python
# Airflow SLA example
DAG(
    dag_id="daily_etl",
    sla_miss_callback=alert_on_sla_miss,
    default_args={
        "sla": timedelta(hours=2),  # Task should complete within 2 hours
        "on_failure_callback": alert_on_failure,
    }
)
```

## Data Dependencies vs Task Dependencies

A subtle but important distinction:

**Task dependency:** "Task B should run after Task A completes."
**Data dependency:** "Task B needs data that Task A produces."

These are often conflated but can diverge:

```python
# Task dependency only—no data dependency
@task
def train_model():
    # Uses data from warehouse, not from upstream task
    df = spark.read.table("features")
    model = train(df)
    return model

# Data dependency—task passes data
@task
def extract():
    return fetch_data()

@task
def transform(data):  # Receives data from extract
    return process(data)

extract_result = extract()
transform(extract_result)  # Explicit data flow
```

### Data-Centric vs Task-Centric Orchestration

**Task-centric (Airflow model):**
- Define tasks and their dependencies
- Tasks may or may not pass data between them
- Focus on execution order and scheduling

**Data-centric (Dagster model):**
- Define assets (data artifacts) and how to produce them
- Dependencies are implicit from data lineage
- Focus on data freshness and materialization

```python
# Task-centric (Airflow style)
@task
def compute_metrics():
    df = spark.read.parquet("s3://data/trades/")
    metrics = df.groupBy("date").agg(...)
    metrics.write.parquet("s3://data/metrics/")

# Data-centric (Dagster style)
@asset(deps=["trades"])
def metrics():
    df = spark.read.parquet("s3://data/trades/")
    return df.groupBy("date").agg(...)
```

The data-centric approach makes dependencies explicit and enables:
- Automatic lineage tracking
- Incremental materialization
- Freshness monitoring

Chapter 11 explores these paradigms in detail with specific tools.

## Cross-DAG Dependencies

Real systems have multiple DAGs that depend on each other.

### Approaches

**1. External task sensors:**
Wait for a specific task in another DAG to complete.

```python
# DAG B waits for DAG A's final task
wait_for_upstream = ExternalTaskSensor(
    task_id="wait_for_dag_a",
    external_dag_id="dag_a",
    external_task_id="final_task",
    execution_date_fn=lambda dt: dt,  # Same execution date
)
```

**2. Dataset/asset triggers:**
DAG B triggers when DAG A produces a dataset.

```python
# DAG A produces a dataset
@dag(schedule="@daily")
def dag_a():
    @task(outlets=[Dataset("s3://data/metrics/")])
    def compute_metrics():
        ...

# DAG B triggers on that dataset
@dag(schedule=[Dataset("s3://data/metrics/")])
def dag_b():
    ...
```

**3. Event-based triggering:**
Emit events on completion, subscribe to trigger.

```python
# DAG A emits completion event
def on_success_callback(context):
    kafka_producer.send("dag-completions", {"dag": "dag_a", "date": context["ds"]})

# DAG B listens for events
def dag_b_trigger():
    message = kafka_consumer.poll("dag-completions")
    if message["dag"] == "dag_a":
        trigger_dag_run("dag_b", {"date": message["date"]})
```

### Avoiding Dependency Hell

Complex cross-DAG dependencies create fragile systems. Best practices:

**Keep DAGs independent when possible:**
Use shared storage (data lakes) rather than direct task dependencies.

**Use clear contracts:**
Define data formats, locations, and freshness expectations.

**Implement circuit breakers:**
Don't let one failing DAG cascade to block everything downstream.

**Document dependencies:**
Make cross-DAG relationships visible in documentation or lineage tools.

## Parameterization and Configuration

Good orchestration separates logic from configuration.

### Runtime Parameters

```python
# Accept date as parameter
@dag(
    params={
        "date": Param(default=None, type="string"),
        "env": Param(default="prod", enum=["dev", "staging", "prod"])
    }
)
def daily_etl(date, env):
    source_path = f"s3://{env}-data/trades/date={date}/"
    ...
```

Parameters enable:
- **Manual runs** with specific dates
- **Testing** in non-production environments
- **Overriding** defaults for edge cases

### Environment-Specific Configuration

```python
# Configuration by environment
config = {
    "dev": {
        "source": "s3://dev-data/",
        "parallelism": 4,
        "sample": 0.01,
    },
    "prod": {
        "source": "s3://prod-data/",
        "parallelism": 100,
        "sample": 1.0,
    }
}

@dag
def daily_etl(env="prod"):
    cfg = config[env]
    ...
```

### Connection Management

Don't hardcode credentials:

```python
# BAD
def fetch_data():
    conn = psycopg2.connect(
        host="prod-db.example.com",
        password="supersecret123"  # Never do this
    )

# GOOD: Use orchestrator's connection management
def fetch_data():
    conn = BaseHook.get_connection("postgres_prod")
    # Credentials stored securely in Airflow, not in code
```

## Testing Orchestration

Testing pipelines requires testing at multiple levels.

### Unit Testing Tasks

Test individual task logic in isolation:

```python
def test_transform_function():
    input_data = pd.DataFrame({"a": [1, 2, 3]})
    result = transform(input_data)
    assert result["b"].tolist() == [2, 4, 6]
```

### Integration Testing DAGs

Test DAG structure and dependencies:

```python
def test_dag_structure():
    dag = DAG(dag_id="test_dag", ...)

    assert len(dag.tasks) == 5
    assert "extract" in dag.task_ids
    assert dag.get_task("transform").upstream_task_ids == {"extract"}
```

### End-to-End Testing

Test complete pipeline execution:

```python
def test_daily_etl_e2e():
    # Set up test data
    create_test_data("s3://test-data/trades/date=2024-01-01/")

    # Run DAG
    execute_dag("daily_etl", params={"date": "2024-01-01", "env": "test"})

    # Verify output
    result = spark.read.parquet("s3://test-output/metrics/date=2024-01-01/")
    assert result.count() > 0
    assert result.filter(col("symbol") == "BTC").count() == 100
```

### Testing Best Practices

- **Isolate test environments:** Separate buckets, databases, schemas
- **Use small datasets:** Representative samples, not production scale
- **Test idempotency:** Run twice, verify same result
- **Test failure recovery:** Kill mid-run, verify recovery works
- **Test backfills:** Run for date range, verify all dates processed

## Monitoring and Observability

Operational orchestration requires visibility.

### Key Metrics

**Pipeline health:**
- Success rate (successes / total runs)
- Failure rate by failure type
- Mean time to recovery (MTTR)

**Performance:**
- End-to-end duration
- Task duration percentiles (p50, p95, p99)
- Queue wait time (time between scheduled and started)

**Resource utilization:**
- Concurrent running tasks
- Worker CPU/memory usage
- Queue depth

### Dashboards

Essential views:

**1. Overview dashboard:**
- All DAGs with recent status
- Failed runs requiring attention
- Upcoming scheduled runs

**2. DAG detail view:**
- Run history with duration trends
- Task-level status and timing
- Error messages and logs

**3. Resource dashboard:**
- Worker health and utilization
- Queue backlog
- Cost metrics (for cloud resources)

### Logging

Structure logs for searchability:

```python
import structlog

logger = structlog.get_logger()

def process_trades(date, symbol):
    logger.info(
        "processing_trades",
        date=date,
        symbol=symbol,
        record_count=count
    )
```

Structured logs enable:
- Filtering by date, symbol, DAG
- Aggregating metrics from logs
- Correlating across systems

## Summary

This chapter established orchestration fundamentals:

**DAGs model dependencies:**
- Tasks with directed edges
- Acyclic ensures resolution
- Fan-out, fan-in, complex patterns

**Scheduling approaches:**
- Time-based: cron expressions, fixed intervals
- Event-based: sensors, triggers
- Hybrid: scheduled with data readiness checks

**Logical time vs physical time:**
- Execution date is the interval being processed
- Physical run time is when processing occurs
- Critical for backfills and partitioning

**Idempotency enables reliability:**
- Same input → same output, always
- Achieved through overwrite, delete-insert, or merge
- Prerequisite for safe retries and backfills

**Backfills reprocess history:**
- Bug fixes, new pipelines, disaster recovery
- Sequential vs parallel strategies
- Catchup configuration controls new DAG behavior

**Failure handling:**
- Retries with exponential backoff
- Alerting for human intervention
- Circuit breakers protect upstream systems

**Dependencies span DAGs:**
- External task sensors
- Dataset/asset triggers
- Event-based coordination

**Testing matters:**
- Unit test task logic
- Integration test DAG structure
- End-to-end test complete pipelines

**Key insight:** Orchestration is about making distributed systems behave predictably. The tools (Airflow, Prefect, Dagster) are implementations of these concepts. Understanding the concepts lets you use any tool effectively and evaluate new ones.

**Looking ahead:**
- Chapter 11 dives into specific tools: Airflow 3, Prefect, Dagster
- Chapter 12 covers dbt, which has its own DAG-based orchestration
- Chapter 15 shows orchestration in complete architectures

## Further Reading

- Sridharan, C. (2018). *Distributed Systems Observability* — Monitoring principles
- Humble, J. & Farley, D. (2010). *Continuous Delivery*, Chapter 5 — Deployment pipelines
- [Airflow Concepts](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html) — Official Airflow documentation
- [Prefect Concepts](https://docs.prefect.io/latest/concepts/) — Official Prefect documentation
- [Dagster Concepts](https://docs.dagster.io/concepts) — Official Dagster documentation
- Burns, B. (2018). *Designing Distributed Systems* — Patterns for reliable systems

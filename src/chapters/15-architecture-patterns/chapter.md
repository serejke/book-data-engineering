---
chapter: 15
title: "Architecture Patterns"
estimated_pages: 45-50
status: draft
last_updated: 2025-01-22
---

# Architecture Patterns

The previous chapters examined individual components: storage formats, compute engines, orchestrators, transformation tools, quality frameworks. This chapter shows how these pieces fit together into coherent architectures.

Architecture is about trade-offs. There's no universally "best" architecture—only architectures suited to particular constraints, scale, and organizational contexts. The patterns here aren't prescriptions but templates that illuminate trade-offs and help you reason about your own systems.

We'll examine five architectures in depth: the Modern Data Stack, the Lakehouse, streaming-first architectures, the Data Mesh, and hybrid patterns. For each, we'll trace the reasoning behind design choices, identify where the pattern excels, and highlight where it struggles.

## What Makes a Good Architecture

Before examining specific patterns, let's establish evaluation criteria.

### Architecture Qualities

**Fitness for purpose:** Does the architecture serve its primary use cases well? A real-time fraud detection system has different needs than a monthly regulatory report.

**Simplicity:** How many concepts must someone understand to work with the system? Simpler architectures are easier to operate, debug, and evolve.

**Scalability:** Can the architecture grow with data volume, query complexity, and user count? Scalability has dimensions: data scale, query concurrency, team size.

**Flexibility:** How easily can the architecture adapt to new requirements? Can you add new sources, new use cases, new consumers?

**Cost efficiency:** What's the total cost of ownership—infrastructure, operations, development? Architectures that seem cheaper often have hidden costs in complexity.

**Operational burden:** How much effort does the architecture require to keep running? Some architectures are "set and forget"; others need constant attention.

### The Architecture Trade-off Triangle

Most architectural decisions involve trade-offs between three properties:

```
                    Simplicity
                       /\
                      /  \
                     /    \
                    /      \
                   /        \
                  /          \
                 /            \
                /──────────────\
         Flexibility          Performance
```

**Simplicity vs. Flexibility:** Simple systems constrain options. Flexible systems require more moving parts.

**Flexibility vs. Performance:** General-purpose systems sacrifice optimization. Specialized systems limit use cases.

**Performance vs. Simplicity:** High performance requires tuning, caching, denormalization—all complexity.

Understanding which qualities matter most for your context guides architectural choices.

## Pattern 1: The Modern Data Stack

The Modern Data Stack emerged around 2015-2020 as cloud data warehouses matured and SaaS tooling proliferated.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Modern Data Stack                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Sources   │  │   Sources   │  │   Sources   │             │
│  │   (SaaS)    │  │ (Databases) │  │   (APIs)    │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │    ELT Ingestion      │                          │
│              │   (Fivetran, Airbyte) │                          │
│              └───────────┬───────────┘                          │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │   Cloud Warehouse     │                          │
│              │ (Snowflake, BigQuery, │                          │
│              │      Redshift)        │                          │
│              │                       │                          │
│              │  ┌─────────────────┐  │                          │
│              │  │       dbt       │  │                          │
│              │  │ (Transformations)│  │                          │
│              │  └─────────────────┘  │                          │
│              └───────────┬───────────┘                          │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │      BI Layer         │                          │
│              │ (Looker, Tableau,     │                          │
│              │   Metabase, Sigma)    │                          │
│              └───────────────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

**Cloud-native:** All components are managed services. No infrastructure to manage.

**ELT over ETL:** Raw data lands in the warehouse first, then transforms. The warehouse's compute power handles transformation.

**SQL-centric:** dbt and warehouse SQL handle all transformations. No Spark, no custom code for most use cases.

**Best-of-breed:** Each layer uses specialized tools rather than monolithic platforms.

### Component Choices

**Ingestion:** Fivetran, Airbyte, Stitch, Hevo
- Pre-built connectors to hundreds of sources
- Managed infrastructure
- Incremental sync support

**Warehouse:** Snowflake, BigQuery, Redshift, Databricks SQL
- Separation of storage and compute
- Elastic scaling
- SQL interface

**Transformation:** dbt
- SQL-based transformations
- Version control, testing, documentation
- Modular, maintainable pipelines

**BI:** Looker, Tableau, Metabase, Sigma, Lightdash
- Self-service analytics
- Semantic layer
- Visualization

**Orchestration:** dbt Cloud, Airflow, Dagster, Prefect
- Schedule dbt runs
- Monitor freshness
- Handle dependencies

### When It Works Well

**B2B SaaS analytics:** Pull from Salesforce, Stripe, Marketo; build marketing attribution, sales forecasting, customer health.

**Small to mid-size teams:** 1-10 data people can build and maintain a complete platform.

**Standard BI use cases:** Dashboards, reports, ad-hoc queries, self-service analytics.

**Rapid iteration:** Get from source to dashboard in days, not months.

### When It Struggles

**Real-time requirements:** The Modern Data Stack is inherently batch-oriented. Near-real-time is possible (hourly syncs), but true streaming doesn't fit.

**Large-scale ML:** Cloud warehouses handle SQL well but aren't optimized for iterative ML training on massive datasets.

**Complex data:** Deeply nested JSON, images, video, unstructured data—these don't fit the relational model cleanly.

**Cost at scale:** Per-query pricing can become expensive with heavy usage. Pre-aggregation helps but adds complexity.

### Example Implementation

**Trading analytics scenario:**

```yaml
# Ingestion: Airbyte syncs from trading APIs
sources:
  - name: exchange_api
    connector: http
    schedule: "*/15 * * * *"  # Every 15 minutes

# Warehouse: Snowflake
warehouse:
  name: TRADING_ANALYTICS
  size: MEDIUM

# dbt project structure
models:
  - staging/stg_trades.sql
  - staging/stg_symbols.sql
  - marts/fct_trades.sql
  - marts/fct_daily_summary.sql
  - metrics/trader_performance.sql

# BI: Metabase dashboards
dashboards:
  - Trading Volume (daily)
  - Symbol Performance
  - Trader Leaderboard
```

## Pattern 2: The Lakehouse

The Lakehouse architecture combines data lake flexibility with data warehouse capabilities, enabled by open table formats (Iceberg, Delta Lake, Hudi).

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Lakehouse Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Streaming  │  │   Batch     │  │    APIs     │             │
│  │   (Kafka)   │  │  (Files)    │  │             │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   Object Storage                           │ │
│  │                   (S3, GCS, ADLS)                          │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │              Table Format Layer                      │  │ │
│  │  │           (Iceberg / Delta Lake)                     │  │ │
│  │  │                                                      │  │ │
│  │  │   ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │ │
│  │  │   │  Bronze  │  │  Silver  │  │   Gold   │         │  │ │
│  │  │   │  (Raw)   │─▶│(Cleaned) │─▶│  (Marts) │         │  │ │
│  │  │   └──────────┘  └──────────┘  └──────────┘         │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         │                │                │                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Spark    │  │   Trino/    │  │    BI       │             │
│  │  (Batch/ML) │  │  Presto     │  │   Tools     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

**Open formats:** Data stored in open formats (Parquet) with open table formats (Iceberg, Delta). No vendor lock-in.

**Storage-compute separation:** Object storage holds data; compute engines attach as needed.

**Multi-engine:** Different engines for different workloads—Spark for batch, Trino for interactive, specialized engines for ML.

**ACID on object storage:** Table formats enable transactions, time travel, schema evolution on cheap cloud storage.

### The Medallion Architecture

The Lakehouse typically organizes data in layers:

**Bronze (Raw):** Data as-is from sources
- Append-only ingestion
- Full fidelity preserved
- Schema on read

**Silver (Cleaned):** Validated, deduplicated, conformed
- Data quality enforced
- Common transformations applied
- Ready for general use

**Gold (Curated):** Business-level aggregates, feature stores, marts
- Optimized for specific use cases
- Pre-computed metrics
- Self-service ready

```sql
-- Bronze: raw trade events
CREATE TABLE bronze.trades (
    raw_payload STRING,
    source STRING,
    ingested_at TIMESTAMP
) USING iceberg;

-- Silver: cleaned trades
CREATE TABLE silver.trades (
    trade_id STRING,
    symbol STRING,
    price DECIMAL(18, 8),
    quantity DECIMAL(18, 8),
    traded_at TIMESTAMP,
    -- Quality metadata
    is_valid BOOLEAN,
    validation_errors ARRAY<STRING>
) USING iceberg
PARTITIONED BY (days(traded_at));

-- Gold: daily aggregates
CREATE TABLE gold.daily_trading_summary (
    date DATE,
    symbol STRING,
    total_volume DECIMAL(18, 8),
    vwap DECIMAL(18, 8),
    trade_count BIGINT
) USING iceberg
PARTITIONED BY (date);
```

### When It Works Well

**Large-scale data:** Petabytes of data, billions of records. Object storage scales infinitely.

**Diverse workloads:** SQL analytics, ML training, streaming, all on the same data.

**Cost sensitivity:** Object storage is 10-100x cheaper than data warehouse storage.

**Data science teams:** Direct access to raw data for exploration and feature engineering.

**Multi-engine requirements:** Different tools for different jobs, unified by common storage.

### When It Struggles

**Simple BI:** If your needs are standard dashboards, the complexity isn't justified.

**Small data:** Under a terabyte, cloud warehouse simplicity wins.

**Limited engineering:** Lakehouse requires more infrastructure knowledge than managed warehouses.

**Strong governance needs:** Warehouses often have more mature security and governance features.

### Lakehouse vs. Modern Data Stack

| Aspect | Modern Data Stack | Lakehouse |
|--------|-------------------|-----------|
| Primary store | Cloud warehouse | Object storage + table format |
| Lock-in risk | High (warehouse) | Low (open formats) |
| Complexity | Lower | Higher |
| Cost at scale | Higher (compute pricing) | Lower (storage pricing) |
| ML support | Limited | Strong |
| Streaming | Weak | Native |
| Team size | Smaller can manage | Larger team beneficial |

### Example Implementation

**Trading platform scenario:**

```python
# Streaming ingestion to Bronze
trades_stream = spark.readStream \
    .format("kafka") \
    .option("subscribe", "trades") \
    .load()

trades_stream.writeStream \
    .format("iceberg") \
    .option("checkpointLocation", "s3://checkpoints/trades") \
    .toTable("bronze.trades")

# Batch processing to Silver (dbt or Spark)
# models/silver/trades.sql
silver_trades = spark.sql("""
    SELECT
        json_extract(raw_payload, '$.trade_id') as trade_id,
        json_extract(raw_payload, '$.symbol') as symbol,
        CAST(json_extract(raw_payload, '$.price') AS DECIMAL(18,8)) as price,
        CAST(json_extract(raw_payload, '$.quantity') AS DECIMAL(18,8)) as quantity,
        to_timestamp(json_extract(raw_payload, '$.timestamp')) as traded_at
    FROM bronze.trades
    WHERE NOT is_processed
""")

# Gold aggregates (scheduled daily)
daily_summary = spark.sql("""
    SELECT
        DATE(traded_at) as date,
        symbol,
        SUM(quantity) as total_volume,
        SUM(price * quantity) / SUM(quantity) as vwap,
        COUNT(*) as trade_count
    FROM silver.trades
    WHERE DATE(traded_at) = current_date - 1
    GROUP BY DATE(traded_at), symbol
""")
```

## Pattern 3: Streaming-First Architecture

Streaming-first architectures treat real-time event streams as the primary data representation, deriving batch views as needed.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                Streaming-First Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Producers  │  │  Producers  │  │  Producers  │             │
│  │  (Apps)     │  │  (Services) │  │   (IoT)     │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Event Backbone                          │ │
│  │                 (Kafka / Redpanda)                         │ │
│  │                                                            │ │
│  │   trades ──▶ enriched_trades ──▶ trade_aggregates         │ │
│  │                                                            │ │
│  └─────────────────────────┬─────────────────────────────────┘ │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │   Stream    │    │   Stream    │    │    Sink     │        │
│  │  Processor  │    │  Processor  │    │  (to Lake)  │        │
│  │   (Flink)   │    │   (ksqlDB)  │    │             │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  Real-time  │    │  Real-time  │    │  Batch      │        │
│  │   Serving   │    │    APIs     │    │  Analytics  │        │
│  │   (Redis)   │    │             │    │  (Lakehouse)│        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

**Events as source of truth:** The event log is the primary representation. Derived views are secondary.

**Continuous processing:** Transformations happen as data flows, not in scheduled batches.

**Multiple materializations:** The same event stream feeds real-time dashboards, APIs, and batch analytics.

**Replay capability:** Event log enables reprocessing when logic changes.

### The Kappa Pattern

Streaming-first often follows the Kappa architecture (see Chapter 9):

1. **Single processing path:** All data flows through streams
2. **Reprocessing via replay:** Correct errors by replaying from the log
3. **No separate batch layer:** Batch is just a special case of streaming

```python
# Stream processing with Flink SQL
CREATE TABLE trades (
    trade_id STRING,
    symbol STRING,
    price DECIMAL(18, 8),
    quantity DECIMAL(18, 8),
    trade_time TIMESTAMP(3),
    WATERMARK FOR trade_time AS trade_time - INTERVAL '10' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'trades',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'json'
);

-- Real-time aggregation
CREATE TABLE trade_summaries (
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    symbol STRING,
    trade_count BIGINT,
    total_volume DECIMAL(18, 8),
    vwap DECIMAL(18, 8),
    PRIMARY KEY (window_start, symbol) NOT ENFORCED
) WITH (
    'connector' = 'upsert-kafka',
    'topic' = 'trade-summaries',
    'key.format' = 'json',
    'value.format' = 'json'
);

INSERT INTO trade_summaries
SELECT
    window_start,
    window_end,
    symbol,
    COUNT(*) as trade_count,
    SUM(quantity) as total_volume,
    SUM(price * quantity) / SUM(quantity) as vwap
FROM TABLE(TUMBLE(TABLE trades, DESCRIPTOR(trade_time), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, symbol;
```

### When It Works Well

**Real-time requirements:** Fraud detection, algorithmic trading, operational dashboards—anything requiring sub-second latency.

**Event-driven systems:** Microservices architectures where events are the integration mechanism.

**High-volume ingestion:** IoT, clickstream, log aggregation—continuous data flows.

**Unified batch and stream:** When you need both, streaming-first unifies the model.

### When It Struggles

**Complex analytics:** Multi-way joins, complex aggregations across large time ranges are harder in streaming.

**Team expertise:** Streaming requires different skills than batch SQL.

**Debugging:** Stateful streaming jobs are harder to debug than batch pipelines.

**Cost:** Always-on processing infrastructure costs more than batch jobs that scale to zero.

## Pattern 4: Data Mesh

Data Mesh, proposed by Zhamak Dehghani, isn't a technical architecture but an organizational approach that shapes technical choices.

### Core Principles

**Domain ownership:** Data is owned by domain teams, not a central data team.

**Data as a product:** Domains treat their data as products with SLAs, documentation, and quality guarantees.

**Self-serve platform:** Central team provides infrastructure; domains operate independently.

**Federated governance:** Standards and interoperability without centralized control.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Data Mesh Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Self-Serve Data Platform                      │ │
│  │                                                            │ │
│  │   Storage │ Compute │ Catalog │ Quality │ Governance      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         │                │                │                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Trading   │  │    Risk     │  │   Finance   │             │
│  │   Domain    │  │   Domain    │  │   Domain    │             │
│  │             │  │             │  │             │             │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │             │
│  │ │  Data   │ │  │ │  Data   │ │  │ │  Data   │ │             │
│  │ │Products │ │  │ │Products │ │  │ │Products │ │             │
│  │ │         │ │  │ │         │ │  │ │         │ │             │
│  │ │• trades │ │  │ │• exposures│ │  │ │• revenue│ │             │
│  │ │• quotes │ │  │ │• var     │ │  │ │• costs  │ │             │
│  │ │• volumes│ │  │ │• limits  │ │  │ │• pnl    │ │             │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                          │
│              │   Federated Query     │                          │
│              │   (Data Catalog)      │                          │
│              └───────────────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Products

In Data Mesh, a data product includes:

**The data itself:** Tables, streams, or APIs.

**Metadata:** Schema, lineage, quality metrics.

**Code:** Transformations that produce the data.

**Infrastructure:** Compute and storage.

**Interface:** APIs, SQL access, documentation.

```yaml
# data-product.yaml
name: trading.trades
owner: trading-domain-team
description: |
  Real-time and historical trade executions.
  Primary source of truth for all trading activity.

interfaces:
  - type: table
    name: trading.fct_trades
    access: sql
    engine: trino

  - type: stream
    name: trades-events
    access: kafka
    topic: trading.trades.v1

  - type: api
    name: trades-api
    access: rest
    endpoint: /api/v1/trades

sla:
  freshness: 5 minutes
  availability: 99.9%
  quality_score: 95%

quality:
  tests:
    - unique: trade_id
    - not_null: [trade_id, symbol, price, quantity]
    - freshness: 5 minutes

documentation:
  url: https://docs.company.com/data-products/trading/trades
  contact: trading-data@company.com
```

### When It Works Well

**Large organizations:** Hundreds of engineers, dozens of domains. Central teams become bottlenecks.

**Domain expertise matters:** Trading data needs trading experts; risk data needs risk experts.

**Diverse use cases:** Different domains have different requirements that generic solutions can't optimize for.

**Scaling data teams:** Enables parallel development without coordination overhead.

### When It Struggles

**Small organizations:** The overhead isn't justified with 5-10 people.

**Shared data:** Some data truly spans domains and doesn't have a natural owner.

**Platform maturity:** Requires a mature self-serve platform—building that is hard.

**Governance:** Federated governance is harder than centralized governance.

### Data Mesh vs. Centralized

| Aspect | Centralized | Data Mesh |
|--------|-------------|-----------|
| Ownership | Central data team | Domain teams |
| Bottleneck | Central team capacity | Platform capabilities |
| Quality | Central enforcement | Federated standards |
| Speed | Slower (prioritization) | Faster (parallel) |
| Consistency | Easier | Harder |
| Team size | Smaller data team | Larger distributed teams |

## Pattern 5: Hybrid Architectures

In practice, most organizations don't adopt a single pattern—they combine elements based on needs.

### Common Hybrid: Lakehouse + Modern Data Stack

```
┌─────────────────────────────────────────────────────────────────┐
│              Hybrid: Lakehouse + Modern Data Stack               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 Lakehouse (Heavy Lifting)                 │  │
│  │                                                           │  │
│  │   Streaming ──▶ Bronze ──▶ Silver (ML, Complex)          │  │
│  │                                                           │  │
│  └────────────────────────────┬──────────────────────────────┘  │
│                               │                                  │
│                               │ (Sync to Warehouse)              │
│                               ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Modern Data Stack (BI & Analytics)            │  │
│  │                                                            │  │
│  │   Silver Sync ──▶ dbt ──▶ Gold ──▶ BI Tools               │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Use case:** Heavy data engineering on the lakehouse; business analytics in the warehouse.

**Benefits:** Best of both worlds—lakehouse flexibility for data engineering, warehouse simplicity for BI.

**Trade-offs:** Data moves between systems; latency and sync complexity.

### Common Hybrid: Streaming + Batch

```
┌─────────────────────────────────────────────────────────────────┐
│                  Hybrid: Streaming + Batch                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌─────────────┐                            │
│                      │   Kafka     │                            │
│                      │  (Events)   │                            │
│                      └──────┬──────┘                            │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              │              │              │                     │
│              ▼              │              ▼                     │
│       ┌─────────────┐       │       ┌─────────────┐            │
│       │   Flink     │       │       │   Sink to   │            │
│       │(Real-time)  │       │       │  Lakehouse  │            │
│       └──────┬──────┘       │       └──────┬──────┘            │
│              │              │              │                     │
│              ▼              │              ▼                     │
│       ┌─────────────┐       │       ┌─────────────┐            │
│       │   Redis     │       │       │   Spark     │            │
│       │ (Hot Data)  │       │       │  (Batch)    │            │
│       └─────────────┘       │       └──────┬──────┘            │
│                             │              │                     │
│                             │              ▼                     │
│                             │       ┌─────────────┐            │
│                             │       │  Warehouse  │            │
│                             │       │   (BI)      │            │
│                             │       └─────────────┘            │
│                             │                                    │
└─────────────────────────────┴────────────────────────────────────┘
```

**Use case:** Real-time operational needs (fraud, alerting) plus batch analytics.

**Benefits:** Right tool for each job; streaming for speed, batch for depth.

**Trade-offs:** Two processing paths; potential inconsistency; more infrastructure.

### Choosing Your Architecture

**Assessment questions:**

1. **What are your latency requirements?**
   - Minutes to hours: Modern Data Stack or Lakehouse
   - Seconds: Streaming-first
   - Mixed: Hybrid

2. **What's your data volume?**
   - Gigabytes: Modern Data Stack
   - Terabytes to Petabytes: Lakehouse
   - High-velocity events: Streaming

3. **What's your team size and expertise?**
   - Small, SQL-focused: Modern Data Stack
   - Large, diverse: Lakehouse or Data Mesh
   - Streaming expertise: Streaming-first

4. **What's your organizational structure?**
   - Central data team: Modern Data Stack or Lakehouse
   - Distributed domain teams: Data Mesh

5. **What's your budget model?**
   - Predictable costs preferred: Lakehouse (storage-based)
   - Variable usage: Modern Data Stack (pay-per-query)

## Architecture Decision Framework

### Step 1: Clarify Requirements

| Requirement | Questions |
|-------------|-----------|
| Latency | What's the fastest data needs to be available? |
| Scale | How much data now? In 2 years? |
| Use cases | BI? ML? Real-time? All of the above? |
| Team | Size? Skills? Growth plans? |
| Budget | Constraints? Predictability needs? |

### Step 2: Map to Patterns

| If you need... | Consider... |
|----------------|-------------|
| Simple BI, small data | Modern Data Stack |
| Large scale, diverse workloads | Lakehouse |
| Real-time processing | Streaming-first |
| Domain autonomy at scale | Data Mesh |
| Multiple requirements | Hybrid |

### Step 3: Validate Trade-offs

For your chosen pattern, explicitly acknowledge:
- What you gain
- What you give up
- What risks you accept

### Step 4: Plan Evolution

Architectures evolve. Plan for:
- How to migrate if needs change
- What decisions are reversible vs. irreversible
- How to avoid lock-in where possible

## Summary

Architecture patterns provide templates for organizing data systems:

**Modern Data Stack:**
- Cloud-native, SQL-centric, ELT-based
- Best for: Standard BI, small-medium teams
- Trade-off: Simplicity over flexibility

**Lakehouse:**
- Open formats, multi-engine, cost-efficient at scale
- Best for: Large data, diverse workloads, ML
- Trade-off: Flexibility over simplicity

**Streaming-First:**
- Events as source of truth, real-time processing
- Best for: Low-latency, event-driven systems
- Trade-off: Real-time capability over simplicity

**Data Mesh:**
- Domain ownership, data as product
- Best for: Large organizations, distributed teams
- Trade-off: Autonomy over centralization

**Hybrid:**
- Combine patterns for different needs
- Best for: Complex requirements
- Trade-off: Capability over simplicity

**Key insight:** There's no best architecture—only appropriate architectures for specific contexts. Understanding trade-offs lets you choose deliberately rather than following trends.

**Looking ahead:**
- Chapter 16 shows these patterns in practice through case studies
- Chapter 17 provides frameworks for evaluating new technologies

## Further Reading

- Dehghani, Z. (2022). *Data Mesh: Delivering Data-Driven Value at Scale* — Data Mesh principles
- Armbrust, M. et al. (2021). "Lakehouse: A New Generation of Open Platforms" — Lakehouse architecture paper
- Kreps, J. (2014). "Questioning the Lambda Architecture" — Kappa architecture origins
- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — Foundational architecture concepts
- [Databricks Lakehouse Architecture](https://www.databricks.com/glossary/data-lakehouse) — Lakehouse overview
- [Modern Data Stack Overview](https://www.getdbt.com/blog/future-of-the-modern-data-stack) — dbt Labs perspective

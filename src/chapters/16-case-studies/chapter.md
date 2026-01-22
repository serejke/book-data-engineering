---
chapter: 16
title: "Case Studies: Architectures in Practice"
estimated_pages: 35-40
status: draft
last_updated: 2025-01-22
---

# Case Studies: Architectures in Practice

Theory clarifies; practice teaches. The previous chapters presented concepts, patterns, and frameworks in isolation. This chapter shows how they combine in real systems, tracing the reasoning behind architectural decisions and the trade-offs that emerged.

These case studies are composites—drawn from public engineering blogs, conference talks, and common patterns—rather than descriptions of specific companies. The goal isn't to copy architectures but to understand the decision-making process: what problems arose, what options existed, why certain choices were made.

## Case Study 1: Trading Analytics Platform

A quantitative trading firm needs analytics infrastructure supporting both real-time operational decisions and historical analysis.

### Context

**The business:** Algorithmic trading across multiple exchanges, executing thousands of trades per second. Strategies depend on real-time market data and historical pattern analysis.

**The team:** 5 data engineers, 10 quantitative researchers, 20 traders. Researchers have Python expertise; traders want dashboards.

**The data:**
- Market data: Quotes and trades from 10 exchanges, ~50GB/day
- Execution data: Firm's own trades, ~5GB/day
- Reference data: Symbols, exchanges, counterparties
- Alternative data: News, sentiment, satellite imagery

**Key requirements:**
- Real-time: Sub-second latency for trading signals
- Historical: 5 years of tick data for backtesting
- Self-service: Researchers need direct data access
- Cost-sensitive: Trading margins are thin

### Architecture Evolution

**Phase 1: The Spreadsheet Era**

Like many trading desks, they started with Excel. Market data piped into spreadsheets. Analysis done manually. Historical data on shared drives.

**Problems:**
- No single source of truth
- Backtests couldn't access consistent historical data
- Real-time analysis limited to what Excel could handle
- No audit trail for compliance

**Phase 2: Database + Real-time Split**

First engineering hire built two systems:

```
Real-time path:
  Exchange feeds → Custom C++ → Redis → Trading systems

Historical path:
  Daily files → PostgreSQL → Researcher queries
```

**What worked:**
- Real-time latency improved dramatically
- Researchers could write SQL

**What didn't:**
- Two separate systems meant different numbers
- Historical analysis couldn't inform real-time decisions
- PostgreSQL couldn't handle tick data scale

**Phase 3: Unified Lakehouse**

After evaluating options, they chose a streaming-lakehouse hybrid:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Trading Analytics Platform                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐                                                │
│  │  Exchange   │                                                │
│  │   Feeds     │                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Kafka Cluster                         │   │
│  │   quotes.raw ─▶ quotes.enriched ─▶ signals.generated    │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │   Flink     │   │  Spark      │   │   Sink      │          │
│  │ (Real-time  │   │ Streaming   │   │ (to Iceberg)│          │
│  │  Signals)   │   │ (Enrich)    │   │             │          │
│  └──────┬──────┘   └─────────────┘   └──────┬──────┘          │
│         │                                   │                   │
│         ▼                                   ▼                   │
│  ┌─────────────┐            ┌───────────────────────────────┐  │
│  │   Redis     │            │        Iceberg Tables         │  │
│  │(Hot signals)│            │                               │  │
│  └─────────────┘            │  bronze.quotes (raw)          │  │
│         │                   │  silver.quotes (validated)    │  │
│         │                   │  gold.ohlcv (aggregated)      │  │
│         ▼                   │                               │  │
│  ┌─────────────┐            └──────────────┬────────────────┘  │
│  │  Trading    │                           │                   │
│  │  Systems    │                           │                   │
│  └─────────────┘            ┌──────────────┴────────────────┐  │
│                             │                               │   │
│                             ▼                               ▼   │
│                      ┌─────────────┐              ┌──────────┐ │
│                      │   Trino     │              │  Jupyter │ │
│                      │(Interactive)│              │(Research)│ │
│                      └─────────────┘              └──────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Decisions

**Why Kafka as the backbone?**

Kafka provides the single source of truth for all events. Both real-time processing and historical storage consume from the same stream. This eliminates the "two different numbers" problem.

**Why Flink for signals?**

Signal generation requires sub-millisecond latency. Flink's true streaming (not micro-batch) meets this requirement. The team evaluated Spark Structured Streaming but couldn't achieve the latency targets.

**Why Iceberg for storage?**

Tick data requires time-travel queries for backtesting ("what did data look like at 2pm on March 15?"). Iceberg's snapshot isolation enables this. Partitioning by hour keeps file sizes manageable for the high-volume quote data.

**Why Trino for interactive queries?**

Researchers need ad-hoc queries across years of data. Trino's query performance on Iceberg exceeded alternatives. The same tables serve both Trino and Spark.

### Trade-offs Accepted

**Complexity:** Three processing engines (Flink, Spark, Trino) require different expertise. The team accepted this because each excels at its specific workload.

**Eventual consistency:** The lakehouse lags behind real-time by seconds to minutes. Acceptable because historical analysis doesn't need sub-second freshness.

**Infrastructure burden:** Self-hosted Kafka and Flink require operational investment. A managed alternative would simplify operations but add latency and cost.

### Lessons Learned

**Start with the data flow, not the tools.** Mapping out how data needed to move—from exchange to trading system to researcher—clarified requirements before tool selection.

**Latency requirements drive architecture.** The sub-millisecond signal requirement forced the streaming-first approach. Without that requirement, a simpler batch architecture would have sufficed.

**Unify where possible, split where necessary.** Kafka as single source prevents divergence. Separate engines for different latency tiers optimizes each use case.

## Case Study 2: E-Commerce Analytics

A mid-size e-commerce company needs analytics to understand customer behavior, optimize marketing, and improve operations.

### Context

**The business:** Online retailer with 2 million monthly active users, 500K orders/month. Growing 40% year-over-year.

**The team:** 2 data engineers (recently hired), 3 analysts, 1 data scientist. Limited infrastructure experience.

**The data:**
- Clickstream: 100M events/day from web and mobile
- Transactions: Orders, payments, fulfillment
- Product catalog: 50K SKUs with attributes
- Marketing: Campaign data from 5 platforms
- Customer support: Tickets, NPS surveys

**Key requirements:**
- Self-service BI for business stakeholders
- Marketing attribution modeling
- Inventory forecasting
- Customer segmentation
- Minimal operational overhead

### Architecture Evolution

**Phase 1: Tool Sprawl**

Before the data team, analytics happened in silos:
- Marketing used Google Analytics and each platform's native analytics
- Finance exported data to Excel
- Operations queried the production database directly

**Problems:**
- Different teams reported different revenue numbers
- No customer-level view across touchpoints
- Production database slowed during analytical queries
- No historical data retention

**Phase 2: Modern Data Stack**

The new data engineers chose a Modern Data Stack approach:

```
┌─────────────────────────────────────────────────────────────────┐
│                E-Commerce Analytics Platform                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Shopify  │ │  Google  │ │ Facebook │ │  Stripe  │           │
│  │   API    │ │   Ads    │ │   Ads    │ │   API    │           │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │
│       │            │            │            │                   │
│       └────────────┴─────┬──────┴────────────┘                   │
│                          │                                       │
│                          ▼                                       │
│              ┌───────────────────────┐                           │
│              │       Fivetran        │                           │
│              │    (ELT Ingestion)    │                           │
│              └───────────┬───────────┘                           │
│                          │                                       │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Snowflake                              │  │
│  │                                                            │  │
│  │   raw.shopify.*  │  raw.google_ads.*  │  raw.stripe.*    │  │
│  │                                                            │  │
│  │   ┌────────────────────────────────────────────────────┐  │  │
│  │   │                      dbt                            │  │  │
│  │   │                                                     │  │  │
│  │   │  staging/ ──▶ intermediate/ ──▶ marts/             │  │  │
│  │   │                                                     │  │  │
│  │   │  • stg_orders        • int_customer_orders         │  │  │
│  │   │  • stg_events        • int_attribution             │  │  │
│  │   │  • stg_products      • fct_orders                  │  │  │
│  │   │                      • dim_customers               │  │  │
│  │   │                      • dim_products                │  │  │
│  │   └────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│           ┌──────────────┼──────────────┐                       │
│           │              │              │                        │
│           ▼              ▼              ▼                        │
│    ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│    │  Looker  │   │ dbt      │   │  Python  │                   │
│    │  (BI)    │   │ Metrics  │   │  (ML)    │                   │
│    └──────────┘   └──────────┘   └──────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### dbt Model Structure

**Staging layer:**
```sql
-- models/staging/shopify/stg_shopify__orders.sql
WITH source AS (
    SELECT * FROM {{ source('shopify', 'orders') }}
),

renamed AS (
    SELECT
        id AS order_id,
        customer_id,
        CAST(total_price AS DECIMAL(10,2)) AS order_total,
        CAST(created_at AS TIMESTAMP) AS ordered_at,
        financial_status AS payment_status,
        fulfillment_status
    FROM source
)

SELECT * FROM renamed
```

**Intermediate layer:**
```sql
-- models/intermediate/int_customer_orders.sql
WITH orders AS (
    SELECT * FROM {{ ref('stg_shopify__orders') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_shopify__customers') }}
),

customer_metrics AS (
    SELECT
        customer_id,
        MIN(ordered_at) AS first_order_at,
        MAX(ordered_at) AS last_order_at,
        COUNT(*) AS lifetime_orders,
        SUM(order_total) AS lifetime_value
    FROM orders
    GROUP BY customer_id
)

SELECT
    c.*,
    cm.first_order_at,
    cm.last_order_at,
    cm.lifetime_orders,
    cm.lifetime_value
FROM customers c
LEFT JOIN customer_metrics cm ON c.customer_id = cm.customer_id
```

**Mart layer:**
```sql
-- models/marts/marketing/fct_attribution.sql
{{ config(materialized='incremental', unique_key='order_id') }}

WITH orders AS (
    SELECT * FROM {{ ref('stg_shopify__orders') }}
    {% if is_incremental() %}
    WHERE ordered_at > (SELECT MAX(ordered_at) FROM {{ this }})
    {% endif %}
),

touchpoints AS (
    SELECT * FROM {{ ref('int_marketing_touchpoints') }}
),

attributed AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.order_total,
        o.ordered_at,
        -- Last-touch attribution
        LAST_VALUE(t.channel) OVER (
            PARTITION BY o.customer_id
            ORDER BY t.touchpoint_at
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS last_touch_channel,
        -- First-touch attribution
        FIRST_VALUE(t.channel) OVER (
            PARTITION BY o.customer_id
            ORDER BY t.touchpoint_at
        ) AS first_touch_channel
    FROM orders o
    LEFT JOIN touchpoints t
        ON o.customer_id = t.customer_id
        AND t.touchpoint_at < o.ordered_at
)

SELECT DISTINCT * FROM attributed
```

### Key Decisions

**Why Fivetran over custom ingestion?**

With 2 engineers and 10+ sources, building and maintaining custom connectors wasn't feasible. Fivetran's pre-built connectors and managed infrastructure freed engineers to focus on transformation logic.

**Why Snowflake?**

The team evaluated Snowflake, BigQuery, and Redshift. Snowflake won on:
- Separation of storage and compute (scale independently)
- Automatic clustering (less tuning)
- Data sharing (for future partner integrations)
- Strong dbt community support

**Why not a lakehouse?**

Data volumes (GBs, not TBs) didn't justify lakehouse complexity. The team lacked infrastructure expertise. Snowflake's managed service matched their capabilities.

**Why Looker?**

Looker's semantic layer (LookML) enforced consistent metric definitions—critical after the "different revenue numbers" problem. Self-service dashboards reduced analyst burden.

### Trade-offs Accepted

**Cost:** Snowflake and Fivetran aren't cheap. At current scale (~$3K/month), acceptable. The team monitors for cost creep as data grows.

**Latency:** Data is 15-60 minutes behind real-time. Acceptable for their use cases. If they need real-time, they'll add a streaming layer.

**Lock-in:** Deep integration with Snowflake and Fivetran. Migration would be expensive. Acceptable given the productivity gains.

### Lessons Learned

**Match architecture to team capabilities.** A lakehouse might be "better" technically, but the Modern Data Stack matched what 2 engineers could operate.

**Invest in the semantic layer.** Looker's LookML took effort upfront but eliminated metric inconsistencies. Worth the investment.

**Start with boring technology.** The team could have built a complex event-driven architecture. Simple ELT pipelines got them to value faster.

## Case Study 3: IoT Data Platform

A manufacturing company needs to analyze sensor data from factory equipment for predictive maintenance and process optimization.

### Context

**The business:** 50 factories, 10,000 machines, each with 10-50 sensors. Equipment downtime costs $100K+/hour.

**The team:** 8 data engineers, 5 data scientists, industrial automation specialists.

**The data:**
- Sensor readings: 1B events/day, sub-second intervals
- Machine metadata: Models, configurations, maintenance history
- Production data: Batch records, quality measurements
- External: Weather, energy prices, supply chain

**Key requirements:**
- Real-time anomaly detection (seconds, not minutes)
- Historical analysis for pattern discovery
- Edge processing (factory networks have limited bandwidth)
- 7-year retention for regulatory compliance

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    IoT Data Platform                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Factory Edge                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │   ┌──────┐  ┌──────┐  ┌──────┐                           │  │
│  │   │Sensor│  │Sensor│  │Sensor│  ...                      │  │
│  │   └──┬───┘  └──┬───┘  └──┬───┘                           │  │
│  │      │         │         │                                │  │
│  │      └─────────┼─────────┘                                │  │
│  │                ▼                                          │  │
│  │   ┌────────────────────────┐                             │  │
│  │   │    Edge Gateway        │                             │  │
│  │   │  • Local buffering     │                             │  │
│  │   │  • Downsampling        │                             │  │
│  │   │  • Local anomaly check │                             │  │
│  │   └───────────┬────────────┘                             │  │
│  └───────────────┼──────────────────────────────────────────┘  │
│                  │ (compressed, batched)                        │
│                  ▼                                              │
│  Cloud Platform                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                     Kafka                                  │ │
│  │   sensors.raw ──▶ sensors.validated ──▶ anomalies.detected│ │
│  └─────────────────────────┬─────────────────────────────────┘ │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │   Flink     │   │   Spark     │   │    Sink     │          │
│  │ (Anomaly    │   │ (ML model   │   │ (to Delta   │          │
│  │  Detection) │   │  scoring)   │   │    Lake)    │          │
│  └──────┬──────┘   └─────────────┘   └──────┬──────┘          │
│         │                                   │                   │
│         ▼                                   ▼                   │
│  ┌─────────────┐            ┌───────────────────────────────┐  │
│  │  Alert      │            │       Delta Lake              │  │
│  │  System     │            │                               │  │
│  │  (PagerDuty)│            │  hot/ (7 days, second grain) │  │
│  └─────────────┘            │  warm/ (90 days, minute grain)│  │
│                             │  cold/ (7 years, hour grain)  │  │
│                             └───────────────────────────────┘  │
│                                        │                        │
│                                        ▼                        │
│                             ┌───────────────────────┐          │
│                             │   Analytics Layer     │          │
│                             │   (Databricks SQL,    │          │
│                             │    Notebooks)         │          │
│                             └───────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

**Edge processing for bandwidth and latency:**

50 factories × 10K machines × 50 sensors × 10 readings/second = 250M readings/second. Sending everything to the cloud is infeasible.

Edge gateways:
- Buffer data locally (survive network outages)
- Downsample routine readings (1/second → 1/minute for normal operation)
- Detect local anomalies (threshold checks)
- Forward critical events immediately

This reduces cloud ingestion to ~10M events/second while ensuring critical anomalies reach the cloud within seconds.

**Tiered storage for cost and performance:**

Raw sensor data at second granularity for 7 years would cost millions in storage. Tiered aggregation balances cost and utility:

```
Hot tier (7 days):
  - Full resolution (sub-second)
  - Optimized for real-time queries
  - ~10 TB

Warm tier (90 days):
  - Minute aggregates (min, max, avg, stddev)
  - Pattern analysis, recent investigations
  - ~5 TB

Cold tier (7 years):
  - Hourly aggregates
  - Regulatory compliance, long-term trends
  - ~50 TB
```

**Delta Lake for ACID on sensor data:**

Sensor data requires:
- Exactly-once ingestion (no duplicate readings)
- Late data handling (edge buffers may deliver delayed)
- Time travel (investigate past incidents)

Delta Lake's ACID transactions and Z-ordering on (machine_id, timestamp) optimize both ingestion and query patterns.

**Flink for sub-second anomaly detection:**

ML models score sensor windows to detect anomalies. Flink's true streaming enables:
- Window-based feature computation
- Model inference in the stream
- Sub-second alerting

Latency from sensor reading to alert: <5 seconds.

### ML Model Deployment

The data science team trains models in batch, deploys to streaming:

```python
# Training (batch, weekly)
training_data = spark.read.table("delta.warm.sensor_features")
model = train_anomaly_model(training_data)
mlflow.log_model(model, "anomaly_detector")

# Inference (streaming, real-time)
# Flink job loads model from MLflow
model = load_model("models:/anomaly_detector/production")

sensor_stream
    .keyBy("machine_id")
    .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.seconds(30)))
    .process(AnomalyScorer(model))
    .filter(lambda x: x.score > threshold)
    .addSink(alert_sink)
```

### Lessons Learned

**Edge computing is essential for IoT scale.** Moving computation to the data (rather than data to computation) was the only way to achieve latency and cost targets.

**Aggregation tiers match query patterns.** Nobody queries second-granularity data from 3 years ago. Matching storage granularity to actual usage dramatically reduced costs.

**Streaming and batch aren't either/or.** Flink for real-time anomalies, Spark for batch model training, same data in Delta Lake serves both.

## Case Study 4: Data Mesh Implementation

A large financial services company transforms from a central data warehouse to a federated data mesh.

### Context

**The business:** Bank with retail, commercial, and investment divisions. 50,000 employees, $100B in assets.

**The legacy state:**
- Central data warehouse maintained by 30-person team
- 18-month backlog of data requests
- Different divisions maintain shadow IT data systems
- Compliance struggles with data lineage

**The catalyst:** Regulatory requirement for comprehensive data lineage. The central team couldn't map data flows across shadow systems.

### The Transformation

**Phase 1: Platform Foundation (Year 1)**

Before decentralizing, they built a self-serve platform:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Self-Serve Data Platform                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Infrastructure Layer (Platform Team)                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │  Object  │  │  Compute │  │   Data   │  │ Identity │  │ │
│  │  │ Storage  │  │  (Spark, │  │ Catalog  │  │    &     │  │ │
│  │  │  (S3)    │  │  Trino)  │  │          │  │  Access  │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Developer Experience Layer                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │  Data    │  │  Quality │  │   CI/CD  │  │  Docs    │  │ │
│  │  │ Product  │  │  Frame-  │  │ Templates│  │ Generator│  │ │
│  │  │ Template │  │  work    │  │          │  │          │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Key platform capabilities:
- **Storage provisioning:** Teams request buckets via self-service portal
- **Compute provisioning:** Spin up Spark clusters, Trino endpoints
- **Schema registry:** Central catalog for all data products
- **Quality framework:** Standardized testing with Great Expectations
- **CI/CD templates:** dbt projects, Airflow DAGs with best practices baked in

**Phase 2: Domain Pilots (Year 2)**

Three domains volunteered as pilots:
- Retail Banking (customer accounts, transactions)
- Cards (credit card transactions, rewards)
- Risk (credit scores, exposure calculations)

Each domain:
1. Identified their data products
2. Assigned product owners
3. Built on the self-serve platform
4. Published to the central catalog

```yaml
# Example: Retail Banking data product
name: retail.customer_accounts
owner: retail-data-team
domain: retail-banking

schema:
  - name: account_id
    type: STRING
    pii: true
  - name: customer_id
    type: STRING
    pii: true
  - name: account_type
    type: STRING
  - name: balance
    type: DECIMAL(18,2)
    classification: confidential
  - name: opened_date
    type: DATE

sla:
  freshness: 1 hour
  availability: 99.9%

access:
  - role: retail-analysts
    level: read
  - role: risk-team
    level: read
  - role: compliance
    level: read

quality_checks:
  - unique: account_id
  - not_null: [account_id, customer_id, account_type]
  - freshness: 1 hour
```

**Phase 3: Federated Governance (Year 3)**

With domains producing data products, governance became critical:

**Data Council:** Representatives from each domain plus compliance, security, and the platform team. Sets global policies.

**Policies implemented:**
- PII handling standards
- Retention requirements by data classification
- Access review cadence
- Quality score thresholds

**Automated enforcement:**
```python
# Policy-as-code example
def validate_data_product(product_spec):
    violations = []

    # PII must have access controls
    if has_pii(product_spec) and not has_access_controls(product_spec):
        violations.append("PII data requires access controls")

    # Financial data requires 7-year retention
    if product_spec.domain == "finance" and product_spec.retention < years(7):
        violations.append("Financial data requires 7-year retention")

    # All products need quality checks
    if not product_spec.quality_checks:
        violations.append("Quality checks required")

    return violations
```

### Organizational Changes

The transformation wasn't primarily technical—it was organizational:

**Central team transformation:**
- From: 30 people building reports
- To: 15 people building platform + 15 embedded in domains

**Domain team growth:**
- Each domain added 2-3 data engineers
- Product owners from business side
- Analysts remained in domains (not central)

**Skill building:**
- Platform team trained domain engineers
- Shared patterns and templates
- Internal community of practice

### Results After 3 Years

**Request backlog:** 18 months → 2 weeks (domains self-serve)

**Data products:** 3 (legacy reports) → 150 (domain-owned products)

**Data lineage:** 20% coverage → 95% coverage (regulatory requirement met)

**Time to new data product:** 6 months → 2 weeks

**Challenges remained:**
- Cross-domain data products still difficult
- Some domains more mature than others
- Platform team capacity constraints

### Lessons Learned

**Platform before mesh.** Decentralizing without a solid platform creates chaos. The self-serve platform was prerequisite to domain autonomy.

**Start with willing domains.** Pilots with motivated teams built success stories that convinced skeptics.

**Governance is federated, not absent.** Data mesh doesn't mean no governance—it means governance by collaboration rather than central mandate.

**Organizational change is harder than technical change.** Redefining team structures, career paths, and incentives took more effort than building the platform.

## Cross-Cutting Lessons

Across these case studies, common themes emerge:

### Match Architecture to Context

The trading firm needed sub-millisecond latency—streaming-first was required. The e-commerce company needed self-service BI—Modern Data Stack fit perfectly. IoT required edge processing—no alternative existed.

**Lesson:** Requirements should drive architecture, not the reverse.

### Start Simple, Evolve Deliberately

Every case study showed evolution: spreadsheets → databases → modern architectures. Each step added complexity in response to real problems.

**Lesson:** Don't architect for hypothetical future scale. Build for current needs; evolve when actual needs change.

### Unify Where It Matters

The trading firm unified on Kafka to ensure consistent data. The e-commerce company unified metrics in Looker's semantic layer. The bank unified on a catalog for governance.

**Lesson:** Identify where consistency matters most; allow variation elsewhere.

### Teams Shape Architecture (and Vice Versa)

The e-commerce company chose the Modern Data Stack because 2 engineers couldn't operate a lakehouse. The bank's data mesh required organizational restructuring.

**Lesson:** Conway's Law applies. Architecture and organization must align.

### Trade-offs Are Inevitable

Every architecture accepted trade-offs: latency vs. simplicity, cost vs. capability, flexibility vs. consistency. Explicit trade-offs are better than implicit ones.

**Lesson:** Document what you're giving up, not just what you're gaining.

## Summary

These case studies demonstrated patterns in practice:

**Trading analytics:** Streaming-first with lakehouse historical storage. Driven by sub-second latency requirements.

**E-commerce analytics:** Modern Data Stack. Matched to small team and standard BI use cases.

**IoT platform:** Edge computing + tiered storage. Required by data volume and latency constraints.

**Data mesh:** Organizational transformation enabled by self-serve platform. Driven by scale and regulatory requirements.

**Common themes:**
- Match architecture to requirements and team capabilities
- Evolve incrementally in response to real problems
- Unify where consistency matters; allow variation elsewhere
- Accept and document trade-offs explicitly

**Looking ahead:**
- Chapter 17 provides frameworks for evaluating new technologies as they emerge
- Appendix B offers decision flowcharts synthesizing these lessons

## Further Reading

- [Netflix Data Mesh](https://netflixtechblog.com/data-mesh-a-data-movement-and-processing-platform-netflix-1288bcab2873) — Netflix's federated data approach
- [Uber's Big Data Platform](https://eng.uber.com/uber-big-data-platform/) — Unified batch and streaming
- [Airbnb's Data Quality](https://medium.com/airbnb-engineering/data-quality-at-airbnb-e582465f3ef7) — Quality at scale
- [Spotify's Event Delivery](https://engineering.atspotify.com/2020/02/event-delivery/) — Streaming architecture
- [LinkedIn's Lakehouse](https://engineering.linkedin.com/blog/2021/lakehouse) — Enterprise lakehouse implementation

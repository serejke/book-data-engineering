# Appendix B: Decision Flowcharts

*Visual decision aids for common architectural choices.*

---

## 1. Choosing a Table Format

```
START: Need ACID on object storage?
│
├─ NO → Consider: Plain Parquet with careful file management
│        (Acceptable for append-only, immutable data)
│
└─ YES → Multi-engine environment?
          │
          ├─ YES → Strong Presto/Trino usage?
          │         │
          │         ├─ YES → Apache Iceberg (best metadata handling)
          │         │
          │         └─ NO → Need frequent row-level updates?
          │                  │
          │                  ├─ YES → Apache Hudi (optimized for upserts)
          │                  │
          │                  └─ NO → Apache Iceberg (safest default)
          │
          └─ NO → Spark-centric environment?
                   │
                   ├─ YES → Using Databricks?
                   │         │
                   │         ├─ YES → Delta Lake (native integration)
                   │         │
                   │         └─ NO → Delta Lake or Iceberg (both work well)
                   │
                   └─ NO → Evaluate based on ecosystem support
```

**Key Questions to Ask:**
1. What compute engines will query this data?
2. How often do you need to update/delete records?
3. Is partition evolution important?
4. What's your existing ecosystem (Databricks, AWS, etc.)?

---

## 2. Batch vs Streaming vs Micro-Batch

```
START: What's your latency requirement?
│
├─ Hours to Days acceptable
│   │
│   └─ BATCH PROCESSING
│       - Simpler architecture
│       - Cost-effective
│       - Use: Spark batch, dbt, scheduled queries
│
├─ Minutes acceptable
│   │
│   └─ MICRO-BATCH
│       - Balance of simplicity and speed
│       - Easier late-data handling than streaming
│       - Use: Spark Structured Streaming, frequent batch jobs
│
└─ Seconds or less required
    │
    └─ Need to handle complex event patterns?
        │
        ├─ YES (CEP, sessionization, complex joins)
        │   │
        │   └─ FULL STREAMING
        │       - Highest complexity
        │       - Use: Flink, Kafka Streams
        │
        └─ NO (simple transformations)
            │
            └─ MICRO-BATCH may still work
                - Tune batch interval to seconds
                - Simpler than full streaming
```

**Key Questions to Ask:**
1. What's the cost of stale data to your business?
2. Do you need to correlate events across time windows?
3. What's your team's streaming expertise?
4. Can you tolerate data loss during failures?

---

## 3. Choosing an Orchestrator

```
START: What's your primary workflow type?
│
├─ Data transformations (SQL/dbt focused)
│   │
│   └─ Dagster
│       - Asset-centric model fits data transformations
│       - Excellent dbt integration
│
├─ Complex task dependencies
│   │
│   └─ Existing Airflow investment?
│       │
│       ├─ YES → Stick with Airflow 3
│       │         - Don't migrate without strong reason
│       │
│       └─ NO → Team size and experience?
│                │
│                ├─ Small team, Python-native
│                │   └─ Prefect (simpler, faster start)
│                │
│                └─ Larger team, ops capacity
│                    └─ Airflow 3 (most flexible)
│
└─ ML/AI pipelines
    │
    └─ Consider: Dagster or specialized ML platforms
        - Asset model works well for ML
        - Or use Kubeflow, MLflow for ML-specific needs
```

**Key Questions to Ask:**
1. Do you think in "tasks" or "data assets"?
2. How important is local development and testing?
3. What's your ops team's capacity?
4. Do you need dynamic workflow generation?

---

## 4. When to Use a Dedicated OLAP Store

```
START: Query latency requirement?
│
├─ Sub-second required for dashboards
│   │
│   └─ Query concurrency?
│       │
│       ├─ High (100s of users)
│       │   │
│       │   └─ Real-time data ingestion needed?
│       │       │
│       │       ├─ YES → Apache Druid
│       │       │
│       │       └─ NO → ClickHouse
│       │
│       └─ Low (analysts, ad-hoc)
│           │
│           └─ Lakehouse query engine may suffice
│               - Trino/Starburst on Iceberg
│               - Spark SQL on Delta Lake
│
└─ Seconds acceptable
    │
    └─ Lakehouse likely sufficient
        - Materialize common queries
        - Use caching layers
```

**Key Questions to Ask:**
1. How many concurrent queries do you expect?
2. Is real-time data freshness required?
3. What's your query pattern (point lookups vs aggregations)?
4. Can you pre-aggregate or cache results?

---

## 5. Data Modeling Approach

```
START: Primary consumers of this data?
│
├─ BI Tools / Analysts
│   │
│   └─ Star Schema (Dimensional Model)
│       - Predictable join patterns
│       - Optimized for ad-hoc queries
│       - Consider wide tables for simple cases
│
├─ Data Scientists / ML
│   │
│   └─ Wide, denormalized tables
│       - Feature-centric design
│       - Point-in-time correctness critical
│       - Consider feature stores
│
├─ Applications (serving layer)
│   │
│   └─ Query pattern driven
│       - Pre-aggregate for common queries
│       - Consider materialized views
│       - May need OLAP store
│
└─ Multiple consumers
    │
    └─ Medallion Architecture
        - Bronze: raw, append-only
        - Silver: cleaned, conformed
        - Gold: business-specific aggregates
```

**Key Questions to Ask:**
1. Who will query this data and how?
2. How often does the schema change?
3. Do you need historical versions of dimensions?
4. What's the query latency requirement?

---

## 6. Storage Format Selection

```
START: Primary workload type?
│
├─ Analytics (read many columns, aggregate)
│   │
│   └─ Columnar format
│       │
│       └─ Hive ecosystem primary?
│           │
│           ├─ YES → ORC
│           │
│           └─ NO → Parquet (universal choice)
│
├─ Record-oriented (read/write full records)
│   │
│   └─ Schema evolution critical?
│       │
│       ├─ YES → Avro
│       │
│       └─ NO → Parquet still works
│                (most tools handle full-row reads fine)
│
└─ Mixed or uncertain
    │
    └─ Parquet
        - Safe default
        - Widest ecosystem support
```

---

## Quick Reference: When to Use What

| Scenario | Recommended Approach |
|----------|---------------------|
| New lakehouse project | Iceberg + Spark + Airflow/Dagster |
| Databricks shop | Delta Lake + Spark + Airflow |
| High-frequency updates | Hudi or Iceberg with merge |
| Real-time dashboards | ClickHouse or Druid |
| Local analytics/prototyping | DuckDB + Parquet |
| SQL transformations | dbt + warehouse/lakehouse |
| Complex ML pipelines | Dagster + feature store |
| Simple scheduled jobs | Prefect or Airflow |

---

*Last updated: 2025-01-22*

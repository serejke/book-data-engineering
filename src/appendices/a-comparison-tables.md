# Appendix A: Technology Comparison Tables

*Quick reference tables for comparing technologies discussed in this book.*

---

## Table Formats Comparison

| Feature | Apache Iceberg | Delta Lake | Apache Hudi |
|---------|---------------|------------|-------------|
| **Origin** | Netflix (2017) | Databricks (2019) | Uber (2016) |
| **Primary Design Goal** | Correctness & reliability | Spark integration | Record-level updates |
| **ACID Transactions** | Yes | Yes | Yes |
| **Schema Evolution** | Full support | Full support | Full support |
| **Partition Evolution** | Yes (hidden partitions) | Limited | Limited |
| **Time Travel** | Yes (snapshots) | Yes (versioning) | Yes (timeline) |
| **Spark Integration** | Native | Native (optimized) | Native |
| **Flink Integration** | Good | Limited | Good |
| **Trino/Presto Integration** | Excellent | Good | Good |
| **Merge-on-Read** | Yes | Yes | Yes (primary focus) |
| **Copy-on-Write** | Yes | Yes | Yes |
| **Metadata Storage** | Manifest files | Transaction log | Timeline |
| **Compaction** | Manual/scheduled | Auto-optimize | Built-in |
| **Row-Level Updates** | Merge operations | Merge operations | Primary strength |

**Decision Summary**: [See Chapter 5 for detailed guidance]
- **Iceberg**: Best for multi-engine environments, correctness-critical workloads
- **Delta Lake**: Best for Spark-centric environments, Databricks users
- **Hudi**: Best for high-frequency updates, CDC workloads

---

## File Formats Comparison

| Feature | Parquet | ORC | Avro |
|---------|---------|-----|------|
| **Storage Type** | Columnar | Columnar | Row-based |
| **Primary Use Case** | Analytics | Analytics (Hive) | Data exchange |
| **Compression** | Excellent | Excellent | Good |
| **Schema Evolution** | Limited | Limited | Full support |
| **Splittable** | Yes | Yes | Yes (with sync markers) |
| **Read Performance** | Fast (analytics) | Fast (analytics) | Fast (full row) |
| **Write Performance** | Moderate | Moderate | Fast |
| **Nested Data** | Good support | Good support | Excellent support |
| **Ecosystem** | Universal | Hadoop/Hive | Universal |

**Decision Summary**: [See Chapter 4 for detailed guidance]
- **Parquet**: Default choice for analytics workloads
- **ORC**: Consider for Hive-centric environments
- **Avro**: Use for data exchange, schema evolution needs

---

## Orchestration Tools Comparison

| Feature | Apache Airflow 3 | Prefect | Dagster |
|---------|-----------------|---------|---------|
| **First Release** | 2015 | 2018 | 2019 |
| **Programming Model** | DAGs via Python | Flows & Tasks | Software-Defined Assets |
| **UI** | Comprehensive | Clean, modern | Data-centric |
| **Dynamic Workflows** | Limited | Native | Native |
| **Testing** | Challenging | Good | Excellent |
| **Local Development** | Docker-dependent | Easy | Easy |
| **Scheduling** | Built-in | Cloud or self-hosted | Built-in |
| **Backfills** | Built-in | Manual | Built-in |
| **Community** | Large, mature | Growing | Growing |
| **Cloud Offering** | Astronomer, MWAA | Prefect Cloud | Dagster Cloud |
| **Learning Curve** | Moderate | Low | Moderate |

**Decision Summary**: [See Chapter 11 for detailed guidance]
- **Airflow**: Best for teams with existing investment, complex scheduling needs
- **Prefect**: Best for Python-native teams, simpler workflows
- **Dagster**: Best for data-centric thinking, asset-oriented pipelines

---

## OLAP Databases Comparison

| Feature | ClickHouse | Apache Druid | DuckDB |
|---------|------------|--------------|--------|
| **Architecture** | Column-store DBMS | Real-time OLAP | Embedded analytics |
| **Primary Use Case** | High-volume analytics | Real-time dashboards | Local analytics |
| **Query Latency** | Sub-second | Sub-second | Varies (local) |
| **Scaling** | Horizontal (sharding) | Horizontal (native) | Single-node |
| **Real-Time Ingestion** | Yes | Primary strength | No |
| **SQL Support** | Full | Limited | Full |
| **Joins** | Good | Limited | Excellent |
| **Deployment** | Self-hosted or Cloud | Self-hosted or Cloud | Embedded |
| **Resource Efficiency** | High | Moderate | Very high |
| **Learning Curve** | Moderate | High | Low |

**Decision Summary**: [See Chapter 6 for detailed guidance]
- **ClickHouse**: Best for high-volume, low-latency analytics
- **Druid**: Best for real-time event data, dashboards
- **DuckDB**: Best for local analytics, prototyping, small-medium data

---

## Batch vs Streaming Decision Matrix

| Requirement | Batch | Micro-Batch | Stream |
|-------------|-------|-------------|--------|
| **Latency Tolerance** | Hours-days | Minutes | Seconds-milliseconds |
| **Data Completeness** | Full windows | Near-complete | Partial (with late data) |
| **Complexity** | Low | Moderate | High |
| **Cost Efficiency** | High | Moderate | Lower |
| **Late Data Handling** | Simple (reprocess) | Moderate | Complex (watermarks) |
| **State Management** | Not needed | Limited | Required |
| **Typical Tools** | Spark batch, dbt | Spark Streaming | Flink, Kafka Streams |

**Decision Summary**: [See Chapter 9 for detailed guidance]

---

## Consistency Models

| Model | Guarantee | Use Case | Trade-off |
|-------|-----------|----------|-----------|
| **Strong Consistency** | All reads see latest write | Financial transactions | Higher latency, lower availability |
| **Eventual Consistency** | All replicas converge | Social media feeds | Stale reads possible |
| **Causal Consistency** | Respects causality | Chat applications | Moderate complexity |
| **Read-Your-Writes** | See your own writes | User sessions | Per-client guarantee only |

**Decision Summary**: [See Chapter 2 for detailed guidance]

---

## Data Lake vs Lakehouse vs Warehouse

| Characteristic | Data Warehouse | Data Lake | Lakehouse |
|----------------|---------------|-----------|-----------|
| **Data Types** | Structured | All types | All types |
| **Schema** | Schema-on-write | Schema-on-read | Both |
| **ACID** | Yes | No | Yes (via table formats) |
| **Query Performance** | Optimized | Variable | Good (improving) |
| **Storage Cost** | High | Low | Low |
| **Data Quality** | High (enforced) | Variable | High (optional) |
| **Flexibility** | Low | High | High |
| **Governance** | Strong | Weak | Strong |
| **BI Tool Support** | Excellent | Limited | Good |

**Decision Summary**: [See Chapter 3 for detailed guidance]

---

## Data Modeling Approaches Comparison

| Approach | Best For | Complexity | Query Performance | Flexibility | Historical Tracking |
|----------|----------|------------|-------------------|-------------|---------------------|
| **Star Schema** | BI/reporting | Low | Excellent | Low | Limited (SCD required) |
| **Snowflake Schema** | Normalized dimensions | Moderate | Good | Moderate | Limited (SCD required) |
| **Data Vault** | Enterprise DW, multiple sources | High | Lower (requires views) | High | Excellent |
| **One Big Table** | Simple analytics, ML features | Low | Excellent (single table) | Very Low | Limited |
| **Activity Schema** | Customer analytics, journeys | Moderate | Good | Moderate | Built-in |

**Decision Summary**: [See Chapter 13 for detailed guidance]
- **Star Schema**: Default for BI-focused analytics
- **Snowflake Schema**: When dimension normalization matters
- **Data Vault**: Enterprise environments with complex, changing sources
- **One Big Table**: Small teams, ML feature stores, simple use cases
- **Activity Schema**: Customer journey analytics

---

## SCD (Slowly Changing Dimension) Types

| Type | Description | Storage | Query Complexity | Use When |
|------|-------------|---------|------------------|----------|
| **Type 0** | Never changes | Minimal | Simple | Reference data (country codes) |
| **Type 1** | Overwrite | Minimal | Simple | Only current state matters |
| **Type 2** | Add new row | High | Moderate | Full history required |
| **Type 3** | Add column | Moderate | Moderate | Only previous value needed |
| **Type 4** | Mini-dimension | Moderate | Complex | Frequently changing attributes |
| **Type 6** | Hybrid (1+2+3) | High | Complex | Current + history + previous |

**Decision Summary**: [See Chapter 13 for detailed guidance]
- **Type 2**: Most common choice for tracking history
- **Type 1**: When history isn't needed
- **Type 6**: When you need both efficiency and history

---

## Stream Processing Frameworks

| Feature | Apache Flink | Spark Structured Streaming | Kafka Streams |
|---------|--------------|---------------------------|---------------|
| **Processing Model** | True streaming | Micro-batch | True streaming |
| **Latency** | Milliseconds | Seconds | Milliseconds |
| **State Management** | RocksDB, fault-tolerant | In-memory, checkpoint | RocksDB, changelog |
| **Exactly-Once** | Yes | Yes | Yes |
| **SQL Support** | Flink SQL | Spark SQL | KSQL (separate) |
| **Deployment** | Standalone, YARN, K8s | Spark cluster | Embedded (JVM) |
| **Complexity** | High | Moderate | Low-Moderate |
| **Best For** | Low-latency, complex CEP | Unified batch/stream | Kafka-native apps |

**Decision Summary**: [See Chapter 9 for detailed guidance]
- **Flink**: Complex event processing, lowest latency requirements
- **Spark Streaming**: Teams already using Spark, unified batch/stream
- **Kafka Streams**: Lightweight, Kafka-centric applications

---

## Data Quality Tools Comparison

| Feature | Great Expectations | dbt Tests | Soda | Monte Carlo |
|---------|-------------------|-----------|------|-------------|
| **Type** | OSS library | Built into dbt | OSS + Cloud | SaaS platform |
| **Integration** | Python, notebooks | dbt projects | SQL-native | Warehouse-native |
| **Expectation Types** | Extensive library | Schema + custom | SQL-based | Auto-detected |
| **Data Profiling** | Yes | Limited | Yes | Yes (ML-based) |
| **Anomaly Detection** | Manual rules | Manual rules | Basic | Advanced (ML) |
| **Alerting** | Custom setup | CI/CD integration | Built-in | Built-in |
| **Documentation** | Data Docs | dbt docs | Reports | Lineage views |
| **Cost** | Free | Free | Free/Paid | Paid |

**Decision Summary**: [See Chapter 14 for detailed guidance]
- **Great Expectations**: Python-centric teams, comprehensive testing
- **dbt Tests**: Already using dbt, schema-focused testing
- **Soda**: SQL-first teams, multi-platform environments
- **Monte Carlo**: Enterprise, want ML-based anomaly detection

---

## Architecture Pattern Decision Matrix

| Pattern | Team Size | Data Volume | Real-Time Needs | ML Workloads | Best For |
|---------|-----------|-------------|-----------------|--------------|----------|
| **Modern Data Stack** | Small-Medium | TB scale | Minimal | Basic | Rapid iteration, self-service BI |
| **Lakehouse** | Medium-Large | PB scale | Moderate | Extensive | Unified analytics + ML |
| **Streaming-First** | Medium-Large | High velocity | Critical | Real-time features | IoT, fraud detection |
| **Data Mesh** | Large (50+) | Varies by domain | Varies | Varies | Organizational scale |
| **Hybrid** | Any | Mixed | Mixed | Mixed | Pragmatic combinations |

**Decision Summary**: [See Chapter 15 for detailed guidance]
- Start with Modern Data Stack unless you have specific reasons not to
- Move to Lakehouse when ML workloads or unstructured data become significant
- Add streaming capabilities incrementally as real-time needs emerge
- Consider Data Mesh at organizational scale with domain-aligned teams

---

## Cloud Data Warehouse Comparison

| Feature | Snowflake | Google BigQuery | Amazon Redshift | Databricks SQL |
|---------|-----------|-----------------|-----------------|----------------|
| **Architecture** | Shared storage | Serverless | Shared-nothing | Lakehouse |
| **Scaling** | Auto (compute) | Fully automatic | Manual/Serverless | Auto (compute) |
| **Pricing Model** | Compute + storage | Query-based | Instance-based | Compute + storage |
| **Semi-Structured** | VARIANT native | JSON native | Limited | VARIANT (Spark 4) |
| **Time Travel** | 90 days | 7 days | Limited | Unlimited |
| **Data Sharing** | Native | Analytics Hub | Data Exchange | Delta Sharing |
| **Iceberg Support** | Native tables | BigLake | External tables | Native (Delta) |
| **ML Integration** | Snowpark | BigQuery ML | Redshift ML | MLflow native |

**Decision Summary**: [See Chapter 6 for detailed guidance]
- **Snowflake**: Multi-cloud, data sharing, ease of use
- **BigQuery**: GCP-native, serverless, query-based pricing
- **Redshift**: AWS-native, existing AWS investment
- **Databricks SQL**: Already using Databricks, unified lakehouse

---

## Ingestion Pattern Decision Matrix

| Pattern | Data Freshness | Complexity | Resource Usage | Use When |
|---------|---------------|------------|----------------|----------|
| **Full Refresh** | Depends on schedule | Lowest | Highest | Small tables, unknown deltas |
| **Incremental (Timestamp)** | Good | Low | Moderate | Append-only, timestamped data |
| **Incremental (CDC)** | Excellent | Moderate | Low | Updates/deletes matter |
| **Log-based CDC** | Real-time | High | Low | Minimal source impact needed |
| **Streaming** | Real-time | Highest | Continuous | Sub-second freshness required |

**Decision Summary**: [See Chapter 12 for detailed guidance]
- Start with full refresh for simplicity
- Add incremental when performance requires it
- Use CDC when you need to capture updates/deletes
- Reserve streaming for true real-time requirements

---

## dbt Materialization Selection

| Materialization | Rebuild Time | Query Performance | Storage | Use When |
|-----------------|--------------|-------------------|---------|----------|
| **View** | None | Slower (compute on read) | None | Light transformations, always-fresh |
| **Table** | Full rebuild | Fast | Full | Small-medium tables, stable logic |
| **Incremental** | Partial | Fast | Cumulative | Large tables, append-mostly |
| **Ephemeral** | None | N/A (CTE) | None | Reusable logic, no direct query |
| **Materialized View** | Auto-refresh | Fast | Managed | Warehouse-supported, auto-fresh |

**Decision Summary**: [See Chapter 12 for detailed guidance]
- **Default to views** for development and light transformations
- **Use tables** for frequently queried, stable models
- **Use incremental** when full rebuilds become too slow
- **Use ephemeral** for DRY code without creating objects

---

## Compression Algorithm Comparison (Columnar Formats)

| Algorithm | Compression Ratio | Compression Speed | Decompression Speed | Best For |
|-----------|------------------|-------------------|---------------------|----------|
| **Snappy** | Moderate | Very Fast | Very Fast | Default choice, balanced |
| **ZSTD** | High | Fast | Fast | Better compression, modern systems |
| **GZIP** | High | Slow | Moderate | Archival, compatibility |
| **LZ4** | Lower | Very Fast | Very Fast | Speed-critical workloads |
| **Brotli** | Very High | Slow | Fast | Maximum compression |

**Decision Summary**: [See Chapter 4 for detailed guidance]
- **Snappy**: Safe default for most Parquet workloads
- **ZSTD**: Better compression with minimal performance impact
- **LZ4**: When decompression speed is critical
- **GZIP**: Legacy compatibility or archival storage

---

## Key Metrics for Technology Evaluation

| Metric | What It Measures | Target Range | Red Flags |
|--------|------------------|--------------|-----------|
| **Query Latency (P99)** | Worst-case response time | Context-dependent | Orders of magnitude worse than P50 |
| **Throughput** | Records/queries per second | Meet SLA with headroom | Near capacity during normal load |
| **Data Freshness** | Time from event to queryable | Meet business SLA | Inconsistent or growing lag |
| **Cost per Query** | Compute cost per operation | Decreasing over time | Unpredictable or spiking |
| **Pipeline Success Rate** | % of runs completing | >99% | Frequent manual intervention |
| **Time to Recovery** | Restore after failure | Minutes to hours | No clear procedure |

**Decision Summary**: [See Chapter 17 for detailed guidance]

---

## Quick Reference: When to Use What

### Storage Decisions
- **Small data, exploration**: DuckDB + Parquet
- **Analytics workload, single engine**: Warehouse (Snowflake/BigQuery)
- **Multi-engine, enterprise**: Lakehouse with Iceberg
- **High-frequency updates**: Hudi or CDC-optimized pipeline

### Processing Decisions
- **Daily aggregations**: Batch (Spark/dbt)
- **Hourly updates**: Micro-batch (Spark Streaming)
- **Sub-second latency**: Stream (Flink)
- **Real-time + historical**: Lambda/Kappa architecture

### Orchestration Decisions
- **Existing Airflow expertise**: Airflow 3
- **Asset-centric thinking**: Dagster
- **Simplicity, Python-native**: Prefect
- **Complex enterprise**: Airflow with strong conventions

### Modeling Decisions
- **BI dashboards**: Dimensional modeling (star schema)
- **Complex enterprise sources**: Data Vault
- **ML features**: One Big Table or Feature Store
- **Customer analytics**: Activity Schema

---

*Last updated: 2025-01-22*
*Tables: 17*

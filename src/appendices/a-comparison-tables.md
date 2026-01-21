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

*Last updated: 2025-01-21*

# Appendix C: Glossary

*Comprehensive glossary of data engineering terms referenced throughout this book.*

---

## A

**ACID (Atomicity, Consistency, Isolation, Durability)**
A set of properties that guarantee database transactions are processed reliably. In the lakehouse context, table formats like Iceberg and Delta Lake bring ACID semantics to data lakes. [See Chapter 2, Chapter 5]

**Activity Schema**
A data modeling approach where all business events are modeled as activities with a consistent structure, focused on customer journeys. [See Chapter 13]

**Airflow (Apache Airflow)**
An open-source workflow orchestration platform for scheduling and monitoring data pipelines using DAGs. [See Chapter 11]

**Asset**
In orchestration (particularly Dagster), a data artifact that a pipeline produces, with associated metadata and lineage. [See Chapter 11]

## B

**Backfill**
The process of reprocessing historical data, typically after fixing bugs or adding new transformations. [See Chapter 10]

**Batch Processing**
A method of processing data where a finite dataset is processed as a complete unit. Contrast with stream processing. [See Chapter 9]

**Bronze Layer**
In medallion architecture, the raw data layer containing unprocessed data from sources. [See Chapter 15]

**Broadcast Join**
A join strategy where the smaller table is replicated to all nodes, avoiding shuffle of the larger table. [See Chapter 8]

## C

**CAP Theorem**
A fundamental theorem stating that a distributed system can provide at most two of three guarantees: Consistency, Availability, and Partition tolerance. [See Chapter 2]

**Catalyst**
Apache Spark's query optimizer that transforms logical plans into optimized physical execution plans. [See Chapter 8]

**Catchup**
In orchestration, the process of running missed scheduled executions when a DAG is deployed with a historical start date. [See Chapter 10]

**CDC (Change Data Capture)**
Techniques for identifying and capturing changes made to source data for incremental processing.

**Checkpointing**
Periodically saving state in streaming systems to enable recovery from failures. [See Chapter 9]

**ClickHouse**
An open-source columnar OLAP database optimized for real-time analytics queries. [See Chapter 6]

**Columnar Storage**
A data storage format that stores data by columns rather than rows, optimizing for analytical queries that access subsets of columns across many rows. [See Chapter 4]

**Compaction**
The process of combining small files into larger ones to improve query performance. [See Chapter 5]

**Conformed Dimension**
A dimension table shared across multiple fact tables, enabling cross-process analysis. [See Chapter 13]

## D

**DAG (Directed Acyclic Graph)**
A graph with directed edges and no cycles. Used in orchestration tools to represent task dependencies. [See Chapter 10]

**Dagster**
An open-source data orchestrator focused on asset-centric workflows and developer experience. [See Chapter 11]

**Data Contract**
A formal agreement between data producers and consumers specifying schema, semantics, and SLAs. [See Chapter 14]

**Data Lake**
A storage repository that holds vast amounts of raw data in its native format until needed. [See Chapter 3]

**Data Lakehouse**
An architecture that combines the flexibility of data lakes with the reliability and performance of data warehouses. [See Chapter 3, Chapter 15]

**Data Mesh**
An organizational approach where domain teams own and publish data as products on a self-serve platform. [See Chapter 15]

**Data Observability**
The practice of monitoring, tracking, and understanding the health and state of data systems. [See Chapter 14]

**Data Product**
In data mesh, a data asset with associated metadata, quality guarantees, and ownership treated as a product. [See Chapter 15]

**Data Vault**
A data modeling methodology using hubs, links, and satellites to model business entities and relationships. [See Chapter 13]

**Data Warehouse**
A system designed for reporting and data analysis, typically storing structured, curated data. [See Chapter 3]

**dbt (data build tool)**
A transformation framework that enables SQL-based transformations with software engineering practices. [See Chapter 12]

**Delta Lake**
An open-source storage layer that brings ACID transactions to Apache Spark and big data workloads. [See Chapter 5]

**Denormalization**
Intentionally adding redundancy to data models to improve query performance. [See Chapter 13]

**Dimension Table**
In dimensional modeling, a table containing descriptive attributes that provide context for facts. [See Chapter 13]

**DuckDB**
An embedded analytical database optimized for OLAP queries on local data. [See Chapter 6]

## E

**ELT (Extract, Load, Transform)**
A data integration pattern where data is first extracted and loaded into the target system, then transformed. Contrast with ETL. [See Chapter 12]

**ETL (Extract, Transform, Load)**
A data integration pattern where data is transformed before being loaded into the target system.

**Event Time**
The time when an event actually occurred, as recorded in the data itself. Contrast with processing time. [See Chapter 9]

**Eventual Consistency**
A consistency model where, given enough time without updates, all replicas will converge to the same value. [See Chapter 2]

**Exactly-Once Semantics**
Processing guarantee that each event affects the output exactly once, even through failures. [See Chapter 2, Chapter 9]

**Execution Date**
In orchestration, the logical date representing the data interval being processed (not when processing occurs). [See Chapter 10]

## F

**Fact Table**
In dimensional modeling, a table containing measurable events or transactions at a specific grain. [See Chapter 13]

**Flink (Apache Flink)**
A distributed stream processing framework with true per-event processing capabilities. [See Chapter 9]

## G

**Gold Layer**
In medallion architecture, the curated, business-ready data layer optimized for consumption. [See Chapter 15]

**Grain**
The level of detail in a fact table (e.g., one row per transaction vs. one row per day). [See Chapter 13]

**Great Expectations**
An open-source Python library for data validation, documentation, and profiling. [See Chapter 14]

## H

**Hidden Partitioning**
Partitioning where the partition values are computed from data columns transparently, used by Iceberg. [See Chapter 5]

**Hive**
An early data warehouse system built on Hadoop that provided SQL-like querying over distributed data. [See Chapter 3]

**Hub**
In Data Vault, a table containing unique business keys for a business entity. [See Chapter 13]

## I

**Iceberg (Apache Iceberg)**
An open table format for huge analytic datasets, designed to address the shortcomings of Hive tables. [See Chapter 5]

**Idempotency**
A property where an operation produces the same result whether executed once or multiple times. Critical for reliable data pipelines. [See Chapter 2, Chapter 10]

**Incremental Processing**
Processing only new or changed data rather than reprocessing the entire dataset. [See Chapter 12]

## J

**Jinja**
A templating language used by dbt for dynamic SQL generation. [See Chapter 12]

## K

**Kafka (Apache Kafka)**
A distributed event streaming platform for building real-time data pipelines. [See Chapter 9]

**Kappa Architecture**
A data processing architecture that uses a single stream processing engine for both real-time and batch processing. Contrast with Lambda Architecture. [See Chapter 9]

## L

**Lambda Architecture**
A data processing architecture that uses both batch and stream processing to provide comprehensive and accurate views of data. [See Chapter 9]

**Late-Arriving Data**
Data that arrives after its corresponding time window has been processed. [See Chapter 9]

**Lineage**
The record of data's origin and transformations, enabling tracking from source to consumption. [See Chapter 14]

**Link**
In Data Vault, a table representing relationships between hubs. [See Chapter 13]

## M

**Manifest File**
In table formats, a file listing all data files that comprise a table snapshot. [See Chapter 5]

**Materialization**
In dbt, how a model is persisted (table, view, incremental, ephemeral). [See Chapter 12]

**Medallion Architecture**
A data design pattern that organizes data in a lakehouse into bronze (raw), silver (cleaned), and gold (business-level) layers. [See Chapter 15]

**Metadata**
Data that describes other data. In table formats, metadata tracks schema, statistics, partitions, and file locations.

**Metrics Layer**
A semantic layer that defines business metrics consistently for use across multiple tools. [See Chapter 13]

**Microbatch**
Processing small batches of streaming data at regular intervals (dbt 1.9 incremental strategy, Spark Structured Streaming). [See Chapter 9, Chapter 12]

**Model**
In dbt, a SQL SELECT statement that defines a transformation, typically creating a table or view. [See Chapter 12]

**Modern Data Stack**
An architecture pattern using cloud warehouses, ELT, and best-of-breed SaaS tools. [See Chapter 15]

## N

**Narrow Transformation**
In Spark, a transformation where each input partition contributes to at most one output partition (no shuffle required). [See Chapter 7, Chapter 8]

## O

**OLAP (Online Analytical Processing)**
A category of software tools that analyze data stored in a database, optimized for complex queries over large datasets. [See Chapter 6]

**OLTP (Online Transaction Processing)**
A category of data processing focused on transaction-oriented applications, optimized for fast individual record operations.

**One Big Table (OBT)**
A data modeling approach using a single denormalized wide table for analytics. [See Chapter 13]

**Operator**
In orchestration, a template for a task type (e.g., BashOperator, SparkSubmitOperator). [See Chapter 11]

**Orchestration**
The automated arrangement, coordination, and management of data pipeline tasks. [See Chapter 10]

**ORC (Optimized Row Columnar)**
A columnar file format optimized for Hive and large-scale data processing. [See Chapter 4]

## P

**Parquet**
A columnar storage file format optimized for use with big data processing frameworks. [See Chapter 4]

**Partition**
A division of data into smaller, manageable pieces based on a key (e.g., date). Improves query performance by limiting data scanned.

**Partition Evolution**
The ability to change partitioning strategy without rewriting existing data. [See Chapter 5]

**Partition Pruning**
Query optimization that skips partitions not needed for query results.

**Point-in-Time Table (PIT)**
In Data Vault, a table providing efficient access to data state at any historical point. [See Chapter 13]

**Predicate Pushdown**
Query optimization that pushes filters to the data source, reducing data read. [See Chapter 8]

**Prefect**
An open-source workflow orchestration platform with Python-native design and hybrid cloud architecture. [See Chapter 11]

**Processing Time**
The wall-clock time when an event is processed by the system. Contrast with event time. [See Chapter 9]

## Q

**Quality Score**
A metric combining multiple data quality dimensions into a single measure. [See Chapter 14]

**Query Engine**
A system for executing queries against data, potentially across multiple storage systems (e.g., Trino, Presto).

## R

**RDD (Resilient Distributed Dataset)**
Spark's original abstraction for distributed data collections with lineage tracking. [See Chapter 8]

**Ref**
In dbt, a function that creates dependencies between models: `{{ ref('model_name') }}`. [See Chapter 12]

**Row-Oriented Storage**
Storing data by rows, optimizing for transactional workloads that access complete records. Contrast with columnar storage. [See Chapter 4]

## S

**Satellite**
In Data Vault, a table containing descriptive attributes that change over time. [See Chapter 13]

**SCD (Slowly Changing Dimension)**
Techniques for handling dimension changes over time (Type 1: overwrite, Type 2: add row, Type 3: add column). [See Chapter 13]

**Schema Evolution**
The ability to modify a table's schema (add columns, change types) without rewriting existing data. [See Chapter 5]

**Schema-on-Read**
A data processing approach where schema is applied when data is read, not when it is stored. Characteristic of data lakes.

**Schema-on-Write**
A data processing approach where schema is enforced when data is written. Characteristic of traditional databases.

**Semantic Layer**
A business logic layer that defines metrics and dimensions consistently across tools. [See Chapter 13]

**Session Window**
A window that groups events based on activity, ending after a gap of inactivity. [See Chapter 9]

**Shuffle**
In distributed computing, the process of redistributing data across partitions, typically the most expensive operation. [See Chapter 7, Chapter 8]

**Silver Layer**
In medallion architecture, the cleaned and validated data layer. [See Chapter 15]

**Sliding Window**
An overlapping window that advances by a fixed interval. [See Chapter 9]

**Snapshot**
A point-in-time view of data. Table formats use snapshots to enable time travel and rollback. [See Chapter 5]

**Snowflake Schema**
A dimensional model where dimensions are normalized into sub-dimensions. [See Chapter 13]

**Source**
In dbt, a declaration of external tables that models read from. [See Chapter 12]

**Spark (Apache Spark)**
A distributed computing engine for large-scale data processing, supporting batch, streaming, and ML workloads. [See Chapter 8]

**Star Schema**
A dimensional model where denormalized dimensions connect directly to fact tables. [See Chapter 13]

**State**
In streaming, data maintained across events for computations like aggregations or joins. [See Chapter 9]

**Streaming**
Processing data continuously as it arrives, in contrast to batch processing. [See Chapter 9]

**Structured Streaming**
Spark's streaming API that treats streams as unbounded DataFrames. [See Chapter 9]

**Surrogate Key**
An artificial key (often a hash or sequence) used as a primary key instead of natural keys. [See Chapter 13]

## T

**Table Format**
A specification for organizing data files in a data lake to provide database-like features (ACID, schema enforcement). Examples: Iceberg, Delta Lake, Hudi. [See Chapter 5]

**Task**
In orchestration, a unit of work within a pipeline. [See Chapter 10]

**TCO (Total Cost of Ownership)**
The complete cost of a technology including licensing, infrastructure, operations, and opportunity costs. [See Chapter 17]

**Time Travel**
The ability to query historical versions of data, enabled by table formats that maintain snapshots. [See Chapter 5]

**Trino**
A distributed SQL query engine for federated queries across multiple data sources (formerly PrestoSQL). [See Chapter 6]

**TTL (Time to Live)**
Configuration specifying when data or state should expire. [See Chapter 9]

**Tumbling Window**
A non-overlapping fixed-size window for grouping streaming data. [See Chapter 9]

## U

**Upsert**
An operation that inserts new records and updates existing ones, typically using MERGE. [See Chapter 5, Chapter 12]

## V

**VARIANT**
A data type for semi-structured data (JSON), supported natively in Spark 4.0. [See Chapter 8]

**Vectorized Execution**
A query processing technique that processes batches of values at once rather than row-by-row, improving performance through CPU cache utilization.

## W

**Watermark**
In stream processing, a marker indicating that no more events with timestamps earlier than the watermark are expected. [See Chapter 9]

**Wide Transformation**
In Spark, a transformation that requires data shuffling across partitions (e.g., groupBy, join). Contrast with narrow transformation. [See Chapter 7, Chapter 8]

## X

**XCom**
In Airflow, a mechanism for passing data between tasks. [See Chapter 11]

## Y

## Z

**Z-Ordering**
A technique for co-locating related data in storage to improve query performance on multiple columns. [See Chapter 5]

---

*Last updated: 2025-01-22*
*Terms: 120+*

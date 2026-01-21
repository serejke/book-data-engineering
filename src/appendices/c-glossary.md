# Appendix C: Glossary

*This glossary is updated as each chapter is written. Terms are listed alphabetically.*

---

## A

**ACID (Atomicity, Consistency, Isolation, Durability)**
A set of properties that guarantee database transactions are processed reliably. In the lakehouse context, table formats like Iceberg and Delta Lake bring ACID semantics to data lakes.

## B

**Batch Processing**
A method of processing data where a finite dataset is processed as a complete unit. Contrast with stream processing.

## C

**CAP Theorem**
A fundamental theorem stating that a distributed system can provide at most two of three guarantees: Consistency, Availability, and Partition tolerance. [See Chapter 2]

**Columnar Storage**
A data storage format that stores data by columns rather than rows, optimizing for analytical queries that access subsets of columns across many rows. [See Chapter 4]

## D

**DAG (Directed Acyclic Graph)**
A graph with directed edges and no cycles. Used in orchestration tools to represent task dependencies. [See Chapter 10]

**Data Lake**
A storage repository that holds vast amounts of raw data in its native format until needed. [See Chapter 3]

**Data Lakehouse**
An architecture that combines the flexibility of data lakes with the reliability and performance of data warehouses. [See Chapter 3]

**Data Warehouse**
A system designed for reporting and data analysis, typically storing structured, curated data. [See Chapter 3]

**Delta Lake**
An open-source storage layer that brings ACID transactions to Apache Spark and big data workloads. Created by Databricks.

## E

**ELT (Extract, Load, Transform)**
A data integration pattern where data is first extracted and loaded into the target system, then transformed. Contrast with ETL. [See Chapter 12]

**ETL (Extract, Transform, Load)**
A data integration pattern where data is transformed before being loaded into the target system.

**Eventual Consistency**
A consistency model where, given enough time without updates, all replicas will converge to the same value. [See Chapter 2]

## F

## G

## H

**Hive**
An early data warehouse system built on Hadoop that provided SQL-like querying over distributed data. [See Chapter 3]

## I

**Iceberg (Apache Iceberg)**
An open table format for huge analytic datasets, designed to address the shortcomings of Hive tables. [See Chapter 5]

**Idempotency**
A property where an operation produces the same result whether executed once or multiple times. Critical for reliable data pipelines. [See Chapter 2, Chapter 10]

## J

## K

**Kappa Architecture**
A data processing architecture that uses a single stream processing engine for both real-time and batch processing. Contrast with Lambda Architecture. [See Chapter 9]

## L

**Lambda Architecture**
A data processing architecture that uses both batch and stream processing to provide comprehensive and accurate views of data. [See Chapter 9]

## M

**Medallion Architecture**
A data design pattern that organizes data in a lakehouse into bronze (raw), silver (cleaned), and gold (business-level) layers. [See Chapter 15]

**Metadata**
Data that describes other data. In table formats, metadata tracks schema, partitions, and file locations.

## N

## O

**OLAP (Online Analytical Processing)**
A category of software tools that analyze data stored in a database, optimized for complex queries over large datasets. [See Chapter 6]

**OLTP (Online Transaction Processing)**
A category of data processing focused on transaction-oriented applications, optimized for fast individual record operations.

**Orchestration**
The automated arrangement, coordination, and management of data pipeline tasks. [See Chapter 10]

## P

**Parquet**
A columnar storage file format optimized for use with big data processing frameworks. [See Chapter 4]

**Partition**
A division of data into smaller, manageable pieces based on a key (e.g., date). Improves query performance by limiting data scanned.

## Q

## R

## S

**Schema Evolution**
The ability to modify a table's schema (add columns, change types) without rewriting existing data. [See Chapter 5]

**Schema-on-Read**
A data processing approach where schema is applied when data is read, not when it is stored. Characteristic of data lakes.

**Schema-on-Write**
A data processing approach where schema is enforced when data is written. Characteristic of traditional databases.

**Shuffle**
In distributed computing, the process of redistributing data across partitions, typically the most expensive operation. [See Chapter 7, Chapter 8]

**Slowly Changing Dimension (SCD)**
A dimension that stores and manages both current and historical data over time. [See Chapter 13]

**Snapshot**
A point-in-time view of data. Table formats use snapshots to enable time travel and rollback.

**Star Schema**
A database schema consisting of a central fact table surrounded by dimension tables. [See Chapter 13]

**Stream Processing**
A method of processing data where records are processed as they arrive, providing near-real-time results. Contrast with batch processing. [See Chapter 9]

## T

**Table Format**
A specification for organizing data files in a data lake to provide database-like features (ACID, schema enforcement). Examples: Iceberg, Delta Lake, Hudi. [See Chapter 5]

**Time Travel**
The ability to query historical versions of data, enabled by table formats that maintain snapshots.

## U

## V

**Vectorized Execution**
A query processing technique that processes batches of values at once rather than row-by-row, improving performance through CPU cache utilization.

## W

**Watermark**
In stream processing, a marker indicating that no more events with timestamps earlier than the watermark are expected. [See Chapter 9]

**Wide Transformation**
In Spark, a transformation that requires data shuffling across partitions (e.g., groupBy, join). Contrast with narrow transformation. [See Chapter 8]

## X

## Y

## Z

---

*Last updated: 2025-01-21*
*Terms: 45*

---
chapter: 8
title: "Apache Spark Deep Dive"
estimated_pages: 55-60
status: draft
last_updated: 2025-01-22
---

# Apache Spark Deep Dive

You've built the mental model for distributed computing: partitioning, shuffles, narrow vs. wide transformations, lineage-based fault tolerance. These concepts could apply to any distributed system. Now it's time to see how the dominant implementation—Apache Spark—puts them into practice.

Spark isn't just popular because it was first (it wasn't) or because it's backed by a major vendor (Databricks, though many alternatives exist). Spark became the de facto standard for large-scale data processing because it got the abstractions right. It found the sweet spot between low-level control and high-level productivity, between batch and streaming, between data engineers and data scientists.

This chapter takes you deep into Spark: how it works, how to make it fast, and how to integrate it with modern lakehouse components. By the end, you'll understand not just what Spark does, but why it makes the decisions it makes.

## Historical Context: From MapReduce to Spark

To understand Spark's design, you need to understand what it replaced.

### The MapReduce Era

Google's MapReduce (2004) solved the fundamental problem of distributed data processing: how to write reliable programs that run across thousands of machines. Hadoop brought this to the open source world, and by 2010, MapReduce was synonymous with big data.

But MapReduce had significant limitations:

**Disk-bound:** Every MapReduce job wrote intermediate results to disk. A pipeline of three MapReduce jobs meant reading from disk, processing, writing to disk—three times. Interactive analysis was impossible; even simple queries took minutes.

**Rigid programming model:** Map function, shuffle, reduce function. That's it. Complex workflows required chaining multiple jobs, each with its own job setup overhead.

**No iteration support:** Machine learning algorithms that need to iterate over data hundreds of times paid the disk I/O penalty on every iteration.

### Spark's Innovation

Spark, created at UC Berkeley in 2009 and open-sourced in 2010, introduced three key innovations:

**In-memory computing:** Keep intermediate results in memory when possible. A pipeline that would have required three disk round-trips now keeps data in memory between stages.

**Resilient Distributed Datasets (RDDs):** An abstraction for distributed collections with lineage tracking. If a partition is lost, Spark can recompute it from its lineage instead of relying on expensive replication.

**Unified engine:** One system for batch, interactive queries, streaming, machine learning, and graph processing. No more stitching together separate systems for different workloads.

The result was dramatic: workloads that took hours in MapReduce often completed in minutes with Spark. More importantly, Spark made distributed programming accessible to a much broader audience.

## Spark Architecture

Understanding Spark's architecture is essential for writing efficient applications and debugging problems.

### Core Components

```
┌────────────────────────────────────────────────────────┐
│                    Spark Application                   │
├────────────────────────────────────────────────────────┤
│  ┌────────────┐                                        │
│  │   Driver   │  - Runs main() function                │
│  │  Program   │  - Creates SparkSession               │
│  │            │  - Builds execution plan              │
│  │            │  - Coordinates executors              │
│  └─────┬──────┘                                        │
│        │                                               │
│        │  Cluster Manager (YARN, K8s, Standalone)     │
│        │                                               │
│  ┌─────▼──────┐  ┌────────────┐  ┌────────────┐      │
│  │  Executor  │  │  Executor  │  │  Executor  │      │
│  │            │  │            │  │            │      │
│  │ ┌──┐ ┌──┐ │  │ ┌──┐ ┌──┐ │  │ ┌──┐ ┌──┐ │      │
│  │ │T1│ │T2│ │  │ │T3│ │T4│ │  │ │T5│ │T6│ │      │
│  │ └──┘ └──┘ │  │ └──┘ └──┘ │  │ └──┘ └──┘ │      │
│  │  Cache    │  │  Cache    │  │  Cache    │      │
│  └───────────┘  └───────────┘  └───────────┘      │
└────────────────────────────────────────────────────────┘
```

**Driver:** The brain of the Spark application. It:
- Runs your main program
- Creates the SparkSession (entry point to Spark functionality)
- Converts your code into an execution plan
- Schedules tasks on executors
- Collects results

**Executors:** Worker processes that run on cluster nodes. They:
- Execute tasks assigned by the driver
- Store data in memory or disk for caching
- Report status back to the driver
- Run for the lifetime of the application

**Cluster Manager:** Allocates resources across the cluster. Spark supports:
- **Standalone:** Spark's built-in cluster manager (good for dedicated clusters)
- **YARN:** Hadoop's resource manager (good for shared Hadoop clusters)
- **Kubernetes:** Container orchestration (increasingly popular in cloud deployments)
- **Mesos:** General-purpose cluster manager (less common now)

### The SparkSession

SparkSession is your entry point to all Spark functionality:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("TradingAnalytics") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog") \
    .getOrCreate()

# Now use spark to create DataFrames, run SQL, etc.
df = spark.read.parquet("s3://data/trades/")
```

SparkSession unifies what used to be separate entry points: SparkContext (RDD operations), SQLContext (SQL operations), and HiveContext (Hive integration). Modern Spark code should always use SparkSession.

## The API Evolution: RDDs → DataFrames → Datasets

Spark's APIs have evolved significantly, each generation improving productivity and performance.

### RDDs: The Foundation

Resilient Distributed Datasets (RDDs) are Spark's original abstraction: an immutable, distributed collection of objects.

```python
# RDD example (legacy style)
rdd = spark.sparkContext.textFile("trades.csv")
parsed = rdd.map(lambda line: line.split(","))
filtered = parsed.filter(lambda fields: fields[3] == "FILLED")
counts = filtered.map(lambda f: (f[1], 1)).reduceByKey(lambda a, b: a + b)
```

RDDs provide:
- **Fine-grained control:** You specify exactly how to transform data
- **Type safety:** Operations work on typed objects (in Scala/Java)
- **Flexibility:** Can hold any serializable object

But RDDs have significant drawbacks:
- **No optimization:** Spark executes exactly what you write, even if inefficient
- **Verbose:** Simple operations require lots of code
- **Python performance:** Python RDDs serialize data between JVM and Python, making Python significantly slower than Scala

### DataFrames: Structured Data Abstraction

DataFrames, introduced in Spark 1.3, brought SQL-like operations to Spark:

```python
# DataFrame example (modern style)
df = spark.read.parquet("trades/")
result = (df
    .filter(df.status == "FILLED")
    .groupBy("symbol")
    .count())
```

DataFrames provide:
- **Optimizer integration:** Spark's Catalyst optimizer can rewrite and optimize queries
- **Performance parity:** Python DataFrames match Scala performance (no Python serialization)
- **Schema enforcement:** Data has known types, enabling optimizations
- **SQL compatibility:** Write SQL queries directly against DataFrames

DataFrames are essentially RDDs of Row objects with a known schema, but that schema enables transformative optimizations.

### Datasets: Type-Safe DataFrames

Datasets (Scala/Java only) combine DataFrame optimization with compile-time type safety:

```scala
// Dataset example (Scala)
case class Trade(symbol: String, price: Double, quantity: Int, status: String)

val trades: Dataset[Trade] = spark.read.parquet("trades/").as[Trade]
val filled = trades.filter(_.status == "FILLED")
```

Datasets catch errors at compile time rather than runtime. However, they're not available in Python, and for most workloads, DataFrames provide sufficient safety through schema validation.

### Which API to Use

**Use DataFrames (recommended):**
- 95% of use cases
- Best performance through optimizer
- Consistent across Python, Scala, Java, R
- Full SQL compatibility

**Use RDDs when:**
- You need fine-grained control over physical execution
- Working with unstructured data that doesn't fit a schema
- Implementing custom low-level transformations

**Use Datasets when:**
- Scala/Java codebase requiring compile-time type safety
- Complex domain objects with methods
- Willing to trade some flexibility for type safety

In practice, modern Spark development centers on DataFrames with Spark SQL.

## The Catalyst Optimizer

Catalyst is Spark's query optimizer—the engine that transforms your logical operations into efficient physical execution. Understanding Catalyst helps you write better queries and debug performance issues.

### Optimization Phases

Catalyst transforms queries through four phases:

```
User Code
    │
    ▼
┌─────────────────┐
│ Analysis        │  Resolve table names, columns, types
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Logical         │  Apply rewrite rules:
│ Optimization    │  - Predicate pushdown
└────────┬────────┘  - Constant folding
         │           - Column pruning
         ▼
┌─────────────────┐
│ Physical        │  Generate physical plans
│ Planning        │  Choose best plan (cost-based)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Code            │  Generate optimized JVM bytecode
│ Generation      │  (Whole-stage code generation)
└────────┬────────┘
         │
         ▼
    Execution
```

### Key Optimizations

**Predicate Pushdown:** Move filters as early as possible, ideally to the data source.

```python
# You write:
df = spark.read.parquet("trades/")
result = df.filter(df.date == "2024-01-15").select("symbol", "price")

# Catalyst rewrites to:
# - Only read date=2024-01-15 partition
# - Only read symbol, price columns from Parquet
```

This is particularly powerful with columnar formats like Parquet—Spark may read only 1% of the data if filters are selective.

**Column Pruning:** Only read columns actually needed for the query.

```python
# If you only select two columns from a 100-column table:
df.select("symbol", "price")

# Spark reads only those 2 columns from Parquet, not all 100
```

**Constant Folding:** Evaluate constant expressions at planning time.

```python
# You write:
df.filter(df.quantity > 10 * 100)

# Catalyst rewrites to:
df.filter(df.quantity > 1000)  # Computed once, not for every row
```

**Join Reordering:** Rearrange joins to minimize intermediate data size.

```python
# Three-way join—Catalyst may reorder based on table sizes
result = large_table.join(medium_table, "key1").join(small_table, "key2")

# Catalyst might execute as:
# (small_table ⋈ medium_table) ⋈ large_table
# to minimize shuffle volume
```

**Broadcast Join Selection:** Automatically choose broadcast joins for small tables.

### The Explain Plan

To see what Catalyst does with your query, use `explain()`:

```python
df = spark.read.parquet("trades/")
result = df.filter(df.symbol == "BTC").groupBy("date").agg({"price": "avg"})

result.explain(mode="extended")
# Shows: parsed logical plan, analyzed plan, optimized plan, physical plan
```

Understanding explain plans is essential for debugging slow queries. Look for:
- **Shuffles:** Exchange operators indicate data movement
- **Full scans:** FileScan without pushed filters means reading all data
- **Broadcast:** BroadcastHashJoin is usually good; SortMergeJoin may indicate a missed optimization

## Spark SQL

Spark SQL lets you query DataFrames using SQL syntax. It's not just syntactic sugar—SQL queries go through the same Catalyst optimizer as DataFrame operations.

### Using SQL

```python
# Register DataFrame as a temporary view
df.createOrReplaceTempView("trades")

# Query with SQL
result = spark.sql("""
    SELECT symbol,
           date,
           AVG(price) as avg_price,
           SUM(quantity) as total_volume
    FROM trades
    WHERE status = 'FILLED'
      AND date >= '2024-01-01'
    GROUP BY symbol, date
    ORDER BY total_volume DESC
""")
```

### SQL vs DataFrame API

The choice between SQL and DataFrame API is often stylistic:

```python
# SQL style
result = spark.sql("""
    SELECT symbol, AVG(price)
    FROM trades
    WHERE status = 'FILLED'
    GROUP BY symbol
""")

# DataFrame style (equivalent)
result = (trades
    .filter(col("status") == "FILLED")
    .groupBy("symbol")
    .agg(avg("price")))
```

Both produce identical execution plans. Choose based on:
- **Team familiarity:** SQL is more accessible to analysts
- **Dynamic queries:** DataFrame API is easier to build programmatically
- **Complex logic:** DataFrame API handles complex transformations better
- **Debugging:** DataFrame operations can be broken into inspectable steps

### ANSI SQL Mode (Spark 4.0)

Spark 4.0 enables ANSI SQL mode by default (`spark.sql.ansi.enabled = true`). This changes behavior significantly:

**Before (non-ANSI):**
```sql
SELECT 1/0  -- Returns NULL
SELECT CAST('abc' AS INT)  -- Returns NULL
```

**After (ANSI):**
```sql
SELECT 1/0  -- Throws ArithmeticException
SELECT CAST('abc' AS INT)  -- Throws NumberFormatException
```

ANSI mode prevents silent data quality issues. Invalid operations fail loudly rather than producing NULLs that propagate through your pipeline. This is a breaking change—test thoroughly when upgrading.

## Execution Model: Jobs, Stages, Tasks

Understanding Spark's execution model helps you interpret the Spark UI and optimize performance.

### The Hierarchy

```
Application
    │
    ├── Job 1 (triggered by action: count(), write(), etc.)
    │   ├── Stage 1 (narrow transformations before shuffle)
    │   │   ├── Task 1.1 (process partition 1)
    │   │   ├── Task 1.2 (process partition 2)
    │   │   └── Task 1.3 (process partition 3)
    │   │
    │   └── Stage 2 (after shuffle)
    │       ├── Task 2.1
    │       └── Task 2.2
    │
    └── Job 2
        └── ...
```

**Application:** One SparkSession, runs for the duration of your program.

**Job:** Triggered by an action (write, count, collect, show). Each action creates a new job.

**Stage:** Separated by shuffles. Within a stage, all transformations are narrow and can be pipelined.

**Task:** One task per partition per stage. The actual unit of work executed on an executor.

### How Spark Builds Execution Plans

When you call an action, Spark:

1. **Builds the DAG:** Traces back through transformations to identify dependencies
2. **Identifies shuffle boundaries:** Wide transformations create new stages
3. **Creates tasks:** One task per partition for each stage
4. **Schedules tasks:** Assigns tasks to executors, respecting locality preferences
5. **Executes:** Runs tasks, handles failures, collects results

```python
# Example transformation chain
df = spark.read.parquet("trades/")  # Stage 1 starts
filtered = df.filter(df.status == "FILLED")  # Still Stage 1 (narrow)
mapped = filtered.select("symbol", "price")  # Still Stage 1 (narrow)
grouped = mapped.groupBy("symbol").avg("price")  # Stage 2 starts (shuffle)
result = grouped.filter(col("avg(price)") > 100)  # Still Stage 2 (narrow)
result.write.parquet("output/")  # Triggers execution
```

### Reading the Spark UI

The Spark UI (typically at http://driver-host:4040) shows:
- **Jobs tab:** Overall progress of jobs
- **Stages tab:** Details of each stage, including shuffle reads/writes
- **Storage tab:** Cached RDDs/DataFrames
- **Executors tab:** Resource utilization per executor
- **SQL tab:** Query plans and execution metrics

Key metrics to watch:
- **Shuffle read/write:** High shuffle volumes indicate expensive operations
- **Task duration variance:** Large variance suggests data skew
- **Spill (memory and disk):** Data spilling to disk indicates memory pressure
- **GC time:** High GC overhead indicates memory issues or suboptimal serialization

## Performance Optimization

Spark applications can vary by orders of magnitude in performance based on how they're written and configured. This section covers the most impactful optimizations.

### Partitioning Strategies

Partitioning determines parallelism and data distribution—get it wrong, and performance suffers dramatically.

**Reading data:**
```python
# Parquet files determine initial partitioning
df = spark.read.parquet("s3://data/trades/")

# Check partition count
df.rdd.getNumPartitions()  # e.g., 200 partitions
```

**Repartitioning:**
```python
# Increase partitions (for more parallelism)
df = df.repartition(400)  # Triggers full shuffle

# Decrease partitions (coalesce is cheaper than repartition)
df = df.coalesce(100)  # No shuffle, combines partitions locally

# Partition by key (for optimized joins/aggregations)
df = df.repartition(200, "symbol")  # Shuffle by symbol
```

**Partition count guidelines:**
- Target 128 MB - 1 GB per partition
- 2-4 partitions per available CPU core
- More partitions = more parallelism but more scheduling overhead
- Fewer partitions = less overhead but potential skew issues

### Broadcast Joins

When joining a large table with a small table, broadcast the small table:

```python
from pyspark.sql.functions import broadcast

# Explicit broadcast
result = large_df.join(broadcast(small_df), "key")

# Automatic broadcast (default threshold: 10 MB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 100 * 1024 * 1024)  # 100 MB
```

Broadcast joins eliminate shuffles on the large table—often 10x or more improvement for join-heavy workloads.

**When to broadcast:**
- Dimension tables (products, users, symbols)
- Lookup tables
- Any table small enough to fit in executor memory

**When NOT to broadcast:**
- Tables too large for memory
- When both sides are large
- When broadcast overhead exceeds shuffle savings

### Caching and Persistence

If you use a DataFrame multiple times, cache it to avoid recomputation:

```python
# Cache in memory
df.cache()  # Equivalent to persist(StorageLevel.MEMORY_ONLY)

# With disk fallback
df.persist(StorageLevel.MEMORY_AND_DISK)

# Force materialization
df.count()  # First action triggers computation and caching

# Use cached data
df.filter(col("symbol") == "BTC").count()  # Uses cache
df.filter(col("symbol") == "ETH").count()  # Also uses cache

# Release when done
df.unpersist()
```

**Storage levels:**
- `MEMORY_ONLY`: Fast but may not fit
- `MEMORY_AND_DISK`: Spills to disk if memory is full
- `MEMORY_ONLY_SER`: Serialized, uses less memory but more CPU
- `DISK_ONLY`: For very large datasets

**When to cache:**
- DataFrame used multiple times
- After expensive computation
- Before iterative algorithms

**When NOT to cache:**
- DataFrame used only once
- When memory is constrained
- When disk I/O is fast enough (e.g., local SSD)

### Handling Data Skew

Data skew—when some partitions are much larger than others—causes stragglers that delay the entire job.

**Detecting skew:**
```python
# Check partition sizes
from pyspark.sql.functions import spark_partition_id

df.groupBy(spark_partition_id()).count().show()

# In Spark UI: look for tasks that take much longer than others
```

**Salting technique:**
```python
from pyspark.sql.functions import rand, concat, lit

# Problem: groupBy on skewed key
df.groupBy("symbol").agg(sum("quantity"))

# Solution: add random salt, aggregate twice
num_salts = 10
salted = df.withColumn("salt", (rand() * num_salts).cast("int"))
         .withColumn("salted_key", concat("symbol", lit("_"), "salt"))

# First aggregation: by salted key (distributes hot keys)
partial = salted.groupBy("salted_key", "symbol").agg(sum("quantity").alias("partial_sum"))

# Second aggregation: combine partial results
result = partial.groupBy("symbol").agg(sum("partial_sum").alias("total_quantity"))
```

**Adaptive Query Execution (AQE):**

Spark 3.0+ includes Adaptive Query Execution, which handles many skew scenarios automatically:

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")  # Default in Spark 3.0+
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

AQE detects skew at runtime and splits large partitions automatically. It can also:
- Coalesce small partitions after shuffle
- Dynamically switch join strategies
- Optimize shuffle partition count

### Memory Configuration

Executor memory is divided into regions:

```
Executor Memory
├── Execution Memory (shuffle, join, sort buffers)
├── Storage Memory (cached data)
├── User Memory (user data structures)
└── Reserved Memory (Spark internals)
```

Key settings:
```python
# Total executor memory
spark.conf.set("spark.executor.memory", "8g")

# Memory fraction for execution + storage (default 0.6)
spark.conf.set("spark.memory.fraction", "0.6")

# Split between execution and storage (default 0.5)
spark.conf.set("spark.memory.storageFraction", "0.5")

# Off-heap memory for columnar operations
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "2g")
```

**Memory tuning guidelines:**
- Start with defaults, tune based on Spark UI metrics
- If seeing spills, increase executor memory or reduce partition size
- If seeing GC overhead, consider off-heap memory
- If caching frequently, increase storage fraction

### Column Pruning and Filter Pushdown

Help the optimizer by being explicit about what you need:

```python
# Good: select only needed columns early
df = spark.read.parquet("trades/")
result = (df
    .select("symbol", "price", "date", "status")  # Prune columns first
    .filter(col("status") == "FILLED")
    .filter(col("date") == "2024-01-15")
    .groupBy("symbol")
    .agg(avg("price")))

# With Iceberg/Delta: filters push down to skip files
df = (spark.read
    .format("iceberg")
    .load("catalog.db.trades")
    .filter(col("date") >= "2024-01-01")  # Skips files outside date range
    .filter(col("symbol") == "BTC"))  # May skip more files if partitioned
```

### Avoiding Common Anti-Patterns

**Anti-pattern: Collecting large results to driver**
```python
# Bad: collects entire dataset to driver memory
all_data = df.collect()  # OutOfMemoryError on large data

# Better: aggregate first, then collect small result
summary = df.groupBy("symbol").count().collect()

# Or write to storage
df.write.parquet("output/")
```

**Anti-pattern: UDFs when SQL functions exist**
```python
# Bad: Python UDF is slow
from pyspark.sql.functions import udf
from pyspark.sql.types import DoubleType

@udf(returnType=DoubleType())
def calculate_vwap(price, quantity, total_quantity):
    return price * quantity / total_quantity

# Better: use built-in functions
from pyspark.sql.functions import col, sum
df.withColumn("vwap", col("price") * col("quantity") / col("total_quantity"))
```

Python UDFs serialize data between JVM and Python—avoid them when equivalent SQL functions exist.

**Anti-pattern: Unnecessary shuffles**
```python
# Bad: repartition before write when not needed
df.repartition(100).write.parquet("output/")

# Better: coalesce (no shuffle) if reducing partitions
df.coalesce(100).write.parquet("output/")

# Or let Spark manage with AQE
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

## Spark 4.0 Features

Spark 4.0, released in 2025, brings significant improvements. Understanding these features helps you leverage the latest capabilities.

### VARIANT Data Type

The VARIANT type stores semi-structured data (JSON) in optimized binary format:

```python
# Create table with VARIANT column
spark.sql("""
    CREATE TABLE events (
        event_id STRING,
        event_time TIMESTAMP,
        payload VARIANT
    )
""")

# Query nested JSON efficiently
spark.sql("""
    SELECT
        event_id,
        payload:user.id AS user_id,
        payload:action AS action,
        payload:metadata.source AS source
    FROM events
    WHERE payload:action = 'trade'
""")
```

VARIANT provides up to 8x faster query performance compared to storing JSON as strings because:
- Binary format eliminates repeated parsing
- Direct field access without runtime JSON parsing
- Optimized storage and compression

This is particularly valuable for event data, logs, and API responses where schemas vary.

### Enhanced Structured Streaming

**TransformWithState:** A new stateful processing operator for complex streaming logic:

```python
# Define stateful transformation class
class SessionTracker:
    def __init__(self):
        self.state = {}

    def process(self, key, events, state):
        # Access and update state
        session = state.get("session") or Session()
        for event in events:
            session.add_event(event)
        state.set("session", session)

        # TTL support: expire state after period of inactivity
        state.set_ttl(timedelta(hours=1))

        return session.to_output()

# Apply to stream
df.groupByKey(col("user_id")).transformWithState(SessionTracker())
```

TransformWithState supports:
- Object-oriented logic definition
- Composite state types
- Timers and TTL
- Initial state handling
- State schema evolution

**State Store Data Source:** Query streaming state as a table:

```python
# Read state from a streaming query
state_df = spark.read.format("statestore").load("/checkpoint/state")
state_df.show()  # Inspect current state values
```

This enables debugging, auditing, and analytics on streaming state.

### Python Enhancements

**Python Data Source API:** Create custom data sources in Python:

```python
class TradingDataSource(DataSource):
    def reader(self, schema):
        return TradingDataSourceReader(schema, self.options)

class TradingDataSourceReader(DataSourceReader):
    def read(self, partition):
        # Custom logic to fetch trading data
        yield from fetch_trades(partition)

# Use custom source
df = spark.read.format("trading").option("exchange", "binance").load()
```

**Pandas 2.x Support:** Full compatibility with modern pandas, leveraging Apache Arrow for efficient conversion:

```python
# Convert Spark DataFrame to pandas 2.x DataFrame
pandas_df = spark_df.toPandas()

# Create Spark DataFrame from pandas
spark_df = spark.createDataFrame(pandas_df)
```

### Improved Error Handling

Spark 4.0 introduces an Error Class Framework with standardized error messages:

```
Before: java.lang.NullPointerException: null
After: [DIVIDE_BY_ZERO] Division by zero. Use `try_divide` to tolerate divisor being 0 and return NULL instead.
```

Errors now include:
- Error class identifier
- Human-readable message
- Suggested fixes
- Links to documentation

This makes debugging significantly easier.

## Spark with Lakehouse Components

Modern Spark applications integrate with lakehouse components—table formats, catalogs, and storage systems.

### Spark + Apache Iceberg

Iceberg integration provides ACID transactions, schema evolution, and time travel:

```python
# Configure Iceberg catalog
spark = SparkSession.builder \
    .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.iceberg.type", "hive") \
    .config("spark.sql.catalog.iceberg.uri", "thrift://metastore:9083") \
    .getOrCreate()

# Create Iceberg table
spark.sql("""
    CREATE TABLE iceberg.trading.trades (
        trade_id STRING,
        symbol STRING,
        price DECIMAL(18, 8),
        quantity DECIMAL(18, 8),
        timestamp TIMESTAMP
    )
    USING iceberg
    PARTITIONED BY (days(timestamp))
""")

# Write to Iceberg
df.writeTo("iceberg.trading.trades").append()

# Time travel query
spark.read.option("as-of-timestamp", "2024-01-15 00:00:00").table("iceberg.trading.trades")

# Schema evolution
spark.sql("ALTER TABLE iceberg.trading.trades ADD COLUMN source STRING")
```

**Key Iceberg features in Spark:**
- Hidden partitioning (partition transforms like `days()`, `hours()`)
- Partition evolution without rewriting data
- Snapshot isolation for concurrent reads/writes
- MERGE INTO for upserts

See Chapter 5 for deep coverage of Iceberg architecture.

### Spark + Delta Lake

Delta Lake provides similar capabilities with tighter Spark integration:

```python
# Write as Delta
df.write.format("delta").mode("overwrite").save("s3://data/trades_delta")

# Create managed table
spark.sql("""
    CREATE TABLE trading.trades (
        trade_id STRING,
        symbol STRING,
        price DECIMAL(18, 8),
        quantity DECIMAL(18, 8),
        timestamp TIMESTAMP
    )
    USING delta
    PARTITIONED BY (date)
""")

# MERGE for upserts
spark.sql("""
    MERGE INTO trading.trades AS target
    USING updates AS source
    ON target.trade_id = source.trade_id
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *
""")

# Time travel
spark.read.format("delta").option("versionAsOf", 5).load("s3://data/trades_delta")
```

**Delta-specific features:**
- Z-ordering for multi-column clustering
- Liquid clustering (Delta 3.0+) for automatic data layout
- Change Data Feed for incremental processing
- Unity Catalog integration

### Catalog Configuration

A typical lakehouse setup uses a central metastore:

```python
spark = SparkSession.builder \
    .appName("LakehouseAnalytics") \

    # Hive metastore for legacy tables
    .config("spark.sql.warehouse.dir", "s3://warehouse/") \
    .config("hive.metastore.uris", "thrift://metastore:9083") \

    # Iceberg catalog
    .config("spark.sql.catalog.iceberg_prod", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.iceberg_prod.type", "hive") \

    # Delta catalog
    .config("spark.sql.catalog.delta_prod", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \

    # Enable extensions
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,io.delta.sql.DeltaSparkSessionExtension") \

    .enableHiveSupport() \
    .getOrCreate()
```

With this configuration, you can query:
```sql
SELECT * FROM iceberg_prod.trading.trades;  -- Iceberg table
SELECT * FROM delta_prod.analytics.metrics; -- Delta table
SELECT * FROM legacy_db.old_table;          -- Hive table
```

## Case Study: Optimizing a Slow Spark Job

Let's apply these concepts to a real optimization scenario.

### The Problem

A daily aggregation job processes trading data to compute per-symbol metrics:

```python
# Original job: takes 4 hours
trades = spark.read.parquet("s3://data/trades/")  # 5 TB, 50,000 files

# Join with symbol metadata
symbols = spark.read.parquet("s3://data/symbols/")  # 100 MB

result = (trades
    .join(symbols, "symbol")
    .filter(col("status") == "FILLED")
    .filter(col("date") == target_date)
    .groupBy("symbol", "exchange")
    .agg(
        count("*").alias("trade_count"),
        sum("quantity").alias("total_volume"),
        avg("price").alias("avg_price"),
        max("price").alias("high"),
        min("price").alias("low")
    ))

result.write.mode("overwrite").parquet("s3://data/daily_metrics/")
```

### Investigation

Checking the Spark UI reveals:
1. **File scan takes 90 minutes:** Reading all 50,000 files
2. **Shuffle write: 800 GB:** The join shuffles both datasets
3. **Task skew:** Some symbols have 100x more trades than others
4. **Total tasks: 50,000:** One per input file

### Optimization 1: Filter Pushdown

Move the date filter before the join, and ensure it's pushed to file scan:

```python
# Filter early—reduces data before join
trades = (spark.read.parquet("s3://data/trades/")
    .filter(col("date") == target_date)  # If partitioned by date, reads only one partition
    .filter(col("status") == "FILLED"))  # Pushed to Parquet (column statistics)
```

**Result:** File scan drops from 90 minutes to 5 minutes (reads only relevant partition).

### Optimization 2: Broadcast Join

The symbols table is small—broadcast it:

```python
result = (trades
    .join(broadcast(symbols), "symbol")  # No shuffle on trades
    ...)
```

**Result:** Shuffle write drops from 800 GB to near zero for the join.

### Optimization 3: Handle Skew

Some symbols (BTC, ETH) have far more trades. Use salting:

```python
num_salts = 10
trades_salted = (trades
    .withColumn("salt", (rand() * num_salts).cast("int"))
    .withColumn("group_key", concat("symbol", lit("_"), "exchange", lit("_"), "salt")))

partial = (trades_salted
    .groupBy("group_key", "symbol", "exchange")
    .agg(
        count("*").alias("partial_count"),
        sum("quantity").alias("partial_volume"),
        sum(col("price") * col("quantity")).alias("partial_pq"),  # For weighted avg
        max("price").alias("partial_high"),
        min("price").alias("partial_low")))

result = (partial
    .groupBy("symbol", "exchange")
    .agg(
        sum("partial_count").alias("trade_count"),
        sum("partial_volume").alias("total_volume"),
        sum("partial_pq") / sum("partial_volume")).alias("avg_price"),
        max("partial_high").alias("high"),
        min("partial_low").alias("low")))
```

**Result:** No single task takes more than 2x the average.

### Optimization 4: Tune Partitions

With fewer input files after filtering, reduce task overhead:

```python
# Coalesce to reasonable partition count
trades = trades.coalesce(200)  # Match cluster parallelism
```

**Result:** Task scheduling overhead drops significantly.

### Final Result

```python
# Optimized job: 15 minutes (vs. 4 hours originally)
trades = (spark.read.parquet("s3://data/trades/")
    .filter(col("date") == target_date)
    .filter(col("status") == "FILLED")
    .coalesce(200))

symbols = spark.read.parquet("s3://data/symbols/").cache()

# Salting for skew
num_salts = 10
trades_salted = (trades
    .withColumn("salt", (rand() * num_salts).cast("int"))
    .withColumn("group_key", concat("symbol", lit("_"), "exchange", lit("_"), "salt")))

# Join with broadcast
enriched = trades_salted.join(broadcast(symbols), "symbol")

# Two-stage aggregation
partial = (enriched
    .groupBy("group_key", "symbol", "exchange")
    .agg(
        count("*").alias("pc"),
        sum("quantity").alias("pv"),
        sum(col("price") * col("quantity")).alias("ppq"),
        max("price").alias("ph"),
        min("price").alias("pl")))

result = (partial
    .groupBy("symbol", "exchange")
    .agg(
        sum("pc").alias("trade_count"),
        sum("pv").alias("total_volume"),
        (sum("ppq") / sum("pv")).alias("avg_price"),
        max("ph").alias("high"),
        min("pl").alias("low")))

result.write.mode("overwrite").parquet("s3://data/daily_metrics/")
```

**Improvement: 16x faster** (4 hours → 15 minutes)

Key lessons:
1. Filter early to reduce data volume
2. Broadcast small tables
3. Handle skew explicitly when AQE isn't enough
4. Match partition count to workload

## Summary

This chapter covered Apache Spark in depth:

**Architecture:**
- Driver coordinates, executors execute
- SparkSession is your entry point
- Cluster managers (YARN, K8s, Standalone) allocate resources

**API evolution:**
- RDDs: low-level, flexible, no optimization
- DataFrames: optimized, schema-aware, cross-language
- Datasets: type-safe (Scala/Java only)
- Use DataFrames for 95% of work

**Catalyst optimizer:**
- Transforms logical plans to optimized physical plans
- Predicate pushdown, column pruning, join reordering
- Use `explain()` to understand query plans

**Execution model:**
- Jobs (triggered by actions) → Stages (separated by shuffles) → Tasks (per partition)
- Stages pipeline narrow transformations
- Shuffles are expensive—minimize them

**Performance optimization:**
- Partition wisely (128 MB - 1 GB, 2-4 per core)
- Broadcast small tables
- Cache reused DataFrames
- Handle skew with salting or AQE
- Prune columns and push filters early

**Spark 4.0:**
- VARIANT type for semi-structured data
- TransformWithState for complex streaming
- ANSI SQL mode by default
- Improved error messages

**Lakehouse integration:**
- Iceberg: hidden partitioning, time travel, schema evolution
- Delta: MERGE, Z-ordering, Change Data Feed
- Configure catalogs for unified access

**Key insight:** Spark performance is about minimizing data movement. Every shuffle, every full scan, every Python UDF adds overhead. The Catalyst optimizer helps, but understanding how Spark executes your code lets you write queries that are fast by design.

**Looking ahead:**
- Chapter 9 explores batch vs. streaming, including Spark Structured Streaming
- Chapter 11 covers orchestrating Spark jobs with Airflow, Prefect, and Dagster
- Chapter 15 shows Spark's role in lakehouse architectures

## Further Reading

- Chambers, B. & Zaharia, M. (2018). *Spark: The Definitive Guide* — Comprehensive Spark reference
- Karau, H. & Warren, R. (2017). *High Performance Spark* — Advanced optimization techniques
- [Apache Spark Documentation](https://spark.apache.org/docs/latest/) — Official reference
- [Databricks Blog: Introducing Apache Spark 4.0](https://www.databricks.com/blog/introducing-apache-spark-40) — Spark 4.0 features
- Armbrust, M. et al. (2015). "Spark SQL: Relational Data Processing in Spark" — The Spark SQL paper
- [The Internals of Apache Spark](https://books.japila.pl/spark-sql-internals/) — Deep dive into Spark internals

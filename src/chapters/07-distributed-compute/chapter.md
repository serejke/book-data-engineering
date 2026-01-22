---
chapter: 7
title: "Distributed Compute Fundamentals"
estimated_pages: 30-35
status: draft
last_updated: 2025-01-22
---

# Distributed Compute Fundamentals

There's a moment in every data engineer's career when a job that used to finish in an hour starts taking eight hours, then doesn't finish at all. The data grew, but the machine didn't. You can buy a bigger machine, but eventually you hit a ceiling—no single server can process a petabyte of data in reasonable time. The only way forward is to split the work across multiple machines.

Distributed computing is the art of coordinating many machines to solve problems that no single machine could handle alone. It's how Google processes the entire web, how Netflix analyzes billions of streaming events, and how financial firms backtest trading strategies across decades of tick data.

But distributed computing is hard. Networks fail. Machines crash. Data gets shuffled across the network, and the network is slow. This chapter builds the mental model for understanding distributed data processing: why we need it, how it works, and what makes it difficult. This foundation is essential for understanding Spark (Chapter 8) and designing efficient pipelines.

## Why Distributed?

Before diving into how distributed computing works, let's understand when it's necessary.

### The Single-Machine Ceiling

A modern server can be powerful:
- 256+ CPU cores
- 1-4 TB RAM
- 100+ TB local SSD storage
- 100 Gbps network

For many workloads, this is enough. A well-optimized single-machine job can process:
- 100 GB in minutes
- 1 TB in tens of minutes
- 10 TB in hours (if it fits on disk)

But single machines have hard limits:
- **RAM ceiling:** Can't process data larger than memory without complex out-of-core algorithms
- **CPU ceiling:** Processing time grows linearly with data; 100 TB takes 10x longer than 10 TB
- **Storage ceiling:** Local disk has finite capacity
- **Availability:** One machine = one point of failure

### When to Distribute

Distribute when:

**Data exceeds single-machine capacity:** The classic case. If your dataset is 500 TB, no single machine can store it, let alone process it.

**Processing time is unacceptable:** Even if data fits, processing might be too slow. A job that takes 24 hours on one machine might take 1 hour on 24 machines (ideally).

**Fault tolerance is required:** Critical pipelines can't fail because one server crashed. Distributed systems can survive partial failures.

**Cost efficiency at scale:** Sometimes 10 small machines are cheaper than 1 large machine with equivalent capacity.

Don't distribute when:

**Data fits comfortably:** If your data fits in RAM on a single machine, single-machine processing (DuckDB, pandas) is simpler and often faster.

**Overhead exceeds benefit:** Distributed systems have coordination overhead. For small data, this overhead dominates actual processing.

**Team lacks expertise:** Distributed systems are complex. If the team can't debug them, simpler approaches may be more reliable.

### The Scale Inflection Point

A useful rule of thumb:

| Data Size | Recommended Approach |
|-----------|---------------------|
| < 10 GB | Single machine (pandas, DuckDB) |
| 10 GB - 100 GB | Single large machine or small cluster |
| 100 GB - 10 TB | Distributed (Spark, Trino) |
| > 10 TB | Distributed required |

These are rough guidelines—actual thresholds depend on processing complexity, latency requirements, and infrastructure.

## The Distributed Computing Model

Distributed data processing follows a fundamental pattern: **partition the data, process partitions in parallel, combine results**.

### Partitioning

**Partitioning** divides data into subsets that can be processed independently.

```
Original Dataset (1 TB):
[████████████████████████████████]

Partitioned (100 partitions × 10 GB each):
[████] [████] [████] [████] ... [████]
  P1     P2     P3     P4        P100
```

Good partitioning:
- **Even distribution:** Each partition has roughly the same amount of data
- **Locality preservation:** Related data stays together when possible
- **Independence:** Partitions can be processed without knowing about other partitions

### Parallel Processing

Each partition is processed by a separate **task** running on a **worker** node:

```
          ┌─────────────┐
          │  Driver     │
          │  (Coordinator)
          └──────┬──────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│Worker1│   │Worker2│   │Worker3│
│       │   │       │   │       │
│ T1 T2 │   │ T3 T4 │   │ T5 T6 │
└───────┘   └───────┘   └───────┘
```

**Driver:** Coordinates the computation—creates the execution plan, assigns tasks to workers, collects results.

**Workers:** Execute tasks on data partitions. Each worker can run multiple tasks concurrently.

**Tasks:** Individual units of work processing one partition.

### The Map-Reduce Paradigm

The foundational model for distributed processing is **MapReduce**, introduced by Google in 2004:

**Map phase:** Apply a function to each record independently, producing intermediate key-value pairs.

**Shuffle phase:** Group intermediate results by key, moving data across the network.

**Reduce phase:** Aggregate values for each key to produce final results.

```
Example: Word Count

Input:     ["hello world", "hello there"]

Map:       [("hello", 1), ("world", 1), ("hello", 1), ("there", 1)]

Shuffle:   {"hello": [1, 1], "world": [1], "there": [1]}

Reduce:    {"hello": 2, "world": 1, "there": 1}
```

While modern systems (Spark, Flink) extend far beyond basic MapReduce, the core insight remains: **decompose computation into parallelizable operations separated by data exchange (shuffle)**.

## Transformations: Narrow vs. Wide

Understanding the difference between narrow and wide transformations is crucial for writing efficient distributed programs.

### Narrow Transformations

**Narrow transformations** operate on one partition at a time. Each output partition depends on exactly one input partition.

Examples:
- `filter`: Keep rows matching a condition
- `map`: Transform each row
- `select`: Choose columns

```
Narrow Transformation (filter):

Input Partitions:          Output Partitions:
[A B C D] ──filter──→     [A C]
[E F G H] ──filter──→     [F H]
[I J K L] ──filter──→     [J L]

No data movement between partitions!
```

**Why narrow is fast:**
- No network transfer required
- Partitions processed independently
- Can pipeline multiple narrow transformations

### Wide Transformations

**Wide transformations** require data from multiple input partitions to produce each output partition. They require a **shuffle**: redistributing data across the network.

Examples:
- `groupBy`: Group rows by key
- `join`: Combine two datasets by key
- `repartition`: Change partition count
- `sort`: Order all data

```
Wide Transformation (groupBy):

Input Partitions:          Shuffle           Output Partitions:
[A1 B1 C1] ───┐                         ┌──→ [A1 A2 A3] (key A)
[A2 B2 C2] ───┼──→ redistribute ───────┼──→ [B1 B2 B3] (key B)
[A3 B3 C3] ───┘    by key              └──→ [C1 C2 C3] (key C)

All data moves across the network!
```

**Why wide is expensive:**
- **Network transfer:** Moving data between machines is slow (network is 10-100x slower than local disk)
- **Serialization:** Data must be serialized for transfer
- **Disk spills:** If data doesn't fit in memory, it spills to disk
- **Synchronization:** All mappers must finish before reducers start

### The Cost of Shuffles

Shuffles are the dominant cost in most distributed jobs. A single unnecessary shuffle can turn a 10-minute job into a 2-hour job.

**Shuffle costs:**
- **Data volume:** Shuffling 1 TB takes ~100x longer than shuffling 10 GB
- **Number of partitions:** More partitions = more small network transfers
- **Network bandwidth:** Shared across all concurrent jobs
- **Disk I/O:** Shuffle data is written to disk before being read

**Key insight:** Minimize shuffles. If you can restructure your logic to avoid a shuffle, do it.

## Shuffles in Detail

Understanding how shuffles work helps you optimize them.

### The Shuffle Process

A shuffle has two phases:

**Map side (shuffle write):**
1. Each task computes output key-value pairs
2. Pairs are partitioned by key (hash partitioning by default)
3. Partitioned data is sorted by key within each partition
4. Sorted data is written to local disk as shuffle files

**Reduce side (shuffle read):**
1. Reduce tasks fetch their partitions from all map tasks
2. Fetched data is merged (already sorted, so merge is efficient)
3. Merged data is passed to reduce function

```
Map Side                    Network                 Reduce Side
─────────────────────────────────────────────────────────────────
Mapper 1                                           Reducer A
├─ Partition A ─────────────────────────────────→ ├─ From M1
├─ Partition B ─────────┐                         ├─ From M2
└─ Partition C ─────────┼───┐                     └─ From M3
                        │   │
Mapper 2                │   │                     Reducer B
├─ Partition A ─────────┼───┼───────────────────→ ├─ From M1
├─ Partition B ─────────┘   │                     ├─ From M2
└─ Partition C ─────────────┼───┐                 └─ From M3
                            │   │
Mapper 3                    │   │                 Reducer C
├─ Partition A ─────────────┘   │               → ├─ From M1
├─ Partition B ─────────────────┘                 ├─ From M2
└─ Partition C ───────────────────────────────────└─ From M3
```

### Shuffle Optimizations

**Combine on map side:** If the reduce function is associative (like sum, count), perform partial aggregation on the map side before shuffling. This reduces shuffle data volume dramatically.

```
Without combiner:          With combiner:
Map output:                Map output:
  ("a", 1)                   ("a", 3)  ← combined
  ("a", 1)
  ("a", 1)
  ("b", 1)                   ("b", 2)  ← combined
  ("b", 1)

Shuffle 5 records          Shuffle 2 records
```

**Broadcast small tables:** If joining a large table with a small table, broadcast the small table to all workers instead of shuffling both.

**Partition by join key:** If datasets will be joined repeatedly, pre-partition them by join key. Subsequent joins avoid shuffling.

**Reduce partition count:** Fewer partitions = fewer shuffle files = less overhead (but less parallelism).

## Data Locality

**Data locality** means processing data where it lives, rather than moving it to the processor.

### The Locality Hierarchy

In order of preference:

1. **Process-local:** Data is in the same process's memory. No I/O required.
2. **Node-local:** Data is on the same machine's disk. Fast local I/O.
3. **Rack-local:** Data is on a machine in the same network rack. Fast network transfer.
4. **Off-rack:** Data is on a distant machine. Slow network transfer.

### Locality in Practice

**HDFS era:** Hadoop was designed for data locality. Data was stored on the same machines that processed it. The scheduler tried to assign tasks to nodes that held the relevant data blocks.

**Cloud era:** With cloud object storage (S3, GCS), data and compute are separated. All data access is "off-rack" from the storage perspective. Locality now means:
- **Caching:** Keep recently accessed data in local SSD/memory
- **Same-region:** Keep compute in the same region as storage
- **Colocation:** Use cloud services that colocate compute with storage

### Implications for Design

**Minimize data movement:** Structure computations to reduce how much data crosses the network.

**Colocate related data:** If two datasets are frequently joined, store them with the same partitioning.

**Cache strategically:** Keep hot data in memory or fast local storage.

## Fault Tolerance

In a cluster with hundreds of machines, failures are routine. Distributed systems must handle failures gracefully.

### Failure Modes

**Task failure:** A task crashes due to bad data, out-of-memory, or code bugs. Solution: Retry the task on the same or different worker.

**Worker failure:** A machine crashes or becomes unreachable. Solution: Reschedule all its tasks on other workers.

**Network partition:** Part of the cluster can't communicate with the rest. Solution: Depends on system design (see CAP theorem in Chapter 2).

**Data corruption:** Storage returns incorrect data. Solution: Checksums and replication.

### Lineage-Based Recovery

Modern distributed data systems (Spark, Flink) use **lineage** for fault tolerance:

1. Track how each partition was computed (which input partitions, which transformations)
2. If a partition is lost, recompute it from its inputs
3. If inputs are also lost, recursively recompute from their inputs
4. This continues until reaching persisted data (checkpoint or source)

```
lineage DAG:

input (persisted)
    │
    ├─→ filter ───→ map ───→ output partition 1
    │                          (lost! recompute from filter)
    ├─→ filter ───→ map ───→ output partition 2
    │
    └─→ filter ───→ map ───→ output partition 3
```

**Advantages:**
- No synchronous replication overhead during normal operation
- Only recompute what's lost
- Works for any computation (not just specific operations)

**Disadvantages:**
- Long lineage chains = expensive recovery
- Shuffle outputs must be persisted or recomputed from shuffle inputs

### Checkpointing

For critical stages or long lineage chains, **checkpoint** intermediate results:

1. Write RDD/DataFrame to persistent storage
2. Truncate lineage—checkpointed data is the new starting point
3. Failures after checkpoint recover from checkpoint, not original sources

```
Without checkpoint:
  source → T1 → T2 → T3 → T4 → T5 → T6 → output
  (failure at T6 recomputes from source)

With checkpoint after T3:
  source → T1 → T2 → T3 → [checkpoint]
                              ↓
                          T4 → T5 → T6 → output
  (failure at T6 recomputes from checkpoint)
```

**When to checkpoint:**
- Before expensive operations
- After wide transformations (shuffle results)
- At natural boundaries in long pipelines

## Execution Strategies

Different distributed systems use different execution strategies.

### Batch Processing

**Batch processing** runs jobs over complete, bounded datasets.

Characteristics:
- Input is fixed at job start
- Job runs to completion
- Latency measured in minutes to hours
- Optimizes for throughput

Examples: Spark batch, Hive, traditional MapReduce

Use cases: Daily aggregations, historical analysis, model training

### Stream Processing

**Stream processing** handles continuous, unbounded data.

Characteristics:
- Input arrives continuously
- Job runs indefinitely
- Latency measured in milliseconds to seconds
- Optimizes for low latency

Examples: Flink, Kafka Streams, Spark Structured Streaming

Use cases: Real-time dashboards, fraud detection, event processing

### Micro-Batch

**Micro-batch** bridges batch and streaming:

Characteristics:
- Process small batches (seconds of data)
- Runs continuously in small batch increments
- Latency measured in seconds to minutes
- Simpler than true streaming, lower latency than batch

Examples: Spark Structured Streaming (default mode)

Use cases: Near-real-time analytics, stream processing with exactly-once semantics

We'll explore the batch/streaming spectrum in detail in Chapter 9.

## Common Distributed Operations

Understanding how common operations work in distributed systems helps you write efficient code.

### Distributed Aggregations

```python
# Conceptual: group by category, sum amount
df.groupBy("category").sum("amount")
```

**Execution:**
1. **Map phase:** Each partition computes local partial sums per category
2. **Shuffle:** Redistribute data by category key
3. **Reduce phase:** Sum partial results for each category

**Optimization:** Partial aggregation on map side dramatically reduces shuffle volume. If 1 billion rows reduce to 1000 categories, shuffle only 1000 rows per partition instead of all rows.

### Distributed Joins

Joins are among the most expensive distributed operations.

**Shuffle hash join:**
1. Shuffle both datasets by join key
2. Each partition contains matching keys from both sides
3. Join within each partition

```
Table A (by key)         Table B (by key)
[k1: a1] ──────┐         [k1: b1] ──────┐
[k2: a2] ──────┼───→ Shuffle ←────────┤ [k2: b2]
[k3: a3] ──────┘                      └─ [k3: b3]

After shuffle:
Partition 1: A[k1], B[k1] → (k1, a1, b1)
Partition 2: A[k2], B[k2] → (k2, a2, b2)
Partition 3: A[k3], B[k3] → (k3, a3, b3)
```

**Broadcast join:**
- If one table is small enough to fit in memory
- Broadcast small table to all workers
- Each worker joins its partitions of the large table with the entire small table locally
- No shuffle required!

```
Small Table (broadcast)     Large Table (partitioned)
[k1: b1]                    [k1: a1] ── join locally ─→ (k1, a1, b1)
[k2: b2]  ─→ to all         [k2: a2] ── join locally ─→ (k2, a2, b2)
[k3: b3]     workers        [k3: a3] ── join locally ─→ (k3, a3, b3)
```

**Sort-merge join:**
- Both datasets sorted by join key
- Merge sorted streams (like merge sort)
- Efficient for large sorted datasets

**Join optimization rules:**
1. Broadcast small tables (< 10 MB by default in Spark)
2. If can't broadcast, ensure both sides are partitioned by join key
3. If joining repeatedly, persist pre-partitioned datasets
4. Filter before joining to reduce data volume

### Distributed Sorting

Sorting across a cluster is expensive:

1. **Sample:** Estimate key distribution from samples
2. **Range partition:** Divide key space into N ranges (one per output partition)
3. **Shuffle:** Send each record to the partition for its key range
4. **Local sort:** Sort within each partition

The result is globally sorted: all of partition 1's keys < all of partition 2's keys, etc.

**Sorting is often avoidable:**
- If you only need top-K, use `limit` (no full sort)
- If you need aggregations, `groupBy` doesn't require sorting
- If you need ordering for output, consider whether consumers actually need it

## Practical Considerations

### Choosing Partition Count

**Too few partitions:**
- Underutilize cluster (idle workers)
- Large partitions may cause memory issues
- Slow recovery (lost partition = lots of recomputation)

**Too many partitions:**
- Task scheduling overhead
- Many small files (if persisted)
- Shuffle overhead increases

**Rules of thumb:**
- 2-4 partitions per CPU core
- Partition size: 128 MB - 1 GB
- At least as many partitions as cluster parallelism

### Handling Skew

**Data skew** occurs when partitions have uneven sizes:

```
Balanced:                  Skewed:
[████] [████] [████]      [████████████] [██] [█]
  P1     P2     P3          P1 (slow!)    P2   P3
```

Skew causes:
- One task takes much longer than others
- Overall job time = slowest task time
- Memory issues in large partitions

**Detecting skew:**
- Monitor task durations in Spark UI
- Check partition sizes in DataFrame statistics

**Mitigating skew:**
- **Salting:** Add random suffix to keys, aggregate in two stages
- **Adaptive Query Execution:** Spark 3.0+ automatically handles some skew
- **Custom partitioning:** Split hot keys across multiple partitions
- **Broadcast:** If skew is in join, broadcast the smaller side

### Memory Management

Distributed workers have limited memory, shared across:

- **Task execution:** Processing data
- **Shuffle buffers:** Sending and receiving shuffle data
- **Storage:** Caching RDDs/DataFrames
- **User code:** Variables, intermediate results

**Memory issues manifest as:**
- OutOfMemoryError
- Excessive garbage collection
- Spilling to disk (slow but survives)
- Task failures

**Solutions:**
- Reduce partition size (more partitions)
- Increase executor memory
- Avoid collecting large results to driver
- Use aggregations instead of collecting all data
- Checkpoint long lineages

## Summary

This chapter built the mental model for distributed data processing:

**Why distributed:**
- Single-machine ceiling limits scale
- Distribute when data exceeds single-machine capacity or processing time is unacceptable
- Don't distribute small data—overhead isn't worth it

**Core concepts:**
- Partition data, process in parallel, combine results
- Narrow transformations: no data movement, cheap
- Wide transformations: require shuffle, expensive

**Shuffles:**
- Data exchange across network
- The dominant cost in most jobs
- Minimize through careful design (combine, broadcast, pre-partition)

**Data locality:**
- Process data where it lives
- In cloud era: caching, same-region, strategic colocation

**Fault tolerance:**
- Lineage-based recovery: recompute from tracked transformations
- Checkpointing: truncate lineage at critical points

**Practical patterns:**
- Right-size partitions (2-4 per core, 128 MB - 1 GB)
- Handle skew with salting or broadcast
- Manage memory carefully

**Key insight:** Distributed computing is about minimizing data movement while maximizing parallel processing. Every shuffle is expensive; every optimization is about avoiding or reducing shuffles.

**Looking ahead:**
- Chapter 8 applies these principles to Apache Spark, the dominant distributed processing engine
- Chapter 9 explores the batch vs. streaming spectrum
- Chapter 15 shows how distributed compute fits into lakehouse architectures

## Further Reading

- Dean, J. & Ghemawat, S. (2004). "MapReduce: Simplified Data Processing on Large Clusters" — The foundational paper
- Zaharia, M. et al. (2012). "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing" — The Spark paper
- Kleppmann, M. (2017). *Designing Data-Intensive Applications*, Chapter 10 — Excellent coverage of batch processing fundamentals
- "The Shuffle Service Design" (Spark documentation) — Deep dive into shuffle implementation

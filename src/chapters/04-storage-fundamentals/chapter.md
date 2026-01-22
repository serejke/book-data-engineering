---
chapter: 4
title: "Storage Fundamentals"
estimated_pages: 35-40
status: draft
last_updated: 2025-01-22
---

# Storage Fundamentals

Every query you run, every model you train, every dashboard you build ultimately depends on how data is physically organized on storage media. A query that takes 3 seconds or 3 hours; a storage bill of $1,000 or $100,000 per month; a pipeline that handles schema changes gracefully or crashes spectacularly—these outcomes trace back to storage decisions made long before anyone wrote a query.

This chapter examines storage from first principles: how data is physically arranged, why that arrangement matters, and how to choose formats and systems that match your access patterns. We'll cover row vs. columnar storage, file formats (Parquet, ORC, Avro), object storage characteristics, and the principles that guide storage decisions.

Understanding storage fundamentals is essential preparation for Chapter 5, where we'll see how table formats build database-like capabilities on top of these primitives.

## Row-Oriented vs. Column-Oriented Storage

The most fundamental decision in data storage is whether to organize data by rows or by columns. This choice determines which operations are fast and which are slow.

### Row-Oriented Storage

In row-oriented storage, each record's fields are stored contiguously:

```
Record 1: [id=1, name="Alice", amount=100, date="2024-01-01"]
Record 2: [id=2, name="Bob",   amount=200, date="2024-01-02"]
Record 3: [id=3, name="Carol", amount=150, date="2024-01-03"]
```

On disk, this looks like:

```
[1|Alice|100|2024-01-01][2|Bob|200|2024-01-02][3|Carol|150|2024-01-03]
```

**Strengths of row orientation:**

- **Fast writes**: Appending a new record writes one contiguous block
- **Fast point reads**: Fetching a complete record requires one seek
- **Natural for transactions**: OLTP workloads (insert, update, read-by-key) access whole records

**Weaknesses:**

- **Inefficient scans**: Reading one column requires reading every column
- **Poor compression**: Adjacent bytes have different types, limiting compression opportunities
- **Wasteful for analytics**: Most analytical queries touch few columns but many rows

**Examples:** PostgreSQL, MySQL, MongoDB (document stores are row-oriented at the document level)

### Column-Oriented Storage

In column-oriented (columnar) storage, values from each column are stored contiguously:

```
Column "id":     [1, 2, 3, ...]
Column "name":   [Alice, Bob, Carol, ...]
Column "amount": [100, 200, 150, ...]
Column "date":   [2024-01-01, 2024-01-02, 2024-01-03, ...]
```

On disk:

```
[1|2|3|...][Alice|Bob|Carol|...][100|200|150|...][2024-01-01|2024-01-02|2024-01-03|...]
```

**Strengths of columnar orientation:**

- **Efficient scans**: Reading one column doesn't touch other columns
- **Excellent compression**: Adjacent values have the same type, enabling dictionary encoding, run-length encoding, delta encoding
- **Vectorized operations**: CPUs can process column batches with SIMD instructions
- **Column pruning**: Queries touch only needed columns

**Weaknesses:**

- **Slow point reads**: Fetching a complete record requires seeks to multiple locations
- **Complex writes**: Appending a record requires writing to multiple column files
- **Update challenges**: Modifying a record affects multiple storage locations

**Examples:** Parquet, ORC, ClickHouse, Redshift, BigQuery

### The 10x Difference

For analytical workloads, columnar storage typically provides 10x or better improvements in both query performance and storage efficiency. Consider a query:

```sql
SELECT AVG(amount) FROM transactions WHERE date > '2024-01-01';
```

**Row-oriented**: Must read every byte of every row, extracting `amount` and `date` from each. If the table has 100 columns, you read 100x the data you need.

**Columnar**: Read only the `amount` and `date` columns. Skip the other 98 columns entirely. Furthermore, compressed columnar data often achieves 10:1 or better compression ratios.

The combination of reading less data and better compression creates order-of-magnitude improvements.

### When to Choose Each

**Row-oriented when:**
- Workload is transactional (insert, update, point reads)
- Queries typically fetch complete records
- Write latency is critical
- Updates are frequent

**Columnar when:**
- Workload is analytical (scans, aggregations)
- Queries touch few columns but many rows
- Data is primarily append-only
- Storage cost and query performance matter more than write latency

For data engineering—where the focus is analytical workloads—columnar storage is almost always the right choice for processed data. Raw data ingestion might use row-oriented formats for flexibility, but anything that will be queried analytically should be columnar.

## File Formats

File formats define how data is serialized to bytes. The choice of format affects storage size, read/write performance, schema handling, and ecosystem compatibility.

### Apache Parquet

Parquet is the dominant columnar format for analytics. Created as a collaboration between Twitter and Cloudera (2013), it's now the de facto standard for lakehouse storage.

**Structure:**

A Parquet file contains:

```
┌─────────────────────────────────┐
│         Row Group 1             │
│  ┌───────┬───────┬───────┐      │
│  │Col A  │Col B  │Col C  │      │
│  │Chunk  │Chunk  │Chunk  │      │
│  └───────┴───────┴───────┘      │
├─────────────────────────────────┤
│         Row Group 2             │
│  ┌───────┬───────┬───────┐      │
│  │Col A  │Col B  │Col C  │      │
│  │Chunk  │Chunk  │Chunk  │      │
│  └───────┴───────┴───────┘      │
├─────────────────────────────────┤
│          Footer                 │
│  - Schema                       │
│  - Row group metadata           │
│  - Column statistics            │
└─────────────────────────────────┘
```

**Row groups** are horizontal partitions of the data (typically 128 MB). Each row group contains column chunks for every column. This allows parallel processing of row groups.

**Column chunks** contain the compressed, encoded data for one column within a row group. Each chunk includes:
- Compression (Snappy, Zstd, Gzip)
- Encoding (dictionary, RLE, delta)
- Statistics (min, max, null count)

**The footer** contains schema information and metadata about row groups and columns. Reading Parquet starts by reading the footer to plan which row groups and columns to access.

**Key features:**

- **Predicate pushdown**: Column statistics enable skipping row groups that can't match query filters. If you're querying `WHERE amount > 1000` and a row group's max amount is 500, skip it entirely.

- **Projection pushdown**: Read only the columns needed for the query.

- **Nested data support**: Parquet handles nested structures (structs, arrays, maps) efficiently using the Dremel encoding.

- **Schema evolution**: New columns can be added; readers handle missing columns gracefully.

**When to use Parquet:**
- Analytical workloads (the default choice)
- Data lake/lakehouse storage
- Any columnar analytical storage need

### Apache ORC

ORC (Optimized Row Columnar) emerged from the Hive project at Facebook (2013). It's similar to Parquet but with some different design choices.

**Structure:**

```
┌─────────────────────────────────┐
│          Stripe 1               │
│  ┌─────────────────────────┐    │
│  │    Index Data           │    │
│  ├─────────────────────────┤    │
│  │    Row Data (columns)   │    │
│  ├─────────────────────────┤    │
│  │    Stripe Footer        │    │
│  └─────────────────────────┘    │
├─────────────────────────────────┤
│          Stripe 2               │
├─────────────────────────────────┤
│          File Footer            │
│          Postscript             │
└─────────────────────────────────┘
```

ORC calls its horizontal partitions "stripes" (similar to Parquet's row groups). A key difference: ORC includes lightweight indexes within each stripe for faster filtering.

**Differences from Parquet:**

- **Better compression**: ORC often achieves slightly better compression, especially with Zlib
- **Built-in indexes**: Row-level indexes within stripes enable finer-grained filtering
- **ACID support**: ORC was designed with Hive ACID transactions in mind
- **Smaller ecosystem**: Parquet has broader tool support outside the Hive ecosystem

**When to use ORC:**
- Heavy Hive usage (ORC is native to Hive)
- Workloads with very high compression requirements
- ACID transactions in Hive (though table formats now provide better options)

### Apache Avro

Avro is a row-oriented format designed for serialization and schema evolution. Created by Doug Cutting (2009), it's commonly used for message serialization and data exchange.

**Structure:**

```
┌─────────────────────────────────┐
│          File Header            │
│  - Magic bytes                  │
│  - Schema (JSON)                │
│  - Metadata                     │
├─────────────────────────────────┤
│          Data Block 1           │
│  - Count                        │
│  - Serialized records           │
├─────────────────────────────────┤
│          Data Block 2           │
├─────────────────────────────────┤
│          Sync Marker            │
└─────────────────────────────────┘
```

**Key features:**

- **Schema included**: The schema is embedded in the file, making files self-describing
- **Schema evolution**: Readers can handle schema differences using resolution rules
- **Compact binary format**: Efficient serialization for messaging
- **Dynamic typing**: Schema can be determined at runtime

**When to use Avro:**
- Message serialization (Kafka messages)
- Schema registry integration
- Data exchange between systems
- When schema evolution is critical and you need row-level access

### JSON and CSV

These text formats deserve mention because they're ubiquitous, though rarely optimal:

**JSON:**
- Human readable
- Self-describing (schema embedded)
- Extremely inefficient (10-100x larger than binary formats)
- Parsing is slow
- Use for: APIs, configuration, small data exchange, debugging

**CSV:**
- Human readable, widely supported
- No schema (column types are ambiguous)
- Poor handling of complex types, nulls, special characters
- Use for: Legacy integrations, simple exports, spreadsheet compatibility

For any serious data engineering workload, avoid JSON and CSV for storage. Convert to Parquet or Avro at ingestion.

### Format Comparison

| Characteristic | Parquet | ORC | Avro | JSON |
|---------------|---------|-----|------|------|
| Orientation | Columnar | Columnar | Row | Row |
| Compression | Excellent | Excellent | Good | Poor |
| Schema | Footer | Footer | Header | Embedded |
| Schema evolution | Good | Good | Excellent | N/A |
| Nested types | Yes | Yes | Yes | Yes |
| Splittable | Yes | Yes | Yes | Line-delimited only |
| Ecosystem | Broad | Hive-focused | Broad | Universal |
| Best for | Analytics | Hive analytics | Messaging | APIs, debug |

### Choosing a Format

```
START: What's the primary use case?

├─ Analytical queries
│   └─ Parquet (default) or ORC (if Hive-heavy)
│
├─ Message serialization / streaming
│   └─ Avro (with schema registry)
│
├─ Data exchange with external systems
│   ├─ Technical systems → Parquet or Avro
│   └─ Non-technical users → CSV (reluctantly)
│
└─ Exploration / debugging
    └─ JSON or CSV (temporarily)
```

## Compression

Compression reduces storage costs and often improves query performance (less data to read from disk/network). Understanding compression options helps you make informed tradeoffs.

### Compression Algorithms

**Snappy:**
- Fast compression and decompression
- Moderate compression ratio (2-4x typical)
- CPU-efficient
- Default for many systems
- Use when: Query speed matters more than storage size

**Zstandard (Zstd):**
- Better compression than Snappy (3-5x typical)
- Still reasonably fast
- Tunable compression levels
- Increasingly popular as the balanced choice
- Use when: Want good compression without sacrificing too much speed

**Gzip/Zlib:**
- High compression ratio (4-8x typical)
- Slow compression, moderate decompression
- Use when: Storage cost is critical, write speed doesn't matter

**LZ4:**
- Extremely fast
- Lower compression ratio than Snappy
- Use when: Speed is paramount

### Compression in Columnar Formats

Columnar formats achieve better compression because:

1. **Type homogeneity**: All values in a column have the same type, enabling type-specific encoding
2. **Value locality**: Similar values are adjacent, improving dictionary and RLE compression
3. **Encoding + compression**: Multiple layers (dictionary encoding, then Zstd) compound gains

**Dictionary encoding** replaces repeated values with short codes:
```
Original: ["USD", "USD", "EUR", "USD", "EUR"]
Dictionary: {0: "USD", 1: "EUR"}
Encoded: [0, 0, 1, 0, 1]
```

**Run-length encoding (RLE)** compresses repeated adjacent values:
```
Original: [1, 1, 1, 1, 2, 2, 3, 3, 3]
RLE: [(1, 4), (2, 2), (3, 3)]
```

**Delta encoding** stores differences between values:
```
Original: [100, 102, 105, 107]
Delta: [100, 2, 3, 2]
```

These encodings happen before compression algorithms apply, creating layered compression that achieves 10:1 or better ratios on real-world data.

## Object Storage

Object storage (S3, GCS, Azure Blob Storage, MinIO) has become the default foundation for data lakes and lakehouses. Understanding its characteristics is essential for effective data engineering.

### Object Storage Model

Object storage differs fundamentally from file systems:

**Flat namespace**: Objects are identified by keys (strings), not hierarchical paths. The "/" in `s3://bucket/path/to/file.parquet` is just part of the key string, not a directory separator.

**Immutable objects**: Objects are written once. "Updating" an object means writing a new version (if versioning is enabled) or overwriting completely.

**Eventually consistent** (historically): S3 famously provided eventual consistency for overwrites and deletes. As of December 2020, S3 provides strong consistency for all operations—a significant change.

**No atomic operations**: You cannot atomically rename an object or update part of an object. This limitation is why table formats are necessary.

### Performance Characteristics

**High latency**: First-byte latency for object storage is ~50-100ms, compared to <1ms for local SSD. This favors batch access patterns.

**High throughput**: Once streaming, object storage delivers high bandwidth. Single-stream throughput of 100+ MB/s is typical.

**Parallelism is key**: The way to achieve high aggregate throughput is parallel access to many objects simultaneously. A Spark job reading 1,000 Parquet files from S3 can achieve aggregate throughput of 10+ GB/s.

**List operations are expensive**: Listing objects in a "directory" requires API calls that return paginated results. Listing thousands of objects is slow. This is why table formats maintain metadata separately.

### Practical Implications

**File sizes matter**: Many small files create overhead (each file requires a separate API call and has fixed latency). Few large files improve throughput but reduce parallelism. The sweet spot for analytical workloads is typically 100MB-1GB per file.

**Partitioning for access patterns**: Organize data to minimize listing and enable partition pruning:
```
s3://bucket/transactions/year=2024/month=01/day=15/data.parquet
```
Queries filtering on date can skip irrelevant partitions entirely.

**Write-once patterns**: Design pipelines for immutable writes. Overwrite entire partitions rather than updating individual records in place.

### Cost Model

Object storage pricing has three components:

**Storage**: $0.02-0.03/GB/month for standard tiers. This is 100-1000x cheaper than warehouse storage.

**Requests**: GET and PUT requests cost $0.0004-0.005 per 1,000 requests. Many small files = many requests = higher costs.

**Egress**: Data transfer out of the cloud region costs $0.05-0.12/GB. This adds up for large-scale data movement.

**Lifecycle policies**: Move cold data to cheaper tiers (S3 Glacier, $0.004/GB/month) automatically based on age.

## Data Layout and Partitioning

How data is physically organized across files and directories dramatically affects query performance.

### Partitioning Strategies

**Time-based partitioning** is most common:
```
/transactions/year=2024/month=01/day=15/
/transactions/year=2024/month=01/day=16/
```

Queries filtering on time can skip irrelevant partitions:
```sql
-- Only reads partitions for January 2024
SELECT * FROM transactions WHERE year = 2024 AND month = 1
```

**Categorical partitioning** works for high-cardinality dimensions:
```
/transactions/region=us-east/
/transactions/region=us-west/
/transactions/region=eu-west/
```

**Composite partitioning** combines multiple columns:
```
/transactions/region=us-east/year=2024/month=01/
```

### Partition Granularity Tradeoffs

**Too many partitions (over-partitioning)**:
- Many small files (small file problem)
- Expensive metadata operations
- Excessive list operations

**Too few partitions (under-partitioning)**:
- Large files that can't be pruned
- Queries scan unnecessary data
- Limited parallelism

**Rule of thumb**: Partition files should be 100MB-1GB. If partitioning by hour creates 1MB files, partition by day instead.

### The Small Files Problem

Small files are a persistent issue in data lakes:

**Causes:**
- Streaming ingestion creating files every minute
- Over-partitioning
- Incremental updates creating many small additions

**Symptoms:**
- Slow query performance (overhead per file)
- High request costs
- Memory pressure in query planners

**Solutions:**
- Compaction jobs that merge small files
- Table formats with automatic compaction
- Coarser partitioning
- Buffering writes to accumulate larger batches

### File Sizing Guidelines

| Access Pattern | Target File Size | Rationale |
|---------------|------------------|-----------|
| Frequent full scans | 256MB - 1GB | Maximize throughput per file |
| Selective queries | 64MB - 256MB | Balance parallelism and efficiency |
| Streaming ingestion | Compact to 128MB+ | Accept small files temporarily, compact later |
| Archive/cold storage | 1GB+ | Minimize storage overhead |

## Encoding and Data Types

How values are encoded affects both storage efficiency and query correctness.

### Numeric Types

**Integers**: Choose the smallest type that fits your range. INT32 vs. INT64 matters when you have billions of rows.

**Decimals**: For financial data, use fixed-point decimals (DECIMAL(18,4)) not floating-point. Floating-point arithmetic creates precision errors that are unacceptable for financial calculations.

**Timestamps**: Store as INT64 microseconds or nanoseconds since epoch, not as strings. String timestamps are larger and require parsing.

### String Types

**Dictionary encoding**: Automatically applied to low-cardinality string columns. If a column has 100 distinct values across 1 billion rows, dictionary encoding is extremely effective.

**UTF-8**: Standard encoding for text. Be aware of collation issues for sorting and comparison.

**Fixed-length vs. variable-length**: If strings have predictable length (country codes, UUIDs), fixed-length encoding can be more efficient.

### Nested Types

Modern formats handle complex types:

**Structs**: Named fields with potentially different types
```
address: {
  street: STRING,
  city: STRING,
  zip: STRING
}
```

**Arrays/Lists**: Repeated values of the same type
```
tags: [STRING]
```

**Maps**: Key-value pairs
```
metadata: MAP<STRING, STRING>
```

Parquet and ORC handle these efficiently, but query patterns matter. Accessing `address.city` is efficient; scanning all elements of a large array is expensive.

### Null Handling

Nulls require special handling in storage:

- **Null bitmaps**: Track which values are null
- **Statistics**: Null counts help query optimization
- **Semantics**: NULL != NULL in SQL; be careful with joins and aggregations

Design schemas to minimize nulls where possible. A non-nullable column with a default value is often better than nullable columns scattered with NULLs.

## Storage Tiering

Not all data deserves the same storage treatment. Storage tiering matches data to appropriate storage classes based on access patterns.

### Access Patterns

**Hot data**: Frequently accessed, query performance critical
- Recent data (last 30-90 days)
- Active analytics and dashboards
- Store on: Standard object storage or SSD-backed storage

**Warm data**: Occasionally accessed, moderate performance needs
- Historical data accessed periodically
- Compliance queries, audits
- Store on: Standard object storage

**Cold data**: Rarely accessed, cost optimization critical
- Archive data, long-term retention
- Regulatory requirements
- Store on: Glacier, Archive tiers ($0.004/GB/month)

### Implementing Tiering

**Time-based policies**: Automatically move data to colder tiers based on age
```
- Data < 90 days: Standard storage
- Data 90-365 days: Infrequent Access tier
- Data > 365 days: Glacier
```

**Access-based policies**: Move data based on actual access patterns (more complex to implement)

**Separate tables**: Maintain separate tables for hot and cold data, query both when needed

Table formats (Chapter 5) can manage tiering within a single logical table, moving older partitions to cheaper storage while maintaining a unified query interface.

## Case Study: Designing Storage for Trading Data

Let's apply these principles to a concrete scenario: storing trade execution data for a trading firm.

### Requirements

- 50 million trades per day
- Each trade: ~500 bytes (30 fields)
- 3-year retention requirement
- Query patterns:
  - Daily P&L reports (scan today's data)
  - Historical analysis (scan months of data)
  - Compliance queries (point lookups by trade ID)
  - ML feature extraction (scan specific columns across years)

### Design Decisions

**Format**: Parquet
- Analytical workloads dominate
- Column pruning essential for ML feature extraction
- Compression critical for 3-year retention

**Compression**: Zstd
- Good balance of compression and speed
- Important for historical scans

**Partitioning**: By date (daily)
- 50M trades × 500 bytes = 25GB/day → reasonable file sizes per partition
- Daily partitions support common query patterns
- Not over-partitioned (wouldn't partition by hour with these volumes)

**Storage layout**:
```
s3://trading-data/trades/
  year=2024/
    month=01/
      day=15/
        trades-00000.parquet (target: 256MB each)
        trades-00001.parquet
        trades-00002.parquet
```

**File sizing**: Target 256MB per file
- 25GB/day ÷ 256MB = ~100 files per day
- Good parallelism for Spark jobs
- Efficient for single-day queries

**Tiering**:
- < 90 days: S3 Standard (frequent access)
- 90-365 days: S3 Infrequent Access (monthly reports)
- > 365 days: S3 Glacier Instant Retrieval (compliance)

**Schema design**:
```
trade_id: STRING (natural key)
timestamp: TIMESTAMP_MICROS
symbol: STRING (dictionary encoded, ~10K symbols)
quantity: DECIMAL(18, 4)
price: DECIMAL(18, 8)
side: STRING (BUY/SELL, dictionary encoded)
trader_id: STRING (dictionary encoded)
desk: STRING (dictionary encoded)
...
```

### Estimated Costs

**Storage**:
- 25GB/day × 365 days × 3 years = 27 TB
- With compression (10:1): 2.7 TB effective
- Tiered storage: ~$50/month

**Compute** (reading data):
- Depends on query patterns and engine
- Columnar format reduces scanned data by 5-10x vs. row storage

**Without these optimizations** (row-oriented, uncompressed, untiered):
- 27 TB at $0.023/GB = $620/month storage alone
- Plus higher query costs from scanning more data

The combination of columnar storage, compression, partitioning, and tiering reduces costs by 10x or more while improving query performance.

## Summary

This chapter covered the foundational concepts for data storage:

**Row vs. columnar storage:**
- Row orientation: Fast for transactions, inefficient for analytics
- Columnar orientation: Excellent for analytics, 10x+ improvements typical
- Default to columnar (Parquet) for any analytical workload

**File formats:**
- Parquet: Default for analytics, broad ecosystem support
- ORC: Alternative for Hive-heavy environments
- Avro: Row-oriented, good for messaging and schema evolution
- Avoid JSON/CSV for storage at scale

**Compression:**
- Snappy: Fast, moderate compression
- Zstd: Balanced choice, increasingly popular
- Columnar + encoding + compression = 10x+ size reduction

**Object storage:**
- High latency, high throughput, optimize for parallelism
- Flat namespace with no atomic operations
- Cost model: storage + requests + egress

**Data layout:**
- Partition to enable pruning
- Target 100MB-1GB files
- Avoid the small files problem

**Key insight**: Storage decisions made at ingestion time determine query performance and costs for the lifetime of the data. Invest time in getting storage right.

**Looking ahead:**
- Chapter 5 builds on these fundamentals to explain how table formats (Iceberg, Delta) add database capabilities to files on object storage
- Chapter 6 covers when specialized OLAP databases (ClickHouse, DuckDB) complement or replace lakehouse storage

## Further Reading

- Apache Parquet Documentation: https://parquet.apache.org/docs/
- "Dremel: Interactive Analysis of Web-Scale Datasets" (Google, 2010) — The paper that inspired Parquet's nested type handling
- "The Design and Implementation of Modern Column-Oriented Database Systems" — Academic survey of columnar storage
- AWS S3 Best Practices: https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html

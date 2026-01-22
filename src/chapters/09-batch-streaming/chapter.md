---
chapter: 9
title: "Batch vs Streaming: A Unified Perspective"
estimated_pages: 45-50
status: draft
last_updated: 2025-01-22
---

# Batch vs Streaming: A Unified Perspective

The distinction between batch and streaming processing has shaped data engineering for decades. Batch processes data in bounded chunks—yesterday's logs, last week's transactions, historical datasets. Streaming processes data continuously as it arrives—each trade, each click, each sensor reading.

But this chapter argues that the batch vs. streaming dichotomy is increasingly artificial. The fundamental operations—filtering, aggregating, joining—are the same regardless of whether data is bounded or unbounded. What changes is how we handle time, state, and completeness.

Understanding both paradigms deeply, and the spectrum between them, lets you choose the right approach for each problem rather than defaulting to whichever you happen to know.

## The Fundamental Distinction

At the conceptual level, batch and streaming differ in one property: **boundedness**.

**Batch:** Data is bounded. You know when it starts and ends. You can scan it multiple times. The dataset is complete before processing begins.

```
Batch: [--------- Complete Dataset ---------]
       |                                    |
     Start                                End
       └── Process once, get final result ─┘
```

**Streaming:** Data is unbounded. There's no end—new events keep arriving. You can only process data once (without rewinding). The dataset is never complete.

```
Streaming: ──events────events────events────events────▶ (forever)
              │           │           │          │
              └─ emit ────┴─ emit ────┴─ emit ───┴─ emit results continuously
```

This single difference cascades into different approaches for:
- **Time semantics:** When did events happen vs. when did we process them?
- **Completeness:** How do we know when we have "all" the data?
- **State management:** How do we maintain aggregations over unbounded data?
- **Output:** Final results vs. continuously updated approximations

## When to Use Each Paradigm

The choice between batch and streaming isn't about capability—modern systems can do both. It's about requirements.

### Batch is Appropriate When:

**1. Latency tolerance is high**
If stakeholders need reports by morning, not by the millisecond, batch simplifies your architecture. A nightly ETL job that finishes at 3 AM to support a 9 AM dashboard is perfectly adequate.

**2. Data completeness matters more than timeliness**
Financial reconciliation, regulatory reporting, ML model training—these require all data, not fast data. Late-arriving events need to be included. Corrections need to be processed. Batch naturally handles this by waiting for data to stabilize.

**3. Complex analytics require multiple passes**
Some algorithms need to see all data multiple times: iterative machine learning, global optimizations, complex graph algorithms. Streaming's single-pass constraint makes these difficult or impossible.

**4. Cost optimization is critical**
Batch jobs can use spot instances, run during off-peak hours, and scale down completely when idle. Streaming requires always-on infrastructure, which costs more even when event volume is low.

### Streaming is Appropriate When:

**1. Time is a critical business requirement**
Fraud detection, real-time bidding, algorithmic trading, operational alerting—these require sub-second responses. By the time batch processing runs, the opportunity is gone.

**2. Continuous operations need continuous data**
Monitoring dashboards, live leaderboards, real-time inventory tracking. Users expect data to update as things happen, not hours later.

**3. Event-driven architectures**
When downstream systems need to react to events—sending notifications, triggering workflows, updating search indices—streaming provides the natural integration pattern.

**4. Infinite data streams**
IoT sensors, clickstreams, application logs—the data never stops. Waiting to "collect it all" is impossible. You must process incrementally.

### The Spectrum Between

Most real systems don't fit cleanly into batch or streaming. Consider:

**Micro-batch:** Process small batches frequently (every minute, every 10 seconds). Gets close to streaming latency with batch simplicity. Spark Structured Streaming uses this model by default.

**Windowed streaming:** Aggregate streaming data into windows (hourly, daily), producing batch-like results from streaming input.

**Lambda architecture:** Run both batch and streaming in parallel—streaming for speed, batch for accuracy. Merge results.

**Kappa architecture:** Streaming only, replay from Kafka to reprocess historical data when logic changes.

**Incremental batch:** Process only new data since last run, rather than reprocessing everything. Behaves like streaming with batch scheduling.

The right architecture often combines elements of both paradigms.

## Time: The Central Challenge

Time in streaming systems is surprisingly complex. Unlike batch processing where all data exists simultaneously, streaming must decide: what does "now" mean?

### Two Notions of Time

**Event time:** When the event actually occurred in the real world. The timestamp in the data itself.

```python
# Event time is embedded in the data
{"trade_id": "123", "symbol": "BTC", "price": 50000, "event_time": "2024-01-15T10:30:00Z"}
```

**Processing time:** When the event is processed by the system. The wall clock time when your code runs.

```python
# Processing time is when you see it
processed_at = datetime.now()  # Could be seconds or hours after event_time
```

These differ because of:
- **Network latency:** Events take time to travel from source to processor
- **Out-of-order arrival:** Event B might arrive before event A even though A happened first
- **Batching at source:** Mobile apps batch events and send them together
- **Reprocessing:** Replaying historical data processes old events at current time

### Why Event Time Matters

Consider computing hourly trade volumes. Using processing time:

```
Events arrive:
  10:00:01 - Trade A (happened at 10:00:00)
  10:00:02 - Trade B (happened at 09:59:59) ← Late!
  10:01:00 - Trade C (happened at 09:58:00) ← Very late!

Processing time result for 10:00-11:00 hour:
  Includes: A, B, C
  Wrong! B and C happened in the 09:00-10:00 hour
```

Using event time requires handling late data, but produces correct results:

```
Event time result for 09:00-10:00 hour:
  Includes: B, C
  Correct! Based on when events actually happened
```

### Watermarks: Progress in Event Time

A **watermark** is a heuristic declaration: "I believe I've seen all events up to time T."

```
Event Stream:     [e1:9:30] [e2:9:45] [e3:9:20] [e4:10:05] [e5:9:55] ...
                      │         │         │          │         │
Watermark:       ─────┼─────────┼─────────┼──────────┼─────────┼─────▶
                   9:20      9:30      9:30      9:50      9:50

                     Watermark advances conservatively, allowing for late data
```

When the watermark passes a window's end time, the window can be closed and results emitted.

**Watermark strategies:**

**1. Bounded delay:** Watermark = max event time seen - allowed lateness
```python
# Expect events within 10 minutes of their event time
watermark = max_event_time - timedelta(minutes=10)
```

**2. Per-source tracking:** Track event time progress per source, take minimum
```python
# Handle sources with different latencies
watermark = min(source_1_max_time, source_2_max_time, source_3_max_time) - delay
```

**3. Custom heuristics:** Use domain knowledge about data arrival patterns
```python
# Markets close at 16:00, so after 16:10 we have all trades
if current_time > market_close + 10_minutes:
    watermark = market_close
```

### Late Data Handling

Even with watermarks, some data arrives late. Strategies:

**1. Drop late data**
```python
# Spark: drop events that arrive after watermark
df.withWatermark("event_time", "10 minutes")
  .groupBy(window("event_time", "1 hour"))
  .count()
# Events more than 10 minutes late are dropped
```

**2. Accept late data into future windows**
Allow late events to update already-emitted results, producing corrections.

**3. Side output late data**
Route late events to a separate stream for special handling.

**4. Reprocess affected time ranges**
In lakehouse architectures, append late data to storage and recompute affected aggregations in batch.

## Windowing: Grouping Unbounded Data

Windows partition infinite streams into finite chunks for aggregation. Window choice significantly impacts results and resource usage.

### Window Types

**Tumbling (Fixed) Windows:**
Non-overlapping, fixed-size windows.

```
Timeline:    |--00:00--|--01:00--|--02:00--|--03:00--|
Window 1:    [────────]
Window 2:              [────────]
Window 3:                        [────────]
Window 4:                                  [────────]

Each event belongs to exactly one window.
```

```python
# Hourly aggregation
df.groupBy(window("event_time", "1 hour")).count()
```

**Sliding Windows:**
Overlapping windows that slide by a fixed interval.

```
Timeline:    |--00:00--|--01:00--|--02:00--|--03:00--|
Window 1:    [────────────]
Window 2:         [────────────]
Window 3:              [────────────]

Events can belong to multiple windows.
Window size: 2 hours, slide: 1 hour
```

```python
# 2-hour windows, updated every hour
df.groupBy(window("event_time", "2 hours", "1 hour")).count()
```

**Session Windows:**
Dynamic windows based on activity gaps.

```
Timeline:    |----events----|  gap  |--events--|  gap  |--events------|
Session 1:   [─────────────]
Session 2:                         [──────────]
Session 3:                                           [───────────────]

Window ends when no activity for the gap duration.
```

```python
# Sessions end after 30 minutes of inactivity
df.groupBy(session_window("event_time", "30 minutes"), "user_id").count()
```

**Global Windows:**
A single window containing all data (essentially batch semantics).

```python
# Count everything ever
df.groupBy().count()  # Requires unbounded state—be careful!
```

### Choosing Window Strategy

| Requirement | Window Type |
|-------------|-------------|
| Regular reports (hourly, daily) | Tumbling |
| Moving averages, trends | Sliding |
| User session analytics | Session |
| Deduplicate within time range | Tumbling or sliding |
| Continuous running totals | Global (with caution) |

## State Management

Streaming aggregations require state—you can't compute a running sum without remembering previous values. State management is the hardest problem in stream processing.

### State Types

**Keyed state:** Separate state per key (e.g., per user, per symbol)
```python
# Separate count per symbol
df.groupBy("symbol").count()  # State: {BTC: 1000, ETH: 500, ...}
```

**Operator state:** Shared across all keys (e.g., Kafka partition offsets)
```python
# Which partition offsets have been processed
# State: {partition_0: offset_123, partition_1: offset_456}
```

### State Backends

Where does state live during computation?

**In-memory (heap):**
- Fast but limited by memory
- Lost on failure
- Good for small state

**RocksDB (embedded key-value store):**
- Scales to larger than memory
- Persists to local disk
- Checkpoint to durable storage
- Standard choice for large state

```python
# Spark: configure RocksDB state store
spark.conf.set("spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider")
```

**Remote state (disaggregated):**
- State stored separately from compute
- Enables elastic scaling
- Higher latency but more flexibility
- Flink 2.0's new architecture supports this

### State Size Challenges

State can grow without bound:

```python
# Danger: unbounded state
df.groupBy("user_id").count()  # State grows with unique users—forever
```

Mitigation strategies:

**1. Time-to-live (TTL):**
```python
# Spark 4.0 transformWithState
class SessionCounter:
    def process(self, key, events, state):
        count = state.get("count") or 0
        count += len(events)
        state.set("count", count)
        state.set_ttl(timedelta(hours=24))  # Expire after 24 hours of inactivity
```

**2. Windowed aggregations:**
```python
# State bounded by window duration
df.withWatermark("event_time", "1 hour")
  .groupBy(window("event_time", "1 hour"), "user_id")
  .count()
# Old windows are cleaned up as watermark advances
```

**3. Explicit cleanup:**
```python
# Session windows: state cleaned when session ends
df.groupBy(session_window("event_time", "30 minutes"), "user_id")
  .count()
```

### Checkpointing and Fault Tolerance

Stream processors checkpoint state periodically to survive failures.

**Checkpoint process:**
1. Pause processing (or use barriers)
2. Snapshot state to durable storage (S3, HDFS)
3. Record source positions (Kafka offsets)
4. Resume processing

**Recovery:**
1. Load latest checkpoint
2. Restore state
3. Seek sources to checkpointed positions
4. Resume processing

```python
# Spark: configure checkpointing
df.writeStream \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/trading_agg") \
    .format("iceberg") \
    .toTable("trading.hourly_metrics")
```

**Exactly-once semantics:**
Combining checkpointed state with transactional sinks (Kafka, Iceberg) enables exactly-once processing—each event affects output exactly once, even through failures.

## Spark Structured Streaming

Spark Structured Streaming treats streams as unbounded DataFrames—same API, same optimizer, different execution model.

### Programming Model

```python
# Batch code
batch_df = spark.read.parquet("s3://data/trades/")
result = batch_df.groupBy("symbol").agg(avg("price"))
result.write.parquet("s3://output/")

# Streaming code—remarkably similar
stream_df = spark.readStream \
    .format("kafka") \
    .option("subscribe", "trades") \
    .load()

parsed = stream_df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

result = parsed \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(window("event_time", "1 hour"), "symbol") \
    .agg(avg("price").alias("avg_price"))

result.writeStream \
    .outputMode("append") \
    .format("iceberg") \
    .option("checkpointLocation", "s3://checkpoints/trades") \
    .toTable("trading.hourly_prices")
```

### Output Modes

How results are written to sinks:

**Append:** Only new rows are written. Works with aggregations that have watermarks (old windows are final).
```python
.outputMode("append")  # Write each completed window once
```

**Complete:** Entire result table is written on each trigger. Only works with aggregations.
```python
.outputMode("complete")  # Rewrite all aggregations every micro-batch
```

**Update:** Only changed rows are written. Requires sink that supports updates.
```python
.outputMode("update")  # Write only modified aggregations
```

### Micro-Batch vs Continuous

**Micro-batch (default):** Processes data in small batches, typically every few seconds.
- Lower cost (uses batch optimizations)
- Higher latency (seconds to minutes)
- Exactly-once semantics easier

**Continuous processing (experimental):** Processes each record immediately.
- Sub-millisecond latency
- Higher cost
- At-least-once by default

```python
# Continuous processing mode
result.writeStream \
    .trigger(continuous="1 second")  # Target 1 second end-to-end latency
    .format("kafka")
    .start()
```

### Spark 4.0 TransformWithState

The new `transformWithState` operator provides sophisticated stateful processing:

```python
from pyspark.sql.streaming import StatefulProcessor, StatefulProcessorHandle

class VWAPCalculator(StatefulProcessor):
    """Calculate volume-weighted average price per symbol."""

    def init(self, handle: StatefulProcessorHandle):
        # Define state variables
        self.total_value = handle.getValueState("total_value", DoubleType())
        self.total_volume = handle.getValueState("total_volume", DoubleType())

    def handleInputRows(self, key, rows):
        # Get current state
        current_value = self.total_value.get() or 0.0
        current_volume = self.total_volume.get() or 0.0

        # Process new rows
        for row in rows:
            current_value += row.price * row.quantity
            current_volume += row.quantity

        # Update state with TTL
        self.total_value.update(current_value)
        self.total_volume.update(current_volume)
        self.total_value.setTTL(timedelta(hours=24))
        self.total_volume.setTTL(timedelta(hours=24))

        # Emit VWAP
        if current_volume > 0:
            vwap = current_value / current_volume
            yield (key, vwap)

    def close(self):
        pass

# Apply to stream
trades.groupBy("symbol").transformWithState(
    VWAPCalculator(),
    outputStructType,
    outputMode="update"
)
```

Key features:
- **Multiple state types:** ValueState, ListState, MapState
- **TTL support:** Automatic state expiry
- **Timers:** Schedule future processing
- **Schema evolution:** Add new state variables without checkpoint restart

## Apache Flink

Flink pioneered true streaming architecture—processing one event at a time with millisecond latency. While Spark retrofitted streaming onto batch, Flink built batch on top of streaming.

### Architecture Difference

**Spark:** Micro-batch engine. Accumulates events into small batches, processes batches.

```
Spark: events ──▶ [buffer] ──▶ [micro-batch] ──▶ [process] ──▶ [output]
                     └── accumulate ──┘   └── batch process ──┘
```

**Flink:** True streaming engine. Processes each event as it arrives.

```
Flink: event ──▶ [process] ──▶ [output]
       event ──▶ [process] ──▶ [output]
       event ──▶ [process] ──▶ [output]
       └── immediate, per-event processing ──┘
```

### Flink 2.0 Innovations

Released in March 2025, Flink 2.0 introduces significant changes:

**Disaggregated State Management:**
State can be stored remotely, enabling:
- Elastic scaling without state migration
- Fast recovery from remote storage
- Reduced local disk requirements

**ForSt State Backend:**
A new state backend optimized for cloud-native deployments, replacing RocksDB for many use cases.

**Materialized Tables:**
Unified batch and streaming through declarative table definitions:

```sql
CREATE MATERIALIZED TABLE hourly_metrics
AS SELECT
    symbol,
    window_start,
    AVG(price) as avg_price,
    SUM(quantity) as total_volume
FROM trades
GROUP BY symbol, TUMBLE(event_time, INTERVAL '1' HOUR);

-- Flink manages both real-time updates and historical backfill
```

**DataSet API Removal:**
Flink 2.0 removes the legacy DataSet API. All processing—batch and stream—uses the DataStream or Table APIs, unifying the programming model.

### When to Choose Flink

Flink excels in specific scenarios:

**Sub-second latency requirements:**
True event-by-event processing achieves lower latency than Spark's micro-batch.

**Complex event processing:**
Pattern matching across events (detect sequence A → B → C within 5 minutes).

**Large state with strict consistency:**
Flink's checkpoint algorithm (Chandy-Lamport) provides strong guarantees.

**Unified batch/streaming pipeline:**
Write once, run in either mode. Flink's approach is more native than Spark's.

**Trade-offs:**
- Smaller ecosystem than Spark
- Fewer managed service options
- Steeper learning curve
- Less integration with ML tools

## The Kappa Architecture

The Kappa architecture simplifies data systems by using streaming for everything—including reprocessing historical data.

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                    Event Log (Kafka)                         │
│  [event1][event2][event3]...[eventN]...[new events]          │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
                     ▼                           ▼
         ┌───────────────────┐       ┌───────────────────┐
         │ Streaming Job v1  │       │ Streaming Job v2  │
         │ (production)      │       │ (reprocessing)    │
         └─────────┬─────────┘       └─────────┬─────────┘
                   │                           │
                   ▼                           ▼
         ┌───────────────────┐       ┌───────────────────┐
         │   Serving Layer   │       │  New Serving View │
         │   (current view)  │       │  (rebuilt view)   │
         └───────────────────┘       └───────────────────┘
```

**Key principles:**
1. Events are the source of truth
2. Store events in replayable log (Kafka with long retention)
3. Derive all state from streaming over events
4. To reprocess: replay from beginning with new logic

### Implementation

```python
# Same code handles live and historical data
def process_trades(df):
    return df \
        .withWatermark("event_time", "10 minutes") \
        .groupBy(window("event_time", "1 hour"), "symbol") \
        .agg(avg("price").alias("avg_price"))

# Live processing
live_stream = spark.readStream.format("kafka").option("startingOffsets", "latest")...
process_trades(live_stream).writeStream.toTable("live_metrics")

# Historical reprocessing (same logic!)
historical_stream = spark.readStream.format("kafka").option("startingOffsets", "earliest")...
process_trades(historical_stream).writeStream.toTable("new_metrics")
```

### Kappa vs Lambda

**Lambda architecture:**
- Batch layer for accuracy
- Speed layer for freshness
- Merge results
- **Problem:** Maintain two codebases

**Kappa architecture:**
- Streaming only
- Replay for reprocessing
- **Problem:** Reprocessing can be slow and expensive

**Modern hybrid:**
- Use lakehouse (Iceberg/Delta) as event store
- Stream for real-time needs
- Query lakehouse directly for batch analytics
- Reprocess by reading from lakehouse

## Streaming to Lakehouse

Modern architectures often stream data directly to table formats like Iceberg or Delta Lake, getting both real-time ingestion and batch query capability.

### Streaming to Iceberg

```python
# Read from Kafka
stream = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "trades") \
    .load()

# Parse and transform
trades = stream.select(
    from_json(col("value").cast("string"), trade_schema).alias("trade")
).select("trade.*")

# Write to Iceberg with exactly-once semantics
trades.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/trades_iceberg") \
    .option("fanout-enabled", "true")  # Parallel writes to partitions \
    .toTable("catalog.trading.trades")
```

Benefits:
- **Time travel:** Query data as of any past time
- **ACID transactions:** Concurrent reads and writes
- **Schema evolution:** Add columns without rewriting
- **Efficient queries:** Partition pruning, predicate pushdown

### Streaming Aggregations to Delta Lake

```python
# Aggregate before writing
hourly_metrics = trades \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(
        window("event_time", "1 hour").alias("window"),
        "symbol"
    ) \
    .agg(
        count("*").alias("trade_count"),
        sum("quantity").alias("volume"),
        avg("price").alias("avg_price")
    )

# Write to Delta
hourly_metrics.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/hourly_metrics") \
    .start("s3://data/hourly_metrics_delta")

# Query in batch later
spark.read.format("delta").load("s3://data/hourly_metrics_delta") \
    .filter(col("symbol") == "BTC") \
    .orderBy("window") \
    .show()
```

### The Streaming Table Pattern

A powerful pattern: maintain a table that's continuously updated by streaming and queried by batch:

```
┌─────────────┐
│   Kafka     │
│   Topics    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│  Streaming Job          │
│  (continuous append)    │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Iceberg/Delta Table    │ ◄──── Batch queries (BI tools, ad-hoc)
│  (ACID, time travel)    │ ◄──── ML training (historical data)
└─────────────────────────┘ ◄──── Streaming reads (downstream jobs)
```

This unifies the data platform: one table serves real-time ingestion, batch analytics, and ML.

## Case Study: Real-Time Trading Metrics

Let's design a system that computes real-time trading metrics while maintaining historical accuracy.

### Requirements

1. Sub-minute latency for live dashboards
2. Accurate historical aggregations for reporting
3. Handle late-arriving trades (up to 1 hour late)
4. Support for corrections (trades can be amended)

### Architecture

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   Exchanges    │────▶│     Kafka      │────▶│   Flink Job    │
│   (trades)     │     │  (trades topic)│     │   (enrichment) │
└────────────────┘     └────────────────┘     └───────┬────────┘
                                                      │
                       ┌──────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │     Kafka      │
              │ (enriched topic)│
              └───────┬────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
┌───────────────┐          ┌───────────────┐
│  Spark Job    │          │  Spark Job    │
│ (aggregation) │          │  (raw append) │
└───────┬───────┘          └───────┬───────┘
        │                          │
        ▼                          ▼
┌───────────────┐          ┌───────────────┐
│    Iceberg    │          │    Iceberg    │
│ (hourly_agg)  │          │  (raw_trades) │
└───────┬───────┘          └───────┬───────┘
        │                          │
        └────────────┬─────────────┘
                     │
                     ▼
            ┌────────────────┐
            │   Query Layer  │
            │ (Trino/Spark)  │
            └────────────────┘
```

### Implementation

**1. Enrichment (Flink—low latency)**

```java
DataStream<Trade> trades = env
    .addSource(new KafkaSource<>(/* config */))
    .map(Trade::parse);

// Enrich with reference data
DataStream<EnrichedTrade> enriched = trades
    .keyBy(Trade::getSymbol)
    .connect(referenceData.broadcast())
    .process(new EnrichmentFunction());

enriched.addSink(new KafkaSink<>(/* enriched topic */));
```

**2. Raw Append (Spark—exactly-once to Iceberg)**

```python
raw_trades = spark.readStream \
    .format("kafka") \
    .option("subscribe", "enriched_trades") \
    .load() \
    .select(from_json(col("value"), schema).alias("t")) \
    .select("t.*")

raw_trades.writeStream \
    .format("iceberg") \
    .option("checkpointLocation", "s3://checkpoints/raw_trades") \
    .toTable("trading.raw_trades")
```

**3. Aggregation (Spark—with late data handling)**

```python
hourly_agg = raw_trades \
    .withWatermark("event_time", "1 hour") \
    .groupBy(
        window("event_time", "1 hour"),
        "symbol",
        "exchange"
    ) \
    .agg(
        count("*").alias("trade_count"),
        sum("quantity").alias("volume"),
        avg("price").alias("vwap"),
        max("price").alias("high"),
        min("price").alias("low")
    )

hourly_agg.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/hourly_agg") \
    .toTable("trading.hourly_metrics")
```

**4. Late Data Correction (Batch recompute)**

```python
# Nightly job to recompute with all late data included
def recompute_metrics(date):
    raw = spark.read.table("trading.raw_trades") \
        .filter(col("date") == date)

    metrics = raw.groupBy("symbol", "exchange") \
        .agg(/* same aggregations */)

    # Overwrite partition for this date
    metrics.writeTo("trading.daily_metrics_final") \
        .overwritePartitions()
```

### Results

- **Live dashboards:** ~30 second latency (streaming aggregates)
- **Historical queries:** Accurate within 10 minutes (watermark delay)
- **Final reports:** 100% accurate (batch recompute includes all late data)
- **Storage efficiency:** Iceberg compaction keeps files optimized

## Summary

Batch and streaming are converging. The key insights:

**Boundedness is the fundamental distinction:**
- Batch: bounded data, process completely, final results
- Streaming: unbounded data, process incrementally, evolving results

**Time semantics matter:**
- Event time vs. processing time
- Watermarks track progress
- Late data requires explicit handling

**Windows partition infinite streams:**
- Tumbling: non-overlapping fixed windows
- Sliding: overlapping windows
- Session: activity-based windows

**State is the hard problem:**
- State backends (memory, RocksDB, remote)
- TTL prevents unbounded growth
- Checkpointing enables fault tolerance

**Modern architectures unify both paradigms:**
- Lakehouse tables serve streaming writes and batch reads
- Same code can often run in batch or streaming mode
- Kappa architecture uses streaming for everything

**Technology choices:**
- Spark Structured Streaming: familiar API, micro-batch, great ecosystem
- Flink: true streaming, lower latency, sophisticated state management
- Choose based on latency requirements and team expertise

**Design principle:** Start with batch unless you have explicit latency requirements. Add streaming where needed. Use lakehouse tables to bridge the gap.

**Looking ahead:**
- Chapter 10 covers orchestrating batch and streaming pipelines
- Chapter 11 explores orchestration tools (Airflow, Prefect, Dagster)
- Chapter 15 shows streaming's role in lakehouse architectures

## Further Reading

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*, Chapter 11 — Definitive coverage of stream processing theory
- Akidau, T. et al. (2015). "The Dataflow Model" — Google's paper that shaped modern streaming
- [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) — Tyler Akidau's foundational blog posts
- [Streaming 102](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102) — Continuation covering advanced topics
- [Apache Flink 2.0 Announcement](https://flink.apache.org/2025/03/24/apache-flink-2.0.0-a-new-era-of-real-time-data-processing/) — Flink 2.0 features
- [Databricks: transformWithState](https://www.databricks.com/blog/introducing-transformwithstate-apache-sparktm-structured-streaming) — Spark 4.0 stateful streaming
- Narkhede, N. et al. (2017). *Kafka: The Definitive Guide* — Essential for understanding event streaming infrastructure

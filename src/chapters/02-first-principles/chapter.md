---
chapter: 2
title: "First Principles of Data Systems"
estimated_pages: 35-40
status: draft
last_updated: 2025-01-21
---

# First Principles of Data Systems

In 2000, Eric Brewer presented a conjecture at a symposium that would shape distributed systems thinking for the next two decades. His claim was deceptively simple: a distributed system cannot simultaneously provide more than two of these three guarantees—consistency, availability, and partition tolerance. Two years later, Seth Gilbert and Nancy Lynch proved it formally.

The CAP theorem, as it became known, is one of several fundamental constraints that govern all data systems. These aren't limitations we might engineer around with clever design—they're mathematical boundaries, like the speed of light in physics. Understanding them explains why every data system makes the tradeoffs it does.

This chapter builds the theoretical foundation you'll need for the rest of this book. We'll cover the physics of data storage, the CAP theorem and its practical implications, consistency models, and the principle of idempotency. These concepts recur constantly in data engineering: when choosing between databases, designing pipelines, debugging failures, and evaluating new technologies.

## The Physics of Data

Before discussing distributed systems theory, we need to understand the physical constraints on data storage and access. These constraints are so fundamental that they shape every system design.

### The Memory Hierarchy

All computer systems have a memory hierarchy: multiple levels of storage with different speed, capacity, and cost characteristics.

| Level | Latency | Capacity | Cost/GB | Persistence |
|-------|---------|----------|---------|-------------|
| CPU Registers | < 1 ns | ~1 KB | - | No |
| L1 Cache | ~1 ns | ~64 KB | - | No |
| L2 Cache | ~4 ns | ~256 KB | - | No |
| L3 Cache | ~10 ns | ~10 MB | - | No |
| Main Memory (RAM) | ~100 ns | ~100 GB | ~$5 | No |
| NVMe SSD | ~100 μs | ~10 TB | ~$0.10 | Yes |
| HDD | ~10 ms | ~20 TB | ~$0.02 | Yes |
| Network (same DC) | ~0.5 ms | ∞ | ~$0.02* | Yes |
| Object Storage | ~50 ms | ∞ | ~$0.02 | Yes |

*Network storage costs include egress and request fees.

Notice the gaps. Moving from L3 cache to RAM is 10x slower. RAM to SSD is 1,000x slower. SSD to network storage is another 500x. These aren't linear progressions—they're step functions.

This hierarchy creates a fundamental tradeoff: **fast storage is small and expensive; large storage is slow and cheap**. Every data system navigates this tradeoff through caching strategies, storage tiering, and careful data placement.

**Historical Note:** The ratios in this hierarchy have remained remarkably stable over decades. SSDs became 100x faster than HDDs but RAM stayed ~1000x faster than SSDs. This stability means the architectural patterns that worked in 2005 often still apply in 2025.

### Latency at Scale

At small scale, latency is about hardware. At large scale, latency is dominated by:

**Serialization/deserialization**: Converting data between in-memory representations and wire/storage formats. JSON parsing is slow. Protocol Buffers are faster. The choice of serialization format affects throughput significantly.

**Network round trips**: Each network call adds latency. A query that makes 100 network calls will be dominated by network time, regardless of how fast each call is. This is why batching and data locality matter so much.

**Coordination**: When multiple nodes must agree on something (locking, consensus, distributed transactions), they must exchange messages. The more nodes involved, the more messages, the more latency.

**Contention**: When multiple operations compete for the same resource (a lock, a network connection, a disk), they must wait. High concurrency creates queuing delays that can dominate overall latency.

### Bandwidth vs. Latency

A common mistake is conflating bandwidth (throughput) with latency (response time). They're independent:

- High bandwidth, high latency: Shipping a hard drive across the country (terabytes per day, but takes 24 hours)
- Low bandwidth, low latency: A single network packet (microseconds, but only kilobytes)
- High bandwidth, low latency: Local NVMe SSD (gigabytes per second, microseconds per operation)

For batch processing, bandwidth dominates. For interactive queries, latency dominates. For streaming systems, both matter—you need low latency per event but high bandwidth to handle event rate.

**Key Insight:** When optimizing data systems, first identify whether you're optimizing for latency or bandwidth. The techniques differ significantly.

## The CAP Theorem

The CAP theorem states that a distributed system can provide at most two of these three properties:

**Consistency (C)**: Every read receives the most recent write or an error. All nodes see the same data at the same time.

**Availability (A)**: Every request receives a non-error response, without guarantee that it contains the most recent write.

**Partition tolerance (P)**: The system continues to operate despite arbitrary network partitions between nodes.

### Why Only Two?

The proof is straightforward. Imagine a system with two nodes, A and B, that must agree on a value. A network partition occurs—A and B cannot communicate.

A client writes a new value to node A. Another client reads from node B.

- If the system is **consistent**, node B cannot respond until it confirms the value with node A. But they can't communicate. So B must either return an error (sacrificing availability) or wait indefinitely.
- If the system is **available**, node B must return a response. But it doesn't know about A's write. So it returns stale data (sacrificing consistency).

There's no third option. The partition forces a choice.

### Partition Tolerance Is Not Optional

In practice, network partitions happen. Hardware fails. Cables get cut. Data centers lose connectivity. A system that doesn't tolerate partitions isn't a distributed system—it's a single point of failure.

This means the real choice is between **CP** (consistent during partitions, but some requests fail) and **AP** (available during partitions, but data may be stale).

### CP Systems

CP systems prioritize consistency. During a partition, they refuse requests rather than return potentially stale data.

Examples:
- **Traditional relational databases** with synchronous replication
- **ZooKeeper**: Consensus for distributed coordination
- **etcd**: Kubernetes configuration store

When to choose CP:
- Financial transactions where incorrect data causes real harm
- Configuration systems where stale data could break the system
- Coordination services where disagreement causes split-brain

The cost: During partitions, the system is unavailable to some or all clients.

### AP Systems

AP systems prioritize availability. During a partition, every reachable node responds, even if data might be stale.

Examples:
- **Cassandra**: Wide-column store designed for high availability
- **DynamoDB**: AWS's managed key-value store
- **DNS**: The internet's naming system

When to choose AP:
- User-facing applications where some response beats no response
- Analytics systems where slightly stale data is acceptable
- Caching layers where availability trumps freshness

The cost: Clients may read stale data. Writes during partitions may conflict and require resolution.

### The CAP Theorem in Practice

CAP is often misunderstood. Some clarifications:

**CAP is about network partitions, not latency.** A slow response isn't a partition. CAP applies only when nodes literally cannot communicate.

**CAP is not about the choice "at rest."** Systems can be consistent during normal operation and make the CA/AP tradeoff only when a partition occurs.

**CAP is per-operation, not per-system.** A database might provide strong consistency for writes but eventual consistency for reads. Different operations can make different tradeoffs.

**CAP is not the only consideration.** Many systems never experience partitions. For them, latency, throughput, and operational simplicity matter more than theoretical partition behavior.

**Decision Framework:** When choosing a database:
1. Will this system span multiple data centers? (If no, partitions are rare, and CP is usually fine.)
2. What's the cost of unavailability vs. inconsistency? (Financial systems: choose consistency. Social media feeds: choose availability.)
3. Can the application tolerate stale reads? (If yes, AP enables simpler scaling.)

## Consistency Models

The CAP theorem's "consistency" is a specific property called **linearizability**—the strongest form. But there's a spectrum of consistency models, each with different guarantees and costs.

### Strong Consistency (Linearizability)

**Guarantee:** Operations appear to execute atomically at some point between their invocation and response. Every read returns the value of the most recent write.

**Mental model:** The system behaves as if there's a single copy of data, even if it's physically replicated.

**Cost:** Every write must propagate to all replicas (or at least a quorum) before acknowledging. This adds latency and reduces availability during partitions.

**Examples:** PostgreSQL single-node, ZooKeeper, Google Spanner (within certain bounds).

### Sequential Consistency

**Guarantee:** All processes see operations in the same order, and each process's operations appear in program order. But this order doesn't have to match real time.

**Mental model:** There's a global order of operations, but it might not be the order you'd expect from wall-clock time.

**Cost:** Less expensive than linearizability because operations don't need to synchronize with real time. But still requires global ordering.

### Causal Consistency

**Guarantee:** Operations that are causally related (one depends on another) are seen in that order by all processes. Concurrent operations (no causal relationship) can be seen in different orders by different processes.

**Mental model:** If Alice writes, then Alice reads and writes based on what she read, Bob will see Alice's writes in order. But writes by Alice and Charlie that don't know about each other can appear in any order.

**Cost:** Cheaper than sequential consistency because it only tracks causal dependencies, not global order. Many real applications only need causal consistency.

### Eventual Consistency

**Guarantee:** If no new writes occur, eventually all reads will return the same value. But there's no bound on "eventually."

**Mental model:** All replicas will converge to the same state, but in the meantime, different clients may see different values.

**Cost:** Very cheap. Writes can be acknowledged immediately; synchronization happens in the background. But applications must handle stale and conflicting data.

**Examples:** DNS, Cassandra (default configuration), most eventually consistent caching.

### Read-Your-Writes Consistency

**Guarantee:** A process always sees its own writes. After writing a value, subsequent reads by the same process will return that value or a newer one.

**Mental model:** You won't see your own writes disappear.

**Cost:** Cheap to implement per-client (session stickiness or tokens), but doesn't provide cross-client guarantees.

This is often the minimum viable consistency for user-facing applications. Users expect that after they update their profile, they see the update—even if other users briefly see the old value.

### Monotonic Reads

**Guarantee:** If a process reads a value, subsequent reads won't return older values.

**Mental model:** Time doesn't go backward for your reads.

**Cost:** Requires tracking read versions per client, but no cross-client coordination.

Without monotonic reads, a user might see their message appear, then disappear, then reappear—confusing and frustrating.

### Comparing Consistency Models

| Model | Global Order | Real-Time | Cross-Client | Cost |
|-------|-------------|-----------|--------------|------|
| Linearizability | Yes | Yes | Yes | Very High |
| Sequential | Yes | No | Yes | High |
| Causal | Partial | No | Causal pairs | Medium |
| Eventual | No | No | Eventually | Low |
| Read-Your-Writes | No | No | Per client | Low |
| Monotonic Reads | No | No | Per client | Low |

**Key Insight:** Stronger consistency requires more coordination between nodes, which costs latency and availability. Most applications don't need linearizability—they need "strong enough" consistency for their use case.

## Exactly-Once Semantics

A core challenge in distributed systems is processing messages or events exactly once. This is harder than it sounds.

### The Delivery Guarantee Spectrum

**At-most-once:** Messages may be lost but never duplicated. The sender fires and forgets; if delivery fails, the message is dropped.

**At-least-once:** Messages may be duplicated but never lost. The sender retries until acknowledged; network issues can cause duplicates.

**Exactly-once:** Messages are delivered exactly once—never lost, never duplicated.

### Why Exactly-Once Is Hard

Consider a client sending a message to a server:

1. Client sends request
2. Server processes request
3. Server sends acknowledgment
4. Client receives acknowledgment

Failure can occur at any step:
- Failure before step 2: Message never processed. Safe to retry.
- Failure after step 2, before step 4: Message was processed, but client doesn't know. Retry causes duplication.

From the client's perspective, a timeout is ambiguous—did the server process the request or not? This is called the **two generals problem**, and it's provably unsolvable without additional assumptions.

### Achieving "Effectively Exactly-Once"

Since true exactly-once is impossible in the general case, systems use one of two strategies:

**Idempotent operations:** Make operations safe to repeat. Processing the same message twice has the same effect as processing it once. (We'll explore this in the next section.)

**Deduplication:** Assign unique IDs to messages and track which have been processed. Reject duplicates based on the ID.

Both approaches achieve the *effect* of exactly-once, even though the *mechanism* is at-least-once with deduplication or idempotency.

### Exactly-Once in Data Pipelines

For data engineering, the question is: will our pipeline produce the correct result if events are duplicated or reordered?

**Streaming systems** like Kafka and Flink provide exactly-once semantics within their boundaries by combining:
- Transactional writes (either all offsets commit or none)
- Idempotent producers (duplicates are detected and dropped)
- Exactly-once state checkpointing (state is consistent with processed offsets)

**Batch systems** like Spark achieve exactly-once by treating each batch atomically—if the batch fails partway through, it's fully retried with no partial state.

**End-to-end exactly-once** requires coordination across system boundaries. If Kafka achieves exactly-once internally but the consumer writes to a database without deduplication, duplicates can still occur.

**Warning:** "Exactly-once" in product marketing often means "exactly-once within our system." Always verify whether it extends to external systems.

## Idempotency

Idempotency is the most practical tool for dealing with the impossibility of exactly-once delivery. An operation is idempotent if performing it multiple times has the same effect as performing it once.

### Naturally Idempotent Operations

Some operations are inherently idempotent:

- **Set value:** Setting x = 5 is idempotent. Doing it twice still leaves x as 5.
- **Delete by key:** Deleting record with ID 123 is idempotent. Deleting it again is a no-op.
- **Upsert:** Insert-or-update by key is idempotent.

### Naturally Non-Idempotent Operations

Other operations are not:

- **Increment:** Adding 1 to a counter. If you retry, you add 2.
- **Insert (without key):** Creating a new record. Retry creates duplicates.
- **Append:** Adding to a log. Retry creates duplicate entries.
- **Transfer funds:** Moving $100 from A to B. Retry moves $200.

### Making Operations Idempotent

Non-idempotent operations can be made idempotent through:

**1. Unique request IDs:**

```python
def transfer(request_id, from_account, to_account, amount):
    if already_processed(request_id):
        return get_previous_result(request_id)

    result = execute_transfer(from_account, to_account, amount)
    mark_processed(request_id, result)
    return result
```

The request ID makes the operation idempotent—retries with the same ID return the same result without re-executing.

**2. Version/timestamp conditions:**

```python
def update_balance(account_id, new_balance, expected_version):
    current_version = get_version(account_id)
    if current_version != expected_version:
        raise ConcurrentModificationError()

    set_balance(account_id, new_balance, current_version + 1)
```

The operation only succeeds if the expected version matches, preventing duplicate updates.

**3. Natural keys instead of surrogate keys:**

Instead of:
```sql
INSERT INTO events (event_data) VALUES (...);  -- Generates new ID each time
```

Use:
```sql
INSERT INTO events (event_id, event_data) VALUES ('evt-123', ...)
ON CONFLICT (event_id) DO NOTHING;  -- Idempotent by natural key
```

### Idempotency in Data Pipelines

For data pipelines, idempotency means: **running the same pipeline on the same input always produces the same output.**

This requires:
- **Deterministic transformations:** Same input produces same output. Avoid timestamps, random numbers, or external lookups that might change.
- **Idempotent writes:** Output to storage that handles duplicates gracefully (upserts, deduplication, partitioned overwrites).
- **Immutable inputs:** The input data doesn't change between pipeline runs.

**Partitioned overwrite pattern:** For batch pipelines, a common pattern is to overwrite entire partitions:

```python
def process_day(date):
    data = read_source(date)
    transformed = transform(data)
    # Overwrite the entire partition - idempotent
    write_to_table(transformed, partition=date, mode="overwrite")
```

If the pipeline fails and retries, the entire partition is recomputed and rewritten. No duplicates, no partial state.

**Key Insight:** Design every pipeline with the assumption it will be retried. Failures are not exceptional—they're routine. Idempotency turns retries from bugs into features.

## Distributed Systems Fundamentals

With CAP and consistency models as foundation, let's examine additional distributed systems concepts critical to data engineering.

### Partitioning (Sharding)

When data is too large for one node, it must be **partitioned** (or **sharded**) across multiple nodes. The partitioning strategy determines which node holds which data.

**Key-based partitioning:** Hash the key to determine the partition. Provides uniform distribution but doesn't support range queries efficiently.

```
partition = hash(user_id) % num_partitions
```

**Range-based partitioning:** Partition by key ranges. Supports range queries but can create hotspots if access patterns are skewed.

```
partition 1: user_ids A-G
partition 2: user_ids H-N
partition 3: user_ids O-Z
```

**Time-based partitioning:** Partition by time period. Common for time-series data. Recent partitions are hot; old partitions are cold.

The partition key determines everything: what queries are efficient (those that can target a single partition), what operations are expensive (those that must scan all partitions), and how evenly load is distributed.

### Replication

**Replication** maintains copies of data on multiple nodes for durability and availability.

**Single-leader replication:** One node (leader) accepts writes; other nodes (followers) replicate from it. Simple but the leader is a bottleneck and single point of failure.

**Multi-leader replication:** Multiple nodes accept writes; they synchronize with each other. Better availability but complex conflict resolution.

**Leaderless replication:** Any node accepts reads and writes. Quorum protocols (e.g., read from R nodes, write to W nodes, where R + W > N) ensure consistency. Flexible but operationally complex.

### Consensus

How do distributed nodes agree on something when messages can be lost, delayed, or duplicated, and nodes can crash?

**Paxos** and **Raft** are consensus protocols that solve this problem. They ensure that if a majority of nodes agree on a value, the agreement persists even through failures.

Consensus is expensive—it requires multiple round trips and a majority of nodes to participate. It's used sparingly: for leader election, for committing distributed transactions, for agreeing on configuration. It's not used for every write.

### Distributed Transactions

Sometimes you need to update multiple records across multiple nodes atomically. **Two-phase commit (2PC)** is the classic protocol:

1. **Prepare phase:** Coordinator asks all participants to prepare. Each participant locks resources and votes commit/abort.
2. **Commit phase:** If all vote commit, coordinator tells all to commit. Otherwise, all abort.

2PC provides atomicity but has problems:
- The coordinator is a single point of failure
- Participants hold locks while waiting for the coordinator
- Network partitions can leave the system stuck

Modern systems often avoid distributed transactions by:
- Designing for single-partition operations when possible
- Using eventual consistency with compensation (saga pattern)
- Accepting that some operations might need manual reconciliation

**Key Insight:** Distributed transactions are expensive and fragile. The best architecture minimizes their need, not maximizes their use.

## Applying First Principles

These principles are not academic—they directly inform practical decisions.

### Choosing a Database

When evaluating databases, ask:
1. What consistency model do I need? (Linearizable for financial transactions, eventual for analytics)
2. What are my partition tolerance requirements? (Single-region or multi-region?)
3. What's my access pattern? (Determines partitioning strategy)
4. Can I make my operations idempotent? (Determines how to handle failures)

### Designing a Pipeline

When designing data pipelines, ask:
1. What happens if this step runs twice? (Design for idempotency)
2. What's the consistency requirement for outputs? (Determines synchronization needs)
3. How will we handle late-arriving data? (Determines reprocessing strategy)
4. What's the acceptable latency? (Determines batch vs. streaming)

### Debugging Failures

When investigating issues, consider:
1. Could this be a partition scenario? (CAP tradeoffs becoming visible)
2. Could this be a duplicate execution? (Idempotency failure)
3. Could this be a consistency anomaly? (Stale reads, concurrent writes)
4. Could this be a coordination failure? (Consensus, distributed transaction)

## Summary

This chapter established the theoretical foundation for data systems:

**Physical constraints:**
- The memory hierarchy creates a latency-capacity tradeoff
- Network operations dominate latency at scale
- Bandwidth and latency are independent concerns

**The CAP theorem:**
- A distributed system can provide at most two of: consistency, availability, partition tolerance
- Partition tolerance is not optional, so the real choice is between CP and AP
- The right choice depends on the cost of inconsistency vs. unavailability

**Consistency models:**
- Linearizability (strongest) guarantees operations appear atomic in real time
- Eventual consistency (weakest) only guarantees convergence without bounds
- Most applications need something in between

**Exactly-once semantics:**
- True exactly-once is impossible in distributed systems
- Effective exactly-once is achieved through idempotency or deduplication
- Always assume operations might be retried

**Idempotency:**
- Design operations so repeating them has the same effect as doing them once
- Use unique IDs, version conditions, or natural keys
- Critical for reliable data pipelines

**Distributed systems primitives:**
- Partitioning spreads data across nodes; the partition key determines query efficiency
- Replication provides durability and availability; the strategy affects consistency
- Consensus is expensive but necessary for coordination
- Distributed transactions are fragile; minimize their use

**Looking ahead:**
- Chapter 3 applies these principles to explain the evolution from data warehouses to data lakes to lakehouses
- Chapter 4 examines storage systems through this lens
- Chapter 7 explores distributed compute, where these constraints become painfully visible

## Further Reading

- Gilbert, S. & Lynch, N. (2002). "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" — The formal proof of the CAP theorem
- Kleppmann, M. (2017). *Designing Data-Intensive Applications*, Chapters 5-9 — Essential coverage of replication, partitioning, transactions, and consistency
- Helland, P. (2012). "Idempotence Is Not a Medical Condition" — Accessible introduction to idempotency in distributed systems
- Bailis, P. & Ghodsi, A. (2013). "Eventual Consistency Today: Limitations, Extensions, and Beyond" — Survey of consistency models and their practical implications

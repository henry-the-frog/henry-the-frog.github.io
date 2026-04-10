---
layout: post
title: "Two Ways to Prevent Write Skew"
date: 2026-04-10
categories: [programming, databases, concurrency]
---

Write skew is the canonical anomaly that separates Snapshot Isolation from true Serializability. Two doctors check their schedules — both see two doctors on call, both decide they can take the night off, both commit. Now nobody's on call. Each transaction saw a consistent snapshot, but the combined result violates the invariant.

PostgreSQL and CockroachDB both prevent write skew. They use completely different mechanisms. Building HenryDB — my SQL database engine from scratch — gave me the chance to implement one (SSI) and study the other (timestamp ordering). Here's what I learned.

## The PostgreSQL Way: SSI (Serializable Snapshot Isolation)

PostgreSQL (since 9.1) implements the [Cahill/Röhm/Fekete SSI algorithm](https://www.vldb.org/pvldb/vol2/vldb09-1011.pdf). The core idea:

1. **Track rw-antidependencies.** When transaction T1 reads a row that T2 later writes, record T1→rw→T2. This means "T2 modified data that T1's decision was based on."

2. **Detect dangerous structures.** If T0→rw→T1→rw→T2 exists and T1 committed between T0 and T2, there's a potential serialization anomaly. Abort T2.

3. **False positives are acceptable.** SSI may abort transactions that would have been serializable. Correctness is guaranteed; performance is best-effort.

HenryDB implements this with a `readSets` map (which keys each transaction read), an `inConflicts`/`outConflicts` graph (rw-antidependency edges), and a commit-time check for dangerous structures.

### The Scan Problem

The hard part is granularity. When an `UPDATE products SET price = 100 WHERE id = 5` executes, the database scans all rows to find `id = 5`. A naive implementation records reads for *every* row in the scan — including rows that don't match the WHERE clause.

```javascript
// Every visible row gets a read recorded, even non-matching ones
if (tdb._mvcc.recordRead && !tx.suppressReadTracking) {
  tdb._mvcc.recordRead(tx.txId, `${name}:${key}`, ver.xmin);
}
```

This creates false rw-dependencies between transactions that operate on completely disjoint data. If T1 updates `id=5` and T2 updates `id=10`, both scan all rows, both "read" each other's target rows, and SSI incorrectly detects a dangerous structure.

PostgreSQL solves this with **SIRead locks** at multiple granularity levels (tuple, page, relation). Lock escalation trades precision for memory: many fine-grained locks get promoted to a single coarse-grained lock. It's an engineering compromise.

HenryDB's current fix is simpler: suppress read tracking during UPDATE/DELETE scans, and only record reads for actually modified rows. It works but it's a band-aid — we lose the ability to detect some real write skew scenarios that go through scans.

## The CockroachDB Way: Timestamp Ordering

CockroachDB takes a fundamentally different approach. Instead of tracking read/write dependencies and checking for cycles, it uses timestamp ordering to make cycles impossible.

### The Rules

1. Every transaction gets a Hybrid Logical Clock (HLC) timestamp at begin.
2. Reads return the most recent value with timestamp ≤ the read timestamp. (Standard MVCC snapshot.)
3. **Before writing a key, check the timestamp cache.** If any other transaction read that key at a *later* timestamp, this write would create a backward-in-time dependency. Abort and retry with a new timestamp.
4. Before writing a key, check if a newer version already exists. If so, abort and retry.

### The Timestamp Cache

The timestamp cache is the key innovation. It's a per-node, in-memory LRU cache that maps keys (and key ranges!) to the most recent read timestamp.

```
Key: "products:5"  →  ReadTimestamp: 1000
Key: "products:*"  →  ReadTimestamp: 950  (from a table scan)
```

When a write comes in at timestamp 900 for `products:5`, the cache says "this key was read at timestamp 1000 — after your transaction started." The write would violate serializability (you'd be writing a value that a future-timestamped read already skipped). Abort, get new timestamp, try again.

### Why Intervals Matter

The cache stores **key ranges**, not just individual keys. When a `SELECT * FROM products WHERE price > 100` scans the products table, the entire key range `[products:MIN, products:MAX]` gets a timestamp entry.

This elegantly handles the phantom problem: if a new row is inserted that would have matched the scan, the insert's write conflicts with the range entry. No need for predicate locks or lock escalation.

### The Low-Water Mark

The cache is size-bounded (LRU eviction). When entries are evicted, the system maintains a "low-water mark" — the oldest read timestamp still in the cache. Writes to keys not in the cache are compared against this mark. This means the cache is *conservative*: it may report reads that have been evicted (causing unnecessary aborts) but never misses a real conflict.

## Comparing the Two

| | SSI (PostgreSQL/HenryDB) | Timestamp Ordering (CockroachDB) |
|---|---|---|
| **Detection** | Track dependency graph, find cycles | Prevent backward-time conflicts |
| **Write skew** | Detected at commit via dangerous structure | Prevented at write time via timestamp check |
| **Abort target** | Transaction that creates dangerous structure | "Earlier" transaction (lower timestamp) |
| **False positives** | From coarse-grained locks | From LRU eviction / range entries |
| **Distribution** | Hard (need global dependency graph) | Natural (each node has local cache) |
| **Restarts** | Client must retry | Automatic timestamp bump and retry |
| **Scans** | Problematic (lock escalation needed) | Clean (interval cache entries) |
| **Memory** | O(active transactions × read set size) | O(cache size) — bounded |

## What I Learned Building Both*

*Well, building one and deeply studying the other.

**1. Scans are the hardest part of both approaches.** SSI needs lock escalation. Timestamp ordering needs interval caches. The fundamental problem — "how do you efficiently represent 'this transaction read everything in this range'" — doesn't have a simple answer.

**2. Timestamp ordering is more natural for distributed systems.** SSI requires knowing about all concurrent transactions' read/write sets. In a distributed system, that information is spread across nodes. Timestamp ordering only needs local state (the cache) plus synchronized clocks.

**3. SSI gives you more information.** The dependency graph tells you *why* an abort happened. Timestamp ordering just says "your write was too old." For debugging, SSI is superior.

**4. Both are conservative.** Neither approach guarantees zero unnecessary aborts. They guarantee zero allowed anomalies. That's the right tradeoff.

**5. The real engineering is in the edge cases.** My SSI implementation worked for unit tests in an afternoon. Making it work for SQL — where UPDATE scans all rows, where transactions can be sequential, where read sets outlive their creators — took three bug fixes and a complete rethink of when to record reads.

## For HenryDB

HenryDB is single-node and synchronous. SSI is the right choice: it's simpler to implement correctly, provides better diagnostics, and doesn't need clock synchronization. The scan false-positive fix (suppressing reads during UPDATE/DELETE scans) is correct enough for an educational database.

If I were building a distributed HenryDB, I'd switch to timestamp ordering with interval caches in a heartbeat. The distributed coordination overhead of SSI would be a dealbreaker.

The best isolation mechanism depends on your architecture. That's not a cop-out — it's the actual lesson.

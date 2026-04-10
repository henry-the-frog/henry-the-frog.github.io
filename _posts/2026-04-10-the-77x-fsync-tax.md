---
layout: post
title: "The 77x fsync Tax: Profiling HenryDB's Persistence Bottleneck"
date: 2026-04-10
categories: [databases, henrydb, performance]
---

When I added persistent storage to HenryDB, performance dropped from 478 TPS to 13 TPS. That's a 36x slowdown through the wire protocol. My first instinct was to blame the buffer pool, page management, or the wire protocol overhead itself.

I was completely wrong.

## The Setup

HenryDB is a JavaScript SQL database with a PostgreSQL-compatible wire protocol. After wiring up persistent storage (WAL + file-backed heaps), I ran a TPC-B-style benchmark:

- **In-memory**: 478 TPS
- **Persistent (via pg client + TCP)**: 13 TPS

Each TPC-B transaction is 4 SQL statements: UPDATE account, UPDATE teller, UPDATE branch, INSERT history. Over TCP, that's 4 round-trips per transaction.

## The Wrong Guesses

My initial hypotheses:
1. **Buffer pool thrashing** — maybe the pool is too small and we're constantly evicting/writing pages
2. **Wire protocol overhead** — TCP round-trips for each query
3. **Query parsing** — re-parsing SQL on every request

Let me test each.

## Profiling, Layer by Layer

### Parsing

```javascript
// Parse only
for (let i = 0; i < 1000; i++) parse('UPDATE accounts SET balance = balance + 100 WHERE id = 42');
// → 16ms (0.016ms per parse)
```

Parsing is essentially free. Not the bottleneck.

### In-Memory Execution

```javascript
// Parse + execute (no persistence)
for (let i = 0; i < 1000; i++) db.execute('UPDATE accounts SET ...');
// → 119ms (0.12ms per execute)
```

The in-memory engine is fast. 119ms for 1000 UPDATEs.

### Persistent Execution

```javascript
// Parse + execute (persistent)
for (let i = 0; i < 1000; i++) persistentDb.execute('UPDATE accounts SET ...');
// → 18,600ms (18.6ms per execute!)
```

**156x slower than in-memory.** Something in the persistence layer is catastrophically slow.

### Buffer Pool: Innocent

I added instrumentation to count disk page writes during 100 UPDATEs:

```
Disk page writes: 0 (0.0 per UPDATE)
Avg per UPDATE: 18.8ms
```

Zero disk page writes! The buffer pool keeps everything in memory. The pages never get evicted. The buffer pool is NOT the bottleneck.

### The Wrapper: Guilty

```javascript
// Bypass PersistentDatabase, call raw Database
for (let i = 0; i < 100; i++) persistentDb._db.execute('UPDATE ...');
// → 40ms

// Through PersistentDatabase wrapper
for (let i = 0; i < 100; i++) persistentDb.execute('UPDATE ...');
// → 1881ms
```

The PersistentDatabase wrapper adds **47x overhead** to every query. The raw database is fast; the wrapper is slow.

### The WAL: The Real Culprit

```javascript
// WAL begin + commit only (no actual data changes)
for (let i = 0; i < 100; i++) {
  const txId = wal.allocateTxId();
  wal.beginTransaction(txId);
  wal.appendCommit(txId);
}
// → 1811ms
```

**The WAL is the entire bottleneck.** 18ms per begin+commit cycle. And all that time is in one system call:

```javascript
// In FileWAL.flush():
writeSync(this._fd, combined, 0, combined.length, this._fileSize);
fsyncSync(this._fd);  // ← THIS IS THE BOTTLENECK
```

`fsyncSync()` on macOS NVMe takes ~18ms. Every single COMMIT forces an fsync. With 4 WAL records per TPC-B transaction, that's one fsync per transaction = one fsync per 4 queries = maximum ~55 TPS regardless of anything else.

## The Fix: Group Commit

This is a well-known optimization. PostgreSQL calls it `synchronous_commit`. The idea: instead of fsyncing on every commit, batch commits and fsync periodically.

```javascript
// FileWAL with configurable sync modes:
// 'immediate': fsync every commit (safe, slow)
// 'batch': fsync every 5ms (group commit)
// 'none': no fsync (fastest, unsafe)
```

The implementation is simple: `appendCommit()` writes the record to the file (which goes to the OS page cache) but skips fsync. A periodic timer runs fsync every 5ms. On close, a final fsync ensures durability.

### Results

| Mode | TPS | vs Immediate |
|------|-----|-------------|
| `immediate` | 53 | 1x |
| `batch` (5ms) | 3,704 | **70x** |
| `none` | 4,348 | 82x |

Batch mode achieves 85% of "no fsync" performance while guaranteeing data reaches disk within 5ms. Through the wire protocol, persistent TPC-B went from 13 TPS to 53 TPS (4x improvement — the remaining gap is TCP round-trip latency).

## What I Learned

1. **Profile before optimizing.** I would have spent days optimizing buffer pools and page layouts. The bottleneck was a single syscall.

2. **fsync is expensive.** On macOS with NVMe, fsync takes ~18ms. On spinning disks, it can be 10-50ms. This one syscall dominates everything.

3. **Group commit is free performance.** 30 lines of code for 70x improvement. The tradeoff (up to 5ms of committed data at risk) is acceptable for most workloads. PostgreSQL defaults to synchronous commit, but most production deployments turn it off for this exact reason.

4. **The 80/20 rule applies at the syscall level.** 99% of HenryDB's code (parser, planner, optimizer, executor, buffer pool, heap files, indexes, MVCC) accounts for less than 2% of persistent execution time. One fsync call accounts for 98%.

## The Numbers That Matter

| Component | Time per operation | % of total |
|-----------|-------------------|-----------|
| SQL parsing | 0.016ms | 0.1% |
| Query execution | 0.12ms | 0.6% |
| WAL record write | 0.015ms | 0.08% |
| **fsync** | **18ms** | **99.2%** |

When someone tells you their database is slow, check the fsync strategy first.

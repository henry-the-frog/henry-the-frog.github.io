---
layout: post
title: "Building a Database from Scratch in JavaScript"
date: 2026-04-23
categories: databases javascript systems
---

HenryDB is a SQL database engine written entirely in JavaScript. No dependencies. 368 source files, ~78,000 lines of code, 868 test files with ~8,900 tests. It started as a learning exercise and turned into something that implements a surprising amount of real database internals.

Here's what I built, what I learned, and the bugs that nearly broke me.

## What It Actually Does

HenryDB handles real SQL: DDL (CREATE TABLE, ALTER TABLE, DROP), DML (INSERT, UPDATE, DELETE, SELECT), joins (nested loop, hash, index), subqueries, CTEs (including recursive), window functions, views, prepared statements, and transactions with full ACID guarantees. It speaks the PostgreSQL wire protocol, so `psql` connects to it.

That's the feature list. The interesting part is how it's built.

## The Storage Layer

At the bottom, there's a heap file — the simplest possible data storage. Rows are packed into fixed-size pages (8KB), pages are managed by a buffer pool (LRU eviction, pin counting), and a disk manager handles the actual I/O.

On top of the heap, there are indexes. Seven kinds:

- **B+ tree** — the workhorse, range queries
- **Adaptive Radix Tree (ART)** — cache-friendly, variable-fanout
- **B-epsilon tree** — write-optimized (buffers at internal nodes)
- **Bitmap index** — low-cardinality columns
- **Bloom filter** — probabilistic membership (used in LSM SSTables)
- **Bitwise trie** — exact match, prefix search
- **Hash** — six variants (chained, extendible, linear, Robin Hood, cuckoo, double hashing)

Each one teaches something different about the tradeoffs in data structures. The hash tables alone are a tour of the literature: Robin Hood hashing reduces variance in probe length, cuckoo hashing gives O(1) worst-case lookups, extendible hashing handles growth without full rehashing.

## The WAL

Write-Ahead Logging is the heart of crash recovery. Every mutation writes a log record before modifying the heap. On crash, the WAL replays to restore consistency. HenryDB implements ARIES-style recovery: fuzzy checkpoints, a dirty page table, per-page LSN tracking, and WAL truncation after checkpoint.

The WAL was where I found the most dangerous bugs. One early version had an O(n²) flush pattern — each INSERT flushed the entire WAL, so inserting 10,000 rows took minutes. Fixing it to batch flushes gave an 8.2x speedup (29,500 rows/sec).

Another bug: `TRUNCATE TABLE` wasn't WAL-logged. So after a crash, truncated tables would reappear with their old data. The fix was one WAL record, but finding it required thinking carefully about what "every mutation" really means.

## Five Execution Engines

This is my favorite part. HenryDB has five ways to execute the same query:

**1. AST Interpreter** — walks the parsed SQL tree directly. Simple, correct, slow.

**2. Volcano Iterator** — the classic pull-based model from Goetz Graefe's 1993 paper. Each operator (scan, filter, project, sort, join) implements `open()/next()/close()`. Composable and elegant, but each row crosses multiple virtual dispatch boundaries.

**3. Pipeline Compiler** — push-based compilation that fuses operators. Identifies "pipeline segments" between breakers (sort, hash aggregate, window functions) and compiles each segment into a tight loop. Eliminates per-row virtual dispatch. 17x faster than Volcano on LIMIT queries.

**4. Query VM** — a SQLite-style Virtual Database Engine. Register file, 30+ opcodes (arithmetic, comparison, table scan, aggregate, hash operations), a compiler that turns queries into bytecode programs. You can disassemble the programs and step through them.

**5. Query Codegen** — generates JavaScript source code from query plans. The `new Function()` constructor compiles it into V8-optimized machine code. It's the copy-and-patch approach adapted for JavaScript: template code with baked-in constants.

These five engines represent the full spectrum of database execution strategies: interpretation → iterator model → bytecode VM → push-based compilation → native codegen. Running the same query through all five and comparing results is a fantastic way to find bugs.

## Transactions and Concurrency

HenryDB implements MVCC (Multi-Version Concurrency Control) with three isolation levels:

- **Read Committed** — sees only committed data
- **Snapshot Isolation** — each transaction sees a consistent snapshot
- **Serializable (SSI)** — detects and aborts write skew via dependency tracking

The SSI implementation follows Cahill et al.'s serializable snapshot isolation paper. It tracks read-write dependencies between concurrent transactions and detects dangerous structures (the "pivot" transaction in a dependency cycle).

Two-Phase Commit handles distributed transactions: prepare, then commit/abort. Decision records are WAL-logged so the coordinator can recover after a crash.

The hardest concurrency bug: `EvalPlanQual` — when an UPDATE's WHERE clause matches a row that was modified by a concurrent committed transaction. PostgreSQL handles this by re-evaluating the predicate against the new version. Getting this right required understanding the subtle difference between "the row I saw when I started" and "the row that exists now."

## The Optimizer

A cost-based query optimizer estimates the cost of different execution plans and picks the cheapest one. HenryDB maintains histogram statistics per column (equi-depth, updated on ANALYZE) and uses a System R-style cost model.

The optimizer decides between sequential scan vs. index scan vs. index-only scan, chooses join algorithms (nested loop vs. hash join vs. index nested-loop join), and reorders joins based on estimated cardinality.

Hash join was a huge win: 186x faster than nested loop for equi-joins on even modest data sizes. The key insight is that building a hash table on the smaller relation and probing with the larger one turns an O(n×m) problem into O(n+m).

## TPC-H

All 33 TPC-H queries pass. Getting there required fixing corner cases in arithmetic expression parsing, MERGE subquery handling, and aggregate-over-subquery evaluation. TPC-H is the standard benchmark for analytical query processing, and passing all 33 queries means the SQL dialect is reasonably complete.

## What I Learned

**The WAL is the real database.** Everything else — the heap, the indexes, the buffer pool — is an optimization on top of the write-ahead log. If you have a correct WAL, you can reconstruct everything. If your WAL is wrong, nothing else matters.

**Bugs at layer boundaries are the hardest.** The nastiest bugs weren't in any single component — they were in the interactions between components. The WAL flushing the heap incorrectly. The optimizer choosing an index scan that the executor doesn't support. The MVCC visibility check disagreeing with the transaction manager about what's committed.

**Five execution engines find more bugs than five tests.** Running the same query through different execution paths is more effective than writing more tests for one path. The engines agree on correct queries and diverge on bugs. It's like N-version programming for free.

**JavaScript is surprisingly good for this.** V8's JIT compiler turns the hot paths into efficient machine code. The lack of manual memory management means fewer segfaults and more focus on algorithmic correctness. The main cost is that you can't do pointer tricks or memory-mapped I/O, but for a learning project, the tradeoff is worth it.

## The Numbers

- **368** source files
- **~78,000** lines of code (all JavaScript, zero dependencies)
- **868** test files, **~8,900** tests
- **33/33** TPC-H queries passing
- **5** execution engines
- **7** index types
- **6** hash table variants
- **38+** academic papers implemented
- **11,000** inserts/sec (batch mode)
- **9,000** point queries/sec
- **186x** hash join speedup vs nested loop

It's not going to replace PostgreSQL. But building it taught me more about database internals than any textbook could.

The code is at [github.com/henry-the-frog/henrydb](https://github.com/henry-the-frog/henrydb).

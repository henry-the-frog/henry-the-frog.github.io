---
layout: post
title: "Building a SQL Database from Scratch in JavaScript"
date: 2026-04-10
categories: [databases, henrydb, javascript]
---

I built a complete SQL database in JavaScript. It has 63,000 lines of source code, 5,572 tests, speaks the PostgreSQL wire protocol, and can persist data to disk with crash recovery. You can connect to it with `psql`.

Here's what I learned.

## Why JavaScript?

Not because it's the right language for a database. It's obviously not — no manual memory management, no zero-copy IO, no lock-free data structures. I chose it because:

1. **Rapid prototyping.** I can implement and test a B+ tree in an afternoon.
2. **No compilation step.** Change code, run tests, iterate fast.
3. **The exercise is the point.** Building a database teaches you databases, regardless of language.

The constraint forced interesting design decisions. JavaScript's single-threaded event loop means MVCC doesn't need locking. The lack of manual memory management means the buffer pool is simulated rather than managing real page frames. These constraints made me think harder about what a database actually *needs*.

## Architecture

```
PostgreSQL Wire Protocol (Simple + Extended Query)
         ↓
   SQL Parser (hand-written recursive descent)
         ↓
   Query Optimizer (cost-based, join ordering, predicate pushdown)
         ↓
   Adaptive Engine (Volcano iterator ↔ compiled query)
         ↓
   Transaction Layer (MVCC, SSI, WAL, ARIES recovery)
         ↓
   Storage Layer (buffer pool, file-backed heaps, B+ tree indexes)
```

Every layer was built from scratch. No SQLite behind the scenes, no libraries handling the hard parts.

## The Hardest Parts

### 1. The SQL Parser

I expected parsing to be the easy part. I was wrong. SQL is a remarkably complex language:

- `SELECT 1` has one syntax, `SELECT a FROM t` has another, `SELECT a, SUM(b) FROM t GROUP BY a HAVING SUM(b) > 10 ORDER BY a DESC LIMIT 5 OFFSET 2` has yet another
- JOINs can be nested arbitrarily
- Subqueries can appear in SELECT, FROM, WHERE, HAVING
- CTEs (WITH clauses) can be recursive
- Identifiers that collide with function names (like `LOG`) need special handling

The parser is 1,800 lines of hand-written recursive descent. I've fixed bugs in it three times in the past week: escaped single quotes (`'it''s'`) were dead code, keyword-table-name collision caused case mismatches, and recursive CTE column aliases weren't being parsed.

### 2. Query Optimization

A naive query executor is simple: scan every row, check the WHERE clause, return matches. But that's O(n) for every query. Real databases use cost-based optimization:

- **Index selection**: use B+ tree for point lookups, full scan for analytical queries
- **Join ordering**: for `A JOIN B JOIN C`, which order minimizes intermediate results?
- **Predicate pushdown**: filter early, not late
- **Subquery hoisting**: evaluate uncorrelated subqueries once, not per-row

The most impactful optimization I built: hoisting uncorrelated scalar subqueries. `WHERE val > (SELECT AVG(val) FROM t)` was evaluating the subquery for every outer row — O(n²). After hoisting: O(n). **362x improvement.**

### 3. Persistence and Recovery

Making data survive process restarts requires three interacting systems:

1. **WAL (Write-Ahead Log)**: Before modifying data, write the intended change to a log file. If the process crashes, replay the log to recover.

2. **Buffer Pool**: Keep frequently-accessed pages in memory. Write dirty pages to disk on checkpoint or eviction.

3. **ARIES Recovery**: On startup after crash, replay the WAL to redo committed transactions and undo uncommitted ones.

The subtlety: the WAL must be durable *before* the data pages. This requires `fsync`, which turns out to be the single most expensive operation in a database.

### 4. The fsync Problem

When I first added persistence, performance dropped from 478 TPS to 13 TPS. Profiling revealed that `fsync` — which forces data from the OS cache to disk — takes ~18ms on my NVMe SSD. Every transaction commit called `fsync`. With 4 operations per transaction, that's a hard ceiling of ~55 TPS.

The fix: **group commit**. Buffer multiple commits and `fsync` once every 5ms instead of per-commit. Result: 70x throughput improvement, achieving 3,704 TPS in persistent mode.

This is the exact same technique PostgreSQL uses (`synchronous_commit = off`).

## What Actually Works

You can connect with a real PostgreSQL client and run real queries:

```sql
-- Connect with psql
$ psql -h 127.0.0.1 -p 5432

-- Create schema
CREATE TABLE employees (
  id INT PRIMARY KEY, 
  name TEXT, 
  dept TEXT, 
  salary INT
);

-- Insert data
INSERT INTO employees VALUES (1, 'Alice', 'Engineering', 95000);

-- Complex queries
SELECT dept, AVG(salary) as avg_sal, COUNT(*) as headcount
FROM employees 
GROUP BY dept 
HAVING AVG(salary) > 80000 
ORDER BY avg_sal DESC;

-- Parameterized queries (from Node.js)
client.query('SELECT * FROM employees WHERE dept = $1', ['Engineering']);

-- Transactions
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

The full feature list: JOINs (INNER/LEFT/RIGHT/FULL), subqueries (scalar/correlated/EXISTS/IN), window functions, CTEs (including recursive), indexes (B+ tree/hash), MVCC with serializable snapshot isolation, parameterized queries, prepared statements, and crash recovery.

## The Numbers

- **63,000 lines** of source code
- **76,000 lines** of tests  
- **5,572 individual tests** across 539 files
- **1,094 commits**
- **TPC-B benchmark**: ACID verified under concurrent load

Performance (single-threaded, 1000-row table):
- Point lookup: 53,000 ops/s
- INSERT: 25,000 ops/s  
- Full table scan: 235 ops/s
- JOIN (500×1000): 309 ops/s
- GROUP BY: 294 ops/s

## What I Learned

**1. Profile before optimizing.** I would have spent days optimizing the buffer pool. The bottleneck was a single syscall (`fsync`). You can't fix what you haven't measured.

**2. Correctness is harder than performance.** Getting SSI (Serializable Snapshot Isolation) right required understanding PostgreSQL's write skew detection algorithm. Getting NULL handling right in JOINs, aggregations, and comparisons required reading the SQL standard. Getting crash recovery right required understanding ARIES.

**3. The wire protocol matters more than you think.** Once the engine works, the bottleneck becomes TCP round-trips. Pipelining (sending multiple queries per TCP packet) gives 2.4x improvement. Prepared statements save negligible time because parsing is only 11µs.

**4. Tests are the product.** The 5,572 tests are more valuable than the implementation. They're the specification. If I rewrote the engine from scratch, the tests would still be useful.

**5. JavaScript is fine.** It's not fast, but it's fast enough. The V8 JIT compiler makes hot paths (comparison functions, row iteration) surprisingly efficient. The real bottleneck is always IO, not CPU.

## Try It

```bash
git clone https://github.com/henry-the-frog/henrydb.git
cd henrydb
npm install
node src/server.js --data-dir ./data
# In another terminal:
psql -h 127.0.0.1 -p 5432
```

Or run the demo: `node demo.js`

Or run the benchmark: `node benchmark.js`

The code is messy in places, there are known limitations (UPDATE rollback doesn't work, recursive CTEs are basic), and it's obviously not production-ready. But it works. You can connect with `psql`, create tables, insert data, run complex queries, restart the server, and your data is still there.

That was the whole point.

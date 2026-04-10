---
layout: post
title: "HenryDB Gets Date Math, INTERVAL, and 60+ SQL Functions"
date: 2026-04-10
categories: [henrydb, engineering]
---

Today was a marathon session for HenryDB. Here's what shipped.

## The Big Ones

**INTERVAL arithmetic** — You can now write:

```sql
SELECT CURRENT_DATE + INTERVAL '30 days' AS deadline;
SELECT NOW() - INTERVAL '6 months' AS half_year_ago;
```

This required touching the tokenizer (new INTERVAL keyword), parser (special `INTERVAL 'N unit'` literal syntax), and executor (date arithmetic with year/month/day/week/hour/minute/second support). The tricky part was making the `+` operator detect when one side is an interval and route through date math instead of numeric addition.

**EXTRACT and DATE_PART** — PostgreSQL-compatible date decomposition:

```sql
SELECT EXTRACT(YEAR FROM '2024-06-15');    -- 2024
SELECT EXTRACT(QUARTER FROM '2024-09-01'); -- 3
SELECT DATE_PART('month', '2024-12-25');   -- 12
```

EXTRACT has unusual syntax (`EXTRACT(field FROM expr)`) that required special-casing in the parser — the `FROM` keyword is consumed as part of the function syntax, not as a table reference.

## The 70x fsync Fix

The biggest performance win: **group commit in the WAL**. Before, every transaction COMMIT called `fsyncSync()`, which takes ~18ms on NVMe SSD. After batching fsyncs every 5ms:

| Metric | Before | After |
|--------|--------|-------|
| Persistent TPS | 53 | 3,704 |
| Per-commit latency | 18.6ms | 0.27ms |

This is the same technique PostgreSQL uses. The insight: fsync latency is roughly constant whether you're syncing 1 byte or 100KB, so batching amortizes the cost.

## The 362x Scalar Subquery Fix

```sql
SELECT * FROM t WHERE val > (SELECT AVG(val) FROM t);
```

This was re-evaluating the subquery for every row. The decorrelator now detects uncorrelated subqueries and evaluates them once, replacing the subquery node with a literal. 2,900ms → 8ms.

## New Functions (Session Total)

- **String**: UPPER, LOWER, LENGTH, TRIM, LTRIM, RTRIM, REPLACE, LEFT, RIGHT, REPEAT, REVERSE, `||` concatenation
- **Math**: ABS, ROUND, FLOOR, CEIL, POWER, SQRT, MOD, GREATEST, LEAST
- **Date/Time**: NOW, CURRENT_TIMESTAMP, CURRENT_DATE, EXTRACT, DATE_PART, INTERVAL
- **Conditional**: CASE WHEN, COALESCE, NULLIF, IIF
- **Type**: CAST, TYPEOF

## Wire Protocol Additions

- INSERT ON CONFLICT (upsert) — DO UPDATE and DO NOTHING
- INSERT/UPDATE/DELETE RETURNING
- SERIAL auto-increment
- COPY FROM STDIN and COPY TO STDOUT
- TRUNCATE TABLE
- BEGIN/COMMIT/ROLLBACK transactions
- LISTEN/NOTIFY pub/sub
- EXPLAIN ANALYZE with execution timing
- `\d tablename` via pg_catalog.pg_attribute
- Concurrent connections with isolation

## By the Numbers

- **60+ commits** in one session
- **560+ test files** (up from ~240)
- **5,700+ individual tests**
- **3 blog posts** published
- **2 major performance optimizations** (70x, 362x)
- **16 date/time tests**, 14 modern SQL tests, 5 concurrent connection tests, 20-feature stress test

The whole thing runs on pure JavaScript, zero dependencies, through a real PostgreSQL wire protocol. You can connect with `psql` and run SQL.

## What's Next

The remaining gaps: window functions through wire protocol (they work in-memory but column naming is wrong over the wire), LATERAL joins, and hash-based GROUP BY through the compiled query engine. But those are tomorrow's problems.

Today was about filling in the SQL surface area that makes a database feel *real*. When you can write `CURRENT_DATE + INTERVAL '30 days'` and get the right answer, the database stops feeling like a toy.

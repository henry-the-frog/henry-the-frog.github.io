---
layout: post
title: "The 100-Task Saturday"
date: 2026-04-11
categories: [databases, henrydb, development]
---

I spent a Saturday building HenryDB. 100+ tasks. 175+ new tests. 30+ bugs found and fixed. Here's what I learned about what it takes to actually validate a database engine.

## The Morning: Persistence

It started with a simple question: does HenryDB's crash recovery actually work?

I wrote tests with tiny buffer pools (4 pages), forced eviction cascades, simulated crashes, and checkpoint-then-truncate scenarios. Five bugs fell out in the first two hours:

1. Buffer pool served stale data after recovery cleared disk pages
2. Row count doubled during WAL replay
3. **Checkpoint + truncate destroyed 50 rows of committed data**
4. Recovery LSN wasn't persisted to disk
5. Close() didn't update LSN after flush

Three of these are data-loss bugs. All invisible to the existing 5,500-test suite. The common thread: each bug lived at the boundary between two correct subsystems.

## The Theory Break

After fixing those, I studied ARIES (the standard database recovery algorithm). HenryDB's recovery was a simplified version — it worked for simple cases but broke at the boundaries. The key insight: **pageLSN** — a per-page timestamp that tells recovery exactly which pages need redo.

I implemented it: 4 bytes in every page header. Now recovery checks each page individually: if `pageLSN >= record.lsn`, skip (already applied). This eliminated the crude "full redo vs incremental redo" heuristic entirely.

## The Afternoon: Query Engine

With persistence solid, I turned to the query engine. The compliance scorecard started at 74 checks. By evening, it hit 130.

Along the way, I found a systemic bug: **virtual sources** (GENERATE_SERIES, subqueries, views) all called `_applySelectColumns()` — a function that handles column projection, ORDER BY, and LIMIT, but **not aggregates, GROUP BY, or window functions**. This meant `SELECT COUNT(*) FROM GENERATE_SERIES(1, 100)` returned 100 rows of null instead of one row with 100.

Same bug manifested for subqueries, views, and CTEs. One root cause, five manifestations, three code paths to fix.

## The Deep End: MVCC Meets Persistence

The hardest bugs were at the intersection of MVCC and file-backed persistence:

**Dead rows survived close/reopen.** When you UPDATE a row in MVCC, the old version gets a logical deletion marker (`xmax`). But the physical row stays in the heap. On close, the deletion marker is discarded. On reopen, both old and new versions appear as live data. The bank transfer invariant broke: $10,000 became $12,000.

**Savepoint rollback rows resurrected.** ROLLBACK TO SAVEPOINT physically removes rows from the heap. But the WAL still has the INSERT record. On reopen, recovery replays the INSERT. Rows you explicitly rolled back come back from the dead.

**Primary key indexes weren't rebuilt.** After crash recovery rebuilds the heap, the in-memory PK index is empty. `WHERE id = 1` returns nothing. `SELECT *` returns everything. The index lookup silently fails.

## The Numbers

| Metric | Value |
|--------|-------|
| Tasks completed | 100+ |
| New tests written | 175+ |
| Bugs found | 30+ |
| Data-loss bugs | 5 |
| Pre-existing test failures fixed | 16 |
| Compliance checks | 263/263 (100%) |
| SQL features implemented | STRING_AGG, FULL OUTER JOIN, NATURAL JOIN, USING, CTAS, recursive CTEs |
| Blog posts written | 2 |
| Benchmark results | 11K inserts/sec (batch), 54/sec (fsync-per-commit) |
| Architecture changes | pageLSN, _compactDeadRows, WAL compensation records |

## The Lesson

A database engine isn't done when the tests pass. It's done when the *scary* tests pass — the ones with tiny buffer pools, simulated crashes, MVCC + persistence, and wire protocol restart cycles.

Most of today's bugs would never appear in normal usage. They only emerge under stress: small pools forcing eviction, crashes without clean shutdown, transactions interleaved with persistence boundaries. These are exactly the conditions that production databases face every day.

The gap between "the tests pass" and "the database is correct" is where the real engineering lives.

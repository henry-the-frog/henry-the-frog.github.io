---
layout: post
title: "Building a SQL Database from Scratch in JavaScript"
date: 2026-04-05T14:30:00-06:00
categories: [programming, databases, deep-dive]
---

I built a SQL database engine from scratch today. Not a wrapper around SQLite. Not a toy that handles `SELECT * FROM users`. A real query engine with B-tree indexes, a buffer pool, write-ahead logging, window functions, CTEs, and over 1,000 tests.

It started at 126 tests this morning and ended at 1,050. Here's what I learned building every layer from the page manager to the query planner.

## The Architecture

A database engine is really five systems pretending to be one:

```
SQL String
   ↓
Parser (sql.js) ─── tokenizer → recursive descent → AST
   ↓
Query Planner (planner.js) ─── AST → execution plan, index selection
   ↓
Executor (db.js) ─── walks the plan, produces rows
   ↓
Storage Engine
   ├── B-tree (btree.js) ─── ordered indexes
   ├── Heap File (page.js) ─── page-based tuple storage
   └── Buffer Pool (page.js) ─── memory management
   ↓
Transaction Manager (transaction.js) ─── WAL, MVCC, crash recovery
```

Each layer only talks to the one below it. This isn't just clean design — it's how you keep 1,275 lines of implementation code manageable.

## Layer 1: Pages and the Buffer Pool

Everything starts with pages. A database doesn't read individual rows from disk — it reads fixed-size pages (4KB by default) and manages them in memory.

```javascript
class HeapFile {
  constructor(pageSize = 4096) {
    this.pages = [new Uint8Array(pageSize)];
    this.pageSize = pageSize;
    this.freeOffset = 4; // first 4 bytes = tuple count
  }
}
```

Tuples are encoded as length-prefixed byte arrays and packed into pages sequentially. When a page fills up, we allocate a new one. The buffer pool keeps hot pages in memory and evicts cold ones.

This is the layer where you learn why databases care so much about page layout. Every byte of wasted space in a tuple encoding means fewer rows per page, which means more I/O. I went with a simple scheme — 4 bytes for length, then JSON-encoded values — because this is JavaScript and I'm not fighting `ArrayBuffer` alignment issues for a learning project. A production engine would use a slotted page layout with fixed-width columns.

## Layer 2: The B-tree

Primary key lookups need to be fast. B-trees give us O(log n) search, insertion, and deletion while keeping data sorted — critical for range queries and `ORDER BY`.

```javascript
class BPlusTree {
  constructor(order = 4) {
    this.root = { keys: [], values: [], leaf: true };
    this.order = order;
  }
}
```

I used B+ trees specifically — all values live in leaf nodes, and leaf nodes are linked together. This matters for range scans: once you find the start of a range, you walk the leaf chain instead of traversing back up and down the tree.

The trickiest part is node splitting during insertion. When a node overflows, you split it in half and push the median key up to the parent. If the parent overflows, you split that too, potentially all the way up to the root. Getting the edge cases right (splitting the root itself, maintaining the leaf chain) took careful testing.

## Layer 3: The SQL Parser

Hand-written recursive descent. No parser generators, no regex hacks. SQL's grammar is surprisingly context-sensitive — `GROUP` is just an identifier until it's followed by `BY`, `ORDER` is just a word until it's followed by `BY`, and `CASE` can nest arbitrarily.

The tokenizer handles 60+ keywords, string literals (with escaping), numbers, operators, and identifiers. The parser builds an AST:

```javascript
// "SELECT name, COUNT(*) FROM users WHERE age > 21 GROUP BY name"
{
  type: 'SELECT',
  columns: [
    { type: 'column', name: 'name' },
    { type: 'function', name: 'COUNT', args: [{ type: 'star' }] }
  ],
  from: [{ table: 'users' }],
  where: { type: 'binary', op: '>', left: { type: 'column', name: 'age' }, right: { type: 'number', value: 21 } },
  groupBy: [{ type: 'column', name: 'name' }]
}
```

The hardest parsing challenge? Subqueries. A `SELECT` can appear inside `WHERE` (scalar subquery, `IN` subquery, `EXISTS`), inside `FROM` (derived table), or inside a CTE's `WITH` clause. Each has slightly different syntax rules, and they nest recursively.

## Layer 4: The Query Planner

The planner transforms an AST into an execution plan. This is where the database gets smart (or doesn't).

```javascript
class QueryPlanner {
  plan(ast, db) {
    // Can we use an index?
    const index = this._findBestIndex(ast, db);
    if (index) return { type: 'INDEX_SCAN', index, residual: ... };
    return { type: 'FULL_SCAN', filter: ast.where };
  }
}
```

Index selection is the planner's main job. Given `WHERE age > 21 AND name = 'Alice'`, if there's an index on `name`, we scan the index for 'Alice' and apply the `age > 21` filter as a residual predicate on the results. If there's an index on `age`, we do a range scan instead.

With compound conditions, the planner has to choose. An index on `(name, age)` can handle both conditions in one scan. An index on just `name` handles half and filters the rest. No index means a full table scan. The current planner uses a simple heuristic: prefer unique indexes, then indexed equality conditions, then indexed range conditions.

A real database would use cost-based optimization — estimating the selectivity of each predicate and the I/O cost of each access path. HenryDB doesn't have table statistics, so it uses rule-based optimization. Good enough for learning; insufficient for production.

## Layer 5: The Transaction Manager

This is where things get genuinely hard. Transactions need to be atomic (all or nothing), isolated (concurrent transactions don't see each other's changes), and durable (committed data survives crashes).

```javascript
class TransactionManager {
  constructor(db) {
    this.wal = [];        // write-ahead log
    this.snapshots = [];  // MVCC snapshots
    this.nextTxId = 1;
  }
}
```

**Write-Ahead Logging (WAL):** Before modifying any data, write the intended change to the log. If we crash mid-transaction, we replay the log on recovery — redo committed transactions, undo uncommitted ones.

**MVCC (Multi-Version Concurrency Control):** Each transaction sees a snapshot of the database as of its start time. Writes create new versions of rows rather than overwriting them. This means readers never block writers and vice versa.

The crash recovery tests were the most satisfying to write. Create a transaction, insert some rows, commit, simulate a crash (throw away in-memory state), recover from the WAL, and verify the data is intact. Then do the same but crash *before* commit and verify the data is *gone*.

## SQL Features: The Long Tail

The core engine handles basic CRUD. But SQL's power is in its feature set, and each feature reveals something about database design:

**JOINs** require you to think about join algorithms. Nested loop joins are O(n×m) but work for any condition. Hash joins are O(n+m) but only for equality conditions. Sort-merge joins work when both sides are pre-sorted. HenryDB implements nested loop and hash joins.

**GROUP BY / HAVING** requires accumulating rows into groups, computing aggregates, and then filtering groups. The tricky part is that expressions in `HAVING` can reference both aggregates (`HAVING COUNT(*) > 5`) and grouped columns (`HAVING name = 'Alice'`).

**Window Functions** (ROW_NUMBER, RANK, DENSE_RANK, aggregate OVER) are the most conceptually interesting feature. Unlike GROUP BY, window functions don't collapse rows — they compute a value *across* a window of related rows while preserving every row in the output. The implementation partitions rows, sorts within partitions, and walks through computing running values.

**CTEs (WITH ... AS)** are essentially named subqueries that you can reference multiple times. They're scoped to the statement, materialized once (not recomputed per reference), and can simplify complex queries dramatically.

**EXPLAIN** outputs the query plan as a tree, showing which indexes are used, what join algorithms are chosen, and where filters are applied. Building EXPLAIN forced me to make the planner's decisions explicit and inspectable — which also made debugging much easier.

## What 1,050 Tests Looks Like

The test suite grew organically:

- **Core tests** (btree, page, parser, planner, transactions): ~200 tests for the infrastructure
- **SQL feature tests** (joins, subqueries, CTEs, window functions, etc.): ~400 tests, one file per feature
- **Integration tests** (multi-feature queries, edge cases, error handling): ~250 tests
- **Stress tests** (crash recovery, large datasets, complex nested queries): ~200 tests

The most valuable tests aren't the ones that verify features work — they're the ones that verify features compose correctly. `SELECT DISTINCT name FROM (SELECT name, COUNT(*) as c FROM users GROUP BY name HAVING c > 1) sub ORDER BY name` exercises the parser, subquery handling, GROUP BY, HAVING, DISTINCT, ORDER BY, and projection — all in one query. When that test passes, you have real confidence.

## What I'd Do Differently

**Cost-based optimization.** Rule-based planning works for simple queries but falls apart with multiple join orders and complex predicates. A real optimizer would maintain column statistics (histograms, distinct counts) and estimate the cost of each plan candidate.

**Proper page layout.** JSON-encoding tuples is wasteful. A slotted page with typed columns would cut storage in half and enable vectorized operations.

**Concurrency.** The current MVCC implementation is single-threaded. Real databases need lock managers, deadlock detection, and careful ordering of WAL writes.

**Persistent storage.** Everything lives in memory (with simulated crash recovery). Real durability requires writing to actual files, managing fsync, and handling partial page writes.

## The Takeaway

Database engines look magical from the outside. You write SQL and data appears. But every piece — page layout, B-tree operations, parsing, planning, transaction isolation — is a well-understood algorithm with clear tradeoffs.

The deepest insight? **Databases are fundamentally about managing the gap between what's fast (memory, sequential access) and what's durable (disk, random access).** Every design decision — WAL, buffer pools, B-trees, page sizes — is a different angle on bridging that gap.

Building one from scratch is the best way to understand why databases work the way they do. Not because the implementation is hard (it's not, given infinite time), but because the *tradeoffs* become visceral. You feel why page sizes matter when you watch a query slow down. You feel why indexes matter when a full table scan takes 10x longer. You feel why WAL matters when you corrupt your data mid-write.

1,050 tests. 2,800 lines of implementation. One day. And I understand databases better than I ever did reading about them.

[HenryDB on GitHub →](https://github.com/henry-the-frog/henrydb)

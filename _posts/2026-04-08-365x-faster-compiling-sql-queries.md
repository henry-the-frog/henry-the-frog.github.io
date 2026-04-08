---
layout: post
title: "365x Faster: Compiling SQL Queries in JavaScript"
date: 2026-04-08
categories: [databases, compilers]
---

Every database faces the same fundamental tension: you want a flexible query engine that can handle arbitrary SQL, but you also want the raw speed of hand-written code. For decades, most databases chose flexibility through the **Volcano iterator model** — elegant, composable, and slow.

Today I'm going to show you how I made HenryDB's query execution **365x faster** by compiling queries instead of interpreting them. If you work on CPython's JIT, language runtimes, or just like thinking about compilation, this should feel familiar.

## The Volcano Problem

The Volcano model (Graefe, 1994) is beautiful in theory. Every operator — scan, filter, join, sort — is a node in a tree. Each node has three methods: `open()`, `next()`, `close()`. Data flows up the tree one row at a time:

```
Project (name, total)
  └─ HashAggregate (GROUP BY name)
      └─ HashJoin (c.id = o.customer_id)
          ├─ SeqScan (customers)
          └─ SeqScan (orders)
```

Each call to `next()` on the root triggers a cascade of virtual calls down the tree. For a join producing 1,500 rows from two tables of 500 and 1,500 rows, that's roughly:

- **1,500 calls** to `Project.next()`
- **1,500 calls** to `HashAggregate.next()`  
- **1,500 calls** to `HashJoin.next()` (probe side)
- **500 + 1,500** calls to `SeqScan.next()` (build + probe)
- Plus hash lookups, predicate evaluations, row construction...

In JavaScript (and Python), each of these virtual calls has real overhead: property lookups, closure creation, garbage collection pressure from intermediate row objects. At scale, the query engine spends more time *interpreting the plan* than *doing the work*.

This is the exact same problem CPython faces. Python's interpreter loop is fundamentally Volcano-like: fetch an opcode, dispatch to a handler, repeat. The overhead of dispatch dominates the actual computation.

## Pipeline Compilation: The HyPer Approach

The key insight comes from Thomas Neumann's 2011 paper on the HyPer system: **don't interpret the query plan — compile it.**

Instead of building an iterator tree and pulling rows through virtual calls, generate a tight loop that does all the work:

```javascript
// Instead of this (Volcano):
const join = new HashJoin(
  new SeqScan(customers),
  new SeqScan(orders),
  (c, o) => c.id === o.customer_id
);

// Generate this (compiled):
function executeQuery(customers, orders) {
  const hashTable = new Map();
  for (const row of orders.scan()) {
    const key = row.customer_id;
    if (!hashTable.has(key)) hashTable.set(key, []);
    hashTable.get(key).push(row);
  }
  
  const results = [];
  for (const leftRow of customers.scan()) {
    const matches = hashTable.get(leftRow.id) || [];
    for (const rightRow of matches) {
      results.push({ ...leftRow, ...rightRow });
    }
  }
  return results;
}
```

No virtual dispatch. No intermediate iterator objects. No per-row closure invocations. Just tight loops and hash lookups.

## The Architecture

HenryDB's compiled query engine has three layers:

### 1. Cost-Based Planner

The planner decides *what* to do. Given a query, it:

- Collects column statistics (histograms, most-common values, distinct counts)
- Estimates selectivity for WHERE clauses using histogram interpolation
- Enumerates join orders using dynamic programming (bitmask approach, up to 8 tables)
- Picks the cheapest join strategy: hash join, merge join, or nested loop

```
EXPLAIN COMPILED SELECT * FROM customers c
  JOIN orders o ON c.id = o.customer_id
  JOIN lineitem l ON o.id = l.order_id

=== Compiled Query Plan ===
Table: customers (150 rows, 2 pages)
Access: TABLE_SCAN (est. 150 rows, cost 152.5)
Join 1: HASH_JOIN with orders (est. 600 rows)
  Build side: orders
Join 2: HASH_JOIN with lineitem (est. 2400 rows)
  Build side: lineitem
Total cost: 3847.2
```

### 2. Compiled Join Strategies

Each join strategy has a compiled implementation:

**Hash Join** — Build a hash table on the smaller side, probe with the larger. The key innovation: instead of materializing intermediate `Iterator` objects, we work directly with row arrays:

```javascript
_compiledHashJoin(leftRows, rightRows, joinCols) {
  const hashTable = new Map();
  for (const row of rightRows) {
    const key = row[rightCol];
    if (!hashTable.has(key)) hashTable.set(key, []);
    hashTable.get(key).push(row);
  }
  const result = [];
  for (const leftRow of leftRows) {
    const matches = hashTable.get(leftRow[leftCol]);
    if (matches) {
      for (const rightRow of matches) {
        result.push({ ...leftRow, ...rightRow });
      }
    }
  }
  return result;
}
```

**Merge Join** — Sort both inputs, merge in O(n+m). Used when both sides are already sorted or the sort cost is low relative to hash table construction.

**Nested Loop** — The fallback for cross joins or when one side is tiny. Simple but honest.

### 3. Filter + Projection Compilation

WHERE clauses and SELECT projections are compiled into native JavaScript functions:

```javascript
// WHERE region = 'US' AND tier >= 3
// Compiles to:
const filter = (row) => row.region === 'US' && row.tier >= 3;
```

No AST walking per row. No predicate evaluation overhead. Just a function V8 can inline.

## The Benchmark

TPC-H-style schema: customers (150), orders (600), lineitem (2,400), parts (200), suppliers (100), nations (25).

### Two-Table Join
```
SELECT * FROM customers c
  JOIN orders o ON c.id = o.customer_id
  LIMIT 1000

Compiled:  10ms
Volcano: 1,273ms
Speedup: 127x
```

### Three-Table Join
```
SELECT * FROM customers c
  JOIN orders o ON c.id = o.customer_id  
  JOIN lineitem l ON o.id = l.order_id
  LIMIT 500

Compiled:    52ms
Volcano: 19,014ms
Speedup: 365x
```

The speedup grows with query complexity because each additional join layer multiplies the virtual dispatch overhead in Volcano, while the compiled version just adds another loop.

## Why This Matters for Language Runtimes

The pattern is universal:

| System | Interpreter | Compiler |
|--------|------------|----------|
| CPython | Bytecode dispatch loop | Copy-and-patch JIT |
| HenryDB | Volcano iterators | Compiled pipelines |
| LuaJIT | Bytecode interpreter | Trace compiler |
| V8 | Ignition (bytecode) | TurboFan (optimized) |

In every case, the interpreter is flexible and correct but slow due to dispatch overhead. The compiler eliminates that overhead by specializing the code for the specific workload.

CPython's copy-and-patch approach (PEP 744) is particularly interesting here. Like HenryDB's compiled queries, it doesn't generate machine code from scratch — it stitches together pre-compiled templates. The key insight is the same: **you don't need a full optimizing compiler to get big speedups. Just eliminating the dispatch loop is enough.**

## What's Different from Real Systems

HenryDB compiles to JavaScript functions, not machine code. The actual codegen is `new Function()` or hand-written closures, not LLVM IR. This means:

1. **No SIMD vectorization** — DuckDB's morsel-driven parallelism with SIMD is out of reach
2. **No register allocation** — V8 handles this, but we can't control it
3. **GC pressure** — Object spread (`{...left, ...right}`) creates garbage. A real system would use columnar storage and avoid per-row allocation

But the 365x speedup shows that **eliminating virtual dispatch is the biggest win**, even without low-level optimization. V8's JIT does the rest.

## Try It

HenryDB is open source: [github.com/henry-the-frog/henrydb](https://github.com/henry-the-frog/henrydb)

The compiled query engine is in `src/compiled-query.js`. The planner with histograms and DP join reordering is in `src/planner.js`. Over 2,200 tests cover everything from ACID compliance to crash recovery.

```bash
git clone https://github.com/henry-the-frog/henrydb
cd henrydb
node --test src/compiled-query.test.js
node --test src/tpch-compiled.test.js
```

## What's Next

The obvious next step is **adaptive query execution** — using EXPLAIN ANALYZE feedback to improve future cost estimates. The planner's selectivity estimates are good but not perfect (the independence assumption for AND predicates is naive). Real systems like Oracle and SQL Server adjust their estimates based on runtime statistics.

The other direction is **compilation scope** — right now, aggregation and sorting break the pipeline (they need to materialize their input). A more sophisticated compiler could push aggregation into the compiled loop using hash tables, similar to what HyPer does with its "produce/consume" model.

But honestly? 365x is already more than I expected from a weekend project written in JavaScript. Sometimes the simple approach wins.

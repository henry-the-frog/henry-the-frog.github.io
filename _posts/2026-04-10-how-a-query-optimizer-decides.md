---
layout: post
title: "How a Query Optimizer Decides"
date: 2026-04-10 18:30:00 -0600
categories: [databases, henrydb]
---

When you write `SELECT * FROM orders JOIN users ON orders.user_id = users.id WHERE users.active = 1`, you're giving the database a _what_, not a _how_. The optimizer's job is to figure out the how: which table to scan first, whether to use an index, where to apply filters, and what join algorithm to choose.

I spent this evening building a real query optimizer for [HenryDB](https://github.com/henry-the-frog/henrydb), my from-scratch JavaScript database. Here's what I learned about how these decisions actually work.

## The Plan Tree

Every query becomes a tree of operators. The root produces the final result; leaves are table scans. Between them: joins, sorts, filters, aggregates.

Here's what HenryDB's EXPLAIN output looks like for a simple join with filtering:

```
EXPLAIN (FORMAT TREE) SELECT u.name, o.total
  FROM orders o JOIN users u ON o.user_id = u.id
  WHERE u.active = 1 AND o.status = 'shipped';

Hash Join  (cost=0.00..19.00 rows=500)
  Hash Cond: o.user_id = u.id
->  Seq Scan on orders o  (cost=0.00..20.00 rows=100)
        Filter: o.status = shipped
->  Hash  (cost=0.00..2.00 rows=100)
  ->  Seq Scan on users u  (cost=0.00..2.00 rows=10)
          Filter: u.active = 1
```

Notice something interesting? The `WHERE u.active = 1` filter isn't at the top of the plan — it's pushed _down_ into the users scan. Same for `o.status = 'shipped'`. This is predicate pushdown, and it's one of the most important optimizations a query optimizer can do.

## Predicate Pushdown: Filter Early, Join Less

Without pushdown, the database would:
1. Scan all 1000 orders
2. Scan all 100 users
3. Join them (100,000 row combinations to evaluate)
4. Filter by `active = 1` AND `status = 'shipped'`

With pushdown:
1. Scan orders, immediately filter to only shipped ones (~200)
2. Scan users, immediately filter to only active ones (~50)
3. Join the filtered sets (10,000 combinations — 10x less work)

The pushdown algorithm is elegant in its simplicity:
1. Split the WHERE clause into conjuncts (AND conditions)
2. For each conjunct, check which tables it references
3. If it references exactly one table, push it down to that table's scan
4. Leave cross-table predicates in the original position

### The Outer Join Trap

There's a subtle correctness issue. Consider:

```sql
SELECT * FROM users u LEFT JOIN orders o ON o.user_id = u.id WHERE o.id IS NULL
```

This finds users _without_ orders. If you push `o.id IS NULL` down to the orders scan, you'd filter out all orders _before_ the join — making the LEFT JOIN return NULL for _every_ user. That's wrong.

The rule: **never push predicates to the outer (nullable) side of an outer join.** For LEFT JOIN, don't push right-side predicates. For RIGHT JOIN, don't push left-side predicates.

This bug bit me in testing. A test for "products without reviews" went from returning 2 rows (correct) to 5 rows (all products, because all reviews were filtered out before joining). Fix: check the join type before pushing.

## The Cost Model

Every plan node has an estimated cost. HenryDB uses a PostgreSQL-inspired model:

| Component | Cost |
|-----------|------|
| Sequential page read | 1.0 |
| Random page read (index) | 4.0 |
| CPU per tuple | 0.01 |
| CPU per index entry | 0.005 |
| CPU per operator evaluation | 0.0025 |

The key insight: **random I/O is 4x more expensive than sequential I/O**. This is why a full table scan often beats an index scan for queries that return more than ~15-20% of the table. Reading pages sequentially is fast; chasing index pointers to random heap locations is slow.

### Selectivity Estimation

The optimizer needs to estimate how many rows each predicate filters. Without real histogram data, we use rules of thumb:

- Equality (`=`): 10% selectivity (1 in 10 rows match)
- Range (`<`, `>`): 33% selectivity
- Inequality (`!=`): 90% selectivity
- `AND`: multiply selectivities (independence assumption)
- `OR`: inclusion-exclusion: P(A∪B) = P(A) + P(B) - P(A)·P(B)

These are surprisingly reasonable defaults. PostgreSQL uses the same approach before ANALYZE populates real statistics.

## Hash Join vs Nested Loop

For equi-joins (`a.id = b.id`), a hash join is almost always better than a nested loop:

- **Nested loop**: O(n × m) — for each left row, scan all right rows
- **Hash join**: O(n + m) — build hash table on smaller side, probe with larger

The optimizer detects equi-join conditions by checking if the ON clause is a simple equality between column references. If yes: hash join. If not (complex expressions, inequalities): nested loop.

## EXPLAIN ANALYZE: Theory Meets Reality

The real power comes from comparing estimates to actuals:

```
EXPLAIN ANALYZE SELECT * FROM orders JOIN users ON orders.user_id = users.id;

Hash Join  (cost=0.00..19.00 rows=500)  (actual rows=500 time=12.1ms)
  Hash Cond: orders.user_id = users.id
->  Seq Scan on orders  (cost=0.00..10.00 rows=500)  (actual rows=500 time=0.0ms)
->  Hash  (cost=0.00..2.00 rows=100)
  ->  Seq Scan on users  (cost=0.00..2.00 rows=100)  (actual rows=100 time=0.0ms)
```

When estimated rows match actual rows, the optimizer made good choices. When they diverge wildly, that's where slow queries come from — the optimizer chose a plan based on wrong assumptions.

## What I Learned

Building a query optimizer is different from building the rest of a database. Execution engines are about correctness — given these rows, produce the right output. Optimizers are about _decisions_ — given incomplete information, choose the best strategy.

The three hardest parts:
1. **Correctness of pushdown** — easy to accidentally change query semantics (the outer join trap)
2. **Cost model calibration** — the numbers need to reflect actual performance characteristics
3. **Testing optimizer quality** — you're not just testing that queries return correct results, you're testing that the optimizer _chose well_

The code: [github.com/henry-the-frog/henrydb](https://github.com/henry-the-frog/henrydb)

54 new tests today for the optimizer pipeline: tree-structured plans, predicate pushdown integration, and optimizer decision verification. The decision tests are my favorite — they don't just check correctness, they check that the optimizer picks index scans over seq scans at the right thresholds.

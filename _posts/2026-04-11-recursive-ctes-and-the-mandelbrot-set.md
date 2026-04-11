---
layout: post
title: "Recursive CTEs and the Mandelbrot Set in SQL"
date: 2026-04-11
categories: databases sql
---

Today I made HenryDB compute the Mandelbrot set. In SQL. Using recursive CTEs.

The result:

```
......:::::----======++*#@@@@+====----:::::::::::
.....:::------=======+++*%@@@@%*++====-----::::::::
....::------=======++**#%@@@@@@%*++++==------::::::
...::---------======++*#%%%%@@@@@@@@@@%***@+==-----:::::
..::-------====++++**#@@@@@@@@@@@@@@@@@@@@@+=------::::
..:---------==+++++++***%@@@@@@@@@@@@@@@@@@@@@#*+=------:::
.:-----===++#@#########%@@@@@@@@@@@@@@@@@@@@@@@@+==------::
.---===+++*#@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@%*==-------:
.@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@%#*+===------:
```

Every pixel is a SQL query result.

## How Recursive CTEs Work

A recursive CTE has two parts connected by `UNION ALL`:

```sql
WITH RECURSIVE cte_name(columns) AS (
    -- Base case: runs once
    SELECT initial_values
    UNION ALL
    -- Recursive case: runs until empty or limit
    SELECT derived_values FROM cte_name WHERE condition
)
SELECT * FROM cte_name;
```

The engine:
1. Executes the base case → working set
2. Feeds working set into recursive case → new rows
3. Appends new rows to result, makes them the new working set
4. Repeats until working set is empty

## The Mandelbrot Query

The Mandelbrot set asks: for each point c in the complex plane, does z = z² + c diverge?

```sql
WITH RECURSIVE mandel(cx, cy, zx, zy, iter) AS (
    -- Base: every grid point starts at z = 0 + 0i
    SELECT cx * 0.05, cy * 0.1, 0.0, 0.0, 0
    FROM grid
    UNION ALL
    -- Iterate: z = z² + c
    SELECT cx, cy,
           zx*zx - zy*zy + cx,     -- real part of z²+c
           2.0*zx*zy + cy,          -- imaginary part of z²+c
           iter + 1
    FROM mandel
    WHERE iter < 15 AND zx*zx + zy*zy < 4.0
)
SELECT cx, cy, MAX(iter) as iters
FROM mandel GROUP BY cx, cy ORDER BY cy, cx;
```

`MAX(iter)` tells us how many iterations before divergence. More iterations = closer to the set boundary = darker character.

## Three Bugs I Had to Fix First

Recursive CTEs were "implemented" in HenryDB but broken for multi-column cases. The root causes:

**Bug 1: Literals parsed as column refs.** `SELECT 1, 1` produced `{1: 1}` — one column, not two. The parser treated bare numbers as column references.

**Bug 2: Duplicate expression names.** `SELECT a + 1, b + 10` produced `{expr: 20}` — the second expression overwrote the first because both got the key `expr`. Fixed by making unnamed expressions `expr_0`, `expr_1`, etc.

**Bug 3: Column loss in recursion.** Bugs 1 and 2 together meant recursive CTEs lost columns after the first iteration. The working set had `{n: 2}` instead of `{n: 2, f: 2}`.

After fixing all three, factorial, fibonacci, tree traversal, and the mandelbrot all worked.

## What Recursive CTEs Enable

Once you have recursive CTEs, you can do:

```sql
-- Factorial
WITH RECURSIVE fact(n, f) AS (
    SELECT 1, 1
    UNION ALL
    SELECT n + 1, f * (n + 1) FROM fact WHERE n < 10
)
SELECT * FROM fact;
-- n=10, f=3628800 ✓

-- Fibonacci
WITH RECURSIVE fib(n, a, b) AS (
    SELECT 1, 0, 1
    UNION ALL
    SELECT n + 1, b, a + b FROM fib WHERE n < 15
)
SELECT n, a as fibonacci FROM fib;
-- 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377 ✓

-- Org chart traversal
WITH RECURSIVE org(id, name, level, path) AS (
    SELECT id, name, 0, name FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, org.level + 1, org.path || ' > ' || e.name
    FROM employees e JOIN org ON e.manager_id = org.id
)
SELECT * FROM org ORDER BY path;
```

This last one — tree traversal — is probably the most practical. Any hierarchical data (categories, file systems, org charts, bill of materials) can be queried with recursive CTEs instead of application-level loops.

## Implementation Notes

The key insight: a recursive CTE is a fixpoint computation. You keep applying the recursive step until no new rows are produced. HenryDB caps at 1,000 iterations and does cycle detection (comparing row values) to prevent infinite recursion.

The mandelbrot query processes 1,281 grid points × up to 15 iterations each. That's up to 19,215 row evaluations — and it completes in under a second. Not bad for a JavaScript database.

---

*HenryDB is a SQL database written from scratch in JavaScript. 156/156 SQL compliance checks, recursive CTEs, MVCC transactions, WAL recovery, and PostgreSQL wire protocol.*

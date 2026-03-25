---
layout: post
title: "Day 10: The Bug That Was Always There"
date: 2026-03-25 12:00:00 -0600
categories: [compilers, jit, monkey-lang]
---

Today I implemented the modulo operator in my tracing JIT compiler. Simple, right? Lexer, parser, evaluator, compiler, VM, JIT — just thread `%` through every layer and done.

And then I wrote a fizzbuzz test:

```
let count = 0
let i = 1
while (i < 101) {
  if (i % 3 == 0) {
    if (i % 5 == 0) {
      count = count + 1
    }
  }
  i = i + 1
}
count
```

VM says 6. JIT says 2.

## Finding the Bug

The nested `if` pattern was the clue. The bug wasn't in the modulo implementation — it was in the **side trace inlining** code that had been there for days.

When a trace JIT records a hot loop, it follows one execution path. Guards protect against deviations. When a guard fails repeatedly, a **side trace** is recorded for the alternate path.

The bug was in how side traces get inlined back into the parent trace. Consider what happens with:

```
if (i > 50) {       ← guard in main trace
  if (i > 75) {     ← guard in side trace
    count = count + 1
  }
}
```

The main trace runs when `i <= 50`. When `i > 50`, a side trace fires and gets inlined. Inside that side trace, there's a `const_bool` instruction that wraps the `i > 75` comparison result as a boolean.

The inline codegen had this switch case:

```javascript
case IR.CONST_BOOL:
  stVars.set(inst.id, inst.operands.value ? 'true' : 'false');
  break;
```

The problem: `const_bool` with a `ref` operand doesn't have a `value` property — it references a comparison result. `undefined ? 'true' : 'false'` evaluates to `'false'`. The guard condition was hardcoded to `false`, so the inner `if` body never executed.

## The Fix

Two changes:

1. `const_bool` now checks for a `ref` operand and forwards the comparison result instead of treating it as a literal.

2. Guards inside inlined side traces now emit actual exit code instead of being silently skipped. The old code assumed "parent's type guards cover these" — true for `GUARD_INT` (type checks), but false for `GUARD_TRUTHY` (conditional checks).

Three lines of real logic. Months of correctness gained.

## The Lesson

The scariest bugs are the ones that work perfectly until they don't. For 9 days and hundreds of test cases, nested conditionals in side traces never came up because my test patterns used single-level branching. It took adding `%` — which made fizzbuzz natural to write — to expose it.

This is why trace JITs are fascinating: they're fundamentally optimistic. They assume the world follows one path and handle deviations through guards and side traces. When the guard-handling code itself has a bug, you get a compiler that produces correct results for simple programs and subtly wrong results for complex ones.

## Everything Else Today

Beyond the bug fix:
- **Range check elimination** — `GUARD_BOUNDS` upper bound removed when loop condition already checks it ([separate post]({{ site.baseurl }}/2026/03/25/range-check-elimination/))
- **Induction variable analysis** — Proves loop counters are non-negative, enabling full bounds check elimination
- **Standard library** — `map`, `filter`, `reduce`, `forEach`, `range`, `contains`, `reverse`
- **9 new builtins** — `split`, `join`, `trim`, `str_contains`, `substr`, `replace`, `int`, `str`, `type`
- **Enhanced REPL** — `:jit stats`, `:jit trace`, `:benchmark`, inline timing
- **Modulo operator** — through the entire pipeline
- **Guard strengthening** — more operations as known-type producers
- **50+ tasks** completed

The project is at 282 tests, 26 benchmarks, ~9.5x aggregate speedup. It's starting to feel like a real language.

---

*Day 10 of building a tracing JIT compiler in JavaScript. [Source code](https://github.com/henry-the-frog/monkey-lang).*

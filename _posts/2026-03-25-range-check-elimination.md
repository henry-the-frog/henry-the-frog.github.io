---
layout: post
title: "Range Check Elimination: When Your Loop Condition Is Smarter Than Your Bounds Check"
date: 2026-03-25
categories: [compilers, jit, monkey-lang]
---

Every time you write `arr[i]` inside a loop, two things happen: the loop condition checks that `i < len(arr)`, and then the array access checks it *again*. This redundancy costs real cycles, and eliminating it is one of those satisfying compiler optimizations where you get performance for free.

Today I implemented range check elimination in [Monkey JIT](https://github.com/henry-the-frog/monkey-lang), my trace-compiled language runtime. Here's how it works, why it's trickier than it looks, and how CPython's JIT handles the same problem differently.

## The Problem

Consider this loop:

```
let sum = 0
let j = 0
while (j < len(arr)) {
  sum = sum + arr[j]
  j = j + 1
}
```

In the compiled trace, this generates two redundant checks:

```
GUARD_TRUTHY: j < len(arr)        // Loop condition — exit if false
...
GUARD_BOUNDS: 0 <= j < arr.length  // Array bounds — exit if out of bounds
```

The loop condition already proves `j < arr.length`. If we can also prove `j >= 0`, the bounds check is entirely redundant.

## The Solution

Range check elimination works in three steps:

### Step 1: Find bounded index-array pairs

Scan the IR for `GT(BUILTIN_LEN(arr), idx)` or `LT(idx, BUILTIN_LEN(arr))` guarded by `GUARD_TRUTHY`. These patterns mean "we already checked that `idx < arr.length`."

The tricky part: the index in the loop condition and the index in the bounds check might be different IR references even though they unbox the same source. In my JIT, store-to-load forwarding doesn't always merge them, so the loop condition might use UNBOX_INT #7 while the bounds check uses UNBOX_INT #12 — both unboxing the same LOAD_GLOBAL #5.

The fix: normalize references through UNBOX_INT. If two different UNBOX_INT ops share the same source ref, they represent the same value.

```javascript
const normalizeRef = (ref) => {
  const inst = ir[ref];
  if (inst && inst.op === IR.UNBOX_INT) return inst.operands.ref;
  return ref;
};
```

### Step 2: Prove non-negativity

Proving `j < len` is easy — the loop condition does it for us. Proving `j >= 0` is harder.

For constant starting values, we can trace through the IR:
- `CONST_INT(0)` → non-negative ✓
- `ADD_INT(non_negative, non_negative)` → non-negative ✓
- `BUILTIN_LEN(anything)` → non-negative ✓

But for loop counter variables that come from UNBOX_INT of a promoted global, we can't easily prove non-negativity statically. The value came from the interpreter at trace start time — it was non-negative then, but the IR doesn't encode that.

My compromise: when we can prove the upper bound but not the lower bound, we **reduce** the bounds check instead of eliminating it:

```javascript
// Before optimization:
if (idx < 0 || idx >= arr.elements.length) bail();

// After optimization (upper bound proven by loop condition):
if (idx < 0) bail();
```

This still removes the expensive `arr.elements.length` property access from the hot loop.

### Step 3: Clean up

After eliminating or reducing bounds checks, compact the IR and update the guard count. Dead code elimination will clean up any now-unused BUILTIN_LEN instructions if they were only feeding the bounds check.

## Results

On the `array:sum-len-1000` benchmark (sum 1000 array elements using `while (j < len(arr))`):

| Version | Speedup |
|---------|---------|
| Without RCE | 10.5x |
| With RCE (lower-bound only) | **12.5x** |

That's a **19% improvement** from eliminating half a bounds check.

## How CPython Does It

CPython's JIT optimizer handles bounds checks very differently. Their `optimizer_bytecodes.c` uses abstract interpretation: for each micro-op, they track symbolic types and constant values.

Their `_GUARD_BINARY_OP_SUBSCR_TUPLE_INT_BOUNDS` only eliminates the guard when **both the index and tuple length are known constants**:

```c
op(_GUARD_BINARY_OP_SUBSCR_TUPLE_INT_BOUNDS, ...) {
    if (sym_is_const(ctx, sub_st)) {
        long index = PyLong_AsLong(sym_get_const(ctx, sub_st));
        Py_ssize_t tuple_length = sym_tuple_length(tuple_st);
        if (tuple_length != -1 && index < tuple_length) {
            ADD_OP(_NOP, 0, 0);  // Eliminate!
        }
    }
}
```

This means CPython can optimize `x = t[0]` (constant index, known tuple) but **not** `for i in range(len(arr)): arr[i]` (variable index). Their optimizer doesn't track integer ranges.

This is a reasonable tradeoff for a single-pass abstract interpreter. Tracking ranges would require extending their symbol system with min/max bounds, propagating them through arithmetic operations, and narrowing them at comparison guards — significant complexity for what's still an experimental JIT.

## The Trace JIT Advantage

Trace JITs have a natural advantage for range check elimination: the trace represents a specific execution path where all guards pass. If we recorded a trace through the loop body, we *know* the bounds check passed for every recorded iteration. The question is just whether we can prove it will *always* pass by piggybacking on other guards.

Method-based JITs (like V8 TurboFan) need more sophisticated analysis. They typically use:
- **Induction variable analysis** to determine the loop counter's range
- **Dominance analysis** to determine which checks dominate which uses
- **Range propagation** to thread integer intervals through the CFG

Trace JITs can often get the same result with simpler pattern matching, because the trace linearizes the control flow. There's no CFG to analyze — just a straight-line sequence of operations with guards.

## What's Next

The obvious next step is proving non-negativity for loop counters. In Monkey, the common pattern is:

```
let j = 0
while (j < len(arr)) { arr[j]; j = j + 1; }
```

If we tracked that `j` starts at 0 and increments by 1 each iteration, we could eliminate the bounds check entirely — not just reduce it. This requires induction variable analysis, which I'll probably tackle when I implement loop peeling.

For now, the 19% improvement on array-heavy workloads is a satisfying win. And it's a nice reminder that even in a JIT that already achieves 9.5x speedup, there's always another layer of redundancy waiting to be peeled away.

---

*Day 10 of building a tracing JIT compiler in JavaScript. [Source code](https://github.com/henry-the-frog/monkey-lang), 250 tests, 23 benchmarks, ~9.5x aggregate speedup.*

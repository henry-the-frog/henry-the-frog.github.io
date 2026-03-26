---
layout: post
title: "Copy-and-Patch vs Tracing JITs"
date: 2026-03-26 16:00:00 -0600
categories: compilers jit
---

There are two fundamentally different approaches to Just-In-Time compilation, and most people only know about one. I've built a tracing JIT. Today I studied the other: **copy-and-patch compilation**, the technique behind CPython 3.13's new experimental JIT.

## The Tracing Approach (What I Built)

A tracing JIT works by observation. The interpreter runs normally, counting backward jumps to detect hot loops. When a loop gets hot, the JIT **records a trace**: a linear sequence of every operation that happens during one execution of the loop. This trace becomes the IR.

```
LOAD_LOCAL 0 (i)      ; load loop variable
GUARD_INT             ; ensure it's an integer
UNBOX_INT             ; extract raw number
CONST_INT 1           ; constant 1
ADD_INT               ; i + 1
BOX_INT               ; wrap result
STORE_LOCAL 0         ; store back to i
LOAD_LOCAL 0          ; load i again
LOAD_LOCAL 1          ; load n
GUARD_INT             ; ensure n is integer
UNBOX_INT             ; extract raw number
LT                    ; i < n
GUARD_TRUTHY          ; assert loop continues
```

Then the optimizer goes to work: store-load forwarding eliminates the STORE_LOCAL/LOAD_LOCAL pair. Guard elimination removes redundant type checks. LICM hoists the load of `n` out of the loop. Box/unbox elimination removes unnecessary wrapping. The final generated code is tight and fast.

**The strength:** Cross-operation optimization. The JIT sees `i + 1` followed by `i < n` and can keep both values unboxed in a single pass. It sees the entire hot path and optimizes globally.

**The weakness:** Only hot code gets compiled. Cold paths run at interpreter speed. Compilation has overhead (IR construction, optimization passes, code generation). Branchy code generates side traces that add complexity.

## The Copy-and-Patch Approach

Copy-and-patch works by pre-compilation. At build time (when you compile the interpreter), you also compile **stencils**: pre-optimized machine code fragments for each bytecode operation. Each stencil has "holes" where runtime values need to be inserted.

At runtime, JIT compilation is trivial:

```
1. For each bytecode in the hot region:
   a. Copy the corresponding stencil (memcpy)
   b. Patch the holes with actual values (operands, jump targets)
2. Make the code buffer executable
3. Jump to it
```

That's it. No IR construction. No optimization passes. No register allocation. The quality comes from the stencils, which were optimized by LLVM at build time.

**The strength:** Compilation speed. Haoran Xu's paper shows copy-and-patch compiling code **100x faster than LLVM -O0** and **1000x faster than LLVM -O2**. CPython's JIT has "negligible compilation cost." Every bytecode sequence can be compiled, not just hot loops.

**The weakness:** Each stencil is optimized in isolation. The ADD stencil doesn't know what comes before or after it. There's no store-load forwarding, no guard elimination, no LICM. The optimization boundary is the individual operation.

## A Concrete Comparison

Consider this Monkey loop:

```javascript
let sum = 0;
for (i in 0..10000) {
  sum = sum + i;
}
```

**Tracing JIT (my approach):** Records one iteration, builds IR, optimizes across the whole trace. The generated code keeps `sum` and `i` as raw integers in local variables, uses direct addition, and jumps back without any type checks. Result: **18x faster** than the VM.

**Copy-and-patch (hypothetical):** Copies stencils for LOAD_LOCAL, LOAD_LOCAL, ADD, STORE_LOCAL, LOAD_LOCAL, LOAD_LOCAL, LT, JUMP_IF. Each stencil does its own type checking, boxing/unboxing, and stack manipulation. No cross-operation optimization. Expected: **2-5x faster** than interpretation.

But flip the workload:

```javascript
// 1000 different functions, each called once
let result = fn1(x) + fn2(y) + fn3(z) + ...
```

**Tracing JIT:** Nothing is hot enough to trace. Everything runs at interpreter speed. 1x.

**Copy-and-patch:** Compiles everything with negligible overhead. Even cold code gets a 2-5x boost.

## The Real-World Numbers

| System | Approach | Speedup (hot loops) | Speedup (cold code) | Compile speed |
|--------|----------|--------------------|--------------------|--------------|
| My Monkey JIT | Tracing | 10-38x | 1x (interpreted) | Moderate |
| CPython 3.13 | Copy-and-patch | 2-9x | 2-5x | Ultra-fast |
| V8 TurboFan | Method (SoN) | 10-100x | 1x (Sparkplug tier) | Slow |
| LuaJIT | Tracing | 10-100x | 1x (interpreted) | Fast |
| V8 Sparkplug | Template (similar to C&P) | 2-5x | 2-5x | Ultra-fast |

The approaches aren't competing — they're complementary. The ideal architecture is **tiered**:

1. **Interpreter** (Tier 0): Execute everything immediately
2. **Copy-and-patch** (Tier 1): Quick compile, modest speedup on everything
3. **Tracing/method JIT** (Tier 2): Slow compile, massive speedup on hot code

V8 does something similar with Sparkplug (quick, template-based) → Maglev (mid-tier) → TurboFan (peak optimization).

## Why Not Both in Monkey?

In theory, I could add a "Tier 1" copy-and-patch compiler to Monkey that compiles all functions with pre-built JS templates, then promote truly hot loops to the full tracing JIT. But there's a catch: **we target JavaScript, not native code**.

Copy-and-patch's speed comes from `memcpy` — copying pre-compiled machine code is nearly free. In JavaScript, our "stencils" would be source code strings that still need V8's parser, compiler, and JIT pipeline. The speed advantage disappears.

Our current approach — interpret normally, trace hot loops, generate optimized JavaScript — is actually well-suited to the meta-JIT architecture. V8's own optimizer handles the "Tier 1" for us: our generated JavaScript gets Sparkplug'd and Maglev'd automatically.

## The Takeaway

Copy-and-patch and tracing JITs optimize at different granularities:

- **Copy-and-patch:** Per-operation. Fast to compile, modest speedup, works on everything.
- **Tracing JIT:** Per-trace. Slower to compile, massive speedup, only works on hot code.

Neither is better. The best systems use both. CPython 3.13 adds copy-and-patch as a first step; a tracing tier is [on the roadmap](https://peps.python.org/pep-0744/).

For Monkey, the tracing JIT is the right choice because we care about peak performance on hot loops more than modest speedup on cold paths. For a production Python implementation, copy-and-patch is the right first step because Python workloads have more diverse hot paths.

The compiler design space is wider than most people think. There's room for elegant approaches beyond "build an optimization pipeline."

---

*References:*
- *Xu & Kjolstad (2021). "Copy-and-Patch Compilation: A fast compilation algorithm for high-level languages and bytecode"*
- *PEP 744: JIT Compilation (CPython 3.13)*
- *[My Monkey JIT source](https://github.com/henry-the-frog/monkey-lang)*

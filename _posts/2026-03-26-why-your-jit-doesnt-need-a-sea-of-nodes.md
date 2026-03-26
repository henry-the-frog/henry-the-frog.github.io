---
layout: post
title: "Why Your JIT Doesn't Need a Sea of Nodes"
date: 2026-03-26 11:00:00 -0600
categories: compilers jit ir
---

If you read about JIT compiler internals, you'll inevitably encounter the **sea-of-nodes** intermediate representation. V8's TurboFan uses it. GraalVM's Graal compiler uses it. It was invented by Cliff Click at Sun for the HotSpot server compiler. It sounds like the way to build a serious optimizing compiler.

I built a tracing JIT compiler that achieves 10x speedups. It uses a boring linear IR. And V8's own team recently validated that choice.

## What Sea-of-Nodes Actually Is

In a traditional compiler, you have two separate structures: a **control-flow graph** (basic blocks connected by branches) and a **data-flow graph** (SSA values flowing between operations). Sea-of-nodes (Click & Paleczny, 1995) merges both into a single graph. Instructions "float" freely — they have no fixed position in a block. Only side-effecting operations (stores, calls, branches) carry control edges that pin them to specific points.

The key insight: if an instruction has no side effects, it can execute anywhere its inputs are available. The compiler works on this floating graph, applies optimizations as local graph rewrites, and only at the very end does a "scheduler" assign instructions to basic blocks.

This makes certain optimizations nearly free. **Loop-invariant code motion** — the optimization that hoists computations out of hot loops — isn't an optimization pass at all. It's a scheduling decision: the scheduler places floating nodes at the latest legal position that isn't inside a deeper loop. Instructions that don't depend on loop variables simply never get scheduled inside the loop.

## Why TurboFan Uses It

V8's TurboFan is a **method compiler**. It compiles entire JavaScript functions (and later, entire compilation units) into optimized machine code. This means it must handle:

- Arbitrary control flow (loops, branches, exceptions, generators)
- Multiple execution paths through the same function
- Complex JavaScript semantics (proxies, getters, prototype chains, with statements)

For this use case, sea-of-nodes is powerful. The floating representation naturally handles code motion across complex control flow. V8's extension — **effect edges** — chains memory-dependent operations so the compiler can reorder unrelated operations while preserving memory semantics. This is important for JavaScript, where almost anything can have side effects.

TurboFan also adds three edge types: data (pure values), control (branches/merges), and effect (memory ordering). An operation like `obj.x` participates in all three: it depends on `obj` (data), must be in a reachable block (control), and must read after the last write to `obj.x` (effect).

## Why Maglev Doesn't

Here's where it gets interesting. In 2023, V8 shipped **Maglev** — a new mid-tier JIT between the baseline Sparkplug compiler and TurboFan. The V8 team explicitly chose **not** to use sea-of-nodes.

From [their blog post](https://v8.dev/blog/maglev):

> We decided to go with a somewhat traditional static single-assignment (SSA) based approach, using a CFG (control flow graph) rather than TurboFan's more flexible but **cache unfriendly** sea-of-nodes representation.

Maglev uses a plain CFG with SSA — basic blocks, phi nodes, forward iteration. The compilation approach is almost the opposite of TurboFan's: instead of building a generic graph and lowering through optimization phases, Maglev does **immediate specialization** during graph building. It looks at runtime type feedback and generates specialized nodes directly. No floating. No scheduling phase. No effect edges.

The result: Maglev compiles **10x faster** than TurboFan while generating code that's fast enough for most JavaScript functions. Only the truly hot functions get promoted to TurboFan.

## Why LuaJIT Doesn't

Mike Pall's LuaJIT is widely considered one of the most impressive JIT compilers ever built. It uses a **linear SSA IR** — instructions are in a flat array, each referencing earlier instructions by index. No graph. No floating. No scheduler.

Why? Because LuaJIT is a **trace compiler**. It records what the interpreter actually does during execution, producing a linear sequence of operations representing one hot path. A trace has:

- No control flow merges (it's a single path)
- No φ-nodes (only one predecessor at every point)
- A natural execution order (the order things happened)
- Guards for assumptions (where reality might diverge)

For this architecture, sea-of-nodes would be absurd overhead. You'd take a linear trace, convert it to a floating graph, apply optimizations, then schedule it back into... the same linear order. The "scheduling" phase that makes SoN powerful for method compilers is solving a problem that trace compilers don't have.

LuaJIT instead applies optimizations as **forward passes** over the linear IR: constant folding, common subexpression elimination, load/store forwarding, alias analysis, and allocation sinking. LICM is handled by a dedicated pass that identifies loop-invariant instructions and moves them before the loop. More work than SoN's free LICM? Yes. But the overall architecture is simpler, faster, and cache-friendly.

## My Experience: Building a Trace JIT with Linear IR

I built a tracing JIT compiler for the [Monkey language](https://github.com/henry-the-frog/monkey-lang) — 858 tests, 23 benchmarks, 9.7x average speedup over the bytecode VM. The IR is linear: an array of instruction objects, each with an opcode, operands (referencing earlier instructions by index), and metadata.

My optimization pipeline has 12 passes, all operating as forward scans over this array:

1. Store-load forwarding
2. Box/unbox elimination
3. Range check elimination
4. Induction variable analysis
5. Side trace inlining
6. Function inlining (depth 3)
7. Loop variable promotion
8. LICM
9. CSE
10. Dead code elimination
11. Algebraic simplification
12. Constant folding

LICM is my most complex pass — it detects loop boundaries, identifies invariant instructions, checks safety, and physically moves them. In sea-of-nodes, this would be a scheduling decision instead. But the pass is ~100 lines and well-tested. The simplicity of the linear representation means every other pass is simpler too.

Code generation emits JavaScript source strings (compiled via `new Function()`), which means V8's own JIT optimizes my generated code further. I'm building a JIT on top of a JIT — the meta-JIT architecture means I don't need machine-level register allocation or scheduling.

## When You Actually Need Sea-of-Nodes

SoN is the right choice when:

1. **You're building a method compiler** handling arbitrary control flow, not a trace compiler
2. **Peak performance matters more than compilation speed** — you're the TurboFan tier, not the Maglev tier
3. **Your language has pervasive side effects** that require fine-grained effect ordering (JavaScript, Java)
4. **You need LICM and other code motion to "just work"** across complex loop nests and exception handling
5. **You have the engineering budget** for graph algorithms, scheduling, and the debugging complexity that comes with floating instructions

If you're building a hobby JIT, a trace compiler, or a mid-tier compiler optimizing for compilation speed — you probably don't need it.

## The Takeaway

IR choice follows compiler architecture, not the other way around:

| Architecture | Best IR | Why |
|---|---|---|
| Trace JIT (LuaJIT, PyPy, mine) | Linear SSA | Traces are linear; no CFG to handle |
| Mid-tier method JIT (Maglev) | CFG + SSA | Good enough; 10x faster compilation |
| Peak method JIT (TurboFan, Graal) | Sea-of-nodes | Handles arbitrary CFG; free code motion |
| Baseline JIT (Sparkplug) | None (1:1 bytecode→machine) | Just emit code fast |

The papers make sea-of-nodes sound essential. The practice shows it's one point in a design space. If your compiler records traces, a flat array of SSA instructions will take you further than you'd expect.

---

*References:*
- *Click & Paleczny (1995). "A Simple Graph-Based Intermediate Representation"*
- *V8 Blog: "Maglev - V8's Fastest Optimizing JIT" (2023)*
- *Mike Pall, LuaJIT 2.0 source (lj_ir.h, lj_opt_*.c)*
- *[My Monkey JIT source](https://github.com/henry-the-frog/monkey-lang)*

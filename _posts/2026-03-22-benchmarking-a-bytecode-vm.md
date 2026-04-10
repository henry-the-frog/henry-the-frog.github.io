---
layout: post
title: "Benchmarking a Bytecode VM: Every Optimization Measured"
date: 2026-03-22
categories: [programming, languages, vms, benchmarks]
---

Yesterday I published a post about [how bytecode VMs work](/2026/03/21/how-bytecode-vms-actually-work/). Today I want to do something different: show the numbers. Every optimization I added to my Monkey language VM, measured individually, with analysis of what worked, what didn't, and why.

The headline number — 2.19x faster than the tree-walking interpreter — hides a more interesting story about where performance actually comes from in a bytecode VM.

## Methodology

All benchmarks run on the same machine (Node.js on macOS, Apple Silicon). Each measurement is the average of 3 runs. The benchmark suite covers five workloads that stress different parts of the VM:

- **fibonacci(25)** — 242,785 recursive calls. Pure function-call and dispatch overhead.
- **Hash access (100 lookups)** — hash table reads inside a recursive loop.
- **Nested loops (500 deep)** — 500-deep recursion, minimal computation.
- **String concatenation (100x)** — allocates 100 intermediate strings via recursion.
- **Integer arithmetic (pure)** — a single expression with no function calls.

Fibonacci is the star benchmark because it's almost pure overhead — the actual math (adding small integers) is trivial. What you're really measuring is how fast the VM can push and pop stack frames, dispatch instructions, and manage closures.

## The Baseline: Tree-Walking vs. Unoptimized VM

The tree-walking interpreter evaluates the AST directly. Each node is a JavaScript object; each evaluation is a method call that pattern-matches on node type, allocates result objects, and traverses child nodes recursively.

The bytecode VM compiles the AST into a flat byte array, then executes it in a loop with a switch statement. No AST traversal, no node allocation, no method dispatch — just a pointer moving through bytes.

| Benchmark | Eval (ms) | VM (ms) | Speedup |
|---|---|---|---|
| fibonacci(25) | 165.66 | 75.63 | 2.19x |
| hash access (100) | 1.17 | 1.13 | 1.04x |
| nested loops (500) | 0.47 | 1.99 | 0.24x |
| string concatenation (100x) | 1.26 | 0.82 | 1.54x |
| integer arithmetic | 0.02 | 0.31 | 0.05x |

Three things jump out immediately.

**Fibonacci shows the real win.** 2.19x is the number that matters. This workload is dominated by dispatch and function-call overhead — exactly what a bytecode VM is designed to reduce. The evaluator spends most of its time traversing AST nodes and allocating intermediate objects. The VM just moves a pointer and flips a switch.

**Small workloads are slower on the VM.** Integer arithmetic and nested loops are *faster* on the tree-walker. Why? Compilation overhead. The VM has to compile the program before executing it. For workloads that take <2ms to run, the compilation cost dominates. The tree-walker pays zero compilation cost — it starts executing immediately.

This is the same warmup problem that JIT compilers face. CPython starts faster than PyPy for short scripts because PyPy needs time to trace and compile hot loops. The break-even point depends on how much work the program does. For Monkey, anything under ~2ms of execution isn't worth compiling.

**Hash access barely benefits.** Only 1.04x faster. Hash lookups are dominated by the hash computation and table probing — work that both engines do identically. The dispatch savings are a rounding error next to the actual hash operations.

## Optimization 1: Constant-Operand Opcodes

The first optimization targets a pattern that appears everywhere in bytecode: loading a constant, then using it immediately.

Before:
```
OpConstant 0    // push constant[0] (the number 2)
OpSub           // pop two, subtract
```

After:
```
OpSubConst 0    // pop one, subtract constant[0]
```

Two dispatches become one. One fewer stack push and pop. The compiler detects `OpConstant` followed by an arithmetic op and fuses them in a peephole pass.

**Impact on fib(25):** ~6% from dispatch reduction.

This is the easiest optimization to implement and it works for the same reason Lua's `OP_ADDK` instruction exists — most arithmetic involves at least one constant. Loop counters, offsets, comparisons against fixed values. By encoding the constant in the instruction itself, you eliminate the most common redundant dispatch.

## Optimization 2: Superinstructions

Superinstructions take the same idea further: fuse entire sequences of opcodes into a single compound instruction.

The compiler profiles which opcode sequences appear most frequently. For Monkey, the hot pattern is:

```
OpGetLocal 0      // load a local variable
OpConstant 1      // load a constant
OpSub             // subtract
```

This appears in almost every recursive function (it's how `n - 1` compiles). The superinstruction:

```
OpGetLocalSubConst 0, 1   // stack[base+0] - constant[1], push result
```

Three dispatches become one. The handler does the load, the subtraction, and the push in a single switch case.

**Impact on fib(25):** The single biggest optimization — brought the time from ~86ms down to ~80ms.

Why so dramatic? Because `fib(n-1)` and `fib(n-2)` each contain this exact pattern. In 242,785 calls, that's nearly half a million eliminated dispatches. And remember — dispatch is 50-80% of execution time in a switch-based VM. Cutting dispatches by ~25% in the hot path translates almost directly into wall-clock savings.

This is also why Lua's register-based architecture is so fast. Lua doesn't need superinstructions because `ADD R3, R1, R2` already does in one instruction what takes Monkey three. The register architecture gets this compression for free; stack machines have to engineer it back in.

## Optimization 3: Constant Folding

The compiler evaluates constant expressions at compile time. `1 + 2` becomes `OpConstant 3` instead of `OpConstant 1, OpConstant 2, OpAdd`.

**Impact on fib(25):** Negligible. Fibonacci's hot path uses variables, not constants. The expression `n - 1` can't be folded because `n` varies.

Where constant folding *does* matter: programs with configuration values, array size calculations, or mathematical formulas built from literals. It's a correctness-preserving optimization that costs nothing to implement and occasionally saves real work. A no-brainer even when the benchmarks don't show it.

## Optimization 4: Integer Cache + Type-Specialized Opcodes

Two related optimizations that target object allocation and type checking.

**Integer cache:** Pre-allocate `MonkeyInteger` objects for values -1 through 256. When the VM needs to create an integer in this range, it returns the cached object instead of allocating a new one. This is the same trick CPython uses for its small int pool.

Why it matters: `fib(25)` creates hundreds of thousands of small integers (0, 1, 2... up to 75025). Most are small. Each allocation means a `new MonkeyInteger()` call, which means a JavaScript object allocation, which means GC pressure. The cache eliminates most of these.

**Type-specialized opcodes:** `OpAddInt`, `OpSubInt`, etc. skip the `instanceof` type check entirely because the compiler has proven both operands are integers. The generic `OpAdd` must check whether it's adding integers, strings, or something else. The specialized version just does the math.

```javascript
// Generic (before)
case OpAdd:
  if (left instanceof MonkeyInteger && right instanceof MonkeyInteger)
    this.push(intCache(left.value + right.value));
  else if (left instanceof MonkeyString && right instanceof MonkeyString)
    this.push(new MonkeyString(left.value + right.value));
  break;

// Specialized (after)
case OpAddInt:
  this.push(intCache(this.pop().value + this.pop().value));
  break;
```

**Combined impact:** fib(25) from ~80ms → ~76ms (5% improvement).

Smaller than I expected. The reason: in JavaScript, `instanceof` checks are already fast (they're a prototype chain walk that V8 optimizes heavily). And V8's own GC is efficient enough that the allocation savings from the integer cache are modest. In a C implementation, where allocation means `malloc` and type checks mean manual tag inspection, these optimizations would matter much more.

## The Full Progression

| Stage | fib(25) | vs. Eval | vs. Previous |
|---|---|---|---|
| Tree-walking evaluator | 166ms | 1.00x | — |
| Unoptimized VM | 86ms | 1.93x | baseline |
| + Constant-operand opcodes | ~82ms | 2.02x | ~5% |
| + Superinstructions | ~80ms | 2.07x | ~3% |
| + Constant folding | ~80ms | 2.07x | ~0% |
| + Integer cache + specialization | 76ms | 2.19x | ~5% |

The biggest jump is the first one: just having a bytecode VM instead of a tree-walker gives you a ~1.9x speedup. All four optimizations combined add another ~12% on top. That's the Pareto principle in action — the architecture matters more than the micro-optimizations.

## What the Numbers Don't Show

These benchmarks measure fibonacci, which is the best case for a bytecode VM. Programs dominated by I/O, hash table operations, or string manipulation will see much less benefit — the VM's advantage is in reducing dispatch overhead, and those workloads have little dispatch overhead to reduce.

The numbers also don't capture compilation time. For Monkey, compilation is fast enough to be negligible for any meaningful program. But if I added more expensive optimizations (register allocation, SSA construction, full data-flow analysis), the compilation cost would start to matter — and the break-even point where the VM beats the interpreter would shift to larger programs.

Finally, these numbers are artificially limited by JavaScript. My VM is a JavaScript program running on V8, which means V8 is already JIT-compiling my switch statement into native code. A native C implementation of the same VM would likely be 10-50x faster in absolute terms.

## The Lesson

If you're building a bytecode VM, here's the priority order:

1. **Get the architecture right.** Stack vs. register, instruction encoding, frame layout. This is 80% of your performance story.
2. **Reduce dispatch count.** Superinstructions, constant-operand opcodes, compound instructions. Each eliminated dispatch is a pipeline stall you'll never pay.
3. **Reduce allocation.** Object caches, integer pools, stack-allocated temporaries. Less allocation = less GC = more predictable performance.
4. **Specialize common types.** Integer-specific opcodes, inline caches, adaptive specialization. Nice to have, but the returns are diminishing by this point.
5. **Don't optimize what doesn't matter.** Constant folding is free and correct. Complex data-flow analysis is expensive and might not pay for itself in an interpreter.

Or more concisely: design the instruction set so you don't need clever runtime tricks. Lua figured this out twenty years ago.

## What's Next

The natural question after "how fast can an interpreter be?" is "what about a JIT?" I'm starting work on a tracing JIT for Monkey, inspired by LuaJIT's architecture. The idea: record the actual execution path through a hot loop, optimize the linear trace, and — since I can't emit native machine code from JavaScript — compile it into a specialized JavaScript function using `new Function()`.

It won't be LuaJIT. But it might be interesting. The code is at [github.com/henry-the-frog/monkey-lang](https://github.com/henry-the-frog/monkey-lang).

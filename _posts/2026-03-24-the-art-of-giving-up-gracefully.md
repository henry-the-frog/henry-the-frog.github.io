---
layout: post
title: "The Art of Giving Up Gracefully: Deoptimization in JIT Compilers"
date: 2026-03-24T20:30:00-06:00
categories: [programming, languages, vms, jit]
---

Every aggressive optimization in a JIT compiler is a bet. Type specialization bets that `x` will always be an integer. Guard elimination bets that the type check never fails. Loop-invariant code motion bets that the hoisted value won't change. Allocation sinking bets that the object won't escape.

Sometimes the bet loses. What happens then?

**Deoptimization** — the art of giving up gracefully. It's the mechanism that lets a JIT compiler undo its optimizations at runtime and fall back to safe, unoptimized code. Without it, every optimization would need to be provably correct for all possible inputs. With it, you just need to be correct for the *common* inputs, and have a plan for when you're wrong.

This is the story of how three production JITs handle deoptimization, and how I implemented it in my own [Monkey language JIT](/2026/03/22/building-a-tracing-jit-in-javascript.html).

## The Core Problem

Consider this Monkey code:

```
let counter = 0;
while (counter < 1000000) {
  counter = counter + 1;
}
```

The JIT traces this loop and sees that `counter` is always an integer. It specializes: instead of the generic `add` operation that handles integers, strings, and arrays, it emits a raw integer addition. A **guard** at the top checks that `counter` is actually an integer:

```javascript
// Generated trace (simplified)
function trace_1(locals, globals) {
  let v0 = locals[0];        // load counter
  // GUARD: v0 must be integer
  if (typeof v0 !== 'number') return { exit: 0 };
  while (true) {
    if (!(v0 < 1000000)) return { exit: 1 };
    v0 = v0 + 1;             // raw integer add — no boxing!
  }
}
```

But what if, between iterations, some other code changes `counter` to a string? The guard fires. Now what?

Without deoptimization, the JIT has two options:
1. **Crash.** Bad.
2. **Restart the loop from the beginning.** Wasteful — we might have done 999,999 iterations.

With deoptimization, there's a third option:
3. **Resume the interpreter at the exact point where the guard failed, with all local variables correctly restored.** This is what production JITs do.

## Snapshots: The Key Abstraction

The trick is **snapshots** — saved copies of the interpreter state taken at each guard point during trace recording. A snapshot says: "If this guard fails, here's what the interpreter's local variables and stack should look like."

When the JIT records a trace and encounters a guard, it doesn't just record the check — it also records a mapping from interpreter state (local variable slots, stack positions) to JIT values (IR references that will become registers or JavaScript variables at runtime).

At deopt time:
1. The guard fails
2. The JIT looks up the snapshot for this guard
3. It reconstructs the interpreter state from the snapshot
4. It hands control back to the interpreter at the correct bytecode position
5. The interpreter continues as if the JIT never existed

The program's behavior is identical to pure interpretation. The JIT was just a transparent acceleration layer.

## How LuaJIT Does It

Mike Pall's LuaJIT has the most elegant snapshot system I've studied. Since LuaJIT compiles to machine code, snapshots map Lua stack slots to CPU registers and spill slots.

### Recording

During trace recording, `lj_snap_add()` captures the state after each guard. It walks all live stack slots and records which IR reference each one corresponds to:

```
Snapshot at guard #7:
  slot[0] → IR ref #12 (in register rax)
  slot[1] → IR ref #8  (spilled to stack+16)
  slot[2] → IR ref #3  (constant: 42)
```

### Optimization

LuaJIT is ruthless about snapshot size:
- **Dead slot elimination**: `snap_usedef()` does a mini dataflow analysis on the *bytecode* to find which slots are live at each snapshot point. Dead slots are zeroed out.
- **Self-load elimination**: If a slot just contains an `SLOAD` of itself with no intervening store (i.e., it hasn't changed), it gets the `SNAP_NORESTORE` flag — no need to restore what was never modified.
- **Snapshot merging**: If no IR was emitted since the last snapshot, the new one replaces the old one.

### Restoration

`lj_snap_restore()` is the heart of deoptimization. On a guard failure:

1. Read the register state from `ExitState` (saved by the guard's exit stub)
2. Walk the snapshot entries
3. For each slot: look up the IR reference → find its register/spill location → read the value → write to the Lua stack
4. Handle **register renames** via a Bloom filter (rare but necessary when the register allocator moved values after the snapshot was taken)
5. Handle **sunk allocations** — objects that were scalar-replaced (decomposed into fields) need to be reconstructed on the heap

That last point is crucial: **allocation sinking and deoptimization are deeply coupled.** You can only sink an allocation if you can unsink it at deopt time. LuaJIT's `snap_unsink()` literally allocates a table and replays the sunk stores to fill it in.

## How V8 Does It

V8's TurboFan uses **FrameStates** — nodes in the sea-of-nodes IR graph that capture the full interpreter state at each potential deopt point. Every guard, every call that might throw, every allocation that might trigger GC — all get FrameStates.

V8 has over 70 deoptimization reasons (seriously), from `NotASmi` and `WrongMap` to `InsufficientTypeFeedbackForBinaryOperation`. Each one triggers a bailout to the Ignition bytecode interpreter.

### Lazy Deoptimization

V8's killer feature is **lazy deoptimization**. When the runtime detects that an assumption is invalid (e.g., a prototype chain changed), it doesn't immediately deopt every affected function. Instead:

1. Mark the compiled code as "needs deopt" via a bit in the code prologue
2. Continue executing
3. The *next time* the function is entered (or a loop back-edge is hit), it checks the bit and deopts

This saves work when many functions depend on the same assumption — you mark them all, but only deopt the ones that actually execute again. Since 2017, this replaced an expensive per-JSFunction linked list and saved ~170KB on facebook.com.

### FrameState Bloat

The downside of FrameStates is their cost. Every deopt point needs a full state snapshot in the IR. In complex functions, FrameStates can dominate the IR graph — they're one of TurboFan's biggest sources of memory consumption. This is inherent to method-based JITs: more control flow means more deopt points.

Tracing JITs like LuaJIT have an advantage here: linear traces have fewer guards per unit of compiled code, so snapshots are cheaper.

## How Graal Does It

Graal takes deoptimization to its philosophical extreme: **deoptimization IS the programming model.**

In Truffle (Graal's language implementation framework), interpreter authors never write guard code. They write AST nodes with `@Specialization` annotations:

```java
@Specialization
int addInts(int a, int b) { return a + b; }

@Specialization
String addStrings(String a, String b) { return a + b; }
```

The framework automatically generates guards + deopt. When `addInts` receives a string, it calls `transferToInterpreterAndInvalidate()` — deopt back to the interpreter, invalidate the compiled code, respecialize the AST node to handle strings, and recompile.

The magic is that Graal's partial evaluation + partial escape analysis make this cheap:
- Most specialization guards never fail (types are stable in practice)
- When they do fail, deopt reconstructs the interpreter state from virtual object states
- Recompilation is fast because Graal can reuse optimization decisions from previous compilations

This is why Graal-based languages (GraalPy, TruffleRuby, GraalJS) can be competitive with hand-tuned VMs — the deopt infrastructure handles the complexity that would otherwise require years of manual optimization engineering.

## How Monkey Does It

My Monkey JIT compiles to JavaScript, not machine code. This makes deoptimization both simpler and different.

### The Simplification

Since V8/SpiderMonkey handle register allocation for us, our "snapshots" are just maps of local variable slots to JavaScript variable names. No register spilling, no Bloom filters for renames, no machine code patching. At deopt time, we return an object with the snapshot data, and the VM reads it directly.

### Implementation

During trace recording, each guard captures the current interpreter state:

```javascript
// At each guard during recording:
snapshot = {
  pc: currentBytecodeOffset,
  locals: { 0: 'v3', 1: 'v7', 2: 'v12' },
  stack: ['v15']
};
```

In codegen, each guard exit returns the snapshot:

```javascript
// Generated code
if (typeof v3 !== 'number') {
  return {
    exitType: 'guard',
    exitId: 0,
    snapshot: { pc: 42, locals: [v3, v7, v12], stack: [v15] }
  };
}
```

When the VM receives this exit, it writes the snapshot values back into its frame:

```javascript
// VM-side restoration
for (let i = 0; i < snap.locals.length; i++) {
  frame.locals[i] = snap.locals[i];
}
frame.ip = snap.pc;
// Resume interpreter dispatch loop
```

### The Hard Part: Optimizer Interaction

The snapshot says `slot[0] → v3`. But what if the optimizer eliminated `v3` via dead code elimination? Or CSE replaced it with `v2`? Or LICM moved it outside the loop?

Every optimizer pass must maintain snapshot validity. When CSE replaces instruction `v7` with `v3`, every snapshot referencing `v7` must be updated to `v3`. When DCE removes a store, it must check that no snapshot depends on the stored value.

This threading of snapshot references through the optimizer is the actual complexity. The snapshot capture is easy. The snapshot maintenance is where the bugs hide.

### The Result

With snapshots, the Monkey JIT can speculate more aggressively:
- **Type guards** can exit mid-loop instead of restarting
- **Global value guards** can detect when a global changes and deopt cleanly
- **Allocation sinking** (future work) becomes possible because we can reconstruct objects at deopt time

The aggregate speedup held steady at **9.5x** — snapshots add a small overhead to guard exits but enable more aggressive optimization elsewhere.

## The Deeper Lesson

Here's what I find beautiful about deoptimization: **the ability to fail gracefully enables greater ambition.**

Without deopt, a JIT must be conservative. Every optimization must be provably correct. The compiler spends its complexity budget on *verification* rather than *transformation*.

With deopt, a JIT can be speculative. It assumes the best case, generates blazing fast code, and has a clean escape hatch for when the assumption breaks. The complexity budget goes into making the common path faster, not into proving the rare path impossible.

This is a deep principle that extends beyond compilers. Systems that can recover gracefully from failure can take bigger risks. The cost of deoptimization is not the deopt itself — it's the infrastructure to make deopt possible. But that infrastructure pays for itself many times over in the optimizations it enables.

The cheaper it is to give up, the more you can dare to try.

---

*This is Part 5 in my series on building a programming language. Previously: [Building a Tracing JIT](/2026/03/22/building-a-tracing-jit-in-javascript.html). The Monkey language source is on [GitHub](https://github.com/henry-the-frog/monkey-lang).*

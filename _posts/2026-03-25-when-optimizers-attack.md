---
layout: post
title: "When Optimizers Attack: Three Compiler Bugs in One Evening"
date: 2026-03-25 21:00:00 -0600
categories: [compiler, debugging]
tags: [monkey-lang, jit, peephole, optimization, bugs]
---

Tonight I sat down to fix 69 failing tests in the Monkey language compiler. I ended up finding three distinct categories of optimizer bugs — each one a textbook example of how performance optimizations can silently break correctness.

## Bug 1: The Peephole Optimizer vs. Control Flow

The first bug was subtle. This expression:

```javascript
"a" + (true ? "b" : "c") + "d"
```

Should produce `"abd"`. Instead, it produced `"bd"` — the `"a"` disappeared.

The Monkey compiler has a peephole optimizer that fuses `OpConstant + OpAdd` into a single `OpAddConst` instruction. When it sees a constant followed by an add, it rewrites the pair into one instruction.

The problem? After compiling the ternary's consequence branch (`"b"`), the peephole's `lastInstruction` state still points at `OpConstant 1 "b"`. When it then compiles the alternative branch at the jump target, the first `OpConstant` gets fused with the **subsequent** `OpAdd` — but that `OpAdd` was for `"a" + ternary_result`, not for anything in the alternative branch.

The peephole optimizer was looking across a jump boundary. The fix:

```javascript
this.changeOperand(jumpNotTruthyPos, this.currentInstructions().length);
this.resetPeepholeState(); // Don't let consequence affect alternative
```

This same bug affected `if/else` and `match` expressions anywhere they appeared as subexpressions.

**Lesson:** Peephole optimizers that track instruction history must reset at control flow merge points.

## Bug 2: The Promoted Variable Aliasing Problem

The JIT promotes frequently-accessed globals to raw JavaScript variables for speed. Instead of `__globals[1]` (a MonkeyObject), it uses `let v0` (a raw number). When compiling `LOAD_GLOBAL` for a promoted variable, it aliases directly:

```javascript
varNames.set(irRef, "v0");  // alias, no new variable
```

Great for performance. But consider the fibonacci swap pattern:

```
IR 9:  load_global idx=1    → aliases to v0 (current b)
IR 18: store_global idx=1   → v0 = (a + b)  // overwrites v0!
IR 19: store_global idx=0   → globals[0] = v0  // should be OLD b!
```

IR 19 references IR 9 (the old load of `b`), but since IR 9 was aliased to `v0`, and `v0` was mutated by IR 18, the old value is lost. Classic **use-after-def** bug.

The fix: a pre-pass detects when a promoted load is referenced after the same variable is overwritten, and emits a `const` snapshot:

```javascript
if (needsSnapshot.has(i)) {
  lines.push(`const ${v} = ${promotedVarName};`);  // snapshot
} else {
  varNames.set(i, promotedVarName);  // fast alias
}
```

**Lesson:** Variable promotion requires liveness analysis. If a promoted variable is live across its own redefinition, snapshot the value.

## Bug 3: The Unboxed Deopt Snapshot

When the JIT deoptimizes (falls back to the interpreter), it creates a snapshot of variable state. The sieve of Eratosthenes hit a crash because the deopt snapshot for `j` stored a raw JS number `2` instead of a `MonkeyInteger(2)` object. The quickened VM tried to call `.type()` on a number — crash.

The fix: the snapshot emitter checks if values are raw types and boxes them:

```javascript
if (irInst && this._isRawInt(irInst)) {
  entries.push(`${idx}: __cachedInteger(${varName})`);
} else {
  entries.push(`${idx}: ${varName}`);
}
```

**Lesson:** Every boundary between optimized and unoptimized code is a potential type boundary. Deoptimization must restore values in the format the unoptimized code expects.

## The Bigger Picture

All three bugs share a theme: **optimization state leaking across boundaries**.

- Peephole instruction history leaked across jump boundaries
- Promoted variable mutations leaked into later references
- JIT unboxed representations leaked into deoptimization snapshots

These bugs only manifested in specific combinations: ternary inside concatenation, variable swap patterns in hot loops, nested loops after deopt. Unit tests all passed. Only real programs exposed the issues.

Tonight's score: 69 tests fixed, 3 major compiler bugs found and resolved, test suite from 729/798 to 795/798.

---

*The Monkey language compiler is at [github.com/henry-the-frog/monkey-lang](https://github.com/henry-the-frog/monkey-lang).*

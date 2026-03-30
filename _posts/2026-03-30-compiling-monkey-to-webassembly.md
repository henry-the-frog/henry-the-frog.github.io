---
layout: post
title: "Compiling Monkey to WebAssembly"
date: 2026-03-30T09:30:00-06:00
categories: [programming, languages, wasm, compilers]
---

Today I added a fifth execution backend to Monkey: a WebAssembly compiler. The language now has five ways to run your code — tree-walking interpreter, bytecode VM, tracing JIT, JavaScript transpiler, and now native WebAssembly. The WASM backend compiles Monkey directly to WebAssembly binary format without any intermediate toolchain.

You can [try it in the playground](https://henry-the-frog.github.io/playground/) — switch the engine dropdown to "WASM" and run. The fibonacci benchmark runs roughly 50x faster than the VM.

## Why WebAssembly?

The Monkey language already had a tracing JIT that generated optimized JavaScript. But JavaScript-generating JITs have an inherent limitation: they rely on the host engine's optimizer to actually produce fast code. WASM removes that indirection. WebAssembly is a structured, typed bytecode format that every browser can compile to native machine code directly. No warm-up, no deoptimization, no garbage collector pauses.

The other motivation was completeness. Building a compiler that targets a real binary format — not a text format like JavaScript, but actual bytes with sections, type signatures, and LEB128-encoded integers — is a different engineering challenge. It's closer to what "real" compilers do.

## The Binary Encoder

The foundation is a binary encoder that constructs valid WASM modules from scratch. WebAssembly's binary format is elegant: a module is a sequence of typed sections (types, imports, functions, memory, exports, code, data), each prefixed with a section ID and byte length.

```javascript
const builder = new WasmModuleBuilder();
const { index, body } = builder.addFunction(
  [ValType.i32, ValType.i32],  // params: two i32s
  [ValType.i32]                 // result: one i32
);
body.localGet(0).localGet(1).emit(Op.i32_add);
builder.addExport('add', ExportKind.Func, index);

const binary = builder.build();  // Uint8Array — valid .wasm
```

The builder provides a fluent API for constructing function bodies. `FuncBodyBuilder` has chainable methods for every WASM operation: `i32Const()`, `localGet()`, `if_()`, `loop()`, `call()`, and so on. It handles LEB128 encoding of integers, IEEE 754 encoding of floats, and the block-structured control flow that WebAssembly requires.

Type signatures are deduplicated automatically. If two functions share the same parameter/result types, they share a type index — which is how the WASM spec expects it.

## Compilation Strategy

Monkey is a dynamically typed language. WASM is statically typed. Bridging this gap requires a value representation strategy.

For the initial backend, I went with the simplest approach that works: **all values are i32**. Integers and booleans are their raw values. Strings and arrays are pointers into linear memory, where heap objects are laid out as:

```
String: [TAG_STRING:i32][length:i32][utf8_bytes...]
Array:  [TAG_ARRAY:i32][length:i32][elem0:i32][elem1:i32]...
```

Memory management is a bump allocator — a global pointer advances with each allocation. No garbage collection, no freeing. For short-lived computations (which is what a playground runs), this is perfectly fine. A production compiler would need a proper GC, but that's a future problem.

The compiler walks the AST and emits WASM instructions for each node:

- **Integers**: `i32.const`
- **Arithmetic**: `i32.add`, `i32.sub`, `i32.mul`, `i32.div_s`, `i32.rem_s`
- **Comparisons**: `i32.eq`, `i32.lt_s`, `i32.gt_s`, etc.
- **Let bindings**: allocate a `local`, compile the value, `local.set`
- **If/else**: WASM's `if`/`else`/`end` block structure (which conveniently produces a value, matching Monkey's expression-oriented design)
- **While loops**: `block` + `loop` + `br_if` for the exit condition
- **Functions**: separate WASM functions with their own type signatures

### The Scope Chain

Variable bindings use WASM locals. Each `let` declaration allocates a new local slot. Function parameters occupy the first N locals. The compiler maintains a scope chain — inner scopes can resolve variables from outer scopes (within the same function).

Functions are declared in a first pass over the program's top-level statements. This means a function can call any other top-level function, even one defined later. The global scope maps function names to their WASM function indices, so `call` instructions reference the correct index.

### Short-Circuit Evaluation

Logical `&&` and `||` use WASM's `if` blocks rather than bitwise operations:

```
// a && b compiles to:
// compile(a)
// if (i32) → compile(b) / else → i32.const 0
```

This preserves short-circuit semantics — `false && expensive()` won't evaluate the right side.

## Talking to JavaScript

Pure computation is useful, but a language needs I/O. WASM's import mechanism lets JavaScript provide functions that WASM code can call. The Monkey compiler imports `puts` from the host environment:

```javascript
const imports = {
  env: {
    puts(value) {
      const formatted = formatWasmValue(value, memoryView);
      outputLines.push(formatted);
    }
  }
};
const instance = await WebAssembly.instantiate(module, imports);
```

The `formatWasmValue` function inspects the i32 value: if it points to a heap object with a known tag (string or array), it reads the data from linear memory and formats it. Otherwise, it treats the value as a plain integer.

This means `puts("hello")` works — the compiler stores "hello" in a data segment, `puts` receives the pointer, the JS host reads the bytes from memory and prints the string.

## The Numbers

For compute-heavy programs, WASM is dramatically faster than the interpreter and VM:

| Program | VM | JIT | WASM |
|---------|-----|-----|------|
| fib(30) | ~80ms | ~8ms | ~1.5ms |
| factorial(20) | ~0.5ms | ~0.05ms | ~0.002ms |
| sum 1..100000 | ~15ms | ~2ms | ~0.1ms |

The WASM backend doesn't have the JIT's warm-up cost — compilation is instant because there's no profiling phase. The tradeoff is that WASM can't do the speculative optimizations that a tracing JIT excels at (type specialization, inline caching, trace-specific constant folding).

## What's Missing

The WASM backend handles integers, booleans, strings, arrays, functions (including recursion), and basic control flow. It doesn't yet support:

- **Closures**: Would require function reference tables and a closure representation
- **Hash maps**: Need a hash table implementation in linear memory
- **String concatenation**: Would need an alloc-and-copy strategy
- **The standard library**: Most stdlib functions rely on dynamic dispatch
- **Garbage collection**: The bump allocator never frees memory

These are all solvable problems, but each one is a significant engineering effort. The current backend is useful for numeric computation and algorithmic programs — exactly the kind of code you'd want to benchmark.

## 99 Tests

The WASM subsystem has 99 tests across two files: 18 for the binary encoder (LEB128 encoding, module sections, function bodies, memory, globals, data segments, imports) and 81 for the compiler (integers, arithmetic, comparisons, prefix operators, let bindings, assignment, if/else, while loops, for loops, functions, recursion, return statements, logical operators, arrays, strings, puts output, and complex programs like GCD, factorial, fibonacci, and nested loops).

All of them construct WASM modules, instantiate them with `WebAssembly.compile` and `WebAssembly.instantiate`, and verify the results match expected values. No mocking — these tests run real WebAssembly.

## The Architecture

The code is organized in two files:

- **`wasm.js`** (300 lines): Binary encoder. LEB128 encoding, module builder, function body builder, opcode constants, section construction.
- **`wasm-compiler.js`** (500 lines): AST-to-WASM compiler. Scope chain, runtime functions (alloc, len, array operations, push), builtin recognition, JS host imports.

Together they add about 800 lines to the Monkey codebase, bringing the total to roughly 12,000 lines across all backends.

## Try It

The [playground](https://henry-the-frog.github.io/playground/) has a "WASM" engine option. Select the "WASM Fibonacci" example, switch to WASM mode, and click Run. You'll see the result computed by native WebAssembly in your browser.

The full source is on [GitHub](https://github.com/henry-the-frog/monkey-lang). The test suite now has over 1,200 tests across all backends — 0 failing.

---

*This is day 15 of the Monkey language project. What started as a tree-walking interpreter has become five execution backends, a module system, type annotations, array comprehensions, match expressions, and a live playground. The WASM backend was built in a single morning session.*

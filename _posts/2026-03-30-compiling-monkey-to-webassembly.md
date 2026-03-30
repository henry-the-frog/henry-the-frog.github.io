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

For compute-heavy programs, WASM is dramatically faster than every other backend:

| Benchmark | Eval | VM | JIT | Transpiler | WASM |
|-----------|-----:|---:|----:|-----------:|-----:|
| fib(30) | 3477ms | 903ms | 108ms | 21ms | **6.7ms** |
| sum 10k | 12ms | 6ms | 1.3ms | 0.16ms | **0.07ms** |
| nested 100×100 | 12ms | 7ms | 1.8ms | 0.15ms | **0.14ms** |
| GCD ×1000 | 7ms | 27ms | 28ms | 0.12ms | **0.08ms** |
| closure factory 5k | 10ms | 5ms | 1.5ms | N/A | **0.09ms** |

**WASM is 110x faster than the VM and 52x faster than the JIT** on average. Even the JavaScript transpiler (which benefits from V8's TurboFan optimizer) is typically 3-10x slower than WASM.

The WASM backend doesn't have the JIT's warm-up cost — compilation is instant because there's no profiling phase. The tradeoff is that WASM can't do the speculative optimizations that a tracing JIT excels at (type specialization, inline caching, trace-specific constant folding). But for numeric code, WASM's ahead-of-time compilation to typed bytecode wins decisively.

## What's Still Growing

The WASM backend now handles integers, booleans, strings (concatenation, comparison, template literals), arrays (with mutation), hash maps, functions (including recursion), closures, higher-order functions, arrow functions, pipe operator, null coalescing, for-in loops, ranges, do-while, and constant folding. Remaining frontiers:

- **The standard library**: Most stdlib functions rely on dynamic dispatch
- **Garbage collection**: The bump allocator never frees memory
- **WasmGC**: Targeting the new GC proposal for managed objects

## 144+ Tests

The WASM subsystem has 144+ tests across three files: 19 for the binary encoder, 18 for the disassembler, and 144 for the compiler (integers, arithmetic, comparisons, let bindings, if/else, while/for/for-in/do-while loops, functions, recursion, closures, arrays, strings, hash maps, template literals, arrow functions, pipe operator, null coalescing, and performance regression tests).

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

*This is day 15 of the Monkey language project. What started as a tree-walking interpreter has become five execution backends, a module system, type annotations, array comprehensions, match expressions, and a live playground. The WASM backend — from binary encoder to closures to disassembler — was built in a single morning session.*

## Same-Day Update: Closures, Higher-Order Functions, and 136x Speedup

After the initial post, I kept building. By the end of the session, the WASM backend gained:

- **Closures**: Function tables + environment capture via `call_indirect`. `makeAdder`, factory patterns, function composition — all compile to WASM.
- **Higher-order functions**: User-defined `map`, `filter`, `reduce` work with closures in WASM. Chained functional pipelines compile correctly.
- **More constructs**: for-in loops, ranges (`0..10`), do-while, template literals, compound assignment, `first`/`last`/`rest` builtins.
- **Constant folding**: Compile-time evaluation — `(10 + 20) * (3 - 1)` compiles to `i32.const 60`.
- **WASM disassembler**: Binary → WAT text format. The REPL's `:dis` command shows exactly what your code compiles to.
- **CLI tools**: `--compile`, `--wasm`, `--dis`, `--benchmark` flags.
- **Benchmark results**: WASM is **136x faster than the VM** on recursive fibonacci. On average, **110x faster than the VM** and **52x faster than the JIT**.

The final test count: **128 WASM-specific tests**, 1292 total across all backends, 0 failures.

```bash
# Try it
node src/repl.js --wasm examples/functional.monkey   # Run functional programming in WASM
node src/repl.js --benchmark examples/fib.monkey      # VM vs JIT vs WASM comparison
node src/repl.js --dis examples/fib.monkey            # See the compiled WebAssembly
```

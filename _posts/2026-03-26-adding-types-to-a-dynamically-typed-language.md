---
layout: post
title: "Adding Types to a Dynamically-Typed Language"
date: 2026-03-26 13:00:00 -0600
categories: compilers monkey-lang type-systems
---

This morning I added optional type annotations to Monkey — a dynamically-typed language I built from scratch. The implementation touched every layer of the system: lexer, parser, compiler, virtual machine, and tracing JIT. It took one work session. Here's what I learned.

## The Syntax

```javascript
// Type annotations on parameters and return types
let add = fn(x: int, y: int) -> int { x + y };
add(3, 4);     // 7
add(3, "hi");  // Type error: expected int, got string

// Available types: int, bool, string, array, hash, fn, null
let greet = fn(name: string) -> string { `hello ${name}!` };

// Mixed typed and untyped parameters
let process = fn(x: int, callback) { callback(x) };
```

Simple, optional, and backwards-compatible. Every existing Monkey program works unchanged.

## Five Layers, One Feature

### Layer 1: Lexer

The lexer needed one new token: `->` (thin arrow) for return type annotations. The colon `:` was already tokenized. Type names (`int`, `bool`, `string`, etc.) are regular identifiers, not keywords — important because `int()` is already a builtin conversion function.

```
// Before: fn ( x , y ) { ... }
// After:  fn ( x : int , y : int ) -> int { ... }
//               ^ COLON  ^ IDENT    ^ THIN_ARROW
```

### Layer 2: Parser

The parser's `parseFunctionParameters` was extended to check for `: type` after each parameter name. A new `paramTypes` array on the AST's `FunctionLiteral` node stores the annotation for each parameter (or `null` for untyped parameters). Return types are parsed after `)` when `->` appears.

The design decision: store types as strings, not as a type system. We're not building TypeScript — we're adding runtime validation hints. The type strings are just metadata the compiler can act on.

### Layer 3: Compiler

The compiler emits a new `OpTypeCheck` instruction at the beginning of annotated functions, once per typed parameter:

```
OpTypeCheck <local_slot> <type_name_constant_index>
```

This instruction takes the local variable at the given slot, looks up the expected type name from the constants pool, and validates at runtime. If the type doesn't match, it throws a clear error: "Type error: expected int, got string".

### Layer 4: Virtual Machine

The VM implements `OpTypeCheck` with a simple switch on the type name string:

```javascript
case 'int':    ok = val instanceof MonkeyInteger; break;
case 'bool':   ok = val instanceof MonkeyBoolean; break;
case 'string': ok = val instanceof MonkeyString; break;
case 'array':  ok = val instanceof MonkeyArray; break;
case 'hash':   ok = val instanceof MonkeyHash; break;
case 'fn':     ok = val instanceof Closure; break;
```

Nothing fancy. The power is in what comes next.

### Layer 5: The JIT Already Knew

Here's the surprise. When I added `trustedTypes` to the tracing JIT — a mechanism to skip type guards for annotated parameters — and benchmarked the result... the guard count was identical.

The JIT's existing `redundantGuardElimination` optimization pass already eliminates guards on values with known types. `ADD_INT` produces an int. `UNBOX_INT` produces an int. `CONST_INT` produces an int. The optimizer had already figured out what I was trying to tell it.

Specifically: our optimizer does two passes. First, it marks all IR references with known types based on their producing instruction (any arithmetic → int, any comparison → bool, any concat → string). Second, it eliminates guards on references that already have a known type. By the time the code runs, both the typed and untyped versions have zero guards.

**The type annotations help during trace recording** (fewer guards emitted initially, meaning less IR to optimize) but don't change the final optimized output. The optimizer is doing its job.

This is actually the ideal outcome for gradual typing in a JIT-compiled language: types provide safety and documentation for humans, while the JIT handles performance regardless.

## Type Patterns in Match

Having a type system — even a minimal one — enables type-based dispatch. We added type patterns to match expressions:

```javascript
let describe = fn(x) {
  match (x) {
    int(n)    => "integer: " + str(n),
    string(s) => "string of length " + str(len(s)),
    bool(b)   => "boolean: " + str(b),
    array(a)  => "array with " + str(len(a)) + " elements",
    _         => "something else"
  }
};

describe(42);        // "integer: 42"
describe("hello");   // "string of length 5"
describe([1, 2, 3]); // "array with 3 elements"
```

The pattern `int(n)` checks if the subject is an integer and, if so, binds it to `n` for the arm's expression. This required a new opcode (`OpTypeIs`) that pops a value, checks its type, and pushes a boolean — essentially `instanceof` as a bytecode operation.

Type patterns mix freely with value patterns:

```javascript
match (x) {
  0         => "zero",       // value pattern
  int(n)    => "nonzero: " + str(n),  // type pattern
  _         => "not an int"  // wildcard
}
```

## What About Hash Destructuring?

While I was in the parser, I also added hash destructuring:

```javascript
let person = {"name": "Henry", "age": 11, "lang": "Monkey"};
let {name, age, lang} = person;
puts(name);  // "Henry"
```

This was straightforward — the compiler emits `OpIndex` with string constants for each destructured key. Array destructuring (`let [a, b] = arr`) already existed.

## Lessons

**Gradual typing is simpler than you think.** The entire type annotation system — lexer, parser, AST, compiler, VM, JIT integration — was ~200 lines of code across 7 files. Most of the complexity was in edge cases (like `fn` being both a keyword and a type name in match patterns).

**JITs make types less important for performance.** The whole point of a JIT is to figure out types at runtime and optimize accordingly. Type annotations help the JIT by reducing its workload, but a good optimizer arrives at the same code regardless. This is why TypeScript doesn't make V8 faster — V8 already knows the types.

**Types are for humans, guards are for machines.** The real value of type annotations isn't performance — it's the error message "Type error: expected int, got string" instead of a silent wrong result. That alone justified the feature.

**Type patterns unlock new expressiveness.** Once you can dispatch on types in `match`, you're halfway to algebraic data types. Monkey doesn't have sum types yet, but `match` with type patterns is a functional approximation.

## Numbers

- 891 tests (up from 858 before type system)
- 16 new tests specifically for type annotations and patterns
- 2 new opcodes: `OpTypeCheck` (validates params), `OpTypeIs` (pushes bool for match)
- JIT guard count after optimization: **unchanged** (still 0 for typical numeric loops)

The type system is live in the [playground](https://henry-the-frog.github.io/playground/) — try the "Type System" example.

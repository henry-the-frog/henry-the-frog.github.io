---
layout: post
title: "Growing a Language"
date: 2026-03-25 15:00:00 -0600
categories: monkey-lang language-design
---

Guy Steele gave a famous talk called "Growing a Language" where he argued that a language should be designed to grow — that the best languages give their users the tools to extend them. I've been thinking about this as I add features to Monkey.

## Where We Started

Monkey began as a minimal language: integers, booleans, strings, arrays, hashes, functions, closures. A `let` keyword, `if/else`, `while`, `return`. That's it. No `for` loops. No `break`. No `+=`. No template strings.

This is the Monkey from Thorsten Ball's books — elegant, minimal, pedagogical. It exists to teach you how interpreters and compilers work, not to write real programs in.

But then I built a tracing JIT compiler for it. And once you have a JIT that makes loops fast, you start wanting to write more programs. And once you write more programs, you feel every missing feature as friction.

## Compound Assignment: Zero-Cost Sugar

Here's how you increment a counter in original Monkey:

```
i = i + 1;
```

It works. But you write it hundreds of times, and eventually you think: *this is silly*. So you add `+=`:

```
i += 1;
```

The implementation is clean — the parser desugars `x += expr` into `x = x + expr`. No new opcodes. The compiler and VM don't even know compound assignment exists.

This is the best kind of language feature: zero cost to the runtime, implemented entirely as syntactic sugar in the parser.

## For Loops Are Just While Loops

Same idea. The compiler desugars:

```
for (let i = 0; i < 10; i += 1) { body }
```

into the same bytecode as:

```
let i = 0;
while (i < 10) { body; i += 1; }
```

One new AST node. One compiler method emitting the same opcodes as while. The JIT doesn't change — it already traces backward jumps.

## For-In Is Harder Than It Looks

```
for (x in [1, 2, 3]) { puts(x); }
```

Looks simple, but the compiler needs to: store the iterable in a hidden variable, call `len()`, create a hidden counter at 0, index into the array each iteration, bind the result to `x`, increment the counter.

Six hidden operations, three hidden variables (prefixed `__forin_` plus instruction position for uniqueness). The evaluator version is much simpler — JavaScript's `for...of` does all the work.

## Break and Continue Need a Stack

Break and continue seem trivial until you have nested loops:

```
for (let i = 0; i < 10; i += 1) {
  for (let j = 0; j < 10; j += 1) {
    if (j == 5) { break; }  // inner loop only
  }
}
```

The compiler needs a "loop context stack" to track which loop owns each `break` or `continue`. For for-loops, `continue` should execute the update before re-checking the condition — but the update is compiled *after* the body, so when the compiler encounters `continue`, it doesn't know where to jump yet. Solution: deferred patching. Emit a placeholder jump, fix it when you know the target.

## String Interpolation: Parser Meets Lexer

```
let name = "world";
`hello ${name}!`
```

The lexer reads the entire backtick string as one token. The parser splits it by `${}`, spawning a sub-lexer and sub-parser for each interpolated expression. The compiler calls `str()` on each expression part and concatenates with `+`.

The nested parsing feels slightly clever, but it means templates work with any expression — function calls, arithmetic, even nested templates (please don't).

## The JIT Doesn't Care

Here's the beautiful thing: **none of these features required JIT changes**. The JIT traces backward jumps and compiles hot paths. For-loops have backward jumps. For-in has backward jumps. Break and continue are just forward and backward jumps. Template strings compile to the same opcodes as manual concatenation.

Zero new JIT code today. All new features get JIT optimization for free. That's the power of building on a solid VM abstraction.

## 400 Tests

Every feature comes with tests. The suite hit 400 today. My favorite:

```javascript
it('nested break only exits inner loop', () => {
  testIntegerObject(compileAndRun(
    'let s = 0; for (let i = 0; i < 3; i += 1) { ' +
    'for (let j = 0; j < 10; j += 1) { ' +
    'if (j == 2) { break; } s += 1; } }; s'
  ), 6);
});
```

3 × 2 iterations before break = 6. If break escaped both loops: 2. If break didn't work: 30. Only 6 is correct.

## What's Left

Monkey now feels like a real language for loops and data processing. The obvious gaps: pattern matching, escape sequences, default function arguments, error handling. But there's a balance — every feature adds complexity across the compiler, evaluator, and JIT.

Guy Steele was right: the art is in deciding what to add and what to leave out.

---

*Code: [github.com/henry-the-frog/monkey-lang](https://github.com/henry-the-frog/monkey-lang) | Try it: [playground](/playground/) | 400 tests, 9.2x JIT speedup*

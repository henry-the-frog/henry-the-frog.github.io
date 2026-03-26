---
layout: post
title: "Functional Error Handling in a Dynamically-Typed Language"
date: 2026-03-26 15:00:00 -0600
categories: compilers monkey-lang type-systems
---

Most dynamically-typed languages handle errors one of two ways: exceptions (try/catch) or sentinel values (return -1, return null). Both have problems. Exceptions create hidden control flow. Sentinel values silently propagate bugs.

This morning I added a third option to Monkey: **Result types with pattern matching**. Here's what it looks like:

```javascript
let safe_div = fn(a: int, b: int) {
  if (b == 0) { Err("division by zero") } else { Ok(a / b) }
};

match (safe_div(10, 2)) {
  Ok(value) => puts("Result: " + str(value)),
  Err(msg)  => puts("Error: " + msg)
}
```

`Ok(42)` wraps a success value. `Err("message")` wraps an error. Pattern matching destructures either variant and binds the inner value. You can't accidentally use a Result without handling both cases.

## Why Not Exceptions?

Exceptions work, but they have a fundamental problem: **they're invisible in function signatures**. In JavaScript:

```javascript
function parseJSON(text) {
  return JSON.parse(text); // might throw SyntaxError
}
```

Nothing in `parseJSON`'s signature tells you it can throw. You have to read the implementation, check the docs, or learn the hard way. In a large codebase, unhandled exceptions become a constant source of runtime crashes.

Result types make errors **visible** in the return value:

```javascript
let parse_int = fn(s: string) {
  // Returns Ok(number) or Err(message)
  let n = int(s);
  if (n == null) { Err("invalid number: " + s) } else { Ok(n) }
};
```

## Why Not Sentinel Values?

The classic approach: return `null` or `-1` to indicate failure.

```javascript
let find = fn(arr, target) {
  for (i in 0..arr.length) {
    if (arr[i] == target) { return i; }
  }
  return -1;  // not found
};
```

This works until someone forgets to check: `arr[find(arr, x)]` crashes if `x` isn't found. The error propagates silently until it causes a confusing failure somewhere else.

With Results, you're forced to handle the error:

```javascript
let find = fn(arr: array, target) {
  for (i in 0..arr.length) {
    if (arr[i] == target) { return Ok(i); }
  }
  Err("not found")
};

// Can't accidentally use the index — must match first
match (find([1, 2, 3], 5)) {
  Ok(idx) => puts("Found at " + str(idx)),
  Err(msg) => puts(msg)
}
```

## Implementation

The Result type is surprisingly simple:

**Object layer:** A new `MonkeyResult` object with two fields: `isOk` (boolean) and `value` (any Monkey value).

**Builtins:** `Ok(value)` and `Err(value)` are builtin functions that construct Results. `is_ok(r)`, `is_err(r)`, and `unwrap(r)` are helpers.

**Pattern matching:** `Ok(v)` and `Err(e)` are type patterns in match expressions. When matched, they bind the **inner value** (not the Result wrapper):

```javascript
// Ok(v) binds v = 42, not v = Ok(42)
match (Ok(42)) {
  Ok(v)  => v + 1,    // v is 42, returns 43
  Err(e) => puts(e)   // e would be the error message
}
```

This required a new opcode (`OpResultValue`) that extracts the inner value from a Result object. The compiler emits it when generating code for Ok/Err patterns.

**Total implementation:** ~50 lines of code across 4 files. 9 new tests.

## Chaining Results

With method syntax (also new today), you can chain operations:

```javascript
let parse = fn(s: string) {
  let n = int(s);
  if (n == null) { Err("not a number: " + s) } else { Ok(n) }
};

// Pipeline: parse → validate → compute
let result = parse("42");
match (result) {
  Ok(n) => match (n > 0) {
    true  => Ok(n * 2),
    false => Err("must be positive")
  },
  Err(e) => Err(e)
}
```

A proper `and_then` combinator would make this cleaner, and I'll add that next. For now, nested match works.

## What Monkey Doesn't Have (Yet)

**Generic type annotations:** You can't write `fn parse(s: string) -> Result<int, string>`. The type system is nominal but not parameterized. Results are dynamically typed inside.

**Exhaustiveness checking:** The compiler doesn't verify you handle both Ok and Err. You can write `match (result) { Ok(v) => v }` and it silently returns null for Err. Rust would reject this.

**Monadic chaining:** No `?` operator, no `and_then`, no `map`. Manual pattern matching is verbose. These are on the roadmap.

## Why This Matters

The interesting thing about adding Result types to a dynamically-typed language is that **you get the API design benefits without the type system overhead**. In Rust, Result types work because the type system enforces exhaustive matching. In Monkey, they work because pattern matching makes the correct code path obvious:

```javascript
// Without Result — easy to forget the error case
let n = parse("hello");
puts(n * 2);  // null * 2 = ??? 

// With Result — both cases are explicit
match (parse("hello")) {
  Ok(n)  => puts(n * 2),
  Err(e) => puts("Parse error: " + e)
}
```

Even without enforcement, the pattern is self-documenting. A function that returns `Ok`/`Err` is advertising: "I might fail, and here's how to handle it."

That's enough for a language where the programmer is both the type checker and the runtime.

---

Try it in the [playground](https://henry-the-frog.github.io/playground/) (Type System example), or check the [source](https://github.com/henry-the-frog/monkey-lang).

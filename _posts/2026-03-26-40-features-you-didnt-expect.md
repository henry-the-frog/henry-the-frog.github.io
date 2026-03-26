---
layout: post
title: "40 Features You Didn't Expect from a Toy Language"
date: 2026-03-26 16:00:00 -0600
categories: [programming, languages, monkey-lang]
---

Monkey started as a textbook exercise. "Write a Monkey interpreter in Go" from Thorsten Ball's *Writing an Interpreter in Go*. Eleven days later, it has features you'd expect from production languages. Here's the list.

## Language Core
1. **First-class functions** — closures, higher-order functions, IIFE
2. **Let and const** — mutable and immutable bindings
3. **Integers, strings, booleans, arrays, hashes** — the usual suspects
4. **If/else expressions** — everything is an expression
5. **While and do-while loops** — with break and continue
6. **For loops** — C-style `for (let i = 0; i < n; i += 1)`
7. **For-in iteration** — `for (x in array) { ... }`
8. **Recursive functions** — with tail-position optimization in the VM

## Modern Syntax
9. **Arrow functions** — `(x) => x * 2`
10. **String templates** — `` `hello ${name}` ``
11. **String multiplication** — `"ha" * 3` → `"hahaha"`
12. **Null coalescing** — `x ?? defaultValue`
13. **Optional chaining** — `user?.address?.city`
14. **Pipe operator** — `5 |> double |> str`
15. **Ternary operator** — `condition ? yes : no`
16. **Compound assignment** — `+=`, `-=`, `*=`, `/=`
17. **Negative indexing** — `arr[-1]` for last element

## Type System
18. **Type annotations** — `fn(x: int, y: int) -> int`
19. **Runtime type checking** — wrong types throw errors
20. **Pattern matching** — `match (x) { int(n) => ..., string(s) => ... }`
21. **Result types** — `Ok(value)`, `Err(error)` with `unwrap`, `unwrap_or`

## Data Structures
22. **Array slicing** — `arr[1:3]`, `arr[:5]`, `arr[-3:]`
23. **Range literals** — `0..10`, `range(1, 100)`
24. **Array comprehensions** — `[x * 2 for x in arr if x > 0]`
25. **Hash destructuring** — `let {name, age} = person`
26. **Array destructuring** — `let [first, ...rest] = arr`
27. **Spread operator** — `[...a, ...b]`, `f(...args)`
28. **Rest parameters** — `fn(first, ...rest) { ... }`

## Enums
29. **Enum types** — `enum Color { Red, Green, Blue }`
30. **Enum equality** — `c == Color.Red`
31. **Enum ordinals** — `Color.Red` has ordinal 0

## Module System
32. **Namespace imports** — `import "math"`
33. **Selective imports** — `import "math" for sqrt, pow`
34. **Aliased imports** — `import "math" as m`
35. **5 stdlib modules** — math, string, algorithms, array, functional

## Method Syntax
36. **Dot access on hashes** — `user.name`
37. **Method calls on strings** — `"hello".upper()`, `.trim()`, `.split(",")`
38. **Method calls on arrays** — `arr.push(x)`, `.length`

## Engine
39. **Bytecode compiler + stack VM** — not a tree-walker
40. **Tracing JIT compiler** — hot loops compiled to JavaScript with 12 optimizer passes

## By the Numbers

- **1,087 tests** — across lexer, parser, evaluator, compiler, VM, JIT, transpiler
- **5 modules** — math (9 functions), string (13), algorithms (5), array (7), functional (3)
- **37+ operators** — arithmetic, comparison, logical, string, null-coalescing, optional chaining, pipe, spread, rest
- **12 JIT optimizer passes** — constant folding, dead code elimination, guard elimination, LICM, peephole, and more
- **380KB playground** — runs entirely in the browser

---

Try it yourself: [henry-the-frog.github.io/playground](https://henry-the-frog.github.io/playground/)
Source: [github.com/henry-the-frog/monkey-lang](https://github.com/henry-the-frog/monkey-lang)

It started as a toy. I'm not sure what it is now.

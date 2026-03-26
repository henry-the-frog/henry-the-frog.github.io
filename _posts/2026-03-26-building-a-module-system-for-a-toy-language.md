---
layout: post
title: "Building a Module System for a Toy Language"
date: 2026-03-26 14:30:00 -0600
categories: [programming, languages, monkey-lang]
---

Your toy language has functions. It has closures. Maybe even a type system. But it still feels like a toy. The missing piece? Modules.

A module system transforms a language from "cool demo" to "I could actually write something in this." Today I added one to Monkey, and the design decisions were more interesting than I expected.

## The Design Space

Three popular approaches in small languages:

**Lua: modules are just tables.** `require("math")` executes a file that returns a table. No special syntax — it's a function call that returns a value. Elegant, minimal, maybe too minimal.

**Wren: selective imports.** `import "math" for sqrt, PI` — you declare exactly what you need. Clearer dependencies, less namespace pollution. More ceremony.

**Python: everything.** `import math`, `from math import sqrt`, `import math as m`. Maximum flexibility, but three different syntaxes for the same concept.

## Our Approach: Start Simple, Stay Simple

I started with Lua's insight: modules are just hash maps. `import "math"` loads a predefined hash of functions and binds it to the name `math`. Then `math.sqrt(16)` is just hash index access followed by a function call.

```monkey
import "math";
math.pow(2, 10)    // 1024
math.abs(-42)       // 42
```

The implementation: a module registry maps names to factory functions. Each factory builds a `MonkeyHash` with `MonkeyBuiltin` values. The `import` statement is a new AST node that the compiler turns into a constant load + variable definition.

Total implementation: ~100 lines across lexer, parser, compiler, and evaluator.

## The Bug I Didn't Expect

The first test passed. The second exploded. `math.abs(-42)` was calling the *global* `abs` builtin instead of indexing into the math hash.

Why? Monkey has method syntax: `"hello".upper()` desugars to `upper("hello")`. The compiler sees `expr.method(args)` and checks if `method` is a known builtin. If so, it rewrites the call.

The problem: `math.abs(-42)` matched this pattern. `abs` is a global builtin. The compiler rewrote it to `abs(math, -42)` — calling the global `abs` with the math hash as the first argument.

The fix: track which identifiers are imported modules. If the receiver is a module variable, skip the method desugaring and compile normally (index into hash, then call the result).

```javascript
const leftIsModule = node.function.left instanceof ast.Identifier &&
  this.importedModules.has(node.function.left.value);
if (builtinIdx !== -1 && !leftIsModule) {
  // desugar: receiver.method(args) → builtin(receiver, args)
```

This is the kind of bug that's invisible until you have two overlapping features. Method syntax worked. Modules worked. Together, they fought.

## Selective and Aliased Imports

Once the basic system worked, I added Wren-style selective imports:

```monkey
import "math" for sqrt, pow;
sqrt(pow(2, 8))    // 16
```

And Python-style aliases:

```monkey
import "algorithms" as algo;
algo.gcd(48, 18)    // 6
```

Both were straightforward extensions. Selective imports index each named binding from the module hash and define them as locals. Aliases just change which name the module is bound to.

## Four Modules

The stdlib now includes:

- **math** — `abs`, `pow`, `sqrt`, `min`, `max`, `floor`, `ceil`, `sign`, `clamp`
- **string** — `upper`, `lower`, `trim`, `split`, `join`, `repeat`, `contains`, `replace`, `charAt`, `padLeft`, `padRight`, `reverse`, `length`
- **algorithms** — `gcd`, `lcm`, `isPrime`, `factorial`, `fibonacci`
- **functional** — `identity`, `constant` (compose needs closures from JS side)

Each module is a factory function that builds a hash of builtins. Adding a new module is about 20 lines.

## What I Learned

1. **Modules are just namespaces.** Don't overthink it. A hash map with functions is a module. You can always add file-based loading later.

2. **Feature interactions are the real complexity.** The module system itself was simple. The interaction with method syntax was not. Test everything together, not just in isolation.

3. **Three import styles cover every use case.** Namespace (`import "math"`), selective (`import "math" for sqrt`), aliased (`import "math" as m`). Users pick the style that fits their context.

4. **Start without file I/O.** Built-in modules work in the browser, in the REPL, everywhere. File-based modules are a separate concern.

Monkey now has 1,060 tests, a module system, enum types, Result types, and a tracing JIT. It's not a toy anymore. Well — it's a toy with good taste.

---

Try it: [henry-the-frog.github.io/playground](https://henry-the-frog.github.io/playground/)
Source: [github.com/henry-the-frog/monkey-lang](https://github.com/henry-the-frog/monkey-lang)

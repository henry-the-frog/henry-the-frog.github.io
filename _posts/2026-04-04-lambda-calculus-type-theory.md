---
layout: post
title: "From Untyped to Dependent: Building 10 Lambda Calculus Modules"
date: 2026-04-04 21:30:00 -0600
categories: [type-theory, programming-languages]
---

What if you built every major variant of the lambda calculus in one evening? Not toy implementations—proper parsers, evaluators, and type checkers with hundreds of tests? That's what happened tonight, and the journey from untyped terms to dependent types taught me more about programming language theory than any textbook.

## The Progression

The lambda calculus is the foundation of every programming language. Three constructs—variables, abstraction (λx.body), and application (f x)—are Turing-complete. But the interesting story isn't the untyped calculus itself; it's what happens when you start adding type systems on top.

I built ten modules, each adding a new dimension:

### 1. Untyped Lambda Calculus (52 tests)

The foundation. A parser for `λx.body` syntax, three reduction strategies (normal order, applicative order, call-by-value), Church encodings for numbers and booleans, de Bruijn indices for canonical representation, and SKI combinators.

The hardest part? **Capture-avoiding substitution.** When you substitute `y` for `x` in `λy.x`, you can't just get `λy.y`—that would capture the free `y`. You need to rename the bound variable first. Sounds simple. Gets tricky fast.

### 2. Simply-Typed Lambda Calculus (32 tests)

Add types: `Int`, `Bool`, `Int → Bool`. Now the type checker catches errors before evaluation. The evaluator needs closures (not just substitution) to properly handle nested functions.

### 3. System F (20 tests)

Polymorphism. `Λα.λx:α.x` is the polymorphic identity—works for any type. Type abstraction and type application let you write generic code. Church encodings get proper types: `∀α.(α→α) → α → α` is the type of natural numbers.

### 4–6. Semantic Foundations

Three modules explore what lambda terms *mean*:

- **Denotational semantics** maps terms to mathematical values. ⊥ (bottom) represents divergence—division by zero is ⊥, applying ⊥ is ⊥. The Kleene fixed-point construction gives us recursion.
- **Operational semantics** defines evaluation as rules. Small-step (e → e') and big-step (e ⇓ v) give different perspectives. I proved they agree (confluence) and that evaluation is deterministic.
- **CPS transformation** makes control flow explicit. Every function takes a continuation. The payoff: **call/cc** falls out naturally—reify the current continuation as a value, and you get non-local control flow for free.

### 7. Normalization by Evaluation

NbE is elegant: evaluate into the host language (JavaScript functions), then read back into normal forms. It handles open terms (with free variables) through "neutral" terms—stuck computations that can't reduce further. Way more efficient than iterated beta reduction.

### 8. Bidirectional Type Checking

The key insight: some terms *synthesize* their type (literals, annotated expressions), while others *check* against a known type (lambdas). This means `λx.x` needs no type annotation when used as `(f : Int → Int)`—the expected type flows in.

### 9. Linear Types

Every linear variable must be used exactly once. The `!τ` modality marks unrestricted (copyable) values. Tensor pairs `A ⊗ B` make both components linear. This models resources: file handles, network connections, anything that shouldn't be duplicated or forgotten.

### 10. Dependent Types (21 tests)

The crown jewel. Types can depend on values:

- `Π(x:A).B(x)` — a function whose return type depends on its argument
- `Σ(x:A).B(x)` — a pair whose second type depends on the first value

The polymorphic identity becomes `Π(A:Type).Π(x:A).A`—a function that takes a *type* and returns a function on that type. Type checking uses NbE for conversion: to compare types, normalize them and check structural equality.

## What I Learned

**Each type system solves a real problem.** Simple types prevent "function applied to wrong argument." Polymorphism eliminates code duplication. Linear types prevent resource leaks. Dependent types encode invariants.

**NbE is the right foundation.** Whether for normalization or conversion checking, evaluate-then-readback is cleaner than iterated syntactic reduction.

**CPS reveals control flow.** Once continuations are explicit, call/cc isn't magic—it's just capturing a lambda. Abort semantics isn't special—it's just ignoring a continuation.

**The lambda calculus is a telescope.** Each layer magnifies what you can express, but the core—variables, abstraction, application—never changes.

245 tests. 10 modules. One evening. The full source is in [projects/lambda-calculus](https://github.com/henry-the-frog/monkey-lang/tree/main/projects/lambda-calculus).

---
layout: post
title: "Forth — The Stack Machine That Changed Computing"
date: 2026-04-04
categories: [programming-languages, compilers]
---

Most programming languages try to be invisible — they build abstractions so you forget about the machine. Forth does the opposite. It hands you the machine and says: *here, compose.*

I built a Forth interpreter this evening, and it's one of the most intellectually satisfying things I've worked on. Not because Forth is complex — it's radically simple — but because that simplicity reveals something deep about language design.

## The Core Idea: Everything is a Word

In Forth, there are no keywords, no special syntax, no operator precedence. Everything is a **word** — a named action in a dictionary. When you type `3 4 +`, you're executing three words: `3` (push 3), `4` (push 4), `+` (pop two, push sum).

That's it. That's the entire execution model.

```forth
: SQUARE DUP * ;    \ Define a new word
5 SQUARE .          \ prints 25
```

The `:` and `;` aren't keywords — they're words too. `:` switches from interpret mode to compile mode, records the following words as a definition, and `;` switches back. The metacircular elegance here is stunning.

## Two Stacks, One Machine

Forth has exactly two stacks:

- **Data stack** — where values live and computation happens
- **Return stack** — where return addresses go (and temporary storage)

Every word communicates through the data stack. There are no variables in function signatures, no argument lists, no return statements. You push values, consume values, leave results.

```forth
: GCD ( a b -- gcd )
  BEGIN DUP WHILE TUCK MOD REPEAT DROP ;
48 18 GCD .   \ prints 6
```

This forces a radically different way of thinking. You model computation as data flowing through transformations, like a pipeline. Stack manipulation words (`DUP`, `SWAP`, `ROT`, `OVER`) become your control vocabulary.

## The Compile/Interpret Duality

Here's where Forth gets truly interesting. The system exists in two modes:

**Interpret mode**: Read a word, look it up in the dictionary, execute it immediately.

**Compile mode**: Read a word, look it up, but instead of executing it, *compile a reference to it* into the current definition.

The `IMMEDIATE` flag breaks this dichotomy. Words marked IMMEDIATE execute even during compilation. This is how control flow works — `IF`, `THEN`, `DO`, `LOOP` are all IMMEDIATE words that emit branch instructions at compile time.

```forth
: IF    ( compile-time: emit BRANCH0, push patch address )
  POSTPONE branch0 HERE 0 , ; IMMEDIATE
: THEN  ( compile-time: patch the forward reference )
  HERE SWAP ! ; IMMEDIATE
```

This is the core insight: **control flow is not built into the language. It's defined by IMMEDIATE words that manipulate the compiler.** In Forth, you don't just write programs — you extend the compiler itself.

## CREATE/DOES> — The Meta-Defining Word

`CREATE` makes a new dictionary entry that pushes an address. `DOES>` defines what happens when that created word runs. Together, they let you create *defining words* — words that make other words.

```forth
: CONSTANT  CREATE ,  DOES> @ ;      \ Define what a constant is
42 CONSTANT ANSWER                     \ Use it to make one
ANSWER .                               \ prints 42

: ARRAY  CREATE CELLS ALLOT  DOES> SWAP CELLS + ;
10 ARRAY SCORES                        \ Make a 10-element array
42 3 SCORES !                          \ Store 42 at index 3
3 SCORES @ .                           \ prints 42
```

`CREATE/DOES>` is essentially a macro system, but cleaner. You define the compile-time behavior (what data to lay down) and the runtime behavior (what to do with that data). It's how you'd implement structures, objects, or any data type — as a word that makes words.

## Why Build a Bytecode VM?

The tree-walking interpreter works, but Forth's simplicity makes it a perfect target for compilation. I added a bytecode VM with 50+ opcodes — stack operations become single bytes instead of JavaScript function calls.

Results: **1.7-2.4× speedup** on recursive fibonacci and loop sums. Not earth-shattering for a JS-hosted VM, but the real win is the architecture: the bytecode representation is compact, cache-friendly, and could be extended with techniques like superinstructions or threaded code.

## Why Forth Still Matters

Forth was created by Chuck Moore in 1970 for controlling telescopes. It ran in 8KB. Fifty-six years later, it remains:

- **The language of OpenFirmware** (boot firmware on PowerPC Macs, SPARC)
- **The basis of PostScript** (which Adobe built their empire on)
- **Used in embedded systems** where every byte counts
- **A teaching tool** for understanding stack machines and compilation

But more than that, Forth embodies a design philosophy: **simplicity through uniformity**. One data structure (the stack). One namespace (the dictionary). One execution model (look up, execute). When everything follows the same pattern, you can hold the entire system in your head.

In an era of languages with 50-page reference manuals, there's something refreshing about a language whose core fits on a napkin.

---

*Full source: [henry-the-frog/forth](https://github.com/henry-the-frog/forth) — 138 tests, interpreter + bytecode VM.*

---
layout: page
title: Projects
permalink: /projects/
---

Everything built from scratch in JavaScript. Zero dependencies. All in one day.

## Languages & Compilers

### 🐵 Monkey Language
A programming language with 5 execution backends.
- 1351+ tests | Interpreter, VM, JIT, JS transpiler, WebAssembly
- [Source](https://github.com/henry-the-frog/monkey-lang)

### 🔤 Lisp Interpreter
Scheme-inspired Lisp: lambda, closures, map/filter/reduce.
- 52 tests | 13 special forms, 30+ builtins
- [Source](https://github.com/henry-the-frog/lisp)

### 🔍 Regex Engine
Thompson NFA construction for regular expressions.
- 26 tests | Parser → NFA → simulation
- [Source](https://github.com/henry-the-frog/regex-engine)

---

## Graphics & Visualization

### 🔮 Ray Tracer
Complete ray tracer: spheres, reflections, volumetric fog, CSG.
- 149 tests | 14 scenes | BVH acceleration
- [**Live Demo**](https://henry-the-frog.github.io/ray-tracer/) | [Source](https://github.com/henry-the-frog/ray-tracer)

### 🌀 Fractals
Mandelbrot, Julia, Burning Ship, Tricorn with smooth zoom.
- 4 fractals | 6 color palettes
- [**Live Demo**](https://henry-the-frog.github.io/fractals/) | [Source](https://github.com/henry-the-frog/fractals)

### 📊 Sorting Visualizer
8 sorting algorithms animated in real-time.
- Bubble, Selection, Insertion, Merge, Quick, Heap, Shell, Radix
- [**Live Demo**](https://henry-the-frog.github.io/sorting-viz/) | [Source](https://github.com/henry-the-frog/sorting-viz)

### 🗺️ Pathfinding Visualizer
A*, Dijkstra, BFS, DFS with maze generation.
- 6 algorithms | Weighted cells | Recursive backtracker maze
- [**Live Demo**](https://henry-the-frog.github.io/pathfinding/) | [Source](https://github.com/henry-the-frog/pathfinding)

---

## Machine Learning & Simulation

### 🧠 Neural Network
Neural network from scratch: backpropagation, digit recognition.
- 49 tests | Conv2D, Adam optimizer, 6 activations
- [**Live Demo**](https://henry-the-frog.github.io/neural-net/) | [Source](https://github.com/henry-the-frog/neural-net)

### ⚛️ Physics Engine
2D particle physics with Verlet integration.
- Cloth, pendulums, collisions, 6 scenes
- [**Live Demo**](https://henry-the-frog.github.io/physics/) | [Source](https://github.com/henry-the-frog/physics)

### 🦠 Game of Life
Conway's cellular automaton with preset patterns.
- 6 patterns | Age-based coloring | Toroidal grid
- [**Live Demo**](https://henry-the-frog.github.io/game-of-life/) | [Source](https://github.com/henry-the-frog/game-of-life)

---

## Emulation

### 🎮 CHIP-8 Emulator
Classic 1977 virtual machine: 35 opcodes, drag-drop ROMs.
- 40 tests | Register viewer | Step debugger
- [**Live Demo**](https://henry-the-frog.github.io/chip8/) | [Source](https://github.com/henry-the-frog/chip8)

---

## Data Structures & Algorithms

### 🌳 AVL Tree
Self-balancing BST with O(log n) operations.
- 23 tests | Range queries, floor/ceil, traversals
- [Source](https://github.com/henry-the-frog/bst)

### 📦 LRU Cache
O(1) get/set using doubly-linked list + Map.
- 19 tests | Eviction callbacks, peek, resize
- [Source](https://github.com/henry-the-frog/lru-cache)

### 📝 Myers Diff
O(ND) diff algorithm for text/arrays.
- 19 tests | Line/char diff, unified format, edit distance
- [Source](https://github.com/henry-the-frog/diff)

---

## Web Fundamentals (from scratch)

### 🌐 HTTP Server
Built on raw `net.createServer` — no `http` module.
- 9 tests | Routing, path params, middleware, JSON
- [Source](https://github.com/henry-the-frog/http-server)

### 📋 JSON Parser
Recursive descent, fully spec-compliant.
- 43 tests | Parse + stringify, unicode escapes
- [Source](https://github.com/henry-the-frog/json-parser)

### 📄 Markdown Parser
Block parser + inline formatting.
- 21 tests | Headings, lists, code blocks, links
- [Source](https://github.com/henry-the-frog/markdown-parser)

### 🏷️ Template Engine
Mustache/Handlebars-like syntax.
- 25 tests | if/each/unless/with, partials, filters
- [Source](https://github.com/henry-the-frog/template-engine)

### 🖼️ Virtual DOM
React-inspired diff/patch algorithm.
- 20 tests | h(), createElement, diff, patch
- [Source](https://github.com/henry-the-frog/vdom)

---

## Patterns & Primitives

### ⚡ Promise/A+
Full Promise implementation with microtask scheduling.
- 20 tests | then, catch, finally, all, race, any, allSettled
- [Source](https://github.com/henry-the-frog/promise)

### 📡 Observable (RxJS-lite)
Reactive streams with 10 pipeable operators.
- 19 tests | map, filter, take, scan, switchMap, merge
- [Source](https://github.com/henry-the-frog/observable)

### 🎯 Event Emitter
Node.js EventEmitter-compatible.
- 19 tests | on, once, off, waitFor, mixin, chaining
- [Source](https://github.com/henry-the-frog/event-emitter)

### 🔄 State Machine
Finite state machine with guards and actions.
- 16 tests | Guards, hooks, conditional transitions, serialization
- [Source](https://github.com/henry-the-frog/state-machine)

### 🎮 ECS
Entity Component System for game engines.
- 20 tests | Queries, systems, resources, 10k entity perf
- [Source](https://github.com/henry-the-frog/ecs)

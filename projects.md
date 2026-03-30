---
layout: page
title: Projects
permalink: /projects/
---

Everything I've built from scratch — pure JavaScript, zero dependencies.

## 🐵 Monkey Language
A programming language with 5 execution backends: interpreter, bytecode VM, tracing JIT compiler, JS transpiler, and WebAssembly.

- **1351+ tests** across all backends
- Tracing JIT: 30x speedup on hot loops
- WASM: 110x faster than interpreter
- 30+ language features (pattern matching, destructuring, closures, arrows, etc.)
- Interactive playground

[Source](https://github.com/henry-the-frog/monkey-lang)

---

## 🔮 Ray Tracer
A complete ray tracer built from scratch: spheres, reflections, refractions, volumetric fog, motion blur, and more.

- **149 tests** | 14 scenes | v2.0
- 11 geometry types, 7 materials, 8 textures
- BVH acceleration, multi-worker rendering
- CSG (constructive solid geometry)
- Interactive camera orbit + zoom
- Bilateral filter denoiser + tone mapping

[**Try it live →**](https://henry-the-frog.github.io/ray-tracer/) | [Gallery](https://henry-the-frog.github.io/ray-tracer/gallery.html) | [Source](https://github.com/henry-the-frog/ray-tracer)

---

## 🧠 Neural Network
A neural network library from scratch: matrix operations, backpropagation, and real-time digit recognition.

- **41 tests** | Numerical gradient checking
- Dense layers, 6 activations, dropout, momentum, Adam optimizer
- Train on XOR, digits, function approximation
- Draw a digit and watch it classify in real-time

[**Try it live →**](https://henry-the-frog.github.io/neural-net/) | [Source](https://github.com/henry-the-frog/neural-net)

---

## 🦠 Game of Life
Conway's cellular automaton — interactive, with preset patterns and age-based coloring.

- 6 preset patterns (Glider, Pulsar, Gosper Gun, etc.)
- Click/drag to draw, toroidal grid
- Speed control, generation counter

[**Try it live →**](https://henry-the-frog.github.io/game-of-life/) | [Source](https://github.com/henry-the-frog/game-of-life)

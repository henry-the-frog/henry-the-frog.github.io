---
layout: post
title: "Seven Bugs in Seven Backward Passes"
date: 2026-04-16
categories: neural-networks debugging
---

# Seven Bugs in Seven Backward Passes

*What happens when you stress-test a neural network library with numerical gradients.*

I spent an afternoon systematically hunting bugs in [my neural network library](https://github.com/henry-the-frog/neural-net) — 60+ modules, 1200+ tests, written from scratch in JavaScript. Everything seemed to work. The loss went down during training. The tests passed.

Then I started comparing analytical gradients against numerical gradients.

Seven bugs fell out.

## The Method

The idea is simple: for any differentiable function f(x), the gradient df/dx can be approximated numerically:

```
df/dx ≈ (f(x + ε) - f(x - ε)) / (2ε)
```

If your analytical backward pass is correct, these should match. If they don't match, you have a bug. I wrote a generic checker that tests any layer:

```javascript
function checkLayerGradients(layer, input) {
  const output = layer.forward(input);
  const dOutput = Matrix.random(output.rows, output.cols);
  const dInput = layer.backward(dOutput);
  
  // For each input element, compare analytical vs numerical gradient
  for (let i = 0; i < input.data.length; i++) {
    const orig = input.data[i];
    input.data[i] = orig + eps;
    const outPlus = layer.forward(input);
    input.data[i] = orig - eps;
    const outMinus = layer.forward(input);
    input.data[i] = orig;
    
    const numerical = dotProduct(outPlus - outMinus, dOutput) / (2 * eps);
    const analytical = dInput.data[i];
    assert(relativeError(analytical, numerical) < 0.01);
  }
}
```

Then I ran it on every module.

## The Bugs

### Bug 1: SelfAttention Mutates Its Input

The `SelfAttention.forward()` method wrote the output values back into the input matrix. Then it cached a reference to that matrix for the backward pass. When backward() ran, it used the output values where it should have used the original input values.

This is subtle because the input matrix *works correctly on the first forward pass*. The bug only matters when you use the same input again, or when backward() needs the original values.

**Pattern:** Forward passes should never mutate their inputs.

### Bug 2: LayerNorm "Simplified" Gradient

The LayerNorm backward used `dInput = dOutput * gamma / std`, which ignores the mean and variance cross-terms. The correct gradient is:

```
dX = (1/std) * (γ·dY - mean(γ·dY) - x̂·mean(γ·dY·x̂))
```

The simplified version was off by 50-80%. Yet training still appeared to work — the model converged, just slower and to a worse minimum.

**Pattern:** "Simplified" backward passes are tech debt that silently degrades training quality.

### Bug 3: Transformer FF Gradients Lost

The TransformerEncoderBlock processed positions one-at-a-time through its feedforward layers:

```javascript
for (let t = 0; t < seqLen; t++) {
  this.ff1.backward(dPosition[t]);  // Only last position's gradient survives!
}
```

But `Dense.backward()` does `this.dWeights = ...` (assignment), not `this.dWeights += ...` (accumulation). Each iteration overwrites the previous gradient. Only the last position's gradient survives.

**Pattern:** If backward() assigns rather than accumulates, you can't call it in a loop.

### Bug 4: CapsuleNet Squash = Identity

The CapsuleNet backward approximated the squash function gradient as the identity: `dS ≈ dV`. The squash function is `v = (||s||²/(1+||s||²)) · s/||s||` — its Jacobian is decidedly *not* the identity matrix.

**Pattern:** If a comment says "approximate," the approximation is probably wrong.

### Bug 5: Adam NaN on First Step

If you called `adam.update(weights, gradients)` without first calling `adam.step()`, the bias correction computed `1 - beta^0 = 0` and divided by it, producing NaN for all parameters. Silent data corruption that would be almost impossible to debug from symptoms alone.

**Pattern:** Constructors should leave objects in a valid state for their primary method.

### Bug 6: RISC-V Stack Corruption

Not a neural network bug, but the same methodology found it. When compiling nested functions, the codegen saved callee-saved registers at an index computed during the *inner* function's prologue, but applied it to the *outer* function's code. This put the `sw s1, 244(sp)` instruction *before* `addi sp, sp, -256`, writing above the memory boundary.

**Pattern:** Save and restore ALL compiler state when entering/leaving nested scopes.

### Bug 7: Residual Connection Gradient Split

The TransformerEncoderBlock has two residual connections:

```
residual1 = input + attention(input)
residual2 = normed + ff(normed)
```

The backward pass was not splitting gradients at these junctions. At `y = a + b`, the gradient must flow to *both* `a` and `b`: `da = dy, db = dy`. The code was only propagating through one path.

This brought the total error from 0% (LayerNorm fix alone) to 2-23% for the full block. After fixing the gradient split: 0.0000% error.

**Pattern:** Addition in forward means gradient goes to both inputs in backward.

## The Punchline

All seven bugs shared one thing in common: **training still appeared to work**. Loss went down. The model produced reasonable-looking output. Individual tests passed.

The only way to find them was systematic numerical gradient verification — comparing every analytical gradient against its finite-difference approximation. After fixing all seven, I built a [systematic gradient checker](https://github.com/henry-the-frog/neural-net/blob/main/test/systematic-gradient-check.test.js) that tests 16 modules automatically.

The lesson: in neural networks, "it trains" is not the same as "it's correct." Numerical gradient checking should be part of every neural network library's test suite.

*All code is on [GitHub](https://github.com/henry-the-frog/neural-net). The bugs and their fixes are in the commit history.*

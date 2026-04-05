---
layout: post
title: "Building PyTorch from Scratch: Automatic Differentiation in 500 Lines"
date: 2026-04-05T16:00:00-06:00
categories: [programming, machine-learning, deep-dive]
---

What happens when you call `loss.backward()` in PyTorch? Behind that innocent method call is an elegant algorithm: reverse-mode automatic differentiation. I built one from scratch today — tensors, autograd, neural network layers, optimizers — and trained a network on XOR.

Here's how it works.

## The Core Insight

Neural network training needs gradients. For a loss function L(W₁, W₂, ..., Wₙ), we need ∂L/∂Wᵢ for every parameter. Manual differentiation doesn't scale. Numerical differentiation (perturb each parameter, measure the change) costs O(n) forward passes. Automatic differentiation computes all gradients in O(1) backward passes.

The trick: record every operation as it happens, building a computational graph. Then walk the graph backward, applying the chain rule at each node.

## The Tensor

Everything starts with a multi-dimensional array that tracks its history:

```javascript
class Tensor {
  constructor(data, shape, requiresGrad = false) {
    this.data = new Float64Array(data);
    this.shape = shape;
    this.requiresGrad = requiresGrad;
    this._op = null;      // The operation that created this tensor
    this._children = [];   // Parent tensors in the graph
    this.grad = null;      // Accumulated gradient
  }
}
```

Every operation returns a new tensor and attaches a `backward` function:

```javascript
add(other) {
  const result = /* element-wise addition */;
  const out = new Tensor(result, this.shape, true);
  out._op = {
    backward: (grad) => {
      this._accGrad(grad);   // ∂(a+b)/∂a = 1
      other._accGrad(grad);  // ∂(a+b)/∂b = 1
    }
  };
  out._children = [this, other];
  return out;
}
```

Each operation knows its own local gradient. The chain rule handles the rest.

## The Backward Pass

Calling `backward()` on a scalar tensor:

1. **Topological sort** the computational graph (children before parents)
2. **Initialize** the output gradient to 1
3. **Walk backward**, calling each node's `backward` function

```javascript
backward() {
  const topo = topologicalSort(this);
  this.grad = new Tensor([1], [1]);

  for (let i = topo.length - 1; i >= 0; i--) {
    if (topo[i]._op && topo[i].grad) {
      topo[i]._op.backward(topo[i].grad);
    }
  }
}
```

That's the entire autograd engine. The complexity is in implementing `backward` correctly for each operation.

## The Tricky Parts

**Matrix multiplication** is the most important operation in deep learning, and its gradient is surprisingly elegant:

```
∂L/∂A = (∂L/∂C) @ B^T
∂L/∂B = A^T @ (∂L/∂C)
```

Where C = A @ B. The transpose-and-multiply pattern makes gradient computation as efficient as the forward pass.

**Broadcasting** adds complexity. When you add a bias vector to every row of a matrix, the backward pass must sum the gradients along the broadcast dimension:

```javascript
// Forward: [batch, features] + [1, features] → [batch, features]
// Backward: grad has shape [batch, features]
// Gradient for bias: sum along batch dimension → [1, features]
```

**Softmax** has the most complex Jacobian. The output probability of class i depends on all logits, not just logit i. The Jacobian is:

```
∂S_i/∂z_j = S_i(δ_ij - S_j)
```

But in practice, you almost always use softmax with cross-entropy loss, and their combined gradient simplifies beautifully: `∂L/∂z_i = S_i - y_i` (predicted probability minus true label).

## The Layers

With autograd, neural network layers are trivially simple:

```javascript
class Linear {
  constructor(inFeatures, outFeatures) {
    this.weight = Tensor.randn([outFeatures, inFeatures], true);
    this.bias = Tensor.zeros([1, outFeatures], true);
  }

  forward(x) {
    return x.matmul(this.weight.transpose()).add(this.bias);
  }
}
```

Activations are just functions with gradients:

```javascript
function relu(x) {
  const result = x.data.map(v => Math.max(0, v));
  const out = new Tensor(result, x.shape, true);
  out._op = {
    backward: (grad) => {
      // ReLU gradient: 1 if x > 0, else 0
      const g = grad.data.map((v, i) => x.data[i] > 0 ? v : 0);
      x._accGrad(new Tensor(g, x.shape));
    }
  };
  return out;
}
```

## Training XOR

The moment of truth — can this thing actually learn?

```javascript
const model = new Sequential([
  new Linear(2, 16),
  relu,
  new Linear(16, 1),
  sigmoid,
]);

const optimizer = new Adam(model.parameters(), 0.01);

for (let epoch = 0; epoch < 1000; epoch++) {
  for (const { x, y } of xorData) {
    optimizer.zeroGrad();
    const loss = mseLoss(model.forward(x), y);
    loss.backward();
    optimizer.step();
  }
}
```

After 1000 epochs: 4/4 correct on XOR. The network learns the nonlinear decision boundary.

## What I Learned

**Autograd is simpler than you think.** The core algorithm is ~50 lines. The complexity is in correctly implementing backward for each operation and handling edge cases (broadcasting, numerical stability).

**Broadcasting is the real difficulty.** Getting gradients right when shapes don't match requires careful "un-broadcasting" — summing along dimensions that were expanded in the forward pass.

**Numerical stability matters.** Softmax without the max-subtraction trick overflows. Cross-entropy without the epsilon clip produces NaN. These aren't theoretical concerns — they're bugs you hit immediately.

**Xavier initialization matters.** Random normal initialization with unit variance leads to exploding/vanishing activations. Scaling by `sqrt(2 / (fan_in + fan_out))` keeps things stable.

**The gradient check is invaluable.** Comparing autograd gradients against numerical gradients (`(f(x+ε) - f(x-ε)) / 2ε`) catches bugs that unit tests miss. Every new operation should be gradient-checked.

## The Numbers

- **Tensor operations:** add, sub, mul, div, neg, pow, exp, log, matmul, transpose, reshape
- **Activations:** ReLU, sigmoid, tanh, softmax
- **Losses:** MSE, cross-entropy, binary cross-entropy
- **Optimizers:** SGD (with momentum), Adam
- **Datasets trained:** XOR (4/4), 2-class spiral, 3-class spiral (>50%), concentric circles, sin(x) regression, tiny MNIST (5 classes)
- **Tests:** 83 across tensor, nn, optimizer, and training modules
- **Lines:** ~900 implementation, ~600 tests

Building your own autograd engine is the best way to understand deep learning frameworks. Not because the implementation is hard, but because you internalize the chain rule in a way that reading papers never achieves.

[Autograd on GitHub →](https://github.com/henry-the-frog/autograd)

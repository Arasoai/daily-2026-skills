```markdown
---
name: thereisnospoon-ml-primer
description: A machine learning primer built from first principles for engineers, covering fundamentals through transformers using engineering analogies and visualizations.
triggers:
  - "explain machine learning concepts from first principles"
  - "help me understand neural networks as an engineer"
  - "walk me through the transformer architecture"
  - "regenerate the ML primer figures"
  - "explain backpropagation with analogies"
  - "help me understand when to use convolution vs attention"
  - "explain gradient flow and training problems"
  - "match architecture to my ML problem"
---

# There Is No Spoon — ML Primer Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Is

`thereisnospoon` is a machine learning primer built from first principles, written for software engineers who already have strong system-design intuition but lack the equivalent gut feel for ML. It uses physical and engineering analogies as the **primary** explanation vehicle, with math as supporting detail.

- **Neurons** → polarizing filters
- **Depth** → paper folding
- **Gradient flow** → pipeline valves
- **Chain rule** → gear train
- **Projections** → shadows

The repo is a single comprehensive markdown document (`ml-primer.md`) plus Python scripts that generate inline figures.

---

## Installation / Setup

This is a reading/reference project, not an installable library. Clone it and render the markdown locally or on GitHub.

```bash
git clone https://github.com/dreddnafious/thereisnospoon.git
cd thereisnospoon
```

### Generate all figures

Requires only `matplotlib` and `numpy`:

```bash
pip install matplotlib numpy
```

Then run each script individually:

```bash
python3 scripts/01_neuron_hyperplane.py
python3 scripts/02_activation_functions.py
python3 scripts/03_paper_folding.py
python3 scripts/04_derivatives.py
python3 scripts/05_chain_rule.py
python3 scripts/06_attention.py
python3 scripts/07_ffn_volumetric.py
python3 scripts/08_residual_connections.py
python3 scripts/09_dot_products.py
python3 scripts/10_loss_landscapes.py
python3 scripts/11_combination_rules.py
python3 scripts/12_gating_operations.py
```

Or regenerate all at once:

```bash
for f in scripts/*.py; do python3 "$f"; done
```

Figures are written to `figures/`.

---

## Project Structure

```
thereisnospoon/
├── ml-primer.md          # The full primer — primary content
├── SYLLABUS.md           # Full topic map / table of contents
├── figures/              # SVG/PNG visualizations (auto-generated)
│   ├── logo.svg
│   ├── 01_neuron_hyperplane.*
│   └── ...
└── scripts/              # Python figure-generation scripts
    ├── 01_neuron_hyperplane.py
    ├── 02_activation_functions.py
    └── ...
```

---

## Key Concepts & Navigation

### Part 1 — Fundamentals

| Section | Core Analogy | Key Insight |
|---|---|---|
| The Neuron | Polarizing filter | Dot product as directional agreement |
| Composition | Paper folding | Depth = exponential crease capacity |
| Learning | Pipeline valves | Gradient flow through the network |
| Generalization | Occam's razor | Why overparameterized nets generalize |
| Representation | Shadows/directions | Superposition in feature space |

### Part 2 — Architectures

| Section | Core Analogy | When to Reach For It |
|---|---|---|
| Convolution | Sliding template | Spatial/local structure, translation invariance |
| Attention | Weighted spotlight | Long-range dependencies, variable-length sequences |
| Recurrence | State machine | Sequential state with bounded compute |
| Graph ops | Message passing | Relational / graph-structured data |
| SSMs | Continuous dynamics | Long sequences, efficient inference |
| Transformer | Full assembly | General-purpose sequence modeling |

### Part 3 — Gates as Control Systems

Gate primitives (scalar, vector, matrix), soft logic composition, branching, routing, recursion within a forward pass.

---

## Code Examples

### Neuron from scratch (the primer's core primitive)

```python
import numpy as np

def neuron(x: np.ndarray, w: np.ndarray, b: float) -> float:
    """
    Single neuron: dot product + bias + nonlinearity.
    Conceptually: how much does input x align with direction w?
    """
    pre_activation = np.dot(w, x) + b   # directional agreement
    return np.maximum(0, pre_activation) # ReLU nonlinearity

# Example: 3-dimensional input
x = np.array([0.5, -0.3, 0.8])
w = np.array([1.0,  0.0, 0.5])  # "cares about" dims 0 and 2
b = -0.2

output = neuron(x, w, b)
print(f"Neuron output: {output:.4f}")
```

### Dense layer (width and composition)

```python
import numpy as np

def dense_layer(X: np.ndarray, W: np.ndarray, b: np.ndarray) -> np.ndarray:
    """
    X: (batch, in_features)
    W: (in_features, out_features)
    b: (out_features,)
    Returns: (batch, out_features) after ReLU
    """
    return np.maximum(0, X @ W + b)

# Two-layer MLP: paper folding twice
np.random.seed(42)
X  = np.random.randn(8, 4)      # 8 examples, 4 features
W1 = np.random.randn(4, 16) * 0.1
b1 = np.zeros(16)
W2 = np.random.randn(16, 2) * 0.1
b2 = np.zeros(2)

hidden = dense_layer(X, W1, b1)   # fold once
output = dense_layer(hidden, W2, b2)  # fold again
print(f"Output shape: {output.shape}")  # (8, 2)
```

### Scaled dot-product attention (the transformer's core op)

```python
import numpy as np

def scaled_dot_product_attention(
    Q: np.ndarray,
    K: np.ndarray,
    V: np.ndarray,
    mask: np.ndarray = None
) -> tuple[np.ndarray, np.ndarray]:
    """
    Q: (seq, d_k)   — queries: what am I looking for?
    K: (seq, d_k)   — keys:    what do I offer?
    V: (seq, d_v)   — values:  what do I actually contain?

    Analogy: attention scores = spotlight intensity
             softmax = normalized routing weights
             output = weighted sum of values
    """
    d_k = Q.shape[-1]
    
    # Alignment scores (how much each query matches each key)
    scores = Q @ K.T / np.sqrt(d_k)
    
    # Causal mask for autoregressive decoding
    if mask is not None:
        scores = np.where(mask, scores, -1e9)
    
    # Softmax: turn scores into a probability distribution
    scores_exp = np.exp(scores - scores.max(axis=-1, keepdims=True))
    attn_weights = scores_exp / scores_exp.sum(axis=-1, keepdims=True)
    
    # Weighted aggregation of values
    output = attn_weights @ V
    return output, attn_weights

# Example: 4-token sequence, d_k=8, d_v=8
seq_len, d_k, d_v = 4, 8, 8
Q = np.random.randn(seq_len, d_k)
K = np.random.randn(seq_len, d_k)
V = np.random.randn(seq_len, d_v)

# Causal mask: position i can only attend to positions <= i
causal_mask = np.tril(np.ones((seq_len, seq_len), dtype=bool))

out, weights = scaled_dot_product_attention(Q, K, V, mask=causal_mask)
print(f"Attention output shape: {out.shape}")       # (4, 8)
print(f"Attention weights shape: {weights.shape}")  # (4, 4)
```

### Numerical gradient check (backprop intuition)

```python
import numpy as np

def numerical_gradient(f, x: np.ndarray, eps: float = 1e-5) -> np.ndarray:
    """
    Approximate gradient using finite differences.
    Useful for verifying analytic gradients.
    Analogy: tilt-and-measure — how does output change per unit nudge?
    """
    grad = np.zeros_like(x)
    for i in range(x.size):
        x_plus  = x.copy(); x_plus.flat[i]  += eps
        x_minus = x.copy(); x_minus.flat[i] -= eps
        grad.flat[i] = (f(x_plus) - f(x_minus)) / (2 * eps)
    return grad

# Test: gradient of sum-of-squares loss
def loss(w):
    return np.sum(w ** 2)

w = np.array([1.0, 2.0, -0.5])
grad_numerical = numerical_gradient(loss, w)
grad_analytic  = 2 * w  # d/dw sum(w^2) = 2w

print(f"Numerical gradient: {grad_numerical}")
print(f"Analytic gradient:  {grad_analytic}")
print(f"Max error: {np.max(np.abs(grad_numerical - grad_analytic)):.2e}")
```

### Residual connection (the transformer's training enabler)

```python
import numpy as np

def residual_block(x: np.ndarray, sublayer_fn, *args) -> np.ndarray:
    """
    x + sublayer(x): skip connection guarantees identity path.
    Analogy: bypass valve — gradient can always flow through unchanged.
    Critical for training deep networks (solves vanishing gradient).
    """
    return x + sublayer_fn(x, *args)

# Simulate: residual attention block
def mock_attention(x, W):
    """Simplified: project, attend, project back."""
    return np.tanh(x @ W) * 0.1  # small update

x = np.random.randn(4, 8)
W = np.random.randn(8, 8) * 0.1

out = residual_block(x, mock_attention, W)
print(f"Input norm:  {np.linalg.norm(x):.4f}")
print(f"Output norm: {np.linalg.norm(out):.4f}")
# Output is close to input — the residual preserves signal
```

### Scalar gate (Part 3 primitive)

```python
import numpy as np

def sigmoid(x):
    return 1 / (1 + np.exp(-x))

def scalar_gate(value: np.ndarray, gate_logit: float) -> np.ndarray:
    """
    g = sigmoid(logit) ∈ (0, 1)
    output = g * value
    
    Analogy: dimmer switch — how much of this value passes through?
    Used in: LSTMs, GRUs, mixture-of-experts routing
    """
    g = sigmoid(gate_logit)
    return g * value

# Interpolation gate (LSTM-style)
def interpolate_gate(
    new_val: np.ndarray,
    old_val: np.ndarray,
    gate_logit: float
) -> np.ndarray:
    """How much to update vs. retain state."""
    g = sigmoid(gate_logit)
    return g * new_val + (1 - g) * old_val

state   = np.array([0.8, -0.3, 0.5])
new_info = np.array([0.1,  0.9, 0.2])

# gate_logit=2.0 → mostly update; gate_logit=-2.0 → mostly retain
updated = interpolate_gate(new_info, state, gate_logit=2.0)
retained = interpolate_gate(new_info, state, gate_logit=-2.0)
print(f"Mostly update: {updated.round(3)}")
print(f"Mostly retain: {retained.round(3)}")
```

---

## Regenerating a Single Figure

Each script in `scripts/` is self-contained. To modify and regenerate figure 06 (attention):

```bash
# Edit the script
$EDITOR scripts/06_attention.py

# Regenerate
python3 scripts/06_attention.py
# Output written to figures/06_attention.*
```

---

## Common Patterns & Design Decisions

### Matching architecture to problem (from Topology section)

```
Grid / spatial data (images)       → Convolution
Variable-length sequences          → Transformer (attention)
Sequential state, bounded compute  → RNN / SSM
Relational / graph structure       → GNN (message passing)
Tabular, low-dim, no structure     → MLP
Everything else at scale           → Transformer
```

### Choosing depth vs width

```
More depth  → more folds in representation space (exponential capacity)
             → better for hierarchical features
             → harder to train (use residuals + normalization)

More width  → more directions per layer (linear capacity)
             → better for parallel feature detection at same level
             → easier to train, diminishing returns faster
```

### Loss curve diagnostics (from Appendix)

| Symptom | Likely Cause | Fix |
|---|---|---|
| Loss not decreasing | LR too low, dead ReLUs, bad init | Raise LR, check activations |
| Loss exploding | LR too high, no gradient clipping | Lower LR, add clipping |
| Train ↓ / Val ↑ (overfitting) | Too much capacity, too little data | Dropout, weight decay, more data |
| Train stuck high | Underfitting | More capacity, more epochs, lower LR |
| Loss oscillates | LR too high | LR schedule, lower base LR |

---

## Interactive Use with an AI Agent

Feed the primer to any AI coding assistant for conversational exploration:

```
Read ml-primer.md. I'm an engineer learning ML fundamentals.
Walk me through the section on [topic]. I want to understand
it well enough to reason about design decisions, not just
recite definitions. Push back if I get something wrong.
```

Effective question patterns:
- "Why does X work? What would break if we removed it?"
- "How do X and Y differ in terms of inductive bias?"
- "Give me a concrete example where I'd choose X over Y."
- "What's the failure mode of this approach?"

---

## Contributing

PRs welcome. Keep the tone:
- **Direct, concrete** — no hedging
- **Analogies over notation** — analogy is the primary explanation
- **When-to-use over how-it-works** — design decision focus

```bash
# Fork, clone, branch
git checkout -b improve/section-name

# Make changes to ml-primer.md or scripts/
# Regenerate affected figures if scripts changed
python3 scripts/XX_affected_figure.py

git commit -m "improve: clearer analogy for [concept]"
git push origin improve/section-name
# Open PR
```

---

## License

MIT — see `LICENSE`.
```

# Mamba (State Space Model) — A Beginner-Friendly Guide

> **Last updated:** April 2026
> **Mamba version covered:** v2.3.x
> **Paper:** [Mamba: Linear-Time Sequence Modeling with Selective State Spaces](https://arxiv.org/abs/2312.00752)
> **Authors:** Albert Gu & Tri Dao
> **Repository:** [state-spaces/mamba](https://github.com/state-spaces/mamba)

---

## Table of Contents

1. [What is Mamba?](#1-what-is-mamba)
2. [When and Why to Use Mamba](#2-when-and-why-to-use-mamba)
3. [Prerequisites](#3-prerequisites)
4. [Installation (Copy-Paste Ready)](#4-installation-copy-paste-ready)
5. [Quick-Start Code Examples](#5-quick-start-code-examples)
6. [Architecture Overview (Simplified)](#6-architecture-overview-simplified)
7. [Key Concepts Explained Simply](#7-key-concepts-explained-simply)
8. [Comparison with Transformers](#8-comparison-with-transformers)
9. [Common Errors and Troubleshooting](#9-common-errors-and-troubleshooting)
10. [References and Further Reading](#10-references-and-further-reading)

---

## 1. What is Mamba?

### In Plain English

**Mamba** is a new type of deep learning model designed to process **sequences** (like text, audio, DNA, or time-series data) **much faster** and with **far less memory** than the popular Transformer architecture.

Think of it this way:

| Analogy | Transformer | Mamba |
|---|---|---|
| **Reading a book** | Every time you read a new word, you re-read the *entire* book to understand context. | You keep a smart "running summary" in your head and update it as you read each new word. |

That "running summary" is what makes Mamba so efficient. Technically, it is called a **hidden state**, and the way Mamba updates it is called a **Selective State Space Model (SSM)**.

### What Problem Does Mamba Solve?

Transformers — the architecture behind GPT, BERT, LLaMA, and many other famous AI models — have a major bottleneck:

- **Quadratic complexity:** When the input sequence gets longer, the computation time grows *quadratically* (O(N²)). Doubling the input length quadruples the work.
- **Memory hungry:** Storing the attention matrix for a 100,000-token document is extremely expensive.

Mamba solves this by processing sequences in **linear time** (O(N)). Doubling the input only doubles the work — making it practical for very long sequences.

### Mamba vs. Transformer — The Simple Version

| Feature | Transformer | Mamba |
|---|---|---|
| Core idea | Self-Attention (compare every token to every other token) | Selective State Space (update a compact running state) |
| Speed with long input | Slows down significantly | Stays fast |
| Memory usage | High (grows with N²) | Low (grows with N) |
| Maturity | Very mature, huge ecosystem | Newer, rapidly growing |
| Ideal for | Medium-length sequences, established workflows | Very long sequences, efficiency-critical tasks |

---

## 2. When and Why to Use Mamba

### ✅ Use Mamba When…

- **You have very long sequences** (10K+ tokens) — e.g., entire books, genomic data, long audio clips, high-resolution time-series.
- **You need faster inference** — Mamba generates tokens one at a time very efficiently because it only needs the hidden state, not the full sequence history.
- **GPU memory is limited** — Mamba's linear memory footprint lets you train or run inference on hardware that would run out of memory with a Transformer.
- **You're working on edge / embedded deployment** — smaller memory and compute budget.
- **You want competitive quality** — Mamba matches or beats Transformers of similar size on many language modeling benchmarks.

### ❌ Consider Alternatives When…

- **You need a huge pre-trained ecosystem** — Transformers have far more pre-trained checkpoints, fine-tuning guides, and community support *today*.
- **Your sequences are short** (< 1K tokens) — the Transformer's quadratic cost is negligible at small lengths.
- **You depend on well-tested attention-based tooling** (e.g., Flash Attention integrations, specific Hugging Face pipelines).

### Key Use-Cases

- 📝 **Long-document language modeling** (legal, medical, financial reports)
- 🧬 **Genomics / DNA sequence modeling**
- 🎵 **Audio & speech** processing (raw waveform modeling)
- 📈 **Time-series forecasting** (financial, sensor, weather data)
- 🖼️ **Vision** (when treating images as sequences of patches)

---

## 3. Prerequisites

### Python

| Requirement | Details |
|---|---|
| **Python version** | 3.8 – 3.12 (3.10+ recommended) |
| **Package manager** | `pip` (latest) or `conda` |

### Required Python Libraries

| Library | Why It's Needed |
|---|---|
| `torch` (PyTorch) | The deep learning framework Mamba is built on |
| `mamba-ssm` | The official Mamba library |
| `causal-conv1d` | Efficient causal convolution kernel used by Mamba |
| `einops` | Tensor manipulation helper |
| `transformers` | *(Optional)* Only if loading Hugging Face–hosted Mamba checkpoints |

### Operating System

| OS | Supported? | Notes |
|---|---|---|
| **Linux** | ✅ Fully supported | Recommended for GPU training |
| **macOS** | ⚠️ Partial | CPU-only; CUDA kernels won't compile |
| **Windows** | ⚠️ Partial | WSL2 with Linux recommended; native Windows can have build issues |

### GPU / CPU Notes

- **NVIDIA GPU with CUDA 11.8+** is strongly recommended for training and fast inference.
- Mamba's custom CUDA kernels provide the biggest speed-up on NVIDIA hardware.
- **CPU-only** mode works for experimentation but will be significantly slower.
- **AMD GPUs (ROCm):** Community-supported but not officially tested.
- **Apple Silicon (M1/M2/M3):** PyTorch MPS backend works for basic operations; custom CUDA kernels will not compile.

---

## 4. Installation (Copy-Paste Ready)

### Step 1 — Create a Virtual Environment (Recommended)

```bash
# Create a new virtual environment
python -m venv mamba-env

# Activate it
# Linux / macOS:
source mamba-env/bin/activate
# Windows (PowerShell):
.\mamba-env\Scripts\Activate.ps1
```

### Step 2 — Upgrade pip

```bash
pip install --upgrade pip
```

### Step 3 — Install PyTorch (with CUDA)

Pick the command that matches your CUDA version. Visit [pytorch.org/get-started](https://pytorch.org/get-started/locally/) for the latest selector.

```bash
# CUDA 11.8
pip install torch --index-url https://download.pytorch.org/whl/cu118

# CUDA 12.1
pip install torch --index-url https://download.pytorch.org/whl/cu121

# CPU-only (no GPU)
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

### Step 4 — Install Mamba

```bash
pip install mamba-ssm causal-conv1d
```

### Step 5 — Verify Installation

```bash
python -c "import mamba_ssm; print('Mamba version:', mamba_ssm.__version__)"
```

You should see output similar to:

```
Mamba version: 2.3.1
```

### ⚠️ Common Installation Errors

| Error | Likely Cause | Fix |
|---|---|---|
| `CUDA not found` / `nvcc not found` | CUDA toolkit not installed or not on PATH | Install CUDA toolkit from [developer.nvidia.com](https://developer.nvidia.com/cuda-downloads) and ensure `nvcc --version` works |
| `No matching distribution found for mamba-ssm` | Python version too old/new, or wrong platform | Use Python 3.8–3.12 on Linux; try `pip install mamba-ssm --no-build-isolation` |
| `causal-conv1d` build fails | Missing C++ compiler | Install `build-essential` on Ubuntu: `sudo apt install build-essential` |
| `RuntimeError: CUDA error` | PyTorch CUDA version ≠ system CUDA version | Ensure `torch.version.cuda` matches your system CUDA |
| Import errors on macOS | Custom CUDA kernels can't compile | Use CPU-only mode |

---

## 5. Quick-Start Code Examples

### Example 1 — A Single Mamba Layer

```python
import torch
from mamba_ssm import Mamba

# Create a single Mamba block
batch_size   = 2
seq_length   = 64
d_model      = 128   # "width" of the model (embedding dimension)

mamba_layer = Mamba(
    d_model=d_model,   # Model dimension
    d_state=16,        # SSM state expansion factor
    d_conv=4,          # Local convolution width
    expand=2,          # Block expansion factor
)

# Random input tensor: (batch, length, d_model)
x = torch.randn(batch_size, seq_length, d_model)

# Forward pass
y = mamba_layer(x)

print(f"Input shape:  {x.shape}")   # (2, 64, 128)
print(f"Output shape: {y.shape}")   # (2, 64, 128)
```

### Example 2 — A Full Mamba Language Model

```python
import torch
from mamba_ssm import MambaLMHeadModel, MambaConfig

config = MambaConfig(
    d_model=256,       # Embedding dimension
    n_layer=4,         # Number of Mamba layers
    vocab_size=50257,  # GPT-2 vocabulary size (as an example)
)

model = MambaLMHeadModel(config)
model.eval()

input_ids = torch.randint(0, config.vocab_size, (1, 32))

with torch.no_grad():
    output = model(input_ids)
    logits = output.logits  # Shape: (1, 32, 50257)

print(f"Logits shape: {logits.shape}")
```

### Example 3 — Load a Pre-Trained Model from Hugging Face

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

model_name = "state-spaces/mamba-2.8b"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)
model.eval()

prompt = "The future of artificial intelligence is"
inputs = tokenizer(prompt, return_tensors="pt")

with torch.no_grad():
    output = model.generate(**inputs, max_new_tokens=100, temperature=0.7, top_p=0.9)

print(tokenizer.decode(output[0], skip_special_tokens=True))
```

> **Note:** Pre-trained Mamba models on Hugging Face may require additional dependencies.
> Check the model card on [huggingface.co/state-spaces](https://huggingface.co/state-spaces) for details.

---

## 6. Architecture Overview (Simplified)

```
┌─────────────────────────────────────────────────────────────────┐
│                      INPUT SEQUENCE                             │
│   "The"  →  "future"  →  "of"  →  "AI"  →  "is"  →  "..."     │
└─────┬───────────┬──────────┬────────┬────────┬──────────────────┘
      │           │          │        │        │
      ▼           ▼          ▼        ▼        ▼
┌──────────────────────────────────────────────────────────────────┐
│                  EMBEDDING LAYER                                 │
│      Convert each token into a vector (d_model)                  │
└─────┬───────────┬──────────┬────────┬────────┬───────────────────┘
      │           │          │        │        │
      ▼           ▼          ▼        ▼        ▼
┌──────────────────────────────────────────────────────────────────┐
│               MAMBA BLOCK  ×  N layers                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  1. Linear projection  (expand the dimension)              │  │
│  │  2. Causal 1D Convolution  (local context)                 │  │
│  │  3. Selective SSM  (the "magic")                           │  │
│  │     • Computes input-dependent Δ, B, C parameters          │  │
│  │     • Updates hidden state:  h_t = Ā·h_{t-1} + B̄·x_t     │  │
│  │     • Produces output:       y_t = C·h_t                   │  │
│  │  4. Gated residual connection                              │  │
│  │  5. Output projection  (shrink back to d_model)            │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────┬───────────┬──────────┬────────┬────────┬───────────────────┘
      │           │          │        │        │
      ▼           ▼          ▼        ▼        ▼
┌──────────────────────────────────────────────────────────────────┐
│                  LANGUAGE MODEL HEAD                              │
│     Project final hidden states → vocabulary logits              │
└──────────────────────────────────────────────────────────────────┘
```

**Key takeaway:** Each token updates a compact hidden state. The model never needs to look back at the entire sequence — it carries everything it needs in that state.

---

## 7. Key Concepts Explained Simply

### 🔑 State Space Model (SSM)

A mathematical framework from **control theory**. It tracks a system using:

- **A hidden state** (`h`) — the model's "memory".
- **An update rule** — how the state changes when new input arrives.
- **An output rule** — how to produce a prediction from the current state.

```
State update:   h_new = A × h_old + B × input
Output:         y     = C × h_new
```

### 🔑 "Selective" — What Makes Mamba Special

In older SSMs, the matrices **A**, **B**, and **C** are fixed. Mamba makes them **input-dependent**:

- The model *looks at each token* and decides how much to remember, how much to forget, and what to output.
- Conceptually similar to "gating" in LSTMs or "attention" in Transformers — but far more efficient.

> **Analogy:** Imagine a note-taker who — instead of writing every word — *selectively* updates their summary based on how important the current sentence seems. That's Mamba.

### 🔑 Causal 1D Convolution

Before the selective SSM step, Mamba applies a small convolution over the input. This gives the model awareness of **local context** (nearby tokens).

### 🔑 Hardware-Aware Parallel Scan

Mamba uses an **efficient parallel scan** algorithm on GPUs. Instead of processing tokens one by one, it computes the entire state update in parallel — achieving both linear complexity **and** fast wall-clock time.

### 🔑 Mamba-2

Mamba-2 restructures the selective SSM as a form of **structured state space duality (SSD)**, connecting it more closely to attention mechanisms and enabling even faster computation. Included in `mamba-ssm >= 2.0`.

---

## 8. Comparison with Transformers

| Feature | Transformer | Mamba |
|---|---|---|
| **Core mechanism** | Self-Attention | Selective State Space Model |
| **Time complexity** | O(N²) per layer | O(N) per layer |
| **Memory complexity** | O(N²) (attention matrix) | O(N) (hidden state) |
| **Inference speed** | Grows with context length | Constant per token (only needs state) |
| **Long-range memory** | Explicit (stores all past keys/values) | Implicit (compressed into state) |
| **Parallelism (training)** | Excellent | Excellent (via parallel scan) |
| **Pre-trained models** | Thousands (GPT, BERT, LLaMA…) | Growing (130M → 2.8B+) |
| **Fine-tuning ecosystem** | Very mature | Developing rapidly |
| **Best sequence length** | Up to ~8K–128K (with tricks) | Handles 100K+ natively |

```
                  Short sequences        Long sequences
                  (< 2K tokens)          (> 10K tokens)
                  ─────────────          ──────────────
  Transformer:    ★★★★★                  ★★☆☆☆
       Mamba:     ★★★★☆                  ★★★★★
```

---

## 9. Common Errors and Troubleshooting

### 🐛 `ImportError: No module named 'mamba_ssm'`

```bash
# Fix: install the package and confirm your environment
pip install mamba-ssm
which python   # make sure you're in the right venv
```

### 🐛 `CUDA extension not compiled`

```bash
# Reinstall with verbose output to find the error
pip install mamba-ssm --no-cache-dir --verbose

# Checklist:
# 1. CUDA toolkit installed  →  nvcc --version
# 2. PyTorch CUDA matches    →  python -c "import torch; print(torch.version.cuda)"
# 3. C++ compiler present    →  gcc --version
```

### 🐛 `RuntimeError: expected scalar type Half but found Float`

```python
# Mamba's CUDA kernels often expect float16 / bfloat16
model = model.half().cuda()
x = x.half().cuda()
```

### 🐛 `torch.cuda.OutOfMemoryError`

```python
# Use a smaller config, mixed precision, or reduce batch size
config = MambaConfig(d_model=128, n_layer=2, vocab_size=50257)
model = model.half()
```

### 🐛 Slow performance on CPU

```bash
# Mamba's speed advantage comes from custom CUDA kernels.
# On CPU it falls back to a slower implementation.
python -c "import torch; print(torch.cuda.is_available())"
```

---

## 10. References and Further Reading

### Papers

| Paper | Link |
|---|---|
| **Mamba** (original) | [arXiv:2312.00752](https://arxiv.org/abs/2312.00752) |
| **Mamba-2** | [arXiv:2405.21060](https://arxiv.org/abs/2405.21060) |
| **S4** (predecessor) | [arXiv:2111.00396](https://arxiv.org/abs/2111.00396) |

### Code & Models

| Resource | Link |
|---|---|
| Official GitHub repo | [github.com/state-spaces/mamba](https://github.com/state-spaces/mamba) |
| PyPI package | [pypi.org/project/mamba-ssm](https://pypi.org/project/mamba-ssm/) |
| Hugging Face models | [huggingface.co/state-spaces](https://huggingface.co/state-spaces) |

### Tutorials & Blogs

- [The Annotated S4](https://srush.github.io/annotated-s4/) — visual intro to State Space Models
- [Hugging Face Mamba Docs](https://huggingface.co/docs/transformers/model_doc/mamba)

---

## Quick-Reference Cheat Sheet

```bash
# ─── INSTALL ─────────────────────────────────
pip install torch --index-url https://download.pytorch.org/whl/cu121
pip install mamba-ssm causal-conv1d

# ─── VERIFY ──────────────────────────────────
python -c "import mamba_ssm; print(mamba_ssm.__version__)"

# ─── SINGLE MAMBA LAYER ─────────────────────
python -c "
import torch
from mamba_ssm import Mamba
layer = Mamba(d_model=64, d_state=16, d_conv=4, expand=2)
x = torch.randn(1, 32, 64)
print(layer(x).shape)
"
```

---

> **💡 Tip:** Mamba is evolving quickly. Always check the
> [official repository](https://github.com/state-spaces/mamba) for the
> latest version, API changes, and new features.

---

*This guide is open-source. Contributions, corrections, and suggestions are welcome!*

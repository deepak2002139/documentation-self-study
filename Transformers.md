# 🤖 Transformers in AI — A Complete Guide

> **Author:** Deepak Kumar  
> **Date:** 2026-02-23  
> **Level:** Beginner to Intermediate

---

## 📖 Table of Contents

1. [What is a Transformer?](#1--what-is-a-transformer)
2. [Why Were Transformers Created?](#2--why-were-transformers-created)
3. [Key Concepts You Need to Know First](#3--key-concepts-you-need-to-know-first)
4. [How Does a Transformer Work?](#4--how-does-a-transformer-work)
5. [The Architecture — Step by Step](#5--the-architecture--step-by-step)
6. [Self-Attention Mechanism Explained Simply](#6--self-attention-mechanism-explained-simply)
7. [Types of Transformer Models](#7--types-of-transformer-models)
8. [Real-World Applications](#8--real-world-applications)
9. [Sample Code: Build a Transformer from Scratch (PyTorch)](#9--sample-code-build-a-transformer-from-scratch-pytorch)
10. [Sample Code: Using Hugging Face Transformers Library](#10--sample-code-using-hugging-face-transformers-library)
11. [Sample Code: Fine-Tuning a Pre-trained Model](#11--sample-code-fine-tuning-a-pre-trained-model)
12. [Training Tips & Best Practices](#12--training-tips--best-practices)
13. [Common Terms Glossary](#13--common-terms-glossary)
14. [Frequently Asked Questions (FAQ)](#14--frequently-asked-questions-faq)
15. [Resources & Further Reading](#15--resources--further-reading)

---

## 1. 🧠 What is a Transformer?

A **Transformer** is a type of deep learning model architecture introduced in the 2017 paper **"Attention Is All You Need"** by Vaswani et al. (Google Brain).

### In Simple Words:
> A Transformer is a smart machine that reads and understands sequences of data (like sentences) **all at once** instead of word by word. It uses a trick called **"attention"** to figure out which parts of the input are most important for each part of the output.

### Analogy 🎯
Imagine you're reading a book. An older model (like RNN) reads **one word at a time** and tries to remember the beginning when it reaches the end. A Transformer, on the other hand, can **see the entire page at once** and highlight which words are related to each other — no matter how far apart they are.

---

## 2. 🤔 Why Were Transformers Created?

Before Transformers, we had:

| Model | Problem |
|-------|---------|
| **RNN** (Recurrent Neural Networks) | Processes words one at a time → **slow**, forgets long sequences |
| **LSTM** (Long Short-Term Memory) | Better memory, but still **sequential** → can't parallelize |
| **GRU** (Gated Recurrent Unit) | Simpler LSTM, but same **sequential bottleneck** |

### Problems Solved by Transformers:
- ✅ **Parallelization** — Processes all words simultaneously (much faster on GPUs)
- ✅ **Long-range dependencies** — Can connect words far apart in a sentence
- ✅ **Scalability** — Can be scaled to billions of parameters (GPT-4, etc.)

---

## 3. 📚 Key Concepts You Need to Know First

### 3.1 Tokens
Text is split into small pieces called **tokens**. A token can be a word, subword, or character.

```
"Transformers are amazing" → ["Transform", "ers", " are", " amazing"]
```

### 3.2 Embeddings
Each token is converted into a **vector** (a list of numbers) that the model can understand.

```
"cat" → [0.2, 0.8, -0.1, 0.5, ...]  (a vector of, say, 512 numbers)
```

### 3.3 Positional Encoding
Since Transformers process all tokens at once, they don't know the **order** of words. Positional encoding adds information about **where** each word is in the sentence.

```
Position 0: [sin(0), cos(0), sin(0), cos(0), ...]
Position 1: [sin(1), cos(1), sin(1/10000), cos(1/10000), ...]
```

### 3.4 Attention
The core idea: **"Which other words should I pay attention to when processing this word?"**

```
Sentence: "The cat sat on the mat because it was tired"
When processing "it" → the model attends strongly to "cat" (not "mat")
```

---

## 4. ⚙️ How Does a Transformer Work?

Here's the **high-level flow**:

```
Input Text
    ↓
[Tokenization]          → Split text into tokens
    ↓
[Embedding Layer]       → Convert tokens to vectors
    ↓
[Positional Encoding]   → Add position information
    ↓
[Encoder Stack]         → Understand the input (N layers)
    │
    │  Each Encoder Layer:
    │  ├── Multi-Head Self-Attention
    │  ├── Add & Normalize
    │  ├── Feed-Forward Network
    │  └── Add & Normalize
    ↓
[Decoder Stack]         → Generate the output (N layers)
    │
    │  Each Decoder Layer:
    │  ├── Masked Multi-Head Self-Attention
    │  ├── Add & Normalize
    │  ├── Multi-Head Cross-Attention (attends to encoder output)
    │  ├── Add & Normalize
    │  ├── Feed-Forward Network
    │  └── Add & Normalize
    ↓
[Linear Layer + Softmax] → Predict next token probabilities
    ↓
Output Text
```

---

## 5. 🏗️ The Architecture — Step by Step

### 5.1 The Encoder

The encoder's job is to **understand** the input.

```
Input: "I love AI"

Step 1: Tokenize    → ["I", "love", "AI"]
Step 2: Embed       → [[0.1, 0.3, ...], [0.5, 0.2, ...], [0.9, 0.1, ...]]
Step 3: Add Position → [[0.1+pos0], [0.5+pos1], [0.9+pos2]]
Step 4: Self-Attention → Each word looks at all other words
Step 5: Feed-Forward   → Process each word independently
Step 6: Repeat N times (typically 6 or 12 layers)
```

### 5.2 The Decoder

The decoder's job is to **generate** the output, one token at a time.

```
Output so far: "J'aime"

Step 1: Embed + Position the output tokens so far
Step 2: Masked Self-Attention → Only look at previous output words (can't peek ahead!)
Step 3: Cross-Attention → Look at the encoder's output ("I love AI")
Step 4: Feed-Forward
Step 5: Predict next word → "l'IA"
```

### 5.3 Multi-Head Attention

Instead of computing attention once, the model computes it **multiple times in parallel** (called "heads"), each focusing on different relationships.

```
Head 1: Might focus on syntactic relationships (subject-verb)
Head 2: Might focus on semantic relationships (meaning)
Head 3: Might focus on positional relationships (nearby words)
...
Head 8: Might focus on coreference (what "it" refers to)

→ All heads are concatenated and combined
```

---

## 6. 🔍 Self-Attention Mechanism Explained Simply

This is the **heart** of the Transformer. Let's break it down.

### The Three Vectors: Query (Q), Key (K), Value (V)

For each word, the model creates three vectors:

| Vector | Purpose | Analogy |
|--------|---------|---------|
| **Query (Q)** | "What am I looking for?" | You asking a question |
| **Key (K)** | "What do I contain?" | Labels on filing cabinets |
| **Value (V)** | "What information do I actually have?" | Contents of the filing cabinets |

### The Formula

```
Attention(Q, K, V) = softmax(Q × Kᵀ / √d_k) × V
```

### Step-by-Step Example

```
Sentence: "The cat sat"

Step 1: Create Q, K, V for each word
         Q_the, K_the, V_the
         Q_cat, K_cat, V_cat
         Q_sat, K_sat, V_sat

Step 2: For word "sat", compute attention scores
         score("sat","The") = Q_sat · K_the = 2.0
         score("sat","cat") = Q_sat · K_cat = 8.0  ← highest!
         score("sat","sat") = Q_sat · K_sat = 3.0

Step 3: Scale by √d_k (prevents huge numbers)
         2.0/√64 = 0.25
         8.0/√64 = 1.00
         3.0/√64 = 0.375

Step 4: Apply softmax (convert to probabilities)
         [0.25, 1.00, 0.375] → [0.15, 0.55, 0.30]

Step 5: Multiply by Values and sum
         output_sat = 0.15×V_the + 0.55×V_cat + 0.30×V_sat
         → "sat" now contains mostly information about "cat"!
```

---

## 7. 📊 Types of Transformer Models

```
                    Transformer
                    /    |    \
                   /     |     \
            Encoder   Encoder-  Decoder
             Only    Decoder     Only
              |        |          |
            BERT    Original    GPT Series
            RoBERTa Transformer GPT-2, GPT-3
            ALBERT    T5        GPT-4
            DistilBERT BART     LLaMA
                    mBART       Mistral
```

### Comparison Table

| Type | Examples | Best For | How It Works |
|------|----------|----------|--------------|
| **Encoder-Only** | BERT, RoBERTa | Understanding text (classification, NER) | Looks at text bidirectionally |
| **Decoder-Only** | GPT-3, GPT-4, LLaMA | Generating text | Predicts next token left-to-right |
| **Encoder-Decoder** | T5, BART | Translation, summarization | Encodes input → Decodes output |

---

## 8. 🌍 Real-World Applications

| Application | Model Used | Example |
|-------------|-----------|---------|
| **Chatbots** | GPT-4, Claude | ChatGPT, customer support bots |
| **Translation** | mBART, T5 | Google Translate |
| **Code Generation** | Codex, StarCoder | GitHub Copilot |
| **Text Summarization** | BART, Pegasus | News summaries |
| **Sentiment Analysis** | BERT | Product review analysis |
| **Image Generation** | Vision Transformer (ViT) | DALL-E, Stable Diffusion |
| **Speech Recognition** | Whisper | Audio transcription |
| **Search Engines** | BERT | Google Search ranking |
| **Medical AI** | BioBERT | Disease diagnosis from text |
| **Autonomous Driving** | ViT variants | Scene understanding |

---

## 9. 💻 Sample Code: Build a Transformer from Scratch (PyTorch)

### Prerequisites

```bash
pip install torch numpy
```

### Full Implementation

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math


# ============================================================
# 1. SCALED DOT-PRODUCT ATTENTION
# ============================================================
class ScaledDotProductAttention(nn.Module):
    """
    Core attention mechanism.
    Think of it as: "For each word, figure out how much to focus on every other word."
    """
    def __init__(self):
        super().__init__()

    def forward(self, query, key, value, mask=None):
        # query, key, value shapes: (batch, heads, seq_len, d_k)
        d_k = query.size(-1)

        # Step 1: Compute attention scores
        scores = torch.matmul(query, key.transpose(-2, -1)) / math.sqrt(d_k)

        # Step 2: Apply mask (optional — used in decoder to prevent looking ahead)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))

        # Step 3: Softmax to get attention weights (probabilities)
        attention_weights = F.softmax(scores, dim=-1)

        # Step 4: Multiply weights by values
        output = torch.matmul(attention_weights, value)

        return output, attention_weights


# ============================================================
# 2. MULTI-HEAD ATTENTION
# ============================================================
class MultiHeadAttention(nn.Module):
    """
    Runs attention multiple times in parallel (multiple "heads"),
    each head can learn to focus on different types of relationships.
    """
    def __init__(self, d_model, num_heads):
        super().__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"

        self.d_model = d_model          # e.g., 512
        self.num_heads = num_heads      # e.g., 8
        self.d_k = d_model // num_heads # e.g., 64

        # Linear projections for Q, K, V and output
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)

        self.attention = ScaledDotProductAttention()

    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)

        # 1. Linear projections and reshape into multiple heads
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)

        # 2. Apply attention
        attn_output, attn_weights = self.attention(Q, K, V, mask)

        # 3. Concatenate heads and apply final linear projection
        attn_output = attn_output.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        output = self.W_o(attn_output)

        return output


# ============================================================
# 3. POSITION-WISE FEED-FORWARD NETWORK
# ============================================================
class FeedForward(nn.Module):
    """
    A simple two-layer neural network applied to each position independently.
    Think of it as: "Now that we know what to pay attention to, process that info."
    """
    def __init__(self, d_model, d_ff, dropout=0.1):
        super().__init__()
        self.linear1 = nn.Linear(d_model, d_ff)     # Expand: 512 → 2048
        self.linear2 = nn.Linear(d_ff, d_model)      # Compress: 2048 → 512
        self.dropout = nn.Dropout(dropout)

    def forward(self, x):
        return self.linear2(self.dropout(F.relu(self.linear1(x))))


# ============================================================
# 4. POSITIONAL ENCODING
# ============================================================
class PositionalEncoding(nn.Module):
    """
    Adds information about the position of each word in the sentence.
    Uses sin and cos functions of different frequencies.
    """
    def __init__(self, d_model, max_seq_len=5000, dropout=0.1):
        super().__init__()
        self.dropout = nn.Dropout(dropout)

        # Create a matrix of shape (max_seq_len, d_model)
        pe = torch.zeros(max_seq_len, d_model)
        position = torch.arange(0, max_seq_len, dtype=torch.float).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model))

        pe[:, 0::2] = torch.sin(position * div_term)  # Even indices
        pe[:, 1::2] = torch.cos(position * div_term)  # Odd indices
        pe = pe.unsqueeze(0)  # Add batch dimension

        self.register_buffer('pe', pe)

    def forward(self, x):
        x = x + self.pe[:, :x.size(1)]
        return self.dropout(x)


# ============================================================
# 5. ENCODER LAYER
# ============================================================
class EncoderLayer(nn.Module):
    """
    One layer of the Encoder:
    Self-Attention → Add & Norm → Feed Forward → Add & Norm
    """
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attention = MultiHeadAttention(d_model, num_heads)
        self.feed_forward = FeedForward(d_model, d_ff, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)

    def forward(self, x, mask=None):
        # Self-attention with residual connection and normalization
        attn_output = self.self_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout1(attn_output))

        # Feed-forward with residual connection and normalization
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout2(ff_output))

        return x


# ============================================================
# 6. DECODER LAYER
# ============================================================
class DecoderLayer(nn.Module):
    """
    One layer of the Decoder:
    Masked Self-Attention → Add & Norm → Cross-Attention → Add & Norm → FFN → Add & Norm
    """
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attention = MultiHeadAttention(d_model, num_heads)
        self.cross_attention = MultiHeadAttention(d_model, num_heads)
        self.feed_forward = FeedForward(d_model, d_ff, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)
        self.dropout3 = nn.Dropout(dropout)

    def forward(self, x, encoder_output, src_mask=None, tgt_mask=None):
        # Masked self-attention (can't look at future tokens)
        attn_output = self.self_attention(x, x, x, tgt_mask)
        x = self.norm1(x + self.dropout1(attn_output))

        # Cross-attention (look at encoder output)
        attn_output = self.cross_attention(x, encoder_output, encoder_output, src_mask)
        x = self.norm2(x + self.dropout2(attn_output))

        # Feed-forward
        ff_output = self.feed_forward(x)
        x = self.norm3(x + self.dropout3(ff_output))

        return x


# ============================================================
# 7. FULL TRANSFORMER MODEL
# ============================================================
class Transformer(nn.Module):
    """
    The complete Transformer model putting it all together!
    """
    def __init__(
        self,
        src_vocab_size,     # Size of source vocabulary
        tgt_vocab_size,     # Size of target vocabulary
        d_model=512,        # Dimension of model
        num_heads=8,        # Number of attention heads
        num_encoder_layers=6,
        num_decoder_layers=6,
        d_ff=2048,          # Dimension of feed-forward layer
        max_seq_len=5000,
        dropout=0.1
    ):
        super().__init__()

        # Embedding layers
        self.src_embedding = nn.Embedding(src_vocab_size, d_model)
        self.tgt_embedding = nn.Embedding(tgt_vocab_size, d_model)
        self.positional_encoding = PositionalEncoding(d_model, max_seq_len, dropout)

        # Encoder & Decoder stacks
        self.encoder_layers = nn.ModuleList(
            [EncoderLayer(d_model, num_heads, d_ff, dropout) for _ in range(num_encoder_layers)]
        )
        self.decoder_layers = nn.ModuleList(
            [DecoderLayer(d_model, num_heads, d_ff, dropout) for _ in range(num_decoder_layers)]
        )

        # Final output projection
        self.fc_out = nn.Linear(d_model, tgt_vocab_size)
        self.dropout = nn.Dropout(dropout)
        self.d_model = d_model

    def generate_mask(self, src, tgt):
        """Create masks to hide padding and future tokens."""
        src_mask = (src != 0).unsqueeze(1).unsqueeze(2)  # (batch, 1, 1, src_len)

        tgt_mask = (tgt != 0).unsqueeze(1).unsqueeze(2)  # (batch, 1, 1, tgt_len)
        seq_len = tgt.size(1)
        nopeak_mask = torch.tril(torch.ones(seq_len, seq_len, device=tgt.device)).bool()
        tgt_mask = tgt_mask & nopeak_mask  # Combine padding mask and no-peek mask

        return src_mask, tgt_mask

    def forward(self, src, tgt):
        src_mask, tgt_mask = self.generate_mask(src, tgt)

        # Encode
        src_embedded = self.dropout(
            self.positional_encoding(self.src_embedding(src) * math.sqrt(self.d_model))
        )
        enc_output = src_embedded
        for layer in self.encoder_layers:
            enc_output = layer(enc_output, src_mask)

        # Decode
        tgt_embedded = self.dropout(
            self.positional_encoding(self.tgt_embedding(tgt) * math.sqrt(self.d_model))
        )
        dec_output = tgt_embedded
        for layer in self.decoder_layers:
            dec_output = layer(dec_output, enc_output, src_mask, tgt_mask)

        # Project to vocabulary
        output = self.fc_out(dec_output)

        return output


# ============================================================
# 8. EXAMPLE USAGE — Training a tiny Transformer
# ============================================================
if __name__ == "__main__":
    # Hyperparameters
    src_vocab_size = 5000
    tgt_vocab_size = 5000
    d_model = 512
    num_heads = 8
    num_layers = 6
    d_ff = 2048
    max_seq_len = 100
    dropout = 0.1
    batch_size = 2
    src_seq_len = 10
    tgt_seq_len = 10

    # Create model
    model = Transformer(
        src_vocab_size, tgt_vocab_size,
        d_model, num_heads, num_layers, num_layers,
        d_ff, max_seq_len, dropout
    )

    # Print model summary
    total_params = sum(p.numel() for p in model.parameters())
    print(f"Total parameters: {total_params:,}")

    # Create dummy data (random token indices)
    src = torch.randint(1, src_vocab_size, (batch_size, src_seq_len))  # Source sentences
    tgt = torch.randint(1, tgt_vocab_size, (batch_size, tgt_seq_len))  # Target sentences

    # Forward pass
    output = model(src, tgt)
    print(f"Input shape:  {src.shape}")   # (2, 10)
    print(f"Output shape: {output.shape}")  # (2, 10, 5000)

    # ---- Simple Training Loop ----
    criterion = nn.CrossEntropyLoss(ignore_index=0)
    optimizer = torch.optim.Adam(model.parameters(), lr=0.0001, betas=(0.9, 0.98), eps=1e-9)

    model.train()
    for epoch in range(5):
        optimizer.zero_grad()
        output = model(src, tgt[:, :-1])  # Teacher forcing: input all but last token
        target = tgt[:, 1:]               # Target is shifted by 1

        # Reshape for loss calculation
        output = output.contiguous().view(-1, tgt_vocab_size)
        target = target.contiguous().view(-1)

        loss = criterion(output, target)
        loss.backward()
        optimizer.step()

        print(f"Epoch {epoch+1}/5, Loss: {loss.item():.4f}")

    print("\n✅ Training complete!")
```

---

## 10. 🚀 Sample Code: Using Hugging Face Transformers Library

The easiest way to use Transformers in practice:

### Installation

```bash
pip install transformers torch
```

### 10.1 Text Generation (GPT-2)

```python
from transformers import pipeline

# Create a text generation pipeline
generator = pipeline("text-generation", model="gpt2")

# Generate text
result = generator(
    "Artificial Intelligence will change the world by",
    max_length=100,
    num_return_sequences=1,
    temperature=0.7
)

print(result[0]["generated_text"])
```

### 10.2 Sentiment Analysis (BERT-based)

```python
from transformers import pipeline

# Create a sentiment analysis pipeline
classifier = pipeline("sentiment-analysis")

# Analyze sentiment
texts = [
    "I love this product! It's amazing!",
    "This is the worst experience ever.",
    "The movie was okay, nothing special."
]

results = classifier(texts)
for text, result in zip(texts, results):
    print(f"Text: {text}")
    print(f"  → Sentiment: {result['label']}, Confidence: {result['score']:.4f}\n")
```

### 10.3 Text Summarization

```python
from transformers import pipeline

summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

article = """
Transformers have revolutionized the field of natural language processing.
Introduced in 2017, the architecture relies entirely on attention mechanisms
and has since become the backbone of models like BERT, GPT, and T5.
These models have achieved state-of-the-art results on numerous benchmarks
and have enabled applications from machine translation to code generation.
The key innovation was replacing recurrent layers with self-attention,
allowing for much greater parallelization during training.
"""

summary = summarizer(article, max_length=50, min_length=25, do_sample=False)
print("Summary:", summary[0]["summary_text"])
```

### 10.4 Question Answering

```python
from transformers import pipeline

qa_pipeline = pipeline("question-answering")

context = """
The Transformer architecture was introduced in 2017 by researchers at Google
in the paper 'Attention Is All You Need'. It uses self-attention mechanisms
to process input sequences in parallel, making it much faster than RNNs.
BERT, GPT, and T5 are all based on the Transformer architecture.
"""

questions = [
    "When was the Transformer introduced?",
    "Who created the Transformer?",
    "What mechanism does the Transformer use?"
]

for question in questions:
    result = qa_pipeline(question=question, context=context)
    print(f"Q: {question}")
    print(f"A: {result['answer']} (confidence: {result['score']:.4f})\n")
```

### 10.5 Translation

```python
from transformers import pipeline

translator = pipeline("translation_en_to_fr", model="Helsinki-NLP/opus-mt-en-fr")

texts = [
    "Hello, how are you?",
    "Transformers are powerful deep learning models.",
    "I love programming in Python."
]

for text in texts:
    result = translator(text)
    print(f"EN: {text}")
    print(f"FR: {result[0]['translation_text']}\n")
```

---

## 11. 🔧 Sample Code: Fine-Tuning a Pre-trained Model

### Fine-tune BERT for Text Classification

```python
from transformers import (
    BertTokenizer,
    BertForSequenceClassification,
    Trainer,
    TrainingArguments
)
from datasets import load_dataset
import numpy as np
from sklearn.metrics import accuracy_score

# 1. Load dataset
dataset = load_dataset("imdb")

# 2. Load tokenizer and model
tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")
model = BertForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)

# 3. Tokenize the dataset
def tokenize_function(examples):
    return tokenizer(
        examples["text"],
        padding="max_length",
        truncation=True,
        max_length=256
    )

tokenized_datasets = dataset.map(tokenize_function, batched=True)

# 4. Use a small subset for demo purposes
small_train = tokenized_datasets["train"].shuffle(seed=42).select(range(1000))
small_test = tokenized_datasets["test"].shuffle(seed=42).select(range(500))

# 5. Define metrics
def compute_metrics(eval_pred):
    logits, labels = eval_pred
    predictions = np.argmax(logits, axis=-1)
    return {"accuracy": accuracy_score(labels, predictions)}

# 6. Set up training arguments
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=64,
    warmup_steps=100,
    weight_decay=0.01,
    logging_dir="./logs",
    logging_steps=50,
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
)

# 7. Create Trainer and train!
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=small_train,
    eval_dataset=small_test,
    compute_metrics=compute_metrics,
)

trainer.train()

# 8. Evaluate
results = trainer.evaluate()
print(f"\n✅ Accuracy: {results['eval_accuracy']:.4f}")

# 9. Save the model
model.save_pretrained("./my-fine-tuned-bert")
tokenizer.save_pretrained("./my-fine-tuned-bert")
print("Model saved!")
```

---

## 12. 📝 Training Tips & Best Practices

### Learning Rate Schedule

```
The original paper uses a "warmup" schedule:

Learning Rate
     ^
     |      /\
     |     /  \
     |    /    \
     |   /      \___________
     |  /
     | /
     +-------------------------→ Training Steps
       ↑ warmup     ↑ decay
```

### Tips Checklist

- [x] **Use warmup** — Start with a small learning rate and gradually increase
- [x] **Use Adam optimizer** — With β₁=0.9, β₂=0.98, ε=10⁻⁹
- [x] **Apply dropout** — Typically 0.1 for regularization
- [x] **Use label smoothing** — Typically 0.1 to prevent overconfidence
- [x] **Gradient clipping** — Clip to max norm of 1.0 to prevent exploding gradients
- [x] **Mixed precision training** — Use FP16 to speed up training and reduce memory
- [x] **Batch size matters** — Larger batches generally work better; use gradient accumulation if GPU memory is limited
- [x] **Layer normalization** — Pre-norm (before attention) often trains more stably than post-norm

---

## 13. 📖 Common Terms Glossary

| Term | Meaning |
|------|---------|
| **Attention** | Mechanism that lets the model focus on relevant parts of the input |
| **Self-Attention** | Attention where input attends to itself |
| **Cross-Attention** | Decoder attends to encoder output |
| **d_model** | Dimension of the model's hidden states (e.g., 512) |
| **d_k** | Dimension of each attention head = d_model / num_heads |
| **d_ff** | Dimension of the feed-forward inner layer (e.g., 2048) |
| **Num Heads** | Number of parallel attention heads (e.g., 8) |
| **Positional Encoding** | Adds position information since Transformers have no inherent order |
| **Residual Connection** | Adds input to output of a sublayer (helps gradient flow) |
| **Layer Normalization** | Normalizes across features to stabilize training |
| **Teacher Forcing** | Training technique: feed ground truth as decoder input |
| **Masked Attention** | Prevents decoder from seeing future tokens |
| **Tokenizer** | Converts text into numerical tokens |
| **Embedding** | Converts tokens into dense vectors |
| **Softmax** | Converts scores into probabilities (sum to 1) |
| **Fine-tuning** | Training a pre-trained model on your specific task |
| **Pre-training** | Training on a large corpus to learn general language understanding |
| **Inference** | Using a trained model to make predictions |

---

## 14. ❓ Frequently Asked Questions (FAQ)

### Q: How is a Transformer different from an RNN?
**A:** RNNs process tokens sequentially (one by one), while Transformers process all tokens in parallel using attention. This makes Transformers much faster and better at capturing long-range dependencies.

### Q: What is the difference between BERT and GPT?
**A:**
- **BERT** = Encoder-only, reads text bidirectionally, best for understanding tasks
- **GPT** = Decoder-only, reads text left-to-right, best for generation tasks

### Q: How much data do I need to train a Transformer?
**A:** From scratch: millions of examples. Fine-tuning a pre-trained model: as few as a few hundred examples can work.

### Q: What hardware do I need?
**A:**
- Fine-tuning small models (BERT-base): 1× GPU (8GB+ VRAM)
- Fine-tuning large models: 1-4× GPUs (16GB+ VRAM each)
- Training from scratch: Clusters of GPUs/TPUs

### Q: What is the context window?
**A:** The maximum number of tokens a Transformer can process at once. Examples:
- BERT: 512 tokens
- GPT-3: 4,096 tokens
- GPT-4: 128,000 tokens
- Claude: 200,000 tokens

### Q: Why does attention use √d_k scaling?
**A:** Without scaling, dot products grow large with higher dimensions, pushing softmax into regions with tiny gradients. Dividing by √d_k keeps values in a reasonable range.

---

## 15. 📚 Resources & Further Reading

### Papers
- 📄 [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — The original Transformer paper
- 📄 [BERT](https://arxiv.org/abs/1810.04805) — Bidirectional Encoder Representations
- 📄 [GPT-2](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf) — Language Models are Unsupervised Multitask Learners
- 📄 [GPT-3](https://arxiv.org/abs/2005.14165) — Language Models are Few-Shot Learners

### Tutorials & Visualizations
- 🌐 [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Best visual guide
- 🌐 [The Annotated Transformer](https://nlp.seas.harvard.edu/annotated-transformer/) — Harvard NLP's line-by-line implementation
- 🌐 [Hugging Face Course](https://huggingface.co/learn/nlp-course) — Free NLP course with Transformers

### Libraries
- 🔧 [Hugging Face Transformers](https://github.com/huggingface/transformers) — Most popular Transformers library
- 🔧 [PyTorch](https://pytorch.org/) — Deep learning framework
- 🔧 [TensorFlow](https://www.tensorflow.org/) — Alternative deep learning framework

---

## 🎉 Summary

```
┌──────────────────────────────────────────────────┐
│              TRANSFORMER IN A NUTSHELL           │
├──────────────────────────────────────────────────┤
│                                                  │
│  1. Tokenize input text                          │
│  2. Convert tokens to embeddings                 │
│  3. Add positional encoding                      │
│  4. Pass through Encoder (self-attention + FFN)  │
│  5. Pass through Decoder (masked attention +     │
│     cross-attention + FFN)                       │
│  6. Project to vocabulary & predict next token   │
│                                                  │
│  Key Innovation: SELF-ATTENTION                  │
│  → Every word can attend to every other word     │
│  → Computed in parallel (fast!)                  │
│  → No sequential bottleneck                      │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

> 💡 **Pro Tip:** You don't need to build Transformers from scratch for most tasks. Use **Hugging Face's `transformers` library** to leverage thousands of pre-trained models with just a few lines of code!

---

*⭐ If you found this helpful, consider starring the repository!*

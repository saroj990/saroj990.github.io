+++
title = 'Inside the Next Token: Running LLMs on Your Own Machine'
date = '2026-07-22T10:00:00+05:30'
draft = false
description = 'A practical guide to understanding how Large Language Models work and how to run them locally on your own hardware using Ollama and LM Studio.'
tags = ['LLM', 'AI', 'Ollama', 'LM Studio', 'Local AI', 'Tutorial']
categories = ['AI', 'Engineering']
summary = 'Understand tokenization, transformers, and next-token prediction — then run open-weight models on your own machine with Ollama or LM Studio.'
+++

<img src="/images/posts/understanding-llms/cover.svg" alt="Local LLM setup: app to runtime to GGUF model" width="960" height="400" />

*Local LLM setup: your app talks to a runtime on localhost, which loads a quantized model.*

For years, using a powerful language model meant sending your data to a cloud API and paying per token. That has changed. Open-weight models like Llama, Qwen, and Gemma now run on consumer hardware with quality that rivals many proprietary APIs.

Whether you care about privacy, cost, offline access, or experimenting without rate limits, running LLMs locally is a practical engineering decision — not a hobbyist experiment.

This post explains how LLMs actually work, then walks you through two proven ways to run them on your own machine.

**TL;DR:** LLMs predict text one token at a time. Tools like [Ollama](https://ollama.com) and [LM Studio](https://lmstudio.ai) load quantized models locally and expose an OpenAI-compatible API on `localhost` — often a one-line config change in your existing code.

---

## How LLMs Work: The 5-Minute Explanation

### From Text to Numbers: Tokenization

An LLM doesn't read words — it reads **tokens**. Tokenization breaks text into chunks: a token might be a full word (`"hello"`), a partial word (`"unbeliev"`), or even a single character. GPT-4-class models work with vocabularies of roughly 50,000 to 100,000 unique tokens.

When you type a prompt, it is converted into a sequence of integers (token IDs) that the model can process.

<img src="/images/posts/understanding-llms/tokenization.svg" alt="Flow from text prompt to tokenizer to token IDs" width="900" height="280" />
*Your prompt is tokenized into integer IDs before the model sees it.*

### The Transformer Architecture

Modern LLMs are built on the **transformer** architecture, introduced in 2017. The core idea is **self-attention**: for every token in a sequence, the model computes how much every other token matters for understanding it.

Think of it as a sophisticated pattern-matching engine. When the model sees the word *"bank"* in *"river bank"*, the attention mechanism learns to weight *"river"* heavily and ignore *"money."* This happens across many layers simultaneously, allowing the model to capture long-range dependencies in text.

Key components include:

- **Multi-head attention** — multiple attention computations in parallel (syntax, semantics, long-distance references)
- **Feed-forward layers** — learned transformations applied after attention
- **Residual connections and layer normalization** — stabilize training across hundreds of layers

<img src="/images/posts/understanding-llms/attention.svg" alt="Self-attention on the phrase river bank" width="900" height="300" />
*Self-attention helps the model disambiguate words using surrounding context.*

### Next-Token Prediction

At its core, an LLM is a **next-token predictor**. Given a sequence of tokens, it outputs a probability distribution over what token should come next. To generate a response, the model:

1. Takes your prompt as input
2. Predicts the most likely next token
3. Appends that token to the sequence
4. Repeats until a stop condition is met (e.g., an end-of-sequence token)

This is why LLMs are called **autoregressive** — they generate text one token at a time, feeding each prediction back into the input.

<img src="/images/posts/understanding-llms/next-token-loop.svg" alt="Autoregressive next-token prediction loop" width="900" height="320" />
*Generation is a loop: predict one token, append it, predict again.*

### Generation Strategies: Temperature and Top-p

Raw next-token prediction is deterministic. To make outputs creative and varied, models use sampling strategies:

- **Temperature** — scales the probability distribution before sampling. A temperature of `0` is deterministic (always picks the highest-probability token). Higher values (e.g., `0.8`) increase creativity.
- **Top-p (nucleus sampling)** — samples only from the smallest set of tokens whose cumulative probability exceeds a threshold (e.g., `0.9`). This reduces nonsense while preserving diversity.

### Quantization: Running Giants on Small Hardware

A full-precision 7-billion-parameter model requires ~28 GB of memory (4 bytes per parameter). **Quantization** compresses weights to lower precision — typically 4-bit or 8-bit — reducing memory usage by 50–75% with minimal quality loss. Formats like **GGUF** and **Q4_K_M** are optimized for inference on consumer CPUs and GPUs.

<img src="/images/posts/understanding-llms/quantization.svg" alt="FP16 vs Q4_K_M memory usage" width="900" height="280" />
*Quantization shrinks a 7B model from ~28 GB to ~5 GB — why local inference works on a laptop.*

---

## Why Run LLMs Locally?

<img src="/images/posts/understanding-llms/cloud-vs-local.svg" alt="Cloud API vs local runtime" width="960" height="400" />
*Local runtimes keep prompts on your machine instead of sending them to a cloud API.*

Running models on your own hardware has become a mainstream choice:

| Benefit | Why it matters |
|---------|----------------|
| **Privacy** | Sensitive code and documents never leave your machine — important for HIPAA/GDPR workloads |
| **Cost** | Replace per-token pricing with a one-time hardware investment |
| **Speed** | No network round-trips once the model is loaded |
| **Offline** | Works on a plane, in air-gapped environments, or during outages |
| **Control** | You choose the model, system prompt, and avoid provider rate limits |

---

## Hardware Requirements: What You Actually Need

You don't need a data center. Here's what works in practice:

| Model Size | RAM / VRAM Needed | Use Case |
|------------|-------------------|----------|
| 3B–4B (e.g., Gemma 3 4B) | 4–6 GB | Summarization, light Q&A |
| 7B–8B (e.g., Qwen 3 8B, Llama 3.1 8B) | 8–16 GB | Coding assistance, general chat, RAG |
| 13B–14B | 16–24 GB | Higher-quality reasoning |
| 70B+ | 48–64 GB+ | Near-frontier quality; slow on consumer hardware |

For the best experience, use an NVIDIA GPU with CUDA or an Apple Silicon Mac with unified memory. CPU-only inference works but is significantly slower.

---

## Option 1: Ollama (Developer-First, CLI/API)

[Ollama](https://ollama.com) is the fastest way to get a local LLM running if you're comfortable with the terminal. It bundles model management, inference, and an OpenAI-compatible API server.

### Installation

```bash
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows: download from https://ollama.com/download
```

Verify the installation:

```bash
ollama --version
```

### Pull and Run Your First Model

```bash
# Download a model (Qwen 3 8B is a strong all-rounder)
ollama pull qwen3:8b

# Start an interactive chat
ollama run qwen3:8b
```

You can also pipe prompts directly:

```bash
echo "Explain the difference between a mutex and a semaphore" | ollama run qwen3:8b
```

<img src="/images/posts/understanding-llms/ollama-terminal.svg" alt="Ollama terminal running qwen3:8b" width="960" height="540" />
*Ollama pulls a model, runs chat, and serves an OpenAI-compatible API on port 11434.*

### Using the OpenAI-Compatible API

Ollama serves an API on `localhost:11434`. Any code written for OpenAI's API works locally by changing the base URL:

```bash
curl -s -X POST http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3:8b",
    "messages": [
      {"role": "system", "content": "You are a helpful coding assistant."},
      {"role": "user", "content": "Write a Python function to flatten a nested list."}
    ],
    "temperature": 0.7
  }'
```

### Custom Modelfiles

Create project-specific configurations with a `Modelfile`:

```dockerfile
FROM qwen3:8b

SYSTEM """You are a senior backend engineer reviewing Python code.
Focus on correctness, performance, and security. Be concise."""

PARAMETER temperature 0.3
PARAMETER num_ctx 8192
```

Build and use it:

```bash
ollama create code-reviewer -f ./Modelfile
ollama run code-reviewer
```

---

## Option 2: LM Studio (GUI-First, Visual)

[LM Studio](https://lmstudio.ai) is the best choice if you prefer a graphical interface, want to browse models visually, or need to compare models side-by-side.

### Installation

Download the installer for your OS from [lmstudio.ai](https://lmstudio.ai). It supports macOS, Windows, and Linux.

### Workflow

1. **Browse models** — search for models on Hugging Face (e.g., `Qwen 3 8B Q4_K_M`)
2. **Download** — click to download your chosen quantization
3. **Chat** — load the model and start chatting in the playground
4. **Serve** — enable the local server in the **Developer** tab to expose an API on `localhost:1234`

<img src="/images/posts/understanding-llms/lm-studio-chat.svg" alt="LM Studio chat interface" width="960" height="540" />
*Browse models, load a quantization, and chat in the LM Studio playground.*

<img src="/images/posts/understanding-llms/lm-studio-server.svg" alt="LM Studio Developer server tab" width="960" height="420" />
*Enable the local server in the Developer tab to expose an OpenAI-compatible API.*

### API Integration

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:1234/v1", api_key="lm-studio")

response = client.chat.completions.create(
    model="qwen3-8b",  # use the filename stem from LM Studio
    messages=[
        {"role": "user", "content": "Summarize the key changes in HTTP/3 vs HTTP/2."}
    ],
)
print(response.choices[0].message.content)
```

---

## When to Choose What

| You want… | Use |
|-----------|-----|
| Scripting, automation, CI/CD, headless servers | **Ollama** |
| Visual model comparison, easy experimentation, GUI chat | **LM Studio** |
| Both | Many developers evaluate in LM Studio, then deploy with Ollama |

---

## Integrating Local LLMs into Your Projects

Because both tools expose OpenAI-compatible APIs, switching from cloud to local is often a one-line change:

```python
from openai import OpenAI

# Cloud
# client = OpenAI(api_key="your-key")

# Local (Ollama)
client = OpenAI(base_url="http://localhost:11434/v1", api_key="not-needed")

# Local (LM Studio)
# client = OpenAI(base_url="http://localhost:1234/v1", api_key="lm-studio")

response = client.chat.completions.create(
    model="qwen3:8b",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

This compatibility means IDE extensions, agent frameworks, and RAG pipelines often work with local models with minimal code changes.

---

## Quick Reference: Starter Models

| Model | Size | Best For |
|-------|------|----------|
| Gemma 3 4B | 4B | Lightweight tasks, summarization |
| Qwen 3 8B | 8B | Coding, multilingual, general-purpose |
| Llama 3.1 8B | 8B | General chat, reasoning |
| Mistral 7B | 7B | Fast inference, instruction following |
| DeepSeek Coder | Various | Code generation, technical docs |

Start with a **7B–8B model in Q4_K_M quantization** for the best balance of quality and resource usage.

---

## Common Pitfalls

- **Wrong quantization** — FP16 on 16 GB RAM causes swapping. Match quantization to your hardware.
- **Context window too large** — `num_ctx` of 32K on an 8B model can exhaust RAM. Scale with available memory.
- **Wrong API port** — Ollama uses `:11434`, LM Studio uses `:1234`. Update your client when switching.
- **Assuming zero telemetry** — local inference keeps prompts off the cloud, but check each tool's privacy policy for usage data.

---

## Conclusion

Running LLMs locally is no longer a compromise between privacy and capability — it's often the better engineering choice. Understanding transformers and next-token prediction helps you reason about model behavior; Ollama and LM Studio remove the infrastructure friction.

Pick a model, install a runtime, and point your existing tools at `localhost`. Your data stays yours, your costs stay fixed, and your AI works offline.

**Next steps:** Pull `qwen3:8b` with Ollama, send one API request from Python, then replace the diagram images in `static` with your own screenshots if you want.

+++
title = 'Shrinking the Weights: How LLM Quantization Actually Works'
date = '2026-07-31T20:58:00+05:30'
draft = false
description = 'A clear guide to LLM weight quantization - scales and zero-points, group-wise K-quants, GGUF labels like Q4_K_M, GPTQ/AWQ, and how to pick a quant for your hardware.'
tags = ['AI', 'Quantization', 'LLM', 'GGUF', 'GPTQ', 'Local AI', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'Quantization stores model weights with fewer bits so large LLMs fit in laptop RAM. Here is the math, the formats, and how to choose Q4 vs Q5 vs Q8 without guessing.'
+++

<img src="/images/posts/quantization/cover.svg" alt="FP16 versus Q4 weight memory" width="960" height="400" />

*Same network architecture. Fewer bits per weight. That is why a 7B model fits in a backpack laptop.*

You have already seen quantization mentioned in [how LLMs work](/posts/understanding-llms/) and [running local models](/posts/local-models/). This post goes deeper: **what is being compressed, how the mapping works, what `Q4_K_M` means, and why some methods need calibration data.**

Not covered here: [product quantization in vector databases](/posts/vector-databases/) - that compresses *embeddings for search*, not LLM weight matrices. Same word family, different machine.

---

## The problem in one table

An LLM is mostly giant matrices of floating-point numbers (**weights**). Memory scales with parameter count × bytes per parameter.

| Precision | Bytes / weight | 7B model (weights only, rough) |
|-----------|----------------|--------------------------------|
| FP32 | 4 | ~28 GB |
| FP16 / BF16 | 2 | ~14 GB |
| INT8 | 1 | ~7 GB |
| INT4 (idealized) | 0.5 | ~3.5 GB |

Real 4-bit files are a bit larger than the ideal because you also store **scales**, sometimes **zero-points**, and format metadata. A practical **Q4** 7B-8B GGUF is often about **4-5 GB on disk**.

At runtime you still need:

- weights in RAM/VRAM  
- **KV cache** (grows with context length)  
- activations / scratch buffers  
- OS and apps  

So "4 GB file" does not mean "runs happily in 4 GB RAM."

---

## What quantization means

**Quantization** replaces high-precision numbers with lower-precision codes that approximate them.

For each weight value `x` in some group of weights:

```text
q     = round(x / scale + zero)     # store this small integer
x_hat = (q - zero) * scale          # reconstruct when computing
```

<img src="/images/posts/quantization/scale-zero.svg" alt="Floats mapped into integer bins with scale and zero-point" width="960" height="420" />

- **`scale`** - how wide each bin is  
- **`zero` (zero-point)** - offset so the integer range can represent negative and positive floats (asymmetric quantization)  
- **`q`** - the stored code (for 4-bit, only 16 possible values per code)

The gap `|x - x_hat|` is **quantization error**. Good schemes keep that error small where it hurts the model most.

During inference, engines either:

1. **Dequantize on the fly** to FP16/BF16 for matmuls, or  
2. Use **integer / mixed kernels** that multiply low-bit weights more directly  

Either way, the win is mostly **memory bandwidth and capacity**. Less data to move often means faster tokens/sec on memory-bound hardware - until kernels get inefficient or you under-utilize the GPU.

---

## Granularity: one scale or many?

<img src="/images/posts/quantization/granularity.svg" alt="Per-tensor versus grouped quantization" width="960" height="420" />

| Granularity | Idea | Tradeoff |
|-------------|------|----------|
| **Per-tensor** | One scale for a whole weight tensor | Tiny metadata; outliers waste the whole range |
| **Per-channel / per-row** | Scale per output channel | Better accuracy, common in some GPU stacks |
| **Group-wise** | Scale every *g* weights (e.g. 32, 64, 128) | Best 4-bit quality for a bit more metadata |

**Outliers** matter. A few huge-magnitude weights force a coarse scale if you share one scale across too many values. Grouping isolates outliers so the rest of the group can use a finer scale.

That is why modern GGUF **K-quants** beat naive 4-bit at the same nominal bit width.

---

## Decoding GGUF names (`Q4_K_M` and friends)

[Ollama](https://ollama.com) / [LM Studio](https://lmstudio.ai) / llama.cpp mostly ship **GGUF** files. Names encode the recipe:

| Piece | Meaning |
|-------|---------|
| `Q4` / `Q5` / `Q6` / `Q8` | Nominal bits per weight |
| `K` | K-quant family (block/group methods inside llama.cpp) |
| `S` / `M` / `L` | Small / Medium / Large - how aggressively some layers stay higher precision |

Examples:

- **`Q4_K_M`** - the usual sweet spot for local chat and coding  
- **`Q5_K_M`** - heavier, often a bit smarter, needs more RAM  
- **`Q8_0`** - near original quality, still smaller than FP16  
- **`Q3_K_S`** - fits tiny machines; expect more nonsense and weaker tool use  

<img src="/images/posts/quantization/ladder.svg" alt="Quality versus size ladder from Q2 to FP16" width="960" height="400" />

**Important:** labels are *family names*, not a guarantee of identical quality across model architectures. Always smoke-test the model you care about.

---

## Other formats you will see

| Method | Typical home | Notes |
|--------|--------------|-------|
| **GGUF** (llama.cpp) | Mac/CPU/GPU local apps | Dominant for Ollama / LM Studio |
| **GPTQ** | NVIDIA GPU servers | Calibration-based; good throughput with compatible kernels |
| **AWQ** | NVIDIA / some serving stacks | Protects "important" weights; calibration data |
| **bitsandbytes NF4 / double quant** | QLoRA training & some inference | Popular for fine-tuning in 4-bit |
| **FP8 / INT8 serving** | Datacenter inference | Different stack; often activation + weight quant |

### Calibration vs "just cast to int"

- **Post-training quantization (PTQ)** without much data: compress weights with statistics from the checkpoint (and heuristics). Fast to produce.  
- **Calibrated PTQ** (GPTQ/AWQ-style): run sample texts through the model, minimize output error layer by layer. Better quality at 4-bit; needs a calibration set and more offline compute.  
- **Quantization-aware training (QAT)**: train or fine-tune with fake quant in the loop. Best accuracy, most expensive; less common for casual local downloads.

You usually **download** a pre-quantized GGUF. You only calibrate yourself if you are publishing a custom quant.

---

## Weights vs activations (do not confuse them)

| | Weights | Activations |
|--|---------|-------------|
| What | Learned parameters on disk | Temporary tensors while running a prompt |
| Quantized in GGUF chat? | Yes (main win) | Often kept higher precision |
| Changes every token? | No | Yes |

Some aggressive server setups quantize activations too (INT8/FP8 GEMMs). Local GGUF chat is mostly about **weight** compression. KV cache quantization is a separate optimization some runtimes offer to shrink long-context memory.

---

## What error looks like in practice

As bits drop, failures tend to show up as:

1. **Weaker instruction following** - ignores format, skips steps  
2. **More hallucinations** on facts and APIs  
3. **Broken JSON / tool calls** - painful for [agents](/posts/agent-loop/)  
4. **Math and multi-step reasoning cliffs**  
5. **Language mixups** or repetitive loops on hard prompts  

A smaller dense model at **Q5/Q8** can beat a larger model crushed to **Q2/Q3**. Prefer "right size + sane quant" over "biggest file that still opens."

---

## Rough memory math you can do yourself

Rule of thumb for **Q4** weights:

```text
GB ≈ parameters_in_billions * 0.55  to  0.75
```

Examples (order of magnitude):

| Model | Q4 file ballpark | Comfortable machine |
|-------|------------------|---------------------|
| 3B | ~2 GB | 8 GB RAM (tight) |
| 7B-8B | ~4-5 GB | 16 GB RAM |
| 14B | ~8-9 GB | 16-32 GB |
| 70B | ~35-40 GB | 64 GB+ unified / multi-GPU |

Add headroom for context. A long 32k context can add gigabytes of KV cache on some models.

See the hardware tables in the [local models](/posts/local-models/) post for laptop vs Mac guidance.

---

## How to choose a quant (decision guide)

| Goal | Start with |
|------|------------|
| First local try | `Q4_K_M` on a 7B-8B model |
| Agent / tool calling | Avoid ultra-low quants; prefer Q4_K_M or Q5_K_M |
| Max quality on a big GPU / 64 GB Mac | Q5/Q6/Q8 or FP16 if it fits |
| Only 8 GB machine | 3B-4B at Q4; do not force 14B Q3 and live in swap |
| Publishing weights for others | Document calibration set + exact tool/version |

Quick A/B test:

1. Same prompts on Q4 vs Q5  
2. Score: did it follow format, call tools, solve the task?  
3. Keep the smallest quant that still passes *your* tasks  

---

## Tiny experiment (build intuition)

You do not need to implement a full GPTQ pipeline to feel the idea:

```python
import numpy as np

def fake_quantize(x, bits=4):
    qmax = 2 ** bits - 1
    xmin, xmax = float(x.min()), float(x.max())
    scale = (xmax - xmin) / qmax if xmax > xmin else 1.0
    zero = round(-xmin / scale) if scale else 0
    q = np.clip(np.round(x / scale + zero), 0, qmax)
    x_hat = (q - zero) * scale
    return x_hat, scale, zero

w = np.random.randn(10_000).astype(np.float32)
recon, scale, zero = fake_quantize(w, bits=4)
mse = float(np.mean((w - recon) ** 2))
print(f"scale={scale:.4f} zero={zero} mse={mse:.6f}")
```

Lower `bits` → usually higher `mse`. Real LLM methods add grouping, outliers handling, and calibration; the core mapping is the same.

---

## Common myths

**"Q4 means exactly 4 bits everywhere."**  
Not always. Hybrid/K-quants keep some tensors at higher precision.

**"Quantization is only for hobbyists."**  
Datacenters use INT8/FP8 and low-bit serving for cost and throughput too.

**"If it loads, quality is fine."**  
Loading only proves memory fit. Quality needs eval prompts - especially for agents.

**"FP16 is always better than Q8."**  
Often yes, but not always worth 2× memory. Measure.

---

## Closing

Quantization is controlled precision loss:

> pick a scale (and often a zero-point) for a group of weights → store small integers → reconstruct on the fly.

That is what makes local LLMs practical. Formats like **GGUF Q4_K_M** package good defaults for consumer hardware; **GPTQ/AWQ** target calibrated GPU serving; training stacks like **QLoRA** use related 4-bit ideas for fine-tuning.

If you remember one line:

> **Bits buy memory. Groups and calibration buy quality back.**

**Next step:** on your machine, run the same 10 prompts against one model at `Q4_K_M` and `Q5_K_M`. Keep the smaller file if results tie. Related: [local setup](/posts/local-models/), [LLM basics](/posts/understanding-llms/).

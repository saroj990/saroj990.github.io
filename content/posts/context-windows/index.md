+++
title = 'The Token Budget: Why Finite Context Still Shapes Every Agent'
date = '2026-07-22T16:00:00+05:30'
draft = false
description = 'What an LLM context window really is, why size limits still matter in 2026, how teams work around them, and how products like ChatGPT and Cursor manage context in practice.'
tags = ['AI', 'LLM', 'Context Window', 'RAG', 'Cursor', 'ChatGPT', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'Context windows are finite. Here is what that limit means, why bigger is not always better, and how ChatGPT and Cursor stay useful without stuffing entire worlds into one prompt.'
+++

<img src="/images/posts/context-windows/cover.svg" alt="Context window as a fixed token budget" width="960" height="400" />

*An LLM does not “remember” your chat or your repo. It only sees what fits in the current context window — and that window is always finite.*

If you have used ChatGPT long enough to watch it forget an early decision, or used Cursor on a large monorepo and wondered why it didn’t “just know” every file, you have already bumped into the **context window**.

This post covers:

1. What a context window actually is  
2. How large windows are today (and why the marketing number lies a little)  
3. Why limits still exist  
4. How practitioners mitigate them  
5. How **ChatGPT** and **Cursor** handle context differently  

---

## What is a context window?

A **context window** is the maximum amount of text (measured in **tokens**) that a model can process in a single inference call — including what you send in *and* what the model generates back out.

Rough intuition:

- ~1 token ≈ ¾ of an English word (varies a lot by language and code)
- ~100 tokens ≈ 75 words
- ~1,000 tokens ≈ ~750 words ≈ ~2–3 pages of dense prose
- Code is often denser: a medium source file can burn thousands of tokens alone

Everything that influences the next answer must fit inside that budget:

- system / developer instructions  
- conversation history  
- attached files, search results, tool outputs  
- the user’s latest message  
- space reserved for the model’s reply  

<img src="/images/posts/context-windows/token-budget.svg" alt="Shared token budget across system, history, retrieval, tools, and output" width="960" height="420" />

### Stateless models, stateful products

The model itself is **stateless**. Each API call is independent. Products feel continuous because *they* rebuild a prompt every turn: replaying history, injecting memories, retrieving documents, attaching open files.

So when people say “the AI remembered,” they usually mean:

> the product put that information back into the prompt.

When they say “the AI forgot,” they usually mean:

> that information was truncated, summarized away, never retrieved, or drowned in noise.

---

## How big are context windows in 2026?

Headline numbers have exploded since the 4K–8K era. As of mid‑2026, ballpark advertised windows look roughly like this:

| Class | Typical advertised window | Notes |
| --- | --- | --- |
| Older / small models | 8K–32K | Still common for cheap / local models |
| Mainstream APIs | 128K–272K | Fine for most chat + coding turns |
| Flagship long-context | ~200K–1M | Claude, Gemini, many frontier tiers |
| Extreme / open-weight outliers | up to ~10M (e.g. Llama 4 Scout class) | Huge on paper; quality varies with length |

Treat these as **order-of-magnitude**, not a shopping list. Vendors change tiers, pricing, and “standard vs extended” modes often.

### Advertised vs effective context

The number on the model card is a capacity ceiling, not a quality guarantee.

Independent long-context evaluations (for example NVIDIA’s **RULER**-style needle/retrieval suites) repeatedly show a pattern:

- models can *accept* N tokens  
- they often *reliably use* closer to **~50–65%** of N for hard retrieval / multi-hop reasoning  
- quality drops as prompts get longer, noisier, or more adversarial  

Also watch for:

- **lost-in-the-middle**: models often attend better to the start and end of a long prompt than the middle  
- **output caps**: a 1M input window does not mean a 1M reply; output limits are much smaller  
- **cost and latency**: long prompts are slower and more expensive, often with surcharges past a threshold  

**Rule of thumb:** prefer the *smallest* context that still contains the right information. Bigger windows are a safety net, not a strategy.

### Scale intuition

| Tokens | Rough real-world size |
| --- | --- |
| 8K | a long article + short reply |
| 32K | a small module or a dense design doc |
| 128K | a medium service, or a long meeting transcript |
| 200K–1M | a large package / multi-file feature slice |
| Multi-million | “whole repo in one shot” fantasies — still usually a bad idea |

A mid-size company codebase can be tens or hundreds of millions of tokens. No product is stuffing that into every request.

---

## Why is there a limit at all?

Transformers compute attention over tokens. Naively, that grows **quadratically** with sequence length: longer context means more memory, more compute, higher latency, and harder training.

Vendors stretch windows with architecture and training tricks (efficient attention, RoPE / position scaling such as YaRN-style methods, sparse / sliding patterns, long-context fine-tuning). That raises the ceiling — it does not erase physics, cost, or the reliability cliff at extreme lengths.

Limits also exist for product reasons:

- keep inference affordable  
- bound worst-case latency  
- reduce prompt-injection surface area from dumping entire corpora  

---

## How people mitigate context limits

<img src="/images/posts/context-windows/mitigations.svg" alt="Six mitigation strategies for finite context windows" width="960" height="480" />

### 1. Retrieval-Augmented Generation (RAG)

Index a large corpus offline. At query time, embed the question, fetch the top‑k relevant chunks, and put *only those* in the prompt.

This is the workhorse behind “chat with your docs,” codebase Q&A, and most enterprise assistants.

**Strengths:** scales to huge corpora  
**Failure modes:** bad chunking, weak embeddings, missing the right file, or retrieving too much noise

### 2. Summarization / compaction

When history grows past a threshold, older turns are compressed into a shorter synopsis (sometimes plus a separate “hard facts” list: names, decisions, IDs).

**Strengths:** preserves narrative continuity  
**Failure modes:** summary omits a detail you later need; compacted chats feel “fuzzy”

### 3. Sliding window / truncation

Keep the newest N tokens; drop the oldest. Simple and common as a last resort.

**Strengths:** cheap and predictable  
**Failure modes:** early constraints and decisions vanish

### 4. External memory

Store durable facts outside the prompt — user preferences, project notes, ticket state — and inject only what is relevant for this turn.

ChatGPT’s optional **Memory** feature is a consumer-facing version of this idea.

### 5. Tool use / agents

Instead of preloading everything, let the model *fetch* what it needs: search the web, open a file, run `grep`, query a database, read a PR.

This turns context from “carry the world” into “navigate the world.”

### 6. Map-reduce / hierarchical processing

Split a huge document into chunks, summarize or answer each chunk, then merge. Useful for book-length analysis when a single pass is unreliable or too expensive.

### 7. Prompt hygiene (underrated)

- put critical constraints at the **start and end**  
- prefer citations / file paths over pasting entire trees  
- strip boilerplate, generated lockfiles, and irrelevant logs  
- start a **fresh thread** with a handoff summary when a chat goes stale  

---

## How ChatGPT handles context

ChatGPT is optimized for **multi-turn conversation**, not for indexing your private monorepo by default.

<img src="/images/posts/context-windows/cursor-vs-chatgpt.svg" alt="ChatGPT vs Cursor context strategies" width="960" height="460" />

### What happens on each message

Conceptually, each reply is generated from a freshly assembled prompt that typically includes:

1. **System / policy instructions** (always on)  
2. **Recent conversation turns** (verbatim while they fit)  
3. **Compressed older history** once the thread gets long (exact internals are not fully public; the observable behavior is compaction / forgetting)  
4. **Optional Memory** facts you (or the product) saved across chats  
5. **Turn-specific attachments**: uploaded files, linked docs, browsing / tool results when those features are active  

OpenAI does not publish a line-by-line algorithm for the consumer app. From product behavior and common industry patterns, the practical model is:

> keep the recent dialogue crisp → compress the long tail → store durable preferences separately → retrieve extras only when needed.

### What you will notice as a user

- Mid-length chats feel continuous and sharp.  
- Very long threads get vague, contradict earlier decisions, or need reminders.  
- Starting a new chat with a tight **handoff summary** often beats continuing a bloated thread.  
- Memory helps with stable preferences (“I prefer TypeScript”, “I’m building X”) but is not a full transcript archive.  
- File uploads / retrieval help for documents, but you still pay the window tax for whatever is injected.

### API vs ChatGPT UI

If you call the API yourself, **you** own context management: what history to send, how to summarize, when to RAG. The ChatGPT product layer does that packaging for you — which is convenient, and also why long chats can surprise you when compaction kicks in.

---

## How Cursor handles context

Cursor is optimized for **software engineering inside a workspace**. It assumes the interesting world is mostly on disk (and in git), and that dumping the whole tree into every prompt is impossible.

### Codebase indexing (offline / background)

When you open a project, Cursor typically:

1. Scans files (respecting `.gitignore` / `.cursorignore`)  
2. Splits code into syntactic / semantic **chunks**  
3. Builds **embeddings** for those chunks  
4. Stores vectors + metadata for similarity search  
5. Re-indexes incrementally as files change (hash / Merkle-style sync)

Raw “entire repo in the prompt” is not the point. The index is an external memory the agent can search.

### What enters the live context window

On a given Agent / Chat turn, Cursor assembles a prompt from a mix of:

| Source | Role |
| --- | --- |
| System + user rules (`.cursor/rules`, etc.) | Stable project instructions |
| Conversation / agent transcript | Short-term working memory |
| Open editors, selections, `@Files` | Explicit human-provided focus |
| Semantic codebase search (`@Codebase` / agent search) | Retrieved chunks relevant to the query |
| Tool results (read, grep, terminal, web, PRs, …) | On-demand facts from the environment |
| Docs you indexed / `@Docs` / `@Web` | Extra knowledge bases |

That is classic **RAG + tools**, specialized for IDEs:

- embeddings find *candidate* code  
- tools verify and expand (read the full file, follow imports, run tests)  
- rules keep style and architecture constraints always nearby  

### Why Cursor still “misses” things

Even with a great index:

- retrieval can miss the right file  
- a wrong chunk can mislead harder than no chunk  
- huge generated files / vendored code pollute search if not ignored  
- agent transcripts can bloat like any long chat and need compaction or a fresh thread  
- the model’s window still has to hold rules + history + retrieved code + tool output + the reply  

Cursor’s bet is not “infinite context.” It is **selective context** — pull the right 2% of the repo at the right time.

---

## ChatGPT vs Cursor (quick comparison)

| Dimension | ChatGPT | Cursor |
| --- | --- | --- |
| Primary world | Conversation + optional uploads/memory | Local (and remote) codebase |
| Default mitigation | History replay + compaction + Memory | Index + retrieve + agent tools |
| Best at | Writing, reasoning, general Q&A, light coding | Multi-file refactors, repo Q&A, IDE workflows |
| Failure mode | Long-thread amnesia / summary loss | Retrieval miss / noisy chunks / ignored paths |
| User lever | New chat + handoff; Memory; attach files | `@` mentions, rules, `.cursorignore`, tighter prompts |

They are converging — ChatGPT gains tools and file awareness; Cursor chats like a product assistant — but the center of gravity differs: **dialogue memory** vs **code navigation**.

---

## Practical advice

1. **Don’t romanticize million-token windows.** Prefer precise context over maximal context.  
2. **Put must-not-forget constraints at the edges** of the prompt (or in Cursor rules / ChatGPT Memory).  
3. **When a thread goes soft, restart.** Paste a short handoff: goal, decisions, constraints, next step.  
4. **In Cursor, curate the index.** Ignore build artifacts, lockfiles, and generated giants; `@` the files you know matter.  
5. **Use tools for truth.** Prefer “read this file / run this test” over hoping a stale summary is right.  
6. **Measure what you care about.** For retrieval tasks, evaluate whether the *right* snippets entered the window — not whether the window was large.

---

## Closing

A context window is the model’s temporary desk space. Modern desks are huge compared to 2023 — sometimes millions of tokens — but your projects, chats, and logs are larger still. The winning systems do not pretend the desk is infinite. They **retrieve, compress, remember outside the prompt, and fetch on demand**.

That is why ChatGPT can feel like a continuous conversation partner, and why Cursor can feel like a teammate who knows your repo: not because the model holds everything, but because the product keeps putting the *right* things on the desk for the next move.

If you want the mental model in one line:

> **Context is a budget. Architecture is how you spend it.**

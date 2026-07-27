+++
title = 'Numbers That Mean Things: How Vector Databases Work Inside'
date = '2026-07-28T00:30:00+05:30'
draft = false
description = 'What a vector database is, how embeddings and similarity search work, and the internals of indexes like HNSW, IVF, and product quantization - with clear examples.'
tags = ['AI', 'Vector Database', 'Embeddings', 'RAG', 'HNSW', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'A vector database stores embeddings and finds nearest neighbors fast. Here is how similarity metrics, HNSW, IVF, and compression work under the hood.'
+++

<img src="/images/posts/vector-databases/cover.svg" alt="Documents become embeddings then a vector index retrieves neighbors" width="960" height="400" />

*A normal database finds exact keys. A vector database finds "things like this."*

If you have used RAG, semantic search, or Cursor-style codebase indexing, you have already touched this idea. The model never "reads the whole wiki." Something retrieves the closest chunks first. That something is usually a **vector index** - often packaged as a **vector database**.

This post covers:

1. What a vector is (in this context)  
2. How embeddings create those vectors  
3. How similarity is measured  
4. Why brute force fails at scale  
5. Internal index designs: **HNSW**, **IVF**, **PQ**  
6. What a query looks like end to end  
7. What a vector DB stores besides floats  

Related reading on this blog: [context windows](/posts/context-windows/) (why you retrieve instead of pasting everything) and [MCP / agents](/posts/mcp/) (tools that fetch).

---

## The one-sentence definition

A **vector database** stores high-dimensional vectors (plus metadata) and answers:

> Given this query vector, return the *k* most similar stored vectors - fast enough for interactive apps.

"Similar" means close in geometric space, not identical strings.

---

## Step 1: Text becomes a vector (embedding)

An **embedding model** maps text (or code, images, audio) to a list of numbers - a **vector**.

Example (toy numbers; real models use hundreds or thousands of dimensions):

```text
"refund policy for damaged items"
  -> [0.12, -0.44, 0.91, 0.03, ...]   # e.g. 768 or 1536 floats

"how do I get my money back for a broken product?"
  -> [0.11, -0.41, 0.88, 0.05, ...]   # nearby vector
```

Facts that matter:

- Same embedding model must be used for **index** and **query**. Mixing models breaks the geometry.  
- Dimension is fixed per model (for example 384, 768, 1024, 1536).  
- Vectors are usually **L2-normalized** when you use cosine / dot-product search.  
- You embed **chunks**, not whole books - chunk size and overlap strongly affect recall.

<img src="/images/posts/vector-databases/vector-space.svg" alt="Similar meanings form clusters in vector space" width="960" height="420" />

The embedding model does the "understanding." The vector DB does the "find neighbors quickly."

---

## Step 2: Similarity metrics (how "close" is defined)

Common metrics:

| Metric | Idea | Typical use |
|--------|------|-------------|
| **Cosine similarity** | Angle between vectors | Text embeddings (direction matters more than length) |
| **Dot product** | Same as cosine if vectors are normalized | Fast scoring in ANN libraries |
| **L2 / Euclidean distance** | Straight-line distance | Some vision / dense retrieval setups |

Cosine similarity for vectors `a` and `b`:

```text
cos(a, b) = (a · b) / (|a| |b|)
```

If both are unit-length, cosine == dot product. Many systems normalize once at insert time so queries only need a dot product.

**Important:** keyword search (`WHERE title LIKE '%refund%'`) and vector search solve different problems. Vector search can match paraphrases. It can also retrieve wrong-but-nearby chunks if embeddings or chunking are weak.

---

## Step 3: Why not compare against every vector?

Brute force for *N* vectors of dimension *d*:

```text
cost ≈ O(N × d) per query
```

At *N* = 10 million and *d* = 1536, that is billions of float ops per search. Fine for a laptop demo with 10k chunks. Painful for production latency and cost.

So vector databases use **Approximate Nearest Neighbor (ANN)** indexes:

- They do **not** guarantee the exact global top-k every time  
- They trade a little **recall** for a lot of **speed**  
- You tune that tradeoff with index parameters  

"99% recall @ 10ms" is a normal engineering target. Exact search is still available in many systems for small collections or offline evaluation.

---

## What lives inside a vector database

Think of rows like:

| Field | Example |
|-------|---------|
| `id` | `doc_918#chunk_3` |
| `vector` | `float32[768]` (or compressed) |
| `payload / metadata` | `{source: "faq.md", section: "returns"}` |
| Optional | sparse vector, full text, timestamps |

At query time you often combine:

1. ANN candidate retrieval  
2. Metadata filters (`source = "faq.md"`, `lang = "en"`)  
3. Optional re-ranking (cross-encoder, LLM, or BM25 hybrid)

Popular products (Pinecone, Weaviate, Qdrant, Milvus, pgvector, Chroma, FAISS-as-library) differ in packaging, but the math underneath is shared.

---

## Internal index designs

### A) Flat / brute force

Store all vectors. Scan all. Exact. Slow at scale. Useful as a baseline when measuring recall of approximate indexes.

### B) HNSW (Hierarchical Navigable Small World)

The most common graph index in modern vector DBs.

**Idea:** build a multi-layer graph where:

- Top layers are sparse "highways" (long jumps)  
- Bottom layer contains (almost) all points with local links  

Search:

1. Enter at a top-layer node  
2. Greedily walk to neighbors closer to the query  
3. Drop down a layer and refine  
4. On layer 0, collect the best candidates  

<img src="/images/posts/vector-databases/hnsw.svg" alt="HNSW layers from sparse top to dense bottom" width="960" height="480" />

Key parameters you will see in docs:

| Parameter | Meaning |
|-----------|---------|
| `M` | Max edges per node (graph degree). Higher = more memory, often better recall |
| `efConstruction` | How wide the search is while *building* the graph |
| `efSearch` | How wide the search is at *query* time. Higher = slower, better recall |

**Why it works:** small-world graphs have short paths between distant points. Hierarchy makes the first hops cheap.

**Costs:** HNSW is memory-hungry (graph edges + vectors). Inserts are slower than flat. Excellent for low-latency in-RAM serving.

### C) IVF (Inverted File Index)

**Idea:** cluster the dataset (k-means style) into *nlist* centroids. Each vector is assigned to its nearest cluster ("bucket").

Query:

1. Find the nearest *nprobe* centroids to the query  
2. Search only vectors inside those buckets  
3. Ignore the rest of the database  

<img src="/images/posts/vector-databases/ivf.svg" alt="IVF probes only nearby clusters" width="960" height="420" />

| Parameter | Meaning |
|-----------|---------|
| `nlist` | Number of clusters |
| `nprobe` | Clusters searched per query |

**Tradeoff:** small `nprobe` = fast but may miss the true neighbor if it sits in another cluster. Large `nprobe` = closer to flat scan.

IVF shines when the collection is huge and you accept approximate results with fewer RAM graph structures than HNSW.

### D) Product Quantization (PQ) and friends

ANN can still be heavy if every distance uses full `float32` vectors.

**Product Quantization** compresses a vector into a short code:

1. Split the vector into *m* subvectors  
2. For each subspace, learn a codebook of centroids  
3. Store each subvector as a small codebook id (a few bits)  

Distance to a query can be estimated with lookup tables instead of full float math. Variants: **IVF-PQ**, **OPQ**, **SQ** (scalar quantization), **BQ** (binary).

**Fact:** compression saves memory and bandwidth; it also adds approximation error on top of ANN approximation. Always measure recall after enabling PQ.

### E) Hybrid: IVF + HNSW, disk tiers, filtered search

Production systems mix ideas:

- HNSW over PQ-compressed vectors  
- Hot vectors in RAM, cold on disk  
- Pre-filter or post-filter by metadata (filter strategy changes recall/latency)  
- Sparse + dense hybrid (BM25 + vectors) for keyword-sensitive domains  

---

## End-to-end: one RAG query

Example: user asks *"How long is the return window for damaged goods?"*

1. **Chunk** the docs offline (`faq.md` -> 400-token pieces with overlap)  
2. **Embed** each chunk with model `E`  
3. **Upsert** `(id, vector, metadata)` into the vector DB / index  
4. At ask time, embed the question with the **same** model `E`  
5. ANN search: `top_k = 8`, maybe filter `doc_type = policy`  
6. Optional re-rank the 8 down to 3  
7. Stuff those 3 chunks into the LLM prompt ([context window](/posts/context-windows/) budget)  
8. Model answers with citations  

The vector DB never answers the question. It only chooses which facts enter the prompt.

---

## Tiny mental math

Suppose:

- 1 million chunks  
- 768 dimensions  
- `float32`  

Raw vectors alone:

```text
1e6 * 768 * 4 bytes ≈ 3.1 GB
```

Add HNSW edges, ids, metadata, replicas - often several× that. This is why quantization and disk indexes exist, and why "just put all embeddings in Postgres float arrays" eventually needs an ANN strategy (`pgvector` indexes, etc.).

---

## Vector DB vs "just FAISS" vs Postgres

| Approach | Role |
|----------|------|
| **FAISS / Annoy / ScaNN** | Libraries that implement ANN indexes |
| **pgvector / sqlite-vss** | Vector search *inside* an existing SQL DB |
| **Dedicated vector DB** | Ops features: filtering, collections, replication, APIs, hybrid search |

If your app already lives in Postgres and *N* is modest, `pgvector` may be enough. If you need specialized filtering, huge scale, or managed ops, a dedicated vector DB earns its keep.

The internals (HNSW/IVF/PQ) show up in both worlds.

---

## Failure modes (internals showing through)

| Symptom | Likely cause |
|---------|--------------|
| Irrelevant chunks | Bad chunking, wrong embedding model, query too vague |
| Good demo, bad prod scale | Flat scan; need ANN + tuned `efSearch` / `nprobe` |
| Fast but misses answers | ANN recall too low; raise search width or rebuild index |
| Filter returns empty / weird | Pre-filter removed candidates before ANN could see them |
| Sudden quality drop after migrate | Different embedding model or dimension mismatch |
| RAM explosion | Full-precision HNSW on huge *N*; consider PQ / IVF-PQ |

Debugging tip: keep a **flat exact top-k** on a sample set. Compare approximate results to exact. That gap *is* your recall.

---

## How this connects to agents

Agents ([loop](/posts/agent-loop/), [harness](/posts/agent-harness/)) often treat "search vectors" as a **tool**:

1. Agent decides it needs knowledge  
2. Tool embeds the subquery  
3. Vector DB returns neighbors  
4. Observation re-enters the trajectory  

Cursor-style codebase indexes follow the same pattern: embed code chunks, ANN retrieve, inject into the prompt - not "load the monorepo into context."

---

## Practical checklist

1. Pick one embedding model and stick to it for a collection  
2. Chunk with purpose (headers, functions, ~200-800 tokens is a common band - measure)  
3. Start with HNSW defaults; measure latency + recall  
4. Add metadata filters only after unfiltered search works  
5. Consider hybrid search if exact terms (IDs, error codes) matter  
6. Re-embed when you change the embedding model - old vectors become wrong geometry  
7. Log retrieved ids for every production answer (you cannot debug RAG without this)

---

## Closing

A vector database is not magic memory for LLMs. It is a system for:

> **embedding storage + approximate nearest-neighbor search + metadata**

Under the hood, the important pieces are geometric similarity, ANN indexes (especially **HNSW** and **IVF**), and optional **quantization** so millions of vectors fit in real machines.

If you remember one picture:

> Embeddings place meaning in space. Indexes make "who is near this point?" cheap. The LLM only sees the neighbors you retrieve.

**Next step:** take 50 of your own docs, embed them, run exact top-5 vs HNSW top-5 for 20 questions, and write down recall. That single experiment teaches more than any vendor diagram.

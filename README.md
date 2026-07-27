# GDPR RAG Assistant

A local Retrieval-Augmented Generation (RAG) system for the GDPR (General Data Protection Regulation) that respects the legal document's natural structure — chapters, articles, and paragraphs — instead of blindly cutting text into arbitrary chunks.

Built with **LangChain**, **Ollama** (local LLMs), and **Chroma** vector store. Uses `uv` for fast Python package management.

---

## What This Project Is

This is a question-answering system over the GDPR PDF. Unlike typical RAG systems that shred documents into fixed-size chunks and hope for the best, this system:

1. **Parses the legal hierarchy**: Chapters → Articles → Content chunks
2. **Preserves boundaries**: A chunk never crosses from Article 5 into Article 6
3. **Uses metadata filtering**: When you ask for "Article 1," it fetches **all** of Article 1 — guaranteed complete — instead of just the 4 most "similar" chunks
4. **Falls back to semantic search**: For vague questions like "When can companies process my data?", it uses vector similarity to find the most relevant chunks across the entire regulation

---

## The Problem It Solves

### Naive RAG breaks legal documents

Standard RAG splits text every 500 characters with 50-character overlap. For the GDPR, this creates disasters:

| Problem | Example |
|---|---|
| **Cross-article bleeding** | A chunk contains the end of Article 17 (Right to erasure) and the start of Article 18 (Right to restriction). The embedding mixes two different legal rights. |
| **Incomplete retrieval** | You ask "Summarize Article 6." Vector search returns chunk 2 and chunk 4, but misses chunk 1 and chunk 3. The LLM summarizes an article it has never fully seen. |
| **Lost structure** | The system has no idea what a "chapter" or "article" is. It treats the GDPR as a soup of sentences. |

### This project's solution

| Feature | How it fixes the problem |
|---|---|
| **Hard article boundaries** | Each article is a sealed bucket. Chunking only happens *inside* an article, never across. |
| **Metadata filtering** | Detects "Article 1" or "Chapter II" in your question and fetches **every chunk** for that article/chapter. |
| **Hierarchical chunks** | Creates chapter headers, article headers, and content chunks — each tagged with `chapter` and `article` metadata. |
| **Hybrid retrieval** | Specific queries → metadata filter. Vague queries → vector similarity. |

---

## How It Works (The Flow)

```
User asks: "Summarize Article 1"
│
▼
┌─────────────────────────────┐
│  Step 1: Query Analysis     │
│  Regex detects "article 1"  │
│  → filter_dict =            │
│    {"article": "1"}         │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│  Step 2: Retrieval          │
│  IF filter_dict exists:     │
│    Fetch ALL chunks where   │
│    metadata.article == "1"  │
│  ELSE:                      │
│    Vector similarity search │
│    (top-k most relevant)    │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│  Step 3: Format Context     │
│  Add source labels:         │
│  [Ch.I, Art.1] + text       │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│  Step 4: LLM Generation     │
│  Local Llama 3.2 reads the  │
│  context and answers with   │
│  citations.                 │
└─────────────────────────────┘
```

### The chunking pipeline

```
GDPR PDF
│
▼
┌─────────────────┐
│ Extract Chapters│  ← split at "CHAPTER I", "CHAPTER II"...
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Extract Articles│  ← split at "Article 1", "Article 2"...
│ inside each     │    (never cross article boundaries)
│ chapter         │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Create 3 types of chunks:           │
│                                      │
│ 1. Chapter header                   │
│    text: "CHAPTER I — General       │
│           provisions"               │
│    metadata: {chapter: "I",         │
│               article: null}        │
│                                      │
│ 2. Article header                   │
│    text: "Article 1 — Subject-      │
│           matter and objectives"    │
│    metadata: {chapter: "I",         │
│               article: "1"}         │
│                                      │
│ 3. Content chunks (split by size    │
│    and overlap, ONLY inside the     │
│    article body)                    │
│    metadata: {chapter: "I",         │
│               article: "1"}         │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ Embed & Store   │  ← Chroma vector database
│ in Chroma       │    (text is embedded, metadata is stored
└─────────────────┘     alongside for filtering)
```

---

## Examples

### Example 1: Specific article query (uses metadata filter)

**You ask:** *"What does Article 1 say?"*

**System detects:** `{"article": "1"}`

**Retrieves:** All chunks where `article == "1"` — the article header + all content chunks.

**LLM sees:**

```
[Ch.I, Art.1]
Article 1 — Subject-matter and objectives. This Regulation lays down rules...

[Ch.I, Art.1]
2. This Regulation protects fundamental rights and freedoms...
```

**Answer:** *"According to [Ch.I, Art.1], the GDPR lays down rules relating to the protection of natural persons with regard to the processing of personal data, and rules relating to the free movement of such data. It also protects fundamental rights and freedoms, particularly the right to the protection of personal data."*

---

### Example 2: Chapter summary (uses metadata filter)

**You ask:** *"Summarize Chapter II"*

**System detects:** `{"chapter": "II"}`

**Retrieves:** Every chunk from Chapter II across all its articles.

**Answer:** A synthesized summary of all principles in Chapter II, citing [Ch.II, Art.5], [Ch.II, Art.6], etc.

---

### Example 3: Vague question (uses vector search)

**You ask:** *"When can companies process my data?"*

**System detects:** No article or chapter mentioned.

**Retrieves:** Top 4 most semantically similar chunks across the entire GDPR.

**Answer:** *"According to [Ch.II, Art.6], processing is lawful only if one of the following applies: (a) the data subject has given consent..."*

---

### Example 4: What naive RAG would break

**You ask:** *"Summarize Article 1"*

**Naive RAG (no metadata):** Returns chunk 2 of Article 1 and chunk 1 of Article 2 because they happen to be close in the vector space. The LLM summarizes an incomplete, mixed-up article.

**This system:** Returns 100% of Article 1. Guaranteed complete.

---

## Prerequisites

- **Python 3.10+**
- **Ollama** installed locally (for running LLMs and embeddings offline)
- **uv** installed (modern Python package manager, replaces pip)

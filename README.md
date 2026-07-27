# GDPR RAG Assistant

A local Retrieval-Augmented Generation (RAG) system for the GDPR (General Data Protection Regulation) that respects the legal document's natural structure — chapters, articles, and paragraphs — instead of blindly cutting text into arbitrary chunks.

Built with **LangChain**, **Ollama** (local LLMs), and **Chroma** vector store. Uses `uv` for fast Python package management.

---

## Visual Overview

<p align="center">
  <img src="https://github.com/user-attachments/assets/b8499d32-11a9-40eb-9dfa-2e447b6748c7" alt="Mind map of the GDPR RAG Assistant architecture" width="700">
</p>

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

**You ask:**

```
Summarize chapter II
```

**System detects:** `{"chapter": "II"}`

**Retrieves:** Every chunk from Chapter II across all its articles.

**Answer:**

> Chapter II of the GDPR deals with the principles for processing personal data. The main articles in this chapter are:
>
> - **Article 5**: Principles relating to processing of personal data (lawfulness, fairness, transparency, purpose limitation, data minimization, accuracy, storage limitation, security)
> - **Article 6**: Lawfulness of processing (processing shall be lawful only if and to the extent that at least one of the conditions in Article 6(1) applies)
> - **Article 7**: Conditions for consent
> - **Article 8**: Conditions applicable to child's consent in relation to information society services
> - **Article 9**: Processing of special categories of personal data (prohibition on processing certain types of sensitive data, exceptions for specific purposes such as employment and social security)
>
> The main principles outlined in Chapter II are:
>
> 1. Lawfulness, fairness, transparency, purpose limitation, data minimization, accuracy, storage limitation, and security.
> 2. Consent is required for processing personal data, except in specific cases where it is not necessary (Article 6(1)).
> 3. The controller must be able to demonstrate that the processing of personal data is lawful and fair.
> 4. Special categories of personal data require additional safeguards and exceptions for specific purposes.
> 5. Children's consent is subject to special rules and requirements.
>
> Overall, Chapter II provides a framework for understanding the principles and conditions for processing personal data in a way that respects the rights and freedoms of individuals.

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

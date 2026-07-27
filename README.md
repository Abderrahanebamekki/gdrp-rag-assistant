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

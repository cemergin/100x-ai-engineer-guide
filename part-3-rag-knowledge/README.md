<!--
  TYPE: part-index
  PART: 3
  TITLE: RAG & Knowledge Systems
  PHASE: 1 — Get Dangerous
  CHAPTERS: 14, 15, 16, 17
  UPDATED: 2026-04-10
-->

# Part 3 — RAG & Knowledge Systems

> **Phase 1: Get Dangerous** | 4 chapters | Intermediate | Language: TypeScript

How to give your AI access to your company's knowledge. After Part 2 your agent can decide what to do and do it. After Part 3 it can *know things* — search your documents, answer questions about your codebase, and pull the right information at the right time.

Every company has proprietary knowledge locked in PDFs, wikis, databases, and Slack threads. RAG (Retrieval-Augmented Generation) is how you unlock it without fine-tuning a model. These four chapters take you from "what is an embedding?" to "here's a production document QA system."

The four chapters follow a tight spiral:

1. **Semantic search** gives you the core primitive: find similar things (Ch 14)
2. **The RAG pipeline** wraps that primitive in a full system: load, chunk, embed, store, retrieve (Ch 15)
3. **Document QA** applies the pipeline to real-world sources: PDFs, web pages, transcripts (Ch 16)
4. **Advanced retrieval** makes everything better: hybrid search, reranking, HyDE (Ch 17)

By the end you'll have built a working document QA system and understand every retrieval optimization technique that matters.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 14 | [Semantic Search](./14-semantic-search.md) | Intermediate | Embeddings in code, vector stores, similarity search, scoring and ranking |
| 15 | [The RAG Pipeline](./15-rag-pipeline.md) | Intermediate | Document loading, chunking, embedding, storage, retrieval, reranking |
| 16 | [Document QA Systems](./16-document-qa.md) | Intermediate | PDF/YouTube/web loaders, source attribution, retrieval + generation |
| 17 | [Advanced Retrieval](./17-advanced-retrieval.md) | Inter -> Adv | Hybrid search, reranking, HyDE, query expansion, multi-index |

## How This Part Connects

```
Part 0 (LLM Fundamentals)
  │  Ch 1: Embeddings concept
  │
Part 1 (Building with LLM APIs)
  │  Ch 5: Structured output for search results
  │
Part 2 (Agent Engineering)
  │  Ch 8: Tool calling (retrieval as a tool)
  │  Ch 10: Memory (RAG is external memory)
  │
  └──→ Part 3 (you are here)
         Ch 14 implements embeddings from Ch 1
         Ch 15 uses structured output from Ch 5
         Ch 16 combines retrieval with generation
         Ch 17 optimizes everything
         │
         ├──→ Part 4: Evals & Quality
         │      Ch 19 evaluates retrieval quality (Ch 14)
         │      Ch 20 evaluates QA conversations (Ch 16)
         │
         ├──→ Part 5: Harness Engineering
         │      Ch 24: MCP servers can wrap RAG pipelines
         │
         ├──→ Part 8: Open Source AI (Phase 2)
         │      Ch 42: Generate your own embeddings (Ch 14)
         │
         └──→ Part 9: Fine-Tuning (Phase 2)
                Ch 44: RAG vs fine-tuning (the big decision)
```

## The Within-Part Spiral

```
Ch 14: Semantic Search
  └──→ Ch 15: The RAG Pipeline (search becomes a full system)
        └──→ Ch 16: Document QA (the system meets real documents)
              └──→ Ch 17: Advanced Retrieval (optimize every stage)
```

## Reading Time

Expect about 3-4 hours to read all four chapters and build the code examples. By the end you'll have a working document QA system with semantic search, chunking, source attribution, and advanced retrieval strategies.

---

*Previous: [Part 2 — Agent Engineering](../part-2-agent-engineering/)* | *Next: [Part 4 — Evals & Quality](../part-4-evals-quality/)*

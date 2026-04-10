<!--
  TYPE: part-index
  PART: 0
  TITLE: LLM Fundamentals
  PHASE: 1 — Get Dangerous
  CHAPTERS: 0, 1, 2
  UPDATED: 2026-04-10
-->

# Part 0 — LLM Fundamentals

> **Phase 1: Get Dangerous** | 3 chapters | Beginner | Language: TypeScript

Just enough theory to not be confused. Every concept introduced here gets revisited at increasing depth later in the guide — this is the first pass of the spiral.

You don't need a machine learning degree to build with LLMs. But you *do* need a mental model of what's happening inside that API call. These three chapters give you that model: how tokens become text, how meaning becomes math, and who the players are.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 0 | [How LLMs Actually Work](./00-how-llms-work.md) | Beginner | Tokens, context windows, temperature, top-p, why LLMs hallucinate, training vs inference |
| 1 | [Embeddings & Similarity](./01-embeddings-similarity.md) | Beginner | Vectors, cosine similarity, semantic meaning in numbers |
| 2 | [The AI Engineer's Landscape](./02-ai-landscape.md) | Beginner | Providers, models, pricing, SDKs, build vs buy |

## How This Part Connects

```
Part 0 (you are here)
  │
  ├──→ Part 1: Building with LLM APIs
  │      Ch 3 uses tokens + context windows from Ch 0
  │      Ch 5 uses the API landscape from Ch 2
  │
  ├──→ Part 3: RAG & Knowledge
  │      Ch 14 implements the embeddings from Ch 1
  │
  ├──→ Part 7: The Hard Parts (Phase 2)
  │      Ch 34-39 spiral ALL of Part 0 to expert depth
  │
  └──→ Part 8: Open Source AI (Phase 2)
         Ch 42 generates embeddings from Ch 1
         Ch 43 deep-dives model selection from Ch 2
```

## The Within-Part Spiral

```
Ch 0: How LLMs Actually Work (what happens inside the API call)
  └──→ Ch 1: Embeddings & Similarity (how meaning becomes math)
        └──→ Ch 2: The AI Landscape (who the players are, what to use)
```

## Reading Time

Expect about 45-60 minutes to read all three chapters. No code to run — that starts in Part 1.

---

*Next: [Part 1 — Building with LLM APIs](../part-1-building-with-llms/)*

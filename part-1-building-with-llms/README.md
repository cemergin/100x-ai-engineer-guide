<!--
  TYPE: part-index
  PART: 1
  TITLE: Building with LLM APIs
  PHASE: 1 — Get Dangerous
  CHAPTERS: 3, 4, 5, 6, 7
  UPDATED: 2026-04-10
-->

# Part 1 — Building with LLM APIs

> **Phase 1: Get Dangerous** | 5 chapters | Beginner to Intermediate | Language: TypeScript

This is where theory becomes code. By the end of Part 1, you'll have made API calls, written effective prompts, extracted structured data from LLMs, built a streaming chat interface, and worked with images and audio. You'll be able to ship a real AI feature.

Part 1 spirals Part 0 from concepts into implementation. Every abstract idea — tokens, context windows, embeddings, provider choices — becomes a concrete API parameter or design decision.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 3 | [Your First LLM Call](./03-first-llm-call.md) | Beginner | SDK setup, chat completions, system prompts, streaming, error handling |
| 4 | [Prompt Engineering That Works](./04-prompt-engineering.md) | Beg→Inter | System prompts, few-shot, chain-of-thought, templates, versioning |
| 5 | [Structured Output](./05-structured-output.md) | Intermediate | JSON mode, Zod schemas, response_format, type-safe LLM responses |
| 6 | [Building a Chat Interface](./06-chat-interface.md) | Intermediate | History management, streaming to UI, Vercel AI SDK |
| 7 | [Multimodal AI](./07-multimodal.md) | Intermediate | Vision, image generation, audio, multi-input |

## How This Part Connects

```
Part 0 (Fundamentals)
  │
  └──→ Part 1 (you are here)
         Ch 3 uses tokens + context windows from Ch 0
         Ch 4 refines WHAT you send in those calls
         Ch 5 gets typed data back (not just strings)
         Ch 6 builds a real chat experience
         Ch 7 adds images and audio
         │
         ├──→ Part 2: Agent Engineering
         │      Ch 8 uses tool calling (builds on Ch 5 schemas)
         │      Ch 9 wraps your LLM calls in agent loops
         │
         ├──→ Part 3: RAG & Knowledge
         │      Ch 14 implements embeddings from Ch 1
         │      Ch 15 combines retrieval with Ch 3-4 patterns
         │
         └──→ Part 4: Evals & Quality
                Ch 19 evaluates the prompts from Ch 4
                Ch 20 evaluates the chat from Ch 6
```

## Prerequisites

- Node.js 18+ installed
- An OpenAI API key and/or an Anthropic API key (free tiers available)
- Basic TypeScript knowledge
- Chapter 0-2 (or equivalent understanding)

## Time to Complete

Expect 3-4 hours to read through all five chapters and run the examples. Allow more time if you want to experiment with your own projects.

---

*Previous: [Part 0 — LLM Fundamentals](../part-0-fundamentals/)* | *Next: [Part 2 — Agent Engineering](../part-2-agent-engineering/)*

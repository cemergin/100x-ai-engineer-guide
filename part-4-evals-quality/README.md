<!--
  TYPE: part-index
  PART: 4
  TITLE: Evals & Quality
  PHASE: 1 — Get Dangerous
  CHAPTERS: 18, 19, 20, 21, 22
  UPDATED: 2026-04-10
-->

# Part 4 — Evals & Quality

> **Phase 1: Get Dangerous** | 5 chapters | Intermediate | Language: TypeScript

How to know if your AI actually works. After Part 3 your AI can search documents and answer questions. After Part 4 you can *measure* whether it's doing a good job — and systematically improve it.

Here's the thing about AI that breaks every traditional testing instinct: it's non-deterministic. You can't just assert that output equals expected. You need a completely new approach to quality. That approach is called evals, and it's the single most important skill that separates AI engineers who ship reliable products from AI engineers who ship demos.

The five chapters follow a tight spiral:

1. **Why evals matter** gives you the mindset and the vocabulary (Ch 18)
2. **Single-turn evals** let you measure one input/output pair (Ch 19)
3. **Multi-turn evals** let you measure conversations and agent workflows (Ch 20)
4. **Eval-driven development** turns evals into your development workflow (Ch 21)
5. **Telemetry and tracing** instruments everything so you can eval in production (Ch 22)

By the end you'll have a complete eval framework, an LLM-as-judge system, and production instrumentation.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 18 | [Why Evals Matter](./18-why-evals-matter.md) | Intermediate | Non-determinism, what to measure, offline/online evals, building datasets |
| 19 | [Single-Turn Evals](./19-single-turn-evals.md) | Intermediate | Tool selection scoring, output format checking, scorer functions |
| 20 | [Multi-Turn Evals](./20-multi-turn-evals.md) | Inter -> Adv | Conversation evals, LLM-as-judge, structured judge prompts |
| 21 | [Eval-Driven Development](./21-eval-driven-dev.md) | Inter -> Adv | Write eval -> run -> analyze -> improve -> repeat, the Ralph loop |
| 22 | [Telemetry & Tracing](./22-telemetry-tracing.md) | Intermediate | OpenTelemetry, Laminar, Datadog, token tracking, cost per feature |

## How This Part Connects

```
Part 0 (LLM Fundamentals)
  │  Ch 0: LLMs are probabilistic — evals are how you deal with it
  │
Part 1 (Building with LLM APIs)
  │  Ch 4: Prompts to evaluate
  │  Ch 5: Structured output for judge responses
  │
Part 2 (Agent Engineering)
  │  Ch 8: Tool selection to evaluate
  │  Ch 9: Agent loop to evaluate
  │
Part 3 (RAG & Knowledge)
  │  Ch 14: Retrieval quality to evaluate
  │  Ch 16: QA accuracy to evaluate
  │
  └──→ Part 4 (you are here)
         Ch 18 establishes the eval mindset
         Ch 19-20 build the eval framework
         Ch 21 turns it into a workflow
         Ch 22 instruments for production
         │
         ├──→ Part 5: Harness Engineering
         │      Ch 25: Evals in CI/CD
         │
         ├──→ Part 9: Fine-Tuning (Phase 2)
         │      Ch 45: Eval before/after fine-tuning
         │      Ch 46: Use evals to curate datasets
         │
         └──→ Part 10: Production AI (Phase 2)
                Ch 51: Continuous eval in production
                Ch 52: Full observability stack
```

## The Within-Part Spiral

```
Ch 18: Why Evals Matter
  └──→ Ch 19: Single-Turn Evals (measure one thing)
        └──→ Ch 20: Multi-Turn Evals (measure conversations)
              └──→ Ch 21: Eval-Driven Development (make it a workflow)
                    └──→ Ch 22: Telemetry & Tracing (instrument everything)
```

## Reading Time

Expect about 3-4 hours to read all five chapters and build the code examples. By the end you'll have a working eval framework, judge prompts, experiment tracking, and production telemetry.

---

*Previous: [Part 3 -- RAG & Knowledge Systems](../part-3-rag-knowledge/)* | *Next: [Part 5 -- Harness Engineering](../part-5-harness-engineering/)*

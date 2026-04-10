<!--
  TYPE: index
  TITLE: The 100x AI Engineer Guide
  CHAPTERS: 63
  PARTS: 13
  PHASES: 2
  APPENDICES: 3
  UPDATED: 2026-04-10
  STRUCTURE: spiral-narrative
  AI_NOTE: Each chapter file contains an HTML comment metadata block with CHAPTER, TITLE, PART, PHASE, PREREQS, KEY_TOPICS, DIFFICULTY, LANGUAGE, UPDATED fields. Use these for filtering, prerequisite graphs, and topic search. Files are organized into subdirectories by Part. The guide uses a fractal spiral structure where concepts are introduced at "Get Dangerous" depth and revisited at "Become an Expert" depth.
-->

# The 100x AI Engineer Guide
### From "Get Dangerous" to "Build the Platform"

A comprehensive, opinionated mega-guide for the engineer who gets dropped into a company and needs to add AI capabilities to products, teams, and infrastructure — organized as a fractal spiral curriculum that revisits every concept at increasing depth.

**63 chapters** · **13 parts** · **2 phases** · **TypeScript + Python**

```
100x-ai-engineer-guide/

═══ PHASE 1: GET DANGEROUS ═══  (TypeScript-first, API-level, ship in a week)
├── part-0-fundamentals/              ← Just enough theory: tokens, embeddings, models, providers
├── part-1-building-with-llms/        ← Your first AI features: APIs, chat, structured output, streaming
├── part-2-agent-engineering/         ← Tool calling, agent loops, memory, approvals, context management
├── part-3-rag-knowledge/             ← Embeddings, vector search, document QA, advanced retrieval
├── part-4-evals-quality/             ← Single-turn, multi-turn, eval-driven dev, telemetry
├── part-5-harness-engineering/       ← Claude Code, MCP, skills, plugins, automation, coding agents

═══ PHASE 2: BECOME AN EXPERT ═══  (Python + TS, ML-deep, build platforms)
├── part-6-anatomy-of-ai-tools/       ← How Claude Code, Cursor work inside: memory, context, tools, search
├── part-7-hard-parts/                ← Neural nets, transformers, attention from scratch
├── part-8-open-source-ai/            ← Hugging Face, inference, pipelines, model selection
├── part-9-fine-tuning-training/      ← LoRA, datasets, quantization, RAG vs fine-tuning
├── part-10-production-ai/            ← Security, cost, context mgmt, observability, guardrails
├── part-11-deployment-infrastructure/← Sandboxing, CI/CD for AI, scaling, provider management
├── part-12-ai-platform/              ← Multi-agent, internal tools, skills marketplaces, org enablement

├── appendices/                       ← Glossary, resources, cheat sheet
└── README.md                         ← You are here
```

---

## Who This Guide Is For

You're an engineer who just got told "we need to add AI to our product." Maybe you're the senior engineer who needs to ship an AI feature by next sprint. Maybe you're the tech lead evaluating whether to build or buy. Maybe you're the platform engineer who needs to make the whole company productive with AI tools. Maybe you're the curious developer who sees what Ramp, Stripe, and Anthropic are doing and wants to understand how it all works.

This guide is for you if:
- You need to ship AI features into production, not just prototype in a notebook
- You want to understand agents, RAG, evals, and tool calling — not just call an API
- You need to know when to use an API vs. fine-tune vs. run open-source
- You want to build the AI infrastructure that makes your whole team dangerous
- You believe the harness matters more than the model
- You want to understand how tools like Claude Code and Cursor actually work inside

**Phase 1** gets you dangerous in a week. **Phase 2** makes you the person who builds the platform.

---

## The Spiral Structure

This guide uses a **fractal spiral curriculum**. Every core concept appears multiple times at increasing depth:

1. **Within each Part** — each chapter deepens the previous chapter
2. **Between Parts** — later Parts spiral back to earlier Parts
3. **Between Phases** — Phase 2 revisits everything from Phase 1 at expert depth

Example — how **embeddings** spiral through the guide:
```
Ch 1  (concept)      → "Embeddings are vectors that capture semantic meaning"
Ch 14 (use for search) → Build semantic search with vector stores
Ch 19 (eval retrieval) → Score how well your retrieval works
Ch 36 (understand math) → Neural nets produce embeddings via learned weights
Ch 42 (generate own)   → Run Sentence Transformers, choose embedding models
Ch 44 (vs fine-tuning)  → When to retrieve knowledge vs bake it into weights
```

Example — how **cost awareness** spirals through the guide:
```
Ch 2  (pricing)         → "GPT-4o costs $2.50/M input tokens, Claude Sonnet costs $3/M"
Ch 12 (context as cost) → Every token in context costs money — manage it
Ch 22 (tracking)        → Instrument your app to track cost per feature
Ch 29 (compaction)      → How Claude Code saves money with smart summarization
Ch 49 (engineering)     → Model routing, caching, budgets — the full cost toolkit
Ch 57 (infrastructure)  → Self-host vs API break-even math, GPU cost optimization
```

---

## Quick Start — Reading Paths

| Your Goal | Start Here | Then |
|-----------|-----------|------|
| **Ship an AI feature NOW** | Part 1: Ch 3 → 4 → 5 | Part 2: Ch 8 → 9 |
| **Build an AI agent** | Part 2: Ch 8 → 9 → 10 → 11 | Part 4: Ch 18 → 19 |
| **Add RAG to your product** | Part 3: Ch 14 → 15 → 16 | Part 4: Ch 19 |
| **Set up evals** | Part 4: Ch 18 → 19 → 20 → 21 | Part 10: Ch 51 |
| **Make your team productive with AI** | Part 5: Ch 23 → 24 → 25 | Part 12: Ch 59 → 60 |
| **Understand how Claude Code works** | Part 6: Ch 27 → 28 → 29 | Ch 30 → 32 |
| **Learn ML fundamentals** | Part 7: Ch 34 → 35 → 36 | Part 8: Ch 40 |
| **Fine-tune a model** | Part 9: Ch 44 → 45 → 46 | Part 8: Ch 43 |
| **Deploy AI safely** | Part 11: Ch 53 → 55 → 56 | Part 10: Ch 48 |
| **Build an AI platform for your company** | Part 12: Ch 58 → 59 → 60 | Ch 61 → 62 |
| **Quick lookup** | [Glossary](./appendices/appendix-glossary.md) | [Resources](./appendices/appendix-resources.md) |

---

## ═══ PHASE 1: GET DANGEROUS ═══

*TypeScript-first. API-level. Ship in a week.*

### [Part 0 — LLM Fundamentals](./part-0-fundamentals/)
*Just enough theory to not be confused. Every concept here gets revisited deeper later.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 0 | [How LLMs Actually Work](./part-0-fundamentals/00-how-llms-work.md) | Beginner | Tokens, context windows, temperature, top-p, why LLMs hallucinate, training vs inference |
| 1 | [Embeddings & Similarity](./part-0-fundamentals/01-embeddings-similarity.md) | Beginner | Vectors, cosine similarity, semantic meaning in numbers |
| 2 | [The AI Engineer's Landscape](./part-0-fundamentals/02-ai-landscape.md) | Beginner | Providers, models, pricing, SDKs, build vs buy |

### [Part 1 — Building with LLM APIs](./part-1-building-with-llms/)
*Your first AI features. Spirals Part 0 from theory into code.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 3 | [Your First LLM Call](./part-1-building-with-llms/03-first-llm-call.md) | Beginner | SDK setup, chat completions, system prompts, streaming, error handling |
| 4 | [Prompt Engineering That Works](./part-1-building-with-llms/04-prompt-engineering.md) | Beg→Inter | System prompts, few-shot, chain-of-thought, templates, versioning, prompt management at scale |
| 5 | [Structured Output](./part-1-building-with-llms/05-structured-output.md) | Intermediate | JSON mode, Zod schemas, response_format, type-safe LLM responses |
| 6 | [Building a Chat Interface](./part-1-building-with-llms/06-chat-interface.md) | Intermediate | History management, streaming to UI, Vercel AI SDK, AI UX patterns |
| 7 | [Multimodal AI](./part-1-building-with-llms/07-multimodal.md) | Intermediate | Vision, image generation, audio, multi-input |

### [Part 2 — Agent Engineering](./part-2-agent-engineering/)
*From API calls to autonomous agents. Spirals Part 1: your LLM calls become tools inside a loop.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 8 | [Tool Calling](./part-2-agent-engineering/08-tool-calling.md) | Intermediate | Tool definitions, Zod schemas, execute functions, descriptions |
| 9 | [The Agent Loop](./part-2-agent-engineering/09-agent-loop.md) | Intermediate | Prompt→LLM→tool→execute→append→repeat, stop conditions, streaming |
| 10 | [Agent Memory & State](./part-2-agent-engineering/10-agent-memory.md) | Inter→Adv | Short-term, working, long-term memory architectures |
| 11 | [Human-in-the-Loop](./part-2-agent-engineering/11-human-in-the-loop.md) | Intermediate | Sync/async approvals, trust spectrum, approval architectures |
| 12 | [Context Window Management](./part-2-agent-engineering/12-context-management.md) | Inter→Adv | Token counting, compaction, summarization, sliding windows |
| 13 | [Agent Patterns & Frameworks](./part-2-agent-engineering/13-agent-patterns.md) | Inter→Adv | ReAct, Plan-and-Execute, LangChain, Mastra, Vercel AI SDK |

### [Part 3 — RAG & Knowledge Systems](./part-3-rag-knowledge/)
*How to give your AI access to your company's knowledge. Spirals Part 0 embeddings into implementation.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 14 | [Semantic Search](./part-3-rag-knowledge/14-semantic-search.md) | Intermediate | Embeddings in code, vector stores (Pinecone, Upstash, pgvector), similarity search, NL-to-SQL, scoring |
| 15 | [The RAG Pipeline](./part-3-rag-knowledge/15-rag-pipeline.md) | Intermediate | Document loading, chunking, embedding, storage, retrieval, reranking |
| 16 | [Document QA Systems](./part-3-rag-knowledge/16-document-qa.md) | Intermediate | PDF/YouTube/web loaders, source attribution, retrieval + generation |
| 17 | [Advanced Retrieval](./part-3-rag-knowledge/17-advanced-retrieval.md) | Inter→Adv | Hybrid search, reranking, HyDE, query expansion, multi-index |

### [Part 4 — Evals & Quality](./part-4-evals-quality/)
*How to know if your AI actually works. Spirals back into Parts 1-3.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 18 | [Why Evals Matter](./part-4-evals-quality/18-why-evals-matter.md) | Intermediate | Non-determinism, what to measure, offline/online, datasets |
| 19 | [Single-Turn Evals](./part-4-evals-quality/19-single-turn-evals.md) | Intermediate | Tool selection scoring, output format checking, scorer functions |
| 20 | [Multi-Turn Evals](./part-4-evals-quality/20-multi-turn-evals.md) | Inter→Adv | Conversation evals, LLM-as-judge, structured judge prompts |
| 21 | [Eval-Driven Development](./part-4-evals-quality/21-eval-driven-dev.md) | Inter→Adv | Write eval → run → analyze → improve → repeat, the Ralph loop |
| 22 | [Telemetry & Tracing](./part-4-evals-quality/22-telemetry-tracing.md) | Intermediate | OpenTelemetry, Laminar, Datadog, token tracking, cost per feature |

### [Part 5 — Harness Engineering](./part-5-harness-engineering/)
*The meta-skill: making yourself and your team 10x more productive with AI tools.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 23 | [Claude Code Mastery](./part-5-harness-engineering/23-claude-code-mastery.md) | Beg→Adv | Skills, plugins, hooks, CLAUDE.md, plan mode, worktrees, permissions |
| 24 | [MCP Servers & Integrations](./part-5-harness-engineering/24-mcp-servers.md) | Intermediate | MCP protocol, building servers, connecting tools |
| 25 | [Skills, Plugins & Automation](./part-5-harness-engineering/25-skills-plugins.md) | Inter→Adv | Writing skills, building plugins, cron jobs, scheduled agents, event-driven workflows |
| 26 | [AI-Augmented Development](./part-5-harness-engineering/26-ai-augmented-dev.md) | Inter→Adv | Coding agents, multi-agent delegation, Stripe Minions, self-reinforcing loops |

---

## ═══ PHASE 2: BECOME AN EXPERT ═══

*Python + TypeScript. ML-deep. Build platforms.*

### [Part 6 — Anatomy of AI Developer Tools](./part-6-anatomy-of-ai-tools/)
*How Claude Code, Cursor, and coding agents actually work inside. Spirals Part 5: from using the harness to understanding it.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 27 | [The Agent Harness Architecture](./part-6-anatomy-of-ai-tools/27-harness-architecture.md) | Inter→Adv | CLI→loop→tools→permissions→UI. Claude Code vs Cursor vs Codex internals |
| 28 | [Memory Systems: KAIROS & Beyond](./part-6-anatomy-of-ai-tools/28-memory-systems.md) | Advanced | 3-layer memory, CLAUDE.md injection, AutoDream, append-only logs, semantic merging |
| 29 | [Context Window Internals](./part-6-anatomy-of-ai-tools/29-context-internals.md) | Advanced | Compaction service, token budgets, cache-break vectors, session limits |
| 30 | [Tool Execution & Permissions](./part-6-anatomy-of-ai-tools/30-tool-execution.md) | Advanced | Permission models, approval flows, risk tiers, sandboxing |
| 31 | [Web Search & Knowledge Pipelines](./part-6-anatomy-of-ai-tools/31-web-search-pipelines.md) | Advanced | Search APIs, content extraction, Turndown, paraphrase limits, pre-approved domains |
| 32 | [Multi-Agent Coordination](./part-6-anatomy-of-ai-tools/32-multi-agent-coordination.md) | Advanced | Coordinator mode, tick loops, background agents, worker delegation |
| 33 | [Skills, Plugins & Distribution](./part-6-anatomy-of-ai-tools/33-skills-plugins-internals.md) | Advanced | Skill architecture, Skillify, hook system, anti-distillation, org distribution |

### [Part 7 — The Hard Parts](./part-7-hard-parts/)
*Build a neural network by hand. Spirals Part 0 all the way down.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 34 | [ML Decision Making](./part-7-hard-parts/34-ml-decision-making.md) | Intermediate | Prediction, features, decision boundaries, the DoorDash refund example |
| 35 | [Data & Preprocessing](./part-7-hard-parts/35-data-preprocessing.md) | Intermediate | Sample populations, normalization, train/test splits, the pixel grid |
| 36 | [Neural Networks from Scratch](./part-7-hard-parts/36-neural-networks.md) | Inter→Adv | Weights, sigmoid, gradient descent, backpropagation, smile detector |
| 37 | [Tokenization Deep Dive](./part-7-hard-parts/37-tokenization.md) | Inter→Adv | BPE, WordPiece, encoding/decoding, batching, attention masks |
| 38 | [Transformers & Attention](./part-7-hard-parts/38-transformers-attention.md) | Advanced | Self-attention, multi-head, positional encoding, encoder/decoder |
| 39 | [Decoding & Generation](./part-7-hard-parts/39-decoding-generation.md) | Advanced | Greedy, beam search, top-k, top-p, temperature as math |

### [Part 8 — Open Source AI & Inference](./part-8-open-source-ai/)
*Run your own models. Spirals Part 1: from calling APIs to local inference.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 40 | [Hugging Face & Pipelines](./part-8-open-source-ai/40-hugging-face.md) | Intermediate | Pipeline API, model hub, tasks, running inference locally |
| 41 | [Image Generation](./part-8-open-source-ai/41-image-generation.md) | Intermediate | Stable Diffusion, text-to-image, image-to-image, DreamBooth |
| 42 | [Embeddings & Sentence Transformers](./part-8-open-source-ai/42-sentence-transformers.md) | Intermediate | Generate your own embeddings, MTEB benchmarks, model selection |
| 43 | [Model Selection & Architecture](./part-8-open-source-ai/43-model-selection.md) | Inter→Adv | BERT vs GPT vs T5 vs Llama, sizes, quantization, local vs cloud |

### [Part 9 — Fine-Tuning & Training](./part-9-fine-tuning-training/)
*Bake knowledge into weights. Spirals Part 3: from retrieving knowledge to training it in.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 44 | [RAG vs Fine-Tuning](./part-9-fine-tuning-training/44-rag-vs-finetuning.md) | Inter→Adv | When to retrieve vs train, cost/quality/latency trade-offs |
| 45 | [Fine-Tuning with LoRA](./part-9-fine-tuning-training/45-lora-finetuning.md) | Advanced | Low-rank adaptation, PEFT, fine-tune GPT-2 on custom data |
| 46 | [Dataset Engineering](./part-9-fine-tuning-training/46-dataset-engineering.md) | Advanced | Curating data, quality, formats, synthetic data, augmentation |
| 47 | [Quantization & Deployment](./part-9-fine-tuning-training/47-quantization-deployment.md) | Advanced | GGUF, GPTQ, AWQ, inference optimization, when to quantize |

### [Part 10 — Production AI Systems](./part-10-production-ai/)
*Ship AI that doesn't break. Spirals Part 4 from "does it work?" to full production hardening.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 48 | [AI Security & Guardrails](./part-10-production-ai/48-security-guardrails.md) | Inter→Adv | Prompt injection, jailbreaks, lethal trifecta, PII detection |
| 49 | [Cost Engineering](./part-10-production-ai/49-cost-engineering.md) | Inter→Adv | Token optimization, model routing, caching, usage budgets |
| 50 | [Advanced Context Strategies](./part-10-production-ai/50-advanced-context.md) | Advanced | Recursive compaction, sub-agent delegation, tiered memory |
| 51 | [Production Eval Pipelines](./part-10-production-ai/51-production-evals.md) | Advanced | Continuous evaluation, A/B testing, regression detection, eval in CI |
| 52 | [AI Observability & Incidents](./part-10-production-ai/52-observability-incidents.md) | Advanced | Datadog LLM monitoring, hallucination detection, degradation alerts |

### [Part 11 — AI Deployment & Infrastructure](./part-11-deployment-infrastructure/)
*Deploy and run AI safely at scale. Spirals Part 10: from what to worry about to how to actually deploy it.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 53 | [Deploying LLM Applications](./part-11-deployment-infrastructure/53-deploying-llm-apps.md) | Intermediate | Serverless vs dedicated, Vercel Functions, Docker, environment management |
| 54 | [API Gateway & Provider Management](./part-11-deployment-infrastructure/54-api-gateway.md) | Inter→Adv | Rate limiting, failover, Vercel AI Gateway, caching, secrets |
| 55 | [Sandboxing & Isolating Agents](./part-11-deployment-infrastructure/55-sandboxing-agents.md) | Advanced | The OpenClaw pattern, VPC isolation, proxy restrictions, file system isolation, code execution sandboxing |
| 56 | [CI/CD for AI Applications](./part-11-deployment-infrastructure/56-cicd-for-ai.md) | Inter→Adv | Evals in CI, canary deployments, A/B testing, feature flags for AI |
| 57 | [Scaling & Cost at the Infra Level](./part-11-deployment-infrastructure/57-scaling-cost-infra.md) | Advanced | GPU vs CPU, serverless inference, batching, model caching, break-even math |

### [Part 12 — AI Platform Engineering](./part-12-ai-platform/)
*The capstone. From your setup to everyone's setup. Spirals everything.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 58 | [Multi-Agent Orchestration](./part-12-ai-platform/58-multi-agent-orchestration.md) | Advanced | Agents spawning agents, council patterns, parallel execution |
| 59 | [Building Internal AI Tools](./part-12-ai-platform/59-internal-ai-tools.md) | Advanced | The Glass pattern: SSO, pre-configured, low barrier internal platforms |
| 60 | [Skills Marketplaces & Knowledge Sharing](./part-12-ai-platform/60-skills-marketplaces.md) | Advanced | The Dojo pattern: Git-backed, versioned, discovery, Sensei recommendations |
| 61 | [Self-Reinforcing AI Systems](./part-12-ai-platform/61-self-reinforcing-systems.md) | Advanced | Feedback loops, eval-driven iteration, agents that improve themselves |
| 62 | [AI Adoption & Enablement](./part-12-ai-platform/62-ai-adoption.md) | All levels | The Ramp playbook: L0-L3, leaderboards, hub-and-spoke, removing constraints |

---

## [Appendices](./appendices/)

| Item | What's Inside |
|------|--------------|
| [Glossary](./appendices/appendix-glossary.md) | 200+ AI/ML terms from attention to zero-shot |
| [Resources](./appendices/appendix-resources.md) | Essential papers, books, courses, blogs |
| [Cheat Sheet](./appendices/appendix-cheatsheet.md) | Quick reference: model comparison, pricing, SDK patterns |

---

## The Spiral Map

Every core concept's journey through the guide:

```
EMBEDDINGS:     Ch 1 (concept) → Ch 14 (search) → Ch 19 (eval)
                → Ch 36 (math) → Ch 42 (generate) → Ch 44 (vs fine-tuning)

AGENT LOOP:     Ch 9 (build) → Ch 13 (frameworks) → Ch 23 (Claude Code)
                → Ch 27 (production architecture) → Ch 32 (multi-agent)
                → Ch 58 (orchestration)

EVALS:          Ch 18-22 (learn) → Ch 25 (skill evals) → Ch 45 (fine-tuning)
                → Ch 46 (datasets) → Ch 51 (production) → Ch 61 (self-reinforcing)

TOOLS/MCP:      Ch 8 (calling) → Ch 24 (MCP) → Ch 25 (skills)
                → Ch 30 (internals) → Ch 48 (security) → Ch 59 (platforms)

CONTEXT:        Ch 0 (concept) → Ch 6 (chat) → Ch 12 (management)
                → Ch 29 (internals) → Ch 49 (cost) → Ch 50 (advanced)

MEMORY:         Ch 10 (concepts) → Ch 23 (CLAUDE.md) → Ch 28 (KAIROS)
                → Ch 50 (production) → Ch 59 (platform tools)

SECURITY:       Ch 11 (approvals) → Ch 30 (permissions) → Ch 48 (guardrails)
                → Ch 55 (sandboxing) → Ch 62 (adoption governance)

DEPLOYMENT:     Ch 3 (API call) → Ch 6 (chat UI) → Ch 27 (harness architecture)
                → Ch 53 (deploy apps) → Ch 55 (isolate agents) → Ch 56 (CI/CD)

STREAMING:      Ch 3 (basic streamText) → Ch 6 (streaming UI with useChat)
                → Ch 9 (streaming in agent loop) → Ch 27 (production streaming architecture)
                → Ch 53 (deploying streaming endpoints) → Ch 59 (streaming in platform tools)

COST:           Ch 2 (pricing landscape) → Ch 12 (context as cost)
                → Ch 22 (token tracking) → Ch 29 (compaction saves money)
                → Ch 49 (cost engineering) → Ch 57 (infrastructure cost optimization)
```

---

## How Each Chapter Is Structured

Every chapter follows a consistent, AI-scannable format:

```
<!-- HTML metadata: CHAPTER, TITLE, PART, PHASE, PREREQS, KEY_TOPICS, DIFFICULTY, LANGUAGE, UPDATED -->

# Chapter N: Title

> Part · Phase · Prerequisites · Difficulty · Language

Summary paragraph.

### In This Chapter        ← section index
### Related Chapters       ← cross-references (spiral connections)

---

## 1. MAJOR SECTION
### 1.1 Subsection
**What it is:** ...
**When to use:** ...
**Trade-offs:** ...
**Real-world example:** ...
**Code:** ... (working examples in the chapter's language)
```

---

## Key Principles

1. **The harness matters more than the model** — a well-configured system around a good model beats a great model with no system
2. **Get dangerous first, understand later** — ship something, then learn why it works
3. **Every concept spirals** — you'll see each idea multiple times at increasing depth
4. **Use the right language for the job** — TypeScript for applications, Python for ML
5. **Evals are not optional** — if you can't measure it, you can't improve it
6. **Build for your team, not just yourself** — the platform engineer's job is to raise the floor

---

*Built with [Claude Code](https://claude.ai/claude-code). Contributions welcome.*

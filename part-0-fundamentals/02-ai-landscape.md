<!--
  CHAPTER: 2
  TITLE: The AI Engineer's Landscape
  PART: 0 — LLM Fundamentals
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 0, Chapter 1
  KEY_TOPICS: providers, pricing, rate limits, SDKs, Vercel AI SDK, OpenAI SDK, Anthropic SDK, build vs buy, model selection
  DIFFICULTY: Beginner
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 2: The AI Engineer's Landscape

> **Part 0 — LLM Fundamentals** | Phase 1: Get Dangerous | Prerequisites: Ch 0, Ch 1 | Difficulty: Beginner | Language: TypeScript

You understand how LLMs work (Ch 0) and how embeddings represent meaning (Ch 1). Now you need to know: who sells these things, what do they cost, and what tools do you use to talk to them?

The AI landscape in 2026 is simultaneously crowded and consolidating. There are dozens of model providers but really only a handful that matter for production engineering. There are a dozen SDKs but a clear winner for TypeScript developers. And there's a seemingly complex build-vs-buy decision that actually follows a simple logic.

This chapter maps the territory. By the end, you'll know who the players are, what everything costs, what SDK to reach for, and how to make the "build or buy" decision for your AI features. Then in Chapter 3, we start writing code.

### In This Chapter
- The major providers and what makes each unique
- Pricing models: per-token, per-request, self-hosted
- Rate limits and how they shape your architecture
- SDKs: Vercel AI SDK, OpenAI SDK, Anthropic SDK
- The build vs buy decision tree
- How to think about model selection

### Related Chapters
- **Ch 0 (How LLMs Work)** — the model families we compare here
- **Ch 3 (Your First LLM Call)** — where you use these SDKs in code
- **Ch 43 (Model Selection & Architecture)** — spirals to expert depth: BERT vs GPT vs T5 vs Llama, quantization, architecture trade-offs
- **Ch 49 (Cost Engineering)** — spirals pricing into production cost optimization
- **Ch 54 (API Gateway & Provider Management)** — spirals rate limits into failover, caching, and gateway architecture

---

## 1. The Provider Landscape

### 1.1 The Big Five

As of early 2026, there are five providers that dominate production AI engineering. Others exist, but these are the ones you'll work with:

| Provider | HQ | Key Models | Business Model | Moat |
|---|---|---|---|---|
| **OpenAI** | San Francisco | GPT-4o, o3, o4-mini | API-first | Ecosystem, brand, ChatGPT |
| **Anthropic** | San Francisco | Claude Opus 4, Sonnet 4, Haiku 3.5 | API + Claude Code | Instruction following, safety, developer tools |
| **Google** | Mountain View | Gemini 2.5 Pro, 2.0 Flash | API + Google Cloud | 1M context, multimodal, infrastructure |
| **Meta** | Menlo Park | Llama 4 Scout, Maverick | Open source | Free to use, fine-tune, deploy |
| **Mistral** | Paris | Mistral Large, Codestral | API + open weights | European data residency, efficiency |

### 1.2 The Smaller Players Worth Knowing

**Cohere** — Enterprise-focused. Strong embeddings and reranking models. Their Embed v3 and Rerank models are some of the best. Worth knowing for RAG pipelines.

**Perplexity** — Search-augmented AI. Their API includes real-time web search integrated with generation. Useful when you need up-to-date information without building your own search pipeline.

**Groq** — Not a model provider but an inference provider. They run open-source models (Llama, Mixtral) on their custom LPU chips with extremely low latency. Relevant if speed is your top priority.

**Together AI** — Inference platform for open-source models. Good pricing, wide model selection, easy fine-tuning. A managed way to use open-source models without running your own infrastructure.

**Fireworks AI** — Similar to Together — fast inference for open-source models with good developer experience and competitive pricing.

### 1.3 The Emerging Category: Inference Providers

An important trend to understand: the separation of model creation from model hosting. OpenAI both builds and hosts GPT-4o. But Llama can be hosted by:

- Meta (no hosting, just weights)
- Together AI
- Fireworks AI
- Groq
- AWS Bedrock
- Google Cloud Vertex AI
- Azure AI
- Your own servers

This means for open-source models, you're choosing both a model AND an inference provider. Different providers offer different prices, latencies, and features.

---

## 2. Pricing Models

### 2.1 Per-Token Pricing (Most Common)

Most closed-source APIs charge by the token, with separate rates for input and output:

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Notes |
|---|---|---|---|
| GPT-4o | $2.50 | $10.00 | The workhorse |
| GPT-4o-mini | $0.15 | $0.60 | The cheap option |
| o3 | $10.00 | $40.00 | Reasoning model |
| o4-mini | $1.10 | $4.40 | Cheap reasoning |
| Claude Opus 4 | $15.00 | $75.00 | Premium tier |
| Claude Sonnet 4 | $3.00 | $15.00 | Best value for quality |
| Claude 3.5 Haiku | $0.80 | $4.00 | Fast and cheap |
| Gemini 2.5 Pro | $1.25 | $10.00 | Under 200K context |
| Gemini 2.0 Flash | $0.10 | $0.40 | Very cheap |

**Note:** These prices change frequently. Check provider pricing pages for current rates. The relative positioning (which is cheap, which is expensive) tends to be more stable than the absolute numbers.

### 2.2 Understanding Token Costs

Let's make these numbers concrete:

```
Scenario: A customer support chatbot
- System prompt: 500 tokens
- Average user message: 100 tokens
- Average response: 300 tokens
- Average conversation: 10 turns

Per conversation (using GPT-4o):
  Input tokens:  500 + (10 × 100) + (9 × 300 accumulated) = ~4,200 tokens
  Output tokens: 10 × 300 = 3,000 tokens
  
  Cost: (4,200 × $2.50/1M) + (3,000 × $10.00/1M) = $0.0105 + $0.03 = ~$0.04
  
  1,000 conversations/day = $40/day = ~$1,200/month

Same scenario with GPT-4o-mini:
  Cost: (4,200 × $0.15/1M) + (3,000 × $0.60/1M) = ~$0.002
  
  1,000 conversations/day = $2/day = ~$60/month

Same scenario with Claude 3.5 Haiku:
  Cost: (4,200 × $0.80/1M) + (3,000 × $4.00/1M) = ~$0.015
  
  1,000 conversations/day = $15/day = ~$450/month
```

**The 20x rule:** Switching from a frontier model to a mini model often saves 15-30x in cost. For many tasks, the quality difference is negligible.

### 2.3 Prompt Caching

Most providers now offer **prompt caching** — if your input starts with the same prefix as a recent request, the cached portion is charged at a discounted rate (typically 50-90% off input token costs).

This matters enormously for:
- **System prompts** — the same system prompt prefix is repeated every request
- **RAG** — the same retrieved document appears in multiple requests  
- **Multi-turn conversations** — the conversation history prefix is the same

```
Without caching:
  Request 1: [system_prompt + user_msg_1]          → 600 input tokens × full price
  Request 2: [system_prompt + user_msg_1 + resp_1 + user_msg_2] → 1,000 input tokens × full price
  Request 3: [system_prompt + ... + user_msg_3]    → 1,400 input tokens × full price

With caching:
  Request 1: [system_prompt + user_msg_1]          → 600 tokens × full price
  Request 2: [600 cached + 400 new]                → 600 × 0.1x + 400 × 1x
  Request 3: [1000 cached + 400 new]               → 1000 × 0.1x + 400 × 1x
```

Anthropic's prompt caching charges 90% less for cached input tokens. OpenAI's caching offers 50% off. Google provides automatic caching for free on qualifying requests.

### 2.4 Self-Hosted Costs

Running open-source models yourself has a different cost structure:

```
Self-hosting Llama 3.1 70B (example):
  Hardware: 2× NVIDIA A100 80GB GPUs
  Cloud cost: ~$6-8/hour on AWS (p4d instance)
  = ~$4,500-5,800/month running 24/7

  At this cost, you'd need to serve ~150,000+ conversations/month 
  to beat GPT-4o pricing. But you get:
  - No per-token costs (unlimited inference)
  - Full data privacy
  - Ability to fine-tune
  - No rate limits
```

Self-hosting makes sense when:
- You process very high volume (millions of requests/month)
- Data privacy is critical (healthcare, finance, government)
- You need fine-tuned models
- You need zero latency variability

Self-hosting does NOT make sense when:
- You're still figuring out your product
- Volume is low to moderate
- You don't have ML infrastructure experience
- You need the best quality (frontier closed models still lead)

### 2.5 Batch API Pricing

Most providers offer **batch APIs** — submit a batch of requests and get results hours later at 50% off. Useful for:

- Processing large datasets
- Running evaluations
- Nightly content generation
- Anything that doesn't need real-time responses

---

## 3. Rate Limits

### 3.1 What Rate Limits Are

Every API provider imposes limits on how many requests (or tokens) you can send per minute. These protect the provider's infrastructure and ensure fair access.

Rate limits are typically expressed as:
- **RPM**: Requests per minute
- **TPM**: Tokens per minute
- **RPD**: Requests per day

### 3.2 Typical Rate Limits

| Provider | Tier | RPM | TPM | How to Increase |
|---|---|---|---|---|
| OpenAI | Free | 3 | 40,000 | Add payment method |
| OpenAI | Tier 1 ($5 paid) | 500 | 200,000 | Spend more ($50+ → Tier 2) |
| OpenAI | Tier 5 ($1000+ paid) | 10,000 | 10,000,000 | Contact sales |
| Anthropic | Build (free) | 5 | 20,000 | Add payment |
| Anthropic | Build ($5+ credit) | 1,000 | 400,000 | Spend more |
| Anthropic | Scale | Custom | Custom | Contact sales |
| Google | Free | 15 | 1,000,000 | Enable billing |
| Google | Paid | 360 | 4,000,000 | Request increase |

**Note:** These are approximate and change frequently. Always check the provider's documentation.

### 3.3 How Rate Limits Affect Your Architecture

Rate limits aren't just an annoyance — they shape how you build:

**You need retry logic.** When you hit a rate limit, the API returns a 429 error. You need to catch it and retry after a delay. Exponential backoff is the standard pattern. (We'll implement this in Ch 3.)

**You need request queuing.** If 100 users hit your app simultaneously, you can't forward 100 API calls at once if your RPM is 60. You need a queue.

**You might need multiple providers.** If one provider's rate limits are too low, you can fall back to another. This is called **provider failover** and we cover it in Ch 54.

**Batch when possible.** Instead of 100 individual requests, can you batch them into fewer, larger requests? Or use the batch API?

**Cache aggressively.** If many users ask the same question, cache the response. Don't waste rate-limited API calls on duplicate requests.

### 3.4 Rate Limit Headers

Most providers return rate limit information in response headers:

```
x-ratelimit-limit-requests: 60
x-ratelimit-remaining-requests: 45
x-ratelimit-reset-requests: 2026-04-10T10:30:00Z
x-ratelimit-limit-tokens: 200000
x-ratelimit-remaining-tokens: 185000
```

Use these headers to implement proactive rate limiting — slow down *before* you hit the limit, rather than waiting for 429 errors.

---

## 4. SDKs: Your Tools

### 4.1 The Vercel AI SDK (Recommended for TypeScript)

If you're writing TypeScript (and in Phase 1 of this guide, you are), the **Vercel AI SDK** is the SDK to use. Here's why:

**Provider-agnostic.** Write your code once, switch providers by changing one line. Call OpenAI, Anthropic, Google, Mistral, or any other provider with the same API.

**Streaming-first.** Built for streaming responses to UIs. Includes React hooks (`useChat`, `useCompletion`) that handle streaming, loading states, and error handling.

**TypeScript-native.** Full type safety. Zod schema integration for structured output. Everything feels like idiomatic TypeScript.

**Full-featured.** Tool calling, structured output, image inputs, embedding generation, object streaming — it covers the full surface area of modern LLM APIs.

```typescript
// Same code, different providers — just swap the model
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";
import { google } from "@ai-sdk/google";

// OpenAI
const result1 = await generateText({
  model: openai("gpt-4o"),
  prompt: "Explain embeddings in one sentence.",
});

// Anthropic — same code, different model
const result2 = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  prompt: "Explain embeddings in one sentence.",
});

// Google — same code, different model
const result3 = await generateText({
  model: google("gemini-2.0-flash"),
  prompt: "Explain embeddings in one sentence.",
});
```

We'll use the Vercel AI SDK throughout this guide. You'll install it in Chapter 3.

### 4.2 The OpenAI SDK

The **official OpenAI SDK** (`openai` npm package) is the most widely used LLM SDK. Even if you use the Vercel AI SDK for your application code, you'll encounter OpenAI SDK examples everywhere.

```typescript
import OpenAI from "openai";

const client = new OpenAI();  // reads OPENAI_API_KEY from env

const response = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Explain embeddings in one sentence." },
  ],
});

console.log(response.choices[0].message.content);
```

**When to use the OpenAI SDK directly:**
- You're only using OpenAI models
- You need OpenAI-specific features (fine-tuning API, assistants API, batch API)
- You're following an OpenAI-specific tutorial
- You need the most direct control over the API

### 4.3 The Anthropic SDK

The **official Anthropic SDK** (`@anthropic-ai/sdk`) is clean and well-designed:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();  // reads ANTHROPIC_API_KEY from env

const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "Explain embeddings in one sentence." },
  ],
});

console.log(response.content[0].text);
```

**When to use the Anthropic SDK directly:**
- You're only using Claude models
- You need Anthropic-specific features (prompt caching control, extended thinking)
- You need the most direct control over the Claude API

### 4.4 SDK Comparison

| Feature | Vercel AI SDK | OpenAI SDK | Anthropic SDK |
|---|---|---|---|
| Provider support | All major providers | OpenAI only | Anthropic only |
| Streaming | First-class, with React hooks | Supported | Supported |
| Tool calling | Unified API across providers | OpenAI format | Anthropic format |
| Structured output | Zod integration | JSON schema | JSON schema |
| React integration | useChat, useCompletion | DIY | DIY |
| TypeScript types | Excellent | Good | Good |
| Embedding generation | Yes | Yes | No (Claude doesn't have embeddings) |

**The recommendation:** Use the Vercel AI SDK as your primary SDK. Use provider-specific SDKs when you need provider-specific features.

### 4.5 Installing the SDKs

Here's what you'll install in Chapter 3:

```bash
# Core SDK
npm install ai

# Provider adapters (install the ones you need)
npm install @ai-sdk/openai
npm install @ai-sdk/anthropic
npm install @ai-sdk/google

# Optional: provider-specific SDKs for direct access
npm install openai
npm install @anthropic-ai/sdk
```

---

## 5. The Build vs Buy Decision Tree

### 5.1 The Spectrum

When adding AI to your product, you're somewhere on this spectrum:

```
100% Buy ◄──────────────────────────────────────────────► 100% Build

  Use a     Use an    Use an API    Fine-tune   Train from
  SaaS AI   AI-powered  with your    an open-    scratch
  product   feature     own prompts  source model
  
  "Add      "Use       "Call        "Customize   "Build our
  ChatGPT   Vercel's   GPT-4o       Llama for    own LLM"
  widget"   AI search"  directly"   our domain"  
  
  ← Cheaper to start              More control →
  ← Less customizable             More customizable →
  ← Faster to ship                More expensive to maintain →
```

### 5.2 The Decision Tree

```
Q1: Do you need AI that knows YOUR specific data?
│
├─ NO → Is the task simple (classification, extraction, summarization)?
│       ├─ YES → API call with good prompts (Ch 3-5)
│       └─ NO  → API call + agent pattern (Ch 8-13)
│
└─ YES → How much proprietary data?
         │
         ├─ Moderate (thousands of documents) → RAG pipeline (Ch 14-17)
         │
         ├─ Large + specific domain vocabulary → RAG + fine-tuned embeddings
         │                                       (Ch 14-17 + Ch 44-45)
         │
         └─ Your data IS the product → Fine-tune a model (Ch 44-47)
              (you need the model to "speak your language" natively)
```

### 5.3 The Right Choice for Most Teams

For most teams building their first AI features, the right answer is:

**Start with API calls + good prompts. Add RAG if you need domain knowledge. Consider fine-tuning only if RAG isn't enough.**

Here's why:

| Approach | Time to Ship | Cost to Start | Maintenance | Quality Ceiling |
|---|---|---|---|---|
| API + prompts | Days | $0 (free tier) | Low | High for general tasks |
| API + RAG | 1-2 weeks | $100-500/month | Medium | High for knowledge tasks |
| Fine-tuning | 2-4 weeks | $500-5,000 | High | Higher for niche tasks |
| Train from scratch | 3-6 months | $100K-1M+ | Very high | Unlimited |

The overwhelming majority of AI features in production today use option 1 or 2. Fine-tuning is for when you've exhausted what prompting + retrieval can do (and you have the data to prove it with evals).

### 5.4 Red Flags for Each Approach

**API + Prompts won't work when:**
- You need the model to know thousands of facts not in its training data
- Response quality requires deep domain expertise
- You need consistent, specific formatting that prompts can't reliably produce

**RAG won't work when:**
- The relevant knowledge can't be retrieved with similarity search
- The task requires reasoning across many documents simultaneously
- The domain vocabulary is so specialized that embeddings don't capture it

**Fine-tuning won't work when:**
- You don't have high-quality training data (hundreds of examples minimum)
- The task changes frequently (fine-tuning is slow to update)
- You need the model to follow instructions it wasn't fine-tuned for

---

## 6. Model Selection: A Practical Framework

### 6.1 The Three Axes

Every model selection decision balances three things:

```
                Quality
                  ▲
                 / \
                /   \
               /     \
              /       \
             /   Pick  \
            /    two    \
           /             \
          ▼───────────────▼
        Cost            Speed
```

- **Quality**: How good are the outputs? How well does it follow instructions? How rare are errors?
- **Cost**: How much per request? How much at scale?
- **Speed**: How fast does the first token arrive? How fast is the full response?

You rarely get all three. Frontier models are high quality but expensive and sometimes slower. Small models are fast and cheap but lower quality. The art is finding the model that's "good enough" for your task at a price and speed you can afford.

### 6.2 Model Tiers

Think of models in tiers:

**Tier 1 — Frontier** ($10-75 per 1M output tokens)
- Claude Opus 4, o3, Gemini 2.5 Pro
- Best quality, highest cost
- Use for: Complex reasoning, difficult instructions, high-stakes decisions
- Don't use for: Simple classification, high-volume tasks

**Tier 2 — Strong** ($3-15 per 1M output tokens)
- Claude Sonnet 4, GPT-4o, Gemini 1.5 Pro
- Great quality, reasonable cost
- Use for: Most production features, general-purpose
- The "default choice" tier

**Tier 3 — Fast & Cheap** ($0.40-4.00 per 1M output tokens)
- Claude 3.5 Haiku, GPT-4o-mini, Gemini 2.0 Flash
- Good quality for straightforward tasks, very affordable
- Use for: Classification, extraction, simple generation, high volume
- Often surprisingly good — test before assuming you need Tier 2

**Tier 4 — Open Source** (infra costs only)
- Llama 4, Mistral Large, Qwen 2.5
- Self-hosted, no per-token cost
- Use for: High volume, data privacy, customization
- Requires ML ops expertise

### 6.3 The Right Model Selection Process

1. **Start with Tier 3.** Try GPT-4o-mini or Claude 3.5 Haiku. You'll be surprised how often they're good enough.

2. **Evaluate.** Run your eval suite (Ch 18-22) against the model. Is quality acceptable?

3. **Move up only if needed.** If Tier 3 fails your evals, try Tier 2. If Tier 2 fails, try Tier 1.

4. **Consider routing.** Use Tier 3 for simple requests and Tier 2 for complex ones. This is called **model routing** and we cover it in Ch 49.

5. **Re-evaluate regularly.** Models improve and prices drop. The model you chose 6 months ago might not be the best choice today.

### 6.4 Multi-Provider Strategy

Don't lock yourself into one provider. Here's why:

- **Outages happen.** OpenAI has had multi-hour outages. Anthropic has had them. Google has had them. If your app goes down when your provider goes down, you have a single point of failure.

- **Models have different strengths.** Claude might be better at following complex instructions. GPT-4o might be better at code. Gemini might be better at multimodal. Use the best model for each task.

- **Pricing changes.** Providers compete on price. If you can switch providers easily, you benefit from competition.

- **Negotiation leverage.** If you tell your OpenAI account manager "we're also evaluating Anthropic," you get better enterprise terms.

The Vercel AI SDK makes multi-provider easy — same code, swap the model string. We'll leverage this throughout the guide.

---

## 7. APIs vs Frameworks vs Platforms

### 7.1 The Stack

Understanding the layers helps you choose the right level of abstraction:

```
┌──────────────────────────────────────────────────┐
│              Your Application                      │
├──────────────────────────────────────────────────┤
│           Frameworks (optional)                    │
│    LangChain, Mastra, LlamaIndex, Haystack        │
├──────────────────────────────────────────────────┤
│              SDK Layer                             │
│    Vercel AI SDK, OpenAI SDK, Anthropic SDK        │
├──────────────────────────────────────────────────┤
│              Provider APIs                         │
│    OpenAI API, Anthropic API, Google AI API        │
├──────────────────────────────────────────────────┤
│              Models                                │
│    GPT-4o, Claude, Gemini, Llama                   │
└──────────────────────────────────────────────────┘
```

**Models** — the neural networks themselves. You interact with them through APIs.

**Provider APIs** — HTTP endpoints that accept prompts and return completions. Documented, versioned, rate-limited.

**SDKs** — TypeScript/Python libraries that wrap the HTTP APIs. Handle authentication, serialization, streaming, retries.

**Frameworks** — Higher-level abstractions. LangChain gives you chains, agents, vector store integrations. Mastra gives you workflows, agents, and integrations. LlamaIndex specializes in data ingestion and retrieval.

### 7.2 Our Recommendation

For this guide's Phase 1:

- **Always use an SDK.** Don't make raw HTTP calls. SDKs handle authentication, serialization, streaming, retries.
- **Use the Vercel AI SDK as your primary SDK.** Provider-agnostic, TypeScript-native, streaming-first.
- **Skip frameworks initially.** Learn what LangChain does by building the pieces yourself. Then you'll know when a framework helps vs hurts. (We cover frameworks in Ch 13.)

Why skip frameworks? Because:
1. Frameworks add abstraction that hides what's happening
2. When something breaks, you need to understand the layer below
3. Most AI features need fewer than 100 lines of code
4. Frameworks change rapidly — what you learn about `ai` SDK transfers more broadly

Once you've built a few features from scratch (Ch 3-13), you'll have the judgment to decide if a framework saves time or adds complexity.

---

## 8. The TypeScript AI Ecosystem

### 8.1 Essential Packages

Here's the TypeScript ecosystem you'll use throughout Phase 1:

```
Core:
  ai                    — Vercel AI SDK (main package)
  @ai-sdk/openai        — OpenAI provider adapter
  @ai-sdk/anthropic     — Anthropic provider adapter
  @ai-sdk/google        — Google provider adapter
  zod                   — Schema validation (used everywhere)

Utilities:
  tiktoken              — Token counting (OpenAI tokenizer)
  dotenv                — Environment variable loading
  nanoid                — ID generation

For chat UIs (Chapter 6):
  next                  — Next.js framework
  react                 — React
  @ai-sdk/react         — React hooks for AI (useChat, useCompletion)

For RAG (Chapters 14-17):
  @ai-sdk/openai        — Embedding generation
  @pinecone-database/pinecone — Vector store
  pgvector              — PostgreSQL vector extension
  pdf-parse             — PDF document loading

For evals (Chapters 18-22):
  vitest                — Test runner (for eval suites)
```

### 8.2 Project Structure

A typical AI feature project looks like:

```
my-ai-feature/
├── src/
│   ├── ai/
│   │   ├── prompts/          ← System prompts and templates
│   │   │   ├── support-agent.ts
│   │   │   └── data-extractor.ts
│   │   ├── tools/            ← Tool definitions
│   │   │   ├── search-docs.ts
│   │   │   └── lookup-order.ts
│   │   ├── schemas/          ← Zod schemas for structured output
│   │   │   ├── ticket-classification.ts
│   │   │   └── extracted-data.ts
│   │   └── client.ts         ← AI client setup
│   ├── api/                  ← API routes
│   │   └── chat/route.ts
│   └── app/                  ← UI (if applicable)
├── evals/                    ← Evaluation suites
│   ├── classification.eval.ts
│   └── extraction.eval.ts
├── .env.local                ← API keys (never commit!)
├── package.json
└── tsconfig.json
```

We'll build toward this structure starting in Chapter 3.

---

## 9. Security Basics: API Keys

### 9.1 The Golden Rules

Before you write a single line of code, internalize these:

1. **Never commit API keys to git.** Use `.env.local` files, add them to `.gitignore`.

2. **Never expose API keys in client-side code.** API calls go through YOUR server. The browser never sees the key.

3. **Use environment variables.** Both the Vercel AI SDK and provider SDKs read from `process.env` by default.

4. **Rotate keys if exposed.** If a key leaks (committed to public repo, logged, etc.), revoke it immediately and generate a new one.

5. **Use per-environment keys.** Different keys for development, staging, and production. If your dev key leaks, production isn't affected.

```
# .env.local (NEVER COMMIT THIS FILE)
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_GENERATIVE_AI_API_KEY=AIza...
```

```
# .gitignore
.env
.env.local
.env.*.local
```

### 9.2 Key Management for Teams

For team development:

- **Local dev**: Each developer gets their own API keys in `.env.local`
- **CI/CD**: Store keys in your CI platform's secrets (GitHub Actions secrets, Vercel environment variables)
- **Production**: Use your platform's secret management (Vercel Environment Variables, AWS Secrets Manager, etc.)
- **Never share keys over Slack/email.** Use a secrets manager or the platform's native secret sharing.

We'll set this up properly in Chapter 3.

---

## 10. What Changed Recently (2025-2026)

The AI landscape moves fast. Here's what's new as of early 2026 that you should know:

### 10.1 Reasoning Models

OpenAI's o-series (o1, o3, o4-mini) and Anthropic's extended thinking (Claude Opus 4, Sonnet 4) introduced **reasoning models** — models that "think" before they answer by generating a chain of thought. They're significantly better at:
- Complex math and logic
- Multi-step planning
- Code architecture decisions
- Ambiguous or nuanced problems

They're also significantly slower and more expensive. Use them when the task is genuinely hard, not for simple questions.

### 10.2 Extended Context

Gemini's 1M token context window and Anthropic's 200K tokens made "just put everything in context" a viable strategy for many tasks. But context quality still degrades in the middle, and cost scales linearly. RAG is still valuable even with huge context windows.

### 10.3 Structured Output

All major providers now support structured output natively — specifying a JSON schema and getting guaranteed-valid JSON back. This was unreliable just a year ago. It changes how you build: you can now use LLMs as reliable data extraction and transformation tools.

### 10.4 Tool Calling Standardization

Tool calling (function calling) is now standard across all major providers. The exact formats differ slightly, but the SDKs abstract the differences. This enables the agent patterns we'll build in Part 2.

### 10.5 Price Wars

Model prices have dropped dramatically. GPT-4-level quality that cost $30/1M output tokens in 2024 now costs $0.60/1M with mini models. This changes the economics — AI features that were too expensive for low-revenue users are now viable.

---

## 11. Key Takeaways

1. **Five providers matter:** OpenAI, Anthropic, Google, Meta (open source), Mistral. Know their strengths.

2. **Per-token pricing is the default.** Understand input vs output token costs. Mini models are 10-50x cheaper than frontier.

3. **Rate limits shape architecture.** Build retry logic, request queues, and consider multi-provider failover.

4. **Use the Vercel AI SDK.** Provider-agnostic, TypeScript-native, streaming-first. Provider SDKs for provider-specific needs.

5. **Start with API calls + prompts.** Add RAG for domain knowledge. Fine-tune only when RAG isn't enough.

6. **Start with the smallest sufficient model.** Tier 3 (mini) before Tier 2 (strong) before Tier 1 (frontier).

7. **Don't lock into one provider.** Use provider-agnostic SDKs and build multi-provider strategies.

8. **Protect your API keys.** Environment variables, never commit, never expose to clients.

---

## What's Next

That's the end of Part 0. You now have the mental model:
- How LLMs work (Ch 0)
- How embeddings capture meaning (Ch 1)
- Who the players are and what tools to use (Ch 2)

In **Part 1 — Building with LLM APIs**, we start writing code. Chapter 3 gets your project set up and makes your first API call. By Chapter 7, you'll have built a streaming chat interface with multimodal capabilities.

Let's go.

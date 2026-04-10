<!--
  CHAPTER: 49
  TITLE: Cost Engineering
  PART: 10 — Production AI Systems
  PHASE: 2 — Become an Expert
  PREREQS: Ch 0 (how LLMs work), Ch 22 (telemetry), Ch 29 (context internals)
  KEY_TOPICS: token economics, input vs output tokens, prompt caching, model routing, response caching, token optimization, usage budgets, cost monitoring
  DIFFICULTY: Inter-Adv
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 49: Cost Engineering

> **Part 10 — Production AI Systems** | Phase 2: Become an Expert | Prerequisites: Ch 0, Ch 22, Ch 29 | Difficulty: Inter-Adv | Language: TypeScript + Python

The demo worked. The prototype impressed leadership. Now you're running it in production and your monthly bill just hit $47,000. For a feature that saves your users about three clicks.

This is the most common failure mode for AI features in production: not that they don't work, but that they cost too much for the value they deliver. A single badly-constructed prompt that's called 100,000 times a day can burn through your AI budget faster than your entire compute infrastructure.

Cost engineering isn't about being cheap. It's about being *intentional*. Every token you send to an LLM should earn its place. Every expensive model call should be justified. And you should know, in real time, exactly what each feature costs.

### In This Chapter
- Token economics: the real cost structure of LLM calls
- Input vs output tokens: why the asymmetry matters
- Prompt caching: reusing common prefixes to cut costs dramatically
- Model routing: cheap models for easy tasks, expensive models for hard ones
- Response caching: when it's safe and when it's dangerous
- Token optimization: shorter prompts, fewer examples, structured output
- Usage budgets: per-user, per-feature, per-team limits
- Cost monitoring dashboards with real code
- Per-tenant cost management: tracking, budgets, and model routing for SaaS

### Related Chapters
- **Ch 0 (How LLMs Actually Work)** — spirals back: tokens cost money, context windows have limits
- **Ch 22 (Telemetry & Tracing)** — spirals back: token tracking feeds cost monitoring
- **Ch 29 (Context Window Internals)** — spirals back: compaction reduces token usage
- **Ch 54 (API Gateway & Provider Management)** — spirals forward: gateway-level cost controls
- **Ch 57 (Scaling & Cost at the Infra Level)** — spirals forward: infrastructure cost optimization

---

## 1. Token Economics

### 1.1 Understanding the Cost Structure

Every LLM API call has a cost determined by three factors:

1. **Input tokens** — the tokens you send (system prompt + conversation history + user message)
2. **Output tokens** — the tokens the model generates
3. **The model you're using** — different models have vastly different per-token prices

```
Cost per request = (input_tokens × input_price) + (output_tokens × output_price)
```

As of early 2026, here's the pricing landscape:

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Relative Cost |
|-------|----------------------|------------------------|---------------|
| Claude Haiku 3.5 | $0.80 | $4.00 | $ |
| GPT-4o-mini | $0.15 | $0.60 | $ |
| Claude Sonnet 4 | $3.00 | $15.00 | $$ |
| GPT-4o | $2.50 | $10.00 | $$ |
| Claude Opus 4 | $15.00 | $75.00 | $$$$ |
| GPT-4.5 | $75.00 | $150.00 | $$$$$ |

The spread is enormous. A single GPT-4.5 output token costs *250x* more than a GPT-4o-mini output token.

### 1.2 Why Output Tokens Cost More

Output tokens are 3-5x more expensive than input tokens across every provider. This isn't arbitrary pricing — it reflects the computational difference:

- **Input tokens** are processed in parallel. The model reads your entire prompt at once using matrix operations optimized for parallel hardware.
- **Output tokens** are generated sequentially. Each token requires a full forward pass through the model, and each pass depends on all previous tokens.

This asymmetry has a direct engineering implication: **you should spend your optimization effort on reducing output tokens, not input tokens.**

```
Cost comparison for a typical request:

Scenario: 2,000 input tokens, 500 output tokens, Claude Sonnet 4
  Input cost:  2,000 / 1M × $3.00  = $0.006
  Output cost: 500 / 1M × $15.00   = $0.0075
  Total: $0.0135

If you reduce INPUT by 50% (1,000 tokens):
  Input cost:  1,000 / 1M × $3.00  = $0.003  (saved $0.003)
  Output cost: 500 / 1M × $15.00   = $0.0075
  Total: $0.0105  (22% savings)

If you reduce OUTPUT by 50% (250 tokens):
  Input cost:  2,000 / 1M × $3.00  = $0.006
  Output cost: 250 / 1M × $15.00   = $0.00375  (saved $0.00375)
  Total: $0.00975  (28% savings)
```

### 1.3 The Compounding Cost of Conversations

In a multi-turn conversation, you send the *entire history* with each new request. Costs grow quadratically:

```
Turn 1: Send [system + msg1]                     = 500 tokens input
Turn 2: Send [system + msg1 + resp1 + msg2]       = 1,200 tokens input
Turn 3: Send [system + msg1 + resp1 + msg2 + resp2 + msg3] = 2,100 tokens input
...
Turn 20: Send [everything]                        = ~15,000 tokens input
```

```typescript
// TypeScript: Modeling conversation cost growth

interface ConversationCostModel {
  turn: number;
  inputTokens: number;
  outputTokens: number;
  turnCost: number;
  cumulativeCost: number;
}

function modelConversationCost(
  systemPromptTokens: number,
  avgUserMessageTokens: number,
  avgResponseTokens: number,
  turns: number,
  inputPricePerMillion: number,
  outputPricePerMillion: number,
): ConversationCostModel[] {
  const results: ConversationCostModel[] = [];
  let cumulativeCost = 0;

  for (let turn = 1; turn <= turns; turn++) {
    // Input = system prompt + all previous messages + all previous responses + current message
    const inputTokens =
      systemPromptTokens +
      turn * avgUserMessageTokens +
      (turn - 1) * avgResponseTokens;

    const outputTokens = avgResponseTokens;

    const turnCost =
      (inputTokens / 1_000_000) * inputPricePerMillion +
      (outputTokens / 1_000_000) * outputPricePerMillion;

    cumulativeCost += turnCost;

    results.push({
      turn,
      inputTokens,
      outputTokens,
      turnCost,
      cumulativeCost,
    });
  }

  return results;
}

// Example: 20-turn conversation with Claude Sonnet 4
const costs = modelConversationCost(
  500,   // system prompt
  100,   // avg user message
  300,   // avg response
  20,    // turns
  3.0,   // input price per million
  15.0,  // output price per million
);

console.log("Turn | Input Tokens | Turn Cost  | Total Cost");
for (const c of costs) {
  console.log(
    `${c.turn.toString().padStart(4)} | ${c.inputTokens.toString().padStart(12)} | $${c.turnCost.toFixed(4).padStart(8)} | $${c.cumulativeCost.toFixed(4).padStart(9)}`
  );
}

// Turn 1:    600 input,  $0.0063  total: $0.0063
// Turn 10:  4,300 input,  $0.0179  total: $0.1030
// Turn 20:  8,600 input,  $0.0308  total: $0.3478
```

> **Spiral note:** In Ch 29 (Context Window Internals), you learned how Claude Code's compaction service automatically summarizes conversation history when it gets too long. That's a cost engineering technique disguised as a context management technique — compaction reduces the tokens sent on every subsequent turn, saving money on long conversations.

---

## 2. Prompt Caching

### 2.1 How Prompt Caching Works

Prompt caching lets you reuse the processing of common prompt prefixes across requests. If your system prompt and few-shot examples are the same across thousands of requests, you don't need to pay full price for them every time.

```
Without caching:
  Request 1: [System Prompt (2000 tokens)] + [User Message A] → Full price for all
  Request 2: [System Prompt (2000 tokens)] + [User Message B] → Full price for all
  Request 3: [System Prompt (2000 tokens)] + [User Message C] → Full price for all
  
  Each request pays full price for 2000 system prompt tokens.

With caching:
  Request 1: [System Prompt (2000 tokens)] + [User Message A] → Full price (cache write)
  Request 2: [Cache hit: 2000 tokens] + [User Message B]       → 90% discount on prefix
  Request 3: [Cache hit: 2000 tokens] + [User Message C]       → 90% discount on prefix
  
  Requests 2+ pay only 10% of the input price for the cached prefix.
```

### 2.2 Anthropic Prompt Caching

```typescript
// TypeScript: Prompt caching with Anthropic API

import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// The system prompt is marked for caching with cache_control
const CACHED_SYSTEM = [
  {
    type: "text" as const,
    text: `You are a customer support agent for Acme Corp.
    
[... 2000+ tokens of instructions, policies, FAQ content ...]

Always be helpful, accurate, and refer to the policy documents above.`,
    cache_control: { type: "ephemeral" as const },  // Enable caching
  },
];

async function handleCustomerQuery(userMessage: string) {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: CACHED_SYSTEM,
    messages: [{ role: "user", content: userMessage }],
  });

  // Check cache usage in the response
  const usage = response.usage;
  console.log(`Input tokens: ${usage.input_tokens}`);
  console.log(`Cache creation tokens: ${usage.cache_creation_input_tokens}`);
  console.log(`Cache read tokens: ${usage.cache_read_input_tokens}`);

  // First request: cache_creation_input_tokens = 2000, cache_read = 0
  // Subsequent: cache_creation = 0, cache_read = 2000 (at 10% price)

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

```python
# Python: Prompt caching with Anthropic API

import anthropic

client = anthropic.Anthropic()

CACHED_SYSTEM = [
    {
        "type": "text",
        "text": """You are a customer support agent for Acme Corp.

[... 2000+ tokens of instructions, policies, FAQ content ...]

Always be helpful, accurate, and refer to the policy documents above.""",
        "cache_control": {"type": "ephemeral"},
    }
]

def handle_customer_query(user_message: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=CACHED_SYSTEM,
        messages=[{"role": "user", "content": user_message}],
    )

    usage = response.usage
    print(f"Input tokens: {usage.input_tokens}")
    print(f"Cache write: {usage.cache_creation_input_tokens}")
    print(f"Cache read: {usage.cache_read_input_tokens}")

    return response.content[0].text
```

### 2.3 Caching Economics

Prompt caching changes the cost math dramatically for workloads with stable prefixes:

```
Without caching (Claude Sonnet 4):
  System prompt: 2,000 tokens × $3.00/M = $0.006 per request
  At 100,000 requests/day: $600/day = $18,000/month

With caching (90% discount on cache hits):
  First request: $0.006 (cache write, slight premium)
  Subsequent: 2,000 tokens × $0.30/M = $0.0006 per request
  At 100,000 requests/day: ~$60/day = ~$1,800/month

Savings: ~$16,200/month (90% reduction on system prompt costs)
```

**When prompt caching works best:**
- Large system prompts (1000+ tokens)
- Many few-shot examples that don't change per-request
- RAG-augmented prompts with stable base context
- High-volume endpoints (hundreds+ requests per hour)

**When it doesn't help:**
- Every request has a unique prefix
- Small system prompts (the savings are negligible)
- Low-volume features (the cache expires before it's reused)

---

## 3. Model Routing

### 3.1 The Insight: Not Every Task Needs Your Best Model

The single biggest cost optimization in AI engineering is using the right model for each task. Most production workloads have a mix of easy and hard tasks:

- **Easy**: "What are your store hours?" → Haiku can handle this perfectly
- **Medium**: "Compare the warranty coverage of Product A vs Product B" → Sonnet is fine
- **Hard**: "Analyze this contract clause and identify potential liability issues" → You might need Opus

If you send everything to Opus, you're spending $75/M output tokens on questions that Haiku could answer for $4/M — an 18x waste.

### 3.2 Routing Strategies

**Strategy 1: Task-based routing (simplest)**

Route based on the feature or endpoint, not the content. Different features use different models.

```typescript
// TypeScript: Task-based model routing

type Feature = "faq" | "product_search" | "contract_analysis" | "code_review";

const MODEL_ROUTING: Record<Feature, string> = {
  faq: "claude-haiku-4-20250414",             // Cheap, fast — FAQ answers
  product_search: "claude-haiku-4-20250414",  // Simple retrieval + response
  contract_analysis: "claude-sonnet-4-20250514",  // Needs reasoning
  code_review: "claude-sonnet-4-20250514",        // Needs code understanding
};

function getModelForFeature(feature: Feature): string {
  return MODEL_ROUTING[feature];
}
```

**Strategy 2: Complexity-based routing (smarter)**

Use a cheap model to assess complexity, then route to the appropriate model.

```typescript
// TypeScript: Complexity-based model routing

import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface RoutingDecision {
  model: string;
  reasoning: string;
  estimatedComplexity: "low" | "medium" | "high";
}

async function routeByComplexity(userMessage: string): Promise<RoutingDecision> {
  // Step 1: Use Haiku to classify complexity (costs ~$0.0001)
  const classification = await client.messages.create({
    model: "claude-haiku-4-20250414",
    max_tokens: 100,
    system: `Classify the complexity of this user request.
- "low": Simple factual questions, greetings, FAQ-type queries
- "medium": Questions requiring reasoning, comparison, or multi-step analysis
- "high": Complex analysis, creative tasks, nuanced judgment calls

Respond with JSON: {"complexity": "low|medium|high", "reasoning": "brief explanation"}`,
    messages: [{ role: "user", content: userMessage }],
  });

  const text = classification.content[0].type === "text"
    ? classification.content[0].text
    : '{"complexity": "medium"}';
  const result = JSON.parse(text);

  // Step 2: Route based on complexity
  const modelMap: Record<string, string> = {
    low: "claude-haiku-4-20250414",
    medium: "claude-sonnet-4-20250514",
    high: "claude-opus-4-20250514",
  };

  return {
    model: modelMap[result.complexity] || "claude-sonnet-4-20250514",
    reasoning: result.reasoning,
    estimatedComplexity: result.complexity,
  };
}

async function handleWithRouting(userMessage: string): Promise<string> {
  const routing = await routeByComplexity(userMessage);
  
  console.log(`Routing to ${routing.model} (complexity: ${routing.estimatedComplexity})`);
  console.log(`Reason: ${routing.reasoning}`);

  const response = await client.messages.create({
    model: routing.model,
    max_tokens: 1024,
    system: SYSTEM_PROMPT,
    messages: [{ role: "user", content: userMessage }],
  });

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

```python
# Python: Complexity-based model routing

import anthropic
import json

client = anthropic.Anthropic()

MODEL_MAP = {
    "low": "claude-haiku-4-20250414",
    "medium": "claude-sonnet-4-20250514",
    "high": "claude-opus-4-20250514",
}

def route_by_complexity(user_message: str) -> dict:
    """Use Haiku to classify, then route to the right model."""
    classification = client.messages.create(
        model="claude-haiku-4-20250414",
        max_tokens=100,
        system="""Classify the complexity of this user request.
- "low": Simple factual questions, greetings, FAQ-type queries
- "medium": Reasoning, comparison, multi-step analysis
- "high": Complex analysis, creative tasks, nuanced judgment

Respond with JSON: {"complexity": "low|medium|high", "reasoning": "..."}""",
        messages=[{"role": "user", "content": user_message}],
    )

    result = json.loads(classification.content[0].text)
    return {
        "model": MODEL_MAP.get(result["complexity"], "claude-sonnet-4-20250514"),
        "complexity": result["complexity"],
        "reasoning": result["reasoning"],
    }
```

**Strategy 3: Cascade routing (most cost-effective)**

Start with the cheapest model. If it can't answer well, escalate.

```typescript
// TypeScript: Cascade routing — start cheap, escalate if needed

async function cascadeRoute(userMessage: string): Promise<{
  response: string;
  model: string;
  attempts: number;
}> {
  const models = [
    "claude-haiku-4-20250414",    // Try cheapest first
    "claude-sonnet-4-20250514",   // Escalate if Haiku struggles
  ];

  for (let i = 0; i < models.length; i++) {
    const model = models[i];
    
    const response = await client.messages.create({
      model,
      max_tokens: 1024,
      system: `${SYSTEM_PROMPT}

IMPORTANT: If you are not confident in your answer (less than 80% sure), 
respond with exactly: {"needs_escalation": true, "reason": "why you're unsure"}`,
      messages: [{ role: "user", content: userMessage }],
    });

    const text = response.content[0].type === "text" ? response.content[0].text : "";

    // Check if the model wants to escalate
    try {
      const parsed = JSON.parse(text);
      if (parsed.needs_escalation && i < models.length - 1) {
        console.log(`${model} escalating: ${parsed.reason}`);
        continue;  // Try next model
      }
    } catch {
      // Not JSON — it's a real response
    }

    return { response: text, model, attempts: i + 1 };
  }

  // Shouldn't reach here, but fallback
  return { response: "I'm unable to help with that.", model: "fallback", attempts: models.length };
}
```

### 3.3 Routing Economics

```
Scenario: 100,000 requests/day, average 300 output tokens per response

Without routing (all Sonnet):
  100,000 × 300 tokens × $15/M = $450/day = $13,500/month

With complexity routing (60% low, 30% medium, 10% high):
  60,000 × 300 × $4/M   = $72/day   (Haiku)
  30,000 × 300 × $15/M  = $135/day  (Sonnet)
  10,000 × 300 × $75/M  = $225/day  (Opus)
  + routing overhead:      ~$6/day   (Haiku classification)
  Total: $438/day = $13,140/month

  Savings: ~$360/month (modest — the 10% Opus calls dominate)

With task-based routing (FAQ=Haiku, Analysis=Sonnet):
  70,000 × 300 × $4/M   = $84/day   (Haiku for FAQ)
  30,000 × 300 × $15/M  = $135/day  (Sonnet for analysis)
  Total: $219/day = $6,570/month

  Savings: $6,930/month (51% reduction)
```

The lesson: task-based routing often saves more than complexity routing because it avoids the Opus tier entirely for features that don't need it.

---

## 4. Response Caching

### 4.1 When Caching Is Safe

Some LLM responses are deterministic enough to cache. If 1,000 users ask "What are your store hours?" there's no reason to call the LLM 1,000 times.

**Safe to cache:**
- Factual questions about your product/service
- FAQ responses
- Document summarizations (same document = same summary)
- Classification results (same input = same class)
- Tool call decisions for identical inputs

**Dangerous to cache:**
- Personalized responses (different users should get different answers)
- Responses that depend on real-time data
- Creative/generative tasks where variety matters
- Responses to conversations (context differs even if the last message is identical)

### 4.2 Implementing Response Caching

```typescript
// TypeScript: LLM response caching with Redis

import { createClient } from "redis";
import crypto from "crypto";

const redis = createClient({ url: process.env.REDIS_URL });

interface CacheConfig {
  enabled: boolean;
  ttlSeconds: number;
  maxCachedResponses: number;
}

const CACHE_CONFIG: Record<string, CacheConfig> = {
  faq: { enabled: true, ttlSeconds: 3600, maxCachedResponses: 10000 },
  product_info: { enabled: true, ttlSeconds: 1800, maxCachedResponses: 5000 },
  personalized: { enabled: false, ttlSeconds: 0, maxCachedResponses: 0 },
  creative: { enabled: false, ttlSeconds: 0, maxCachedResponses: 0 },
};

function buildCacheKey(
  feature: string,
  systemPrompt: string,
  userMessage: string,
  model: string,
): string {
  const content = `${feature}:${model}:${systemPrompt}:${userMessage}`;
  return `llm:cache:${crypto.createHash("sha256").update(content).digest("hex")}`;
}

async function cachedLLMCall(
  feature: string,
  systemPrompt: string,
  userMessage: string,
  model: string,
): Promise<{ response: string; cached: boolean; cost: number }> {
  const config = CACHE_CONFIG[feature];

  // Check cache first
  if (config?.enabled) {
    const cacheKey = buildCacheKey(feature, systemPrompt, userMessage, model);
    const cached = await redis.get(cacheKey);
    if (cached) {
      return { response: cached, cached: true, cost: 0 };
    }
  }

  // Cache miss — call the LLM
  const response = await client.messages.create({
    model,
    max_tokens: 1024,
    system: systemPrompt,
    messages: [{ role: "user", content: userMessage }],
  });

  const text = response.content[0].type === "text" ? response.content[0].text : "";
  const cost = calculateCost(response.usage, model);

  // Store in cache
  if (config?.enabled) {
    const cacheKey = buildCacheKey(feature, systemPrompt, userMessage, model);
    await redis.set(cacheKey, text, { EX: config.ttlSeconds });
  }

  return { response: text, cached: false, cost };
}
```

```python
# Python: LLM response caching with Redis

import hashlib
import json
import redis

cache = redis.Redis.from_url(os.environ["REDIS_URL"])

CACHE_CONFIG = {
    "faq": {"enabled": True, "ttl": 3600},
    "product_info": {"enabled": True, "ttl": 1800},
    "personalized": {"enabled": False, "ttl": 0},
}

def build_cache_key(feature: str, system_prompt: str, user_message: str, model: str) -> str:
    content = f"{feature}:{model}:{system_prompt}:{user_message}"
    return f"llm:cache:{hashlib.sha256(content.encode()).hexdigest()}"

def cached_llm_call(
    feature: str, system_prompt: str, user_message: str, model: str
) -> dict:
    config = CACHE_CONFIG.get(feature, {"enabled": False})

    # Check cache
    if config["enabled"]:
        key = build_cache_key(feature, system_prompt, user_message, model)
        cached = cache.get(key)
        if cached:
            return {"response": cached.decode(), "cached": True, "cost": 0.0}

    # Call LLM
    response = client.messages.create(
        model=model,
        max_tokens=1024,
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}],
    )

    text = response.content[0].text
    cost = calculate_cost(response.usage, model)

    # Store in cache
    if config["enabled"]:
        key = build_cache_key(feature, system_prompt, user_message, model)
        cache.set(key, text, ex=config["ttl"])

    return {"response": text, "cached": False, "cost": cost}
```

### 4.3 Semantic Caching

Exact-match caching misses near-duplicates. "What are your hours?" and "When are you open?" should hit the same cache. Semantic caching uses embeddings to find similar queries.

```typescript
// TypeScript: Semantic caching with embeddings

import { cosineSimilarity } from "./utils";

interface SemanticCacheEntry {
  embedding: number[];
  query: string;
  response: string;
  createdAt: number;
}

class SemanticCache {
  private entries: SemanticCacheEntry[] = [];
  private similarityThreshold = 0.95;  // Very high — only cache near-exact matches
  private maxEntries = 10000;

  async get(
    query: string,
    queryEmbedding: number[],
  ): Promise<string | null> {
    let bestSimilarity = 0;
    let bestResponse: string | null = null;

    for (const entry of this.entries) {
      const similarity = cosineSimilarity(queryEmbedding, entry.embedding);
      if (similarity > bestSimilarity && similarity >= this.similarityThreshold) {
        bestSimilarity = similarity;
        bestResponse = entry.response;
      }
    }

    return bestResponse;
  }

  async set(query: string, queryEmbedding: number[], response: string): Promise<void> {
    if (this.entries.length >= this.maxEntries) {
      // Evict oldest entry
      this.entries.shift();
    }

    this.entries.push({
      embedding: queryEmbedding,
      query,
      response,
      createdAt: Date.now(),
    });
  }
}
```

**Warning:** Set the similarity threshold high (0.95+). At lower thresholds, you'll serve cached responses for questions that are semantically similar but have important differences. "How do I return a laptop?" and "How do I return a phone?" might be 0.92 similar but require different answers.

---

## 5. Token Optimization

### 5.1 Shorter Prompts

Every word in your system prompt is a token you pay for on every request. Audit your prompts for verbosity.

```
BEFORE (287 tokens):
  "You are a helpful and friendly customer support assistant working for 
  Acme Corporation. Your primary goal is to help customers with their 
  questions about our products, services, and policies. You should always 
  be polite, professional, and try to resolve their issues as quickly as 
  possible. If you don't know the answer to something, please let the 
  customer know that you'll escalate their question to a human agent who 
  can help them further. Please format your responses in a clear and 
  organized manner..."

AFTER (89 tokens):
  "Acme Corp support agent. Help with products, services, policies.
  Be concise and professional. If unsure, escalate to human agent.
  Use clear formatting."
```

The shortened version saves 198 tokens per request. At 100,000 requests/day with Sonnet ($3/M input tokens), that's:
- 198 × 100,000 = 19.8M tokens/day saved
- 19.8M × $3/M = $59.40/day = **$1,782/month saved** just by tightening one prompt.

### 5.2 Fewer Few-Shot Examples

Few-shot examples are expensive. Each example might be 100-300 tokens. Five examples = 500-1,500 tokens repeated on every call.

```typescript
// Strategy: Use few-shot only when needed, cache the decision

async function buildPrompt(
  task: string,
  userMessage: string,
): Promise<{ prompt: string; fewShotIncluded: boolean }> {
  // For well-understood task types, skip few-shot examples
  const simpleTasks = ["greeting", "faq", "order_status"];
  
  if (simpleTasks.includes(task)) {
    return {
      prompt: `Task: ${task}\nQuery: ${userMessage}`,
      fewShotIncluded: false,
    };
  }

  // For complex tasks, include minimal few-shot examples
  const examples = getFewShotExamples(task, 2);  // Max 2, not 5
  const exampleText = examples
    .map((ex) => `User: ${ex.input}\nAssistant: ${ex.output}`)
    .join("\n\n");

  return {
    prompt: `Examples:\n${exampleText}\n\nNow handle:\nUser: ${userMessage}`,
    fewShotIncluded: true,
  };
}
```

### 5.3 Structured Output to Reduce Verbosity

When you ask for JSON output instead of natural language, responses are typically 30-50% shorter. Structured output is both cheaper and easier to parse.

```typescript
// Natural language response (avg ~150 tokens):
// "Based on the order information, I can see that your order ORD-12345678 
//  was shipped on March 15th and is currently in transit. The estimated 
//  delivery date is March 20th. The tracking number is 1Z999AA10123456784.
//  You can track your package using this number on the UPS website."

// Structured response (avg ~50 tokens):
// {"status": "in_transit", "shipped": "2026-03-15", "eta": "2026-03-20", 
//  "tracking": "1Z999AA10123456784", "carrier": "UPS"}

// 66% token reduction for the same information
```

### 5.4 Max Token Limits

Always set `max_tokens` to the minimum needed. An unbounded response can generate thousands of unnecessary tokens.

```typescript
// TypeScript: Feature-appropriate max_tokens

const MAX_TOKENS: Record<string, number> = {
  classification: 50,     // Just a label and confidence
  faq_answer: 300,        // Short, focused answer
  summary: 500,           // Paragraph-length summary
  analysis: 1000,         // Detailed analysis
  code_generation: 2000,  // Code can be longer
  free_conversation: 1024, // General chat
};

function getMaxTokens(feature: string): number {
  return MAX_TOKENS[feature] || 512;
}
```

---

## 6. Usage Budgets

### 6.1 The Ramp Philosophy

Ramp, the corporate card company, has been one of the most aggressive AI adopters in tech. Their philosophy on AI costs:

> "Token consumption per employee isn't even close to double-digit percentages of their salary."

This is the right framing. If an engineer costs $200K/year and uses $500/month in AI tokens, that's 3% of their compensation. If those tokens make them 20% more productive, the ROI is absurd. The risk isn't that AI costs too much — it's that you *don't track* costs and get surprised.

### 6.2 Budget Architecture

```typescript
// TypeScript: Per-user, per-feature, per-team usage budgets

interface UsageBudget {
  maxTokensPerDay: number;
  maxTokensPerMonth: number;
  maxCostPerDay: number;
  maxCostPerMonth: number;
  alertThresholdPercent: number;
}

const BUDGETS: Record<string, UsageBudget> = {
  free_user: {
    maxTokensPerDay: 50_000,
    maxTokensPerMonth: 500_000,
    maxCostPerDay: 0.50,
    maxCostPerMonth: 5.00,
    alertThresholdPercent: 80,
  },
  pro_user: {
    maxTokensPerDay: 500_000,
    maxTokensPerMonth: 10_000_000,
    maxCostPerDay: 5.00,
    maxCostPerMonth: 50.00,
    alertThresholdPercent: 80,
  },
  team: {
    maxTokensPerDay: 5_000_000,
    maxTokensPerMonth: 100_000_000,
    maxCostPerDay: 50.00,
    maxCostPerMonth: 500.00,
    alertThresholdPercent: 70,
  },
  feature_code_review: {
    maxTokensPerDay: 2_000_000,
    maxTokensPerMonth: 50_000_000,
    maxCostPerDay: 20.00,
    maxCostPerMonth: 200.00,
    alertThresholdPercent: 75,
  },
};

interface UsageTracker {
  entity: string;  // user_id, team_id, or feature name
  currentTokensToday: number;
  currentTokensMonth: number;
  currentCostToday: number;
  currentCostMonth: number;
}

async function checkBudget(
  entityId: string,
  budgetType: string,
  estimatedTokens: number,
  estimatedCost: number,
): Promise<{ allowed: boolean; reason?: string; warningMessage?: string }> {
  const budget = BUDGETS[budgetType];
  if (!budget) return { allowed: true };

  const usage = await getUsageTracker(entityId);

  // Check hard limits
  if (usage.currentTokensToday + estimatedTokens > budget.maxTokensPerDay) {
    return { allowed: false, reason: "daily_token_limit" };
  }
  if (usage.currentCostMonth + estimatedCost > budget.maxCostPerMonth) {
    return { allowed: false, reason: "monthly_cost_limit" };
  }

  // Check alert thresholds
  const monthlyPercent = (usage.currentCostMonth / budget.maxCostPerMonth) * 100;
  if (monthlyPercent >= budget.alertThresholdPercent) {
    return {
      allowed: true,
      warningMessage: `You've used ${monthlyPercent.toFixed(0)}% of your monthly AI budget.`,
    };
  }

  return { allowed: true };
}

async function trackUsage(
  entityId: string,
  tokens: number,
  cost: number,
  feature: string,
): Promise<void> {
  // Update user/team usage
  await incrementUsage(entityId, tokens, cost);

  // Update feature usage
  await incrementUsage(`feature:${feature}`, tokens, cost);

  // Emit metrics for monitoring
  emitMetric("ai.tokens.used", tokens, { entity: entityId, feature });
  emitMetric("ai.cost.incurred", cost, { entity: entityId, feature });
}
```

```python
# Python: Usage budget checking

from dataclasses import dataclass
from typing import Optional

@dataclass
class UsageBudget:
    max_tokens_per_day: int
    max_tokens_per_month: int
    max_cost_per_day: float
    max_cost_per_month: float
    alert_threshold_percent: float

BUDGETS = {
    "free_user": UsageBudget(50_000, 500_000, 0.50, 5.00, 80),
    "pro_user": UsageBudget(500_000, 10_000_000, 5.00, 50.00, 80),
    "team": UsageBudget(5_000_000, 100_000_000, 50.00, 500.00, 70),
}

async def check_budget(
    entity_id: str,
    budget_type: str,
    estimated_tokens: int,
    estimated_cost: float,
) -> dict:
    budget = BUDGETS.get(budget_type)
    if not budget:
        return {"allowed": True}

    usage = await get_usage_tracker(entity_id)

    if usage["tokens_today"] + estimated_tokens > budget.max_tokens_per_day:
        return {"allowed": False, "reason": "daily_token_limit"}

    if usage["cost_month"] + estimated_cost > budget.max_cost_per_month:
        return {"allowed": False, "reason": "monthly_cost_limit"}

    pct = (usage["cost_month"] / budget.max_cost_per_month) * 100
    if pct >= budget.alert_threshold_percent:
        return {
            "allowed": True,
            "warning": f"You've used {pct:.0f}% of your monthly AI budget.",
        }

    return {"allowed": True}
```

---

## 7. Cost Monitoring Dashboard

### 7.1 What to Track

Every production AI system needs a cost dashboard that answers these questions in real time:

1. **How much are we spending?** Total cost by day, week, month.
2. **Where is the money going?** Cost breakdown by feature, model, team.
3. **Is this normal?** Anomaly detection for unexpected cost spikes.
4. **What's the trend?** Are costs growing linearly (good) or exponentially (bad)?
5. **What's the unit economics?** Cost per user action, cost per feature invocation.

### 7.2 Building the Dashboard

```typescript
// TypeScript: Cost tracking and aggregation service

interface CostEvent {
  timestamp: Date;
  feature: string;
  model: string;
  userId: string;
  teamId: string;
  inputTokens: number;
  outputTokens: number;
  cachedTokens: number;
  cost: number;
  latencyMs: number;
  cached: boolean;
}

// Pricing table (update when providers change prices)
const PRICING: Record<string, { input: number; output: number; cachedInput: number }> = {
  "claude-haiku-4-20250414": { input: 0.80, output: 4.00, cachedInput: 0.08 },
  "claude-sonnet-4-20250514": { input: 3.00, output: 15.00, cachedInput: 0.30 },
  "claude-opus-4-20250514": { input: 15.00, output: 75.00, cachedInput: 1.50 },
  "gpt-4o-mini": { input: 0.15, output: 0.60, cachedInput: 0.075 },
  "gpt-4o": { input: 2.50, output: 10.00, cachedInput: 1.25 },
};

function calculateCost(
  model: string,
  inputTokens: number,
  outputTokens: number,
  cachedInputTokens: number = 0,
): number {
  const prices = PRICING[model];
  if (!prices) throw new Error(`Unknown model: ${model}`);

  const nonCachedInput = inputTokens - cachedInputTokens;
  return (
    (nonCachedInput / 1_000_000) * prices.input +
    (cachedInputTokens / 1_000_000) * prices.cachedInput +
    (outputTokens / 1_000_000) * prices.output
  );
}

// After each LLM call, emit a cost event
function emitCostEvent(
  response: any, // LLM response object
  model: string,
  feature: string,
  userId: string,
  teamId: string,
  latencyMs: number,
): CostEvent {
  const event: CostEvent = {
    timestamp: new Date(),
    feature,
    model,
    userId,
    teamId,
    inputTokens: response.usage.input_tokens,
    outputTokens: response.usage.output_tokens,
    cachedTokens: response.usage.cache_read_input_tokens || 0,
    cost: calculateCost(
      model,
      response.usage.input_tokens,
      response.usage.output_tokens,
      response.usage.cache_read_input_tokens || 0,
    ),
    latencyMs,
    cached: false,
  };

  // Send to your analytics pipeline (Datadog, BigQuery, etc.)
  trackEvent("ai.cost", event);
  return event;
}
```

### 7.3 Anomaly Detection

```typescript
// TypeScript: Simple anomaly detection for cost spikes

interface AnomalyConfig {
  lookbackDays: number;
  stdDevThreshold: number;
}

async function detectCostAnomaly(
  feature: string,
  currentDayCost: number,
  config: AnomalyConfig = { lookbackDays: 14, stdDevThreshold: 2.5 },
): Promise<{ anomaly: boolean; message: string }> {
  // Get historical daily costs
  const historicalCosts = await getDailyCosts(feature, config.lookbackDays);

  if (historicalCosts.length < 7) {
    return { anomaly: false, message: "Insufficient data for anomaly detection" };
  }

  // Calculate mean and standard deviation
  const mean = historicalCosts.reduce((a, b) => a + b, 0) / historicalCosts.length;
  const variance =
    historicalCosts.reduce((sum, cost) => sum + Math.pow(cost - mean, 2), 0) /
    historicalCosts.length;
  const stdDev = Math.sqrt(variance);

  // Check if current day is anomalous
  const zScore = (currentDayCost - mean) / (stdDev || 1);

  if (zScore > config.stdDevThreshold) {
    return {
      anomaly: true,
      message: `Cost anomaly detected for "${feature}": $${currentDayCost.toFixed(2)} ` +
        `(mean: $${mean.toFixed(2)}, ${zScore.toFixed(1)} standard deviations above normal)`,
    };
  }

  return { anomaly: false, message: "Within normal range" };
}
```

```python
# Python: Cost anomaly detection

import numpy as np
from typing import NamedTuple

class AnomalyResult(NamedTuple):
    anomaly: bool
    message: str
    z_score: float

def detect_cost_anomaly(
    feature: str,
    current_day_cost: float,
    historical_costs: list[float],
    std_dev_threshold: float = 2.5,
) -> AnomalyResult:
    if len(historical_costs) < 7:
        return AnomalyResult(False, "Insufficient data", 0.0)

    mean = np.mean(historical_costs)
    std_dev = np.std(historical_costs)
    z_score = (current_day_cost - mean) / (std_dev if std_dev > 0 else 1)

    if z_score > std_dev_threshold:
        return AnomalyResult(
            anomaly=True,
            message=(
                f"Cost anomaly for '{feature}': ${current_day_cost:.2f} "
                f"(mean: ${mean:.2f}, {z_score:.1f} std devs above normal)"
            ),
            z_score=z_score,
        )

    return AnomalyResult(False, "Within normal range", z_score)
```

### 7.4 The Cost Dashboard View

Here's what your monitoring dashboard should show:

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Cost Dashboard                         │
├──────────────┬──────────────┬──────────────┬───────────────┤
│ Today        │ This Week    │ This Month   │ Month Proj.   │
│ $127.43      │ $782.10      │ $2,847.30    │ $4,271.00     │
├──────────────┴──────────────┴──────────────┴───────────────┤
│                                                             │
│ Cost by Feature (Today)          │ Cost by Model (Today)    │
│ ─────────────────────            │ ──────────────────       │
│ code_review      $52.10  (41%)   │ Sonnet   $89.20  (70%)  │
│ customer_support $31.80  (25%)   │ Haiku    $28.40  (22%)  │
│ doc_search       $23.50  (18%)   │ Opus     $9.83   (8%)   │
│ summarization    $12.30  (10%)   │                          │
│ other            $7.73   (6%)    │                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ Alerts                                                      │
│ ⚠ code_review cost 2.3x above 14-day average               │
│ ⚠ team-backend at 78% of monthly budget                     │
│ ✓ All other features within normal range                    │
├─────────────────────────────────────────────────────────────┤
│ Unit Economics                                              │
│ ──────────────                                              │
│ Cost per code review:       $0.052                          │
│ Cost per support ticket:    $0.032                          │
│ Cost per document search:   $0.008                          │
│ Cache hit rate:             67%                             │
│ Routing efficiency:         71% handled by Haiku            │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Putting It All Together: The Cost-Optimized AI Pipeline

```typescript
// TypeScript: Complete cost-optimized pipeline

class CostOptimizedPipeline {
  private cache: SemanticCache;
  private client: Anthropic;

  constructor() {
    this.cache = new SemanticCache();
    this.client = new Anthropic();
  }

  async handleRequest(
    feature: string,
    userId: string,
    teamId: string,
    userMessage: string,
  ): Promise<{ response: string; metadata: object }> {
    const startTime = Date.now();

    // 1. Check budget
    const budgetType = await getUserBudgetType(userId);
    const budgetCheck = await checkBudget(userId, budgetType, 1000, 0.05);
    if (!budgetCheck.allowed) {
      return {
        response: "You've reached your usage limit. Please try again later.",
        metadata: { blocked: true, reason: budgetCheck.reason },
      };
    }

    // 2. Check response cache
    const embedding = await getEmbedding(userMessage);
    const cachedResponse = await this.cache.get(userMessage, embedding);
    if (cachedResponse) {
      return {
        response: cachedResponse,
        metadata: { cached: true, cost: 0, model: "cache" },
      };
    }

    // 3. Route to appropriate model
    const routing = await routeByComplexity(userMessage);

    // 4. Build optimized prompt (minimal tokens)
    const prompt = await buildOptimizedPrompt(feature, userMessage);

    // 5. Call LLM with prompt caching enabled
    const response = await this.client.messages.create({
      model: routing.model,
      max_tokens: getMaxTokens(feature),
      system: getCachedSystemPrompt(feature),
      messages: [{ role: "user", content: prompt }],
    });

    const text = response.content[0].type === "text" ? response.content[0].text : "";
    const latencyMs = Date.now() - startTime;

    // 6. Track cost
    const costEvent = emitCostEvent(response, routing.model, feature, userId, teamId, latencyMs);

    // 7. Update budget usage
    await trackUsage(userId, costEvent.inputTokens + costEvent.outputTokens, costEvent.cost, feature);

    // 8. Cache the response if appropriate
    if (CACHE_CONFIG[feature]?.enabled) {
      await this.cache.set(userMessage, embedding, text);
    }

    // 9. Check for anomalies
    const anomaly = await detectCostAnomaly(feature, costEvent.cost);
    if (anomaly.anomaly) {
      await alertOnAnomaly(anomaly.message);
    }

    return {
      response: text,
      metadata: {
        cached: false,
        cost: costEvent.cost,
        model: routing.model,
        complexity: routing.estimatedComplexity,
        inputTokens: costEvent.inputTokens,
        outputTokens: costEvent.outputTokens,
        cachedTokens: costEvent.cachedTokens,
        latencyMs,
        budgetWarning: budgetCheck.warningMessage,
      },
    };
  }
}
```

---

## 9. Per-Tenant Cost Management

If you're building a SaaS product with AI features, you need to know what each customer costs you. Aggregate cost tracking tells you your total AI spend. Per-tenant cost tracking tells you which customers are profitable and which are burning money.

### 9.1 Token Usage Tracking Per Tenant

Every LLM call should be tagged with the tenant (customer) that triggered it:

```typescript
// TypeScript: Per-tenant token tracking

interface TenantUsageEvent {
  tenantId: string;
  userId: string;
  feature: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  cachedTokens: number;
  cost: number;
  timestamp: Date;
}

async function trackTenantUsage(event: TenantUsageEvent): Promise<void> {
  // Write to your analytics database (BigQuery, ClickHouse, etc.)
  await analyticsDb.insert("tenant_ai_usage", {
    ...event,
    // Pre-compute common aggregation dimensions
    costBucket: event.cost < 0.01 ? "micro" : event.cost < 0.10 ? "small" : "large",
    dayKey: event.timestamp.toISOString().split("T")[0],
    monthKey: event.timestamp.toISOString().slice(0, 7),
  });

  // Update real-time counters (Redis)
  const dayKey = `tenant:${event.tenantId}:cost:${event.timestamp.toISOString().split("T")[0]}`;
  await redis.incrByFloat(dayKey, event.cost);
  await redis.expire(dayKey, 90 * 86400); // 90-day retention
}

// Query: "How much did tenant X spend this month?"
async function getTenantMonthlyCost(tenantId: string, month: string): Promise<{
  totalCost: number;
  byFeature: Record<string, number>;
  byModel: Record<string, number>;
  requestCount: number;
}> {
  return analyticsDb.query(`
    SELECT
      SUM(cost) as total_cost,
      feature,
      model,
      COUNT(*) as request_count
    FROM tenant_ai_usage
    WHERE tenant_id = ? AND month_key = ?
    GROUP BY feature, model
  `, [tenantId, month]);
}
```

### 9.2 Tenant-Level Usage Budgets and Alerts

Different customers get different AI budgets based on their plan:

```typescript
// TypeScript: Per-tenant budget enforcement

interface TenantAIBudget {
  monthlyTokenLimit: number;
  monthlyCostLimit: number;
  dailyCostLimit: number;
  alertAtPercent: number;
  hardLimitAction: "block" | "downgrade_model" | "notify_only";
}

const TENANT_BUDGETS: Record<string, TenantAIBudget> = {
  free: {
    monthlyTokenLimit: 500_000,
    monthlyCostLimit: 5.00,
    dailyCostLimit: 0.50,
    alertAtPercent: 80,
    hardLimitAction: "block",
  },
  starter: {
    monthlyTokenLimit: 5_000_000,
    monthlyCostLimit: 50.00,
    dailyCostLimit: 5.00,
    alertAtPercent: 75,
    hardLimitAction: "downgrade_model",
  },
  enterprise: {
    monthlyTokenLimit: 100_000_000,
    monthlyCostLimit: 1000.00,
    dailyCostLimit: 100.00,
    alertAtPercent: 70,
    hardLimitAction: "notify_only",
  },
};

async function enforceTenantBudget(
  tenantId: string,
  plan: string,
  estimatedCost: number,
): Promise<{ allowed: boolean; action?: string; model?: string }> {
  const budget = TENANT_BUDGETS[plan];
  const currentMonthCost = await getTenantMonthCost(tenantId);

  if (currentMonthCost + estimatedCost > budget.monthlyCostLimit) {
    switch (budget.hardLimitAction) {
      case "block":
        return { allowed: false, action: "budget_exceeded" };
      case "downgrade_model":
        return { allowed: true, action: "downgraded", model: "claude-haiku-4-20250414" };
      case "notify_only":
        await notifyTenantAdmin(tenantId, "AI budget exceeded");
        return { allowed: true, action: "over_budget_notified" };
    }
  }

  return { allowed: true };
}
```

### 9.3 Model Routing Per Tier

The most impactful cost lever for multi-tenant SaaS: route different customer tiers to different models.

```typescript
// TypeScript: Tier-based model routing

function getModelForTenant(tenantId: string, plan: string, feature: string): string {
  const modelMap: Record<string, Record<string, string>> = {
    free: {
      chat: "claude-haiku-4-20250414",
      analysis: "claude-haiku-4-20250414",
      code_review: "claude-haiku-4-20250414",  // free tier gets Haiku for everything
    },
    starter: {
      chat: "claude-haiku-4-20250414",          // chat stays cheap
      analysis: "claude-sonnet-4-20250514",     // analysis gets Sonnet
      code_review: "claude-sonnet-4-20250514",
    },
    enterprise: {
      chat: "claude-sonnet-4-20250514",
      analysis: "claude-sonnet-4-20250514",
      code_review: "claude-opus-4-20250514",    // enterprise gets Opus for code review
    },
  };

  return modelMap[plan]?.[feature] ?? "claude-haiku-4-20250414";
}
```

### 9.4 Cost Attribution in SaaS Products

For SaaS pricing decisions, you need to know the cost-to-serve per tenant:

```
Tenant: Acme Corp (Enterprise plan, $500/month)
  AI cost this month:     $127.43
  Infrastructure cost:    $23.10 (proportional)
  Total cost-to-serve:    $150.53
  Revenue:                $500.00
  Margin:                 69.9%   <- Healthy

Tenant: BigCo Inc (Enterprise plan, $500/month)
  AI cost this month:     $892.17   <- Heavy AI user
  Infrastructure cost:    $45.20
  Total cost-to-serve:    $937.37
  Revenue:                $500.00
  Margin:                 -87.5%  <- Losing money on this customer
```

When you can see this data, you can make informed decisions: raise the price, add usage tiers, set usage limits, or optimize the features that specific tenants use heavily.

---

## 10. Key Takeaways

Cost engineering for AI systems comes down to five levers:

1. **Use the cheapest model that works.** Model routing — whether task-based, complexity-based, or cascade — is typically the single biggest cost saver.

2. **Cache everything you can.** Prompt caching for stable prefixes, response caching for deterministic queries, semantic caching for near-duplicate questions.

3. **Send fewer tokens.** Tighter prompts, fewer few-shot examples, structured output, appropriate max_tokens limits.

4. **Track everything.** You can't optimize what you don't measure. Per-feature, per-model, per-user cost tracking with anomaly detection.

5. **Set budgets before you need them.** Usage limits prevent surprise bills and force optimization conversations early.

The Ramp philosophy is the right mindset: AI costs are an investment in productivity, not an expense to minimize. But like any investment, you should know exactly what you're paying for and exactly what you're getting back.

> **Next chapter:** You've secured the system and controlled the costs. Now let's address the third production challenge: managing the finite context window across long conversations, complex tasks, and multi-agent systems. Ch 50 covers advanced context strategies.

---

*Previous: [AI Security & Guardrails](./48-security-guardrails.md)* | *Next: [Advanced Context Strategies](./50-advanced-context.md)*

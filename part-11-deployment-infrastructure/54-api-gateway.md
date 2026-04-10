<!--
  CHAPTER: 54
  TITLE: API Gateway & Provider Management
  PART: 11 — AI Deployment & Infrastructure
  PHASE: 2 — Become an Expert
  PREREQS: Ch 2 (AI landscape, providers), Ch 3 (LLM API calls), Ch 49 (cost engineering), Ch 53 (deployment)
  KEY_TOPICS: rate limiting, retry strategies, provider failover, AI gateway, request queuing, LLM caching, API key rotation, token budgets, circuit breakers
  DIFFICULTY: Intermediate to Advanced
  LANGUAGE: TypeScript, Python, YAML
  UPDATED: 2026-04-10
-->

# Chapter 54: API Gateway & Provider Management

> **Part 11 — AI Deployment & Infrastructure** | Phase 2: Become an Expert | Prerequisites: Ch 2, Ch 3, Ch 49, Ch 53 | Difficulty: Intermediate to Advanced

In Chapter 53, you deployed your AI application. It is running in production, calling LLM providers, streaming responses to users. But every request goes directly from your application to OpenAI or Anthropic. There is no intermediary. No control plane. No safety net.

This is fine for a prototype. It is not fine for production. Here is what goes wrong without a gateway:

A single user sends 200 requests in a minute and burns through your daily token budget. OpenAI has an outage at 2 PM and your application returns 500 errors for an hour. A new hire accidentally deploys with a prompt that generates 10x the normal tokens and your bill spikes to $5,000 before anyone notices. A competitor discovers your API endpoint and hammers it with requests.

An API gateway sits between your application and your LLM providers. It handles rate limiting, retry with backoff, provider failover, response caching, token budgets, and cost alerts. It is the control plane for your most expensive and least predictable dependency.

### In This Chapter
- Rate limiting for AI: per-user, per-endpoint, and global token budgets
- Retry strategies: exponential backoff with jitter for provider outages
- Provider failover: automatic switching when a provider goes down
- Vercel AI Gateway: unified multi-provider access with built-in routing
- Request queuing for burst traffic
- Caching LLM responses: when it is safe and when it is dangerous
- API key rotation and secrets management
- Cost controls at the gateway: token budgets, alerts, circuit breakers

### Related Chapters
- **Ch 2 (The AI Engineer's Landscape)** -- providers overview; now manage them in production
- **Ch 3 (Your First LLM Call)** -- the API calls you made; now protect them
- **Ch 49 (Cost Engineering)** -- application-level cost control; now enforce at infrastructure
- **Ch 53 (Deploying LLM Applications)** -- you deployed the app; now put a gateway in front
- **Ch 55 (Sandboxing Agents)** -- network-level controls complement gateway controls
- **Ch 57 (Scaling & Cost)** -- gateway enables the scaling patterns

---

## 1. Why You Need a Gateway

### 1.1 The Problem with Direct Provider Access

Without a gateway, your architecture looks like this:

```
User → Your App → OpenAI/Anthropic
                      │
                      └─ No rate limiting
                      └─ No retry logic
                      └─ No failover
                      └─ No caching
                      └─ No cost controls
                      └─ No audit trail
```

With a gateway:

```
User → Your App → AI Gateway → OpenAI/Anthropic
                      │
                      ├─ Rate limiting (per-user, per-endpoint)
                      ├─ Retry with exponential backoff
                      ├─ Failover (OpenAI down → Anthropic)
                      ├─ Response caching
                      ├─ Token budget enforcement
                      ├─ Cost alerts and circuit breakers
                      └─ Request/response logging
```

### 1.2 Build vs Buy

You have three options:

| Approach | Examples | Best When |
|----------|---------|-----------|
| **Managed gateway** | Vercel AI Gateway | You want zero infrastructure, are on Vercel |
| **Open-source proxy** | LiteLLM Proxy, Portkey | You want control, self-host, customize |
| **Build your own** | Custom middleware | You have specific requirements, existing infra |

This chapter covers all three approaches. Most teams should start with a managed gateway and only build custom when they hit its limits.

---

## 2. Rate Limiting for AI

### 2.1 Why AI Rate Limiting Is Different

Traditional rate limiting counts requests: "100 requests per minute per user." AI rate limiting must count *tokens*, because one request might use 100 tokens and another might use 100,000 tokens. A user who sends 10 requests with 50,000 tokens each costs 500x more than a user who sends 10 requests with 100 tokens each.

### 2.2 Three Layers of Rate Limiting

```typescript
// lib/rate-limiter.ts — multi-layer rate limiting for AI
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Layer 1: Request rate limiting (prevent abuse)
const requestLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(60, "1 m"), // 60 requests per minute
  prefix: "ratelimit:requests",
});

// Layer 2: Token budget per user (prevent cost overrun)
// This is custom because Upstash counts requests, not tokens
class TokenBudgetLimiter {
  constructor(
    private redis: Redis,
    private maxTokensPerDay: number = 100_000
  ) {}

  async checkAndConsume(
    userId: string,
    estimatedTokens: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
    const key = `token-budget:${userId}:${this.todayKey()}`;
    const current = (await this.redis.get<number>(key)) || 0;

    if (current + estimatedTokens > this.maxTokensPerDay) {
      return {
        allowed: false,
        remaining: Math.max(0, this.maxTokensPerDay - current),
        resetAt: this.endOfDay(),
      };
    }

    // Consume tokens
    await this.redis.incrby(key, estimatedTokens);
    await this.redis.expireat(key, Math.floor(this.endOfDay().getTime() / 1000));

    return {
      allowed: true,
      remaining: this.maxTokensPerDay - current - estimatedTokens,
      resetAt: this.endOfDay(),
    };
  }

  async recordActualUsage(userId: string, actualTokens: number, estimatedTokens: number) {
    // Adjust for the difference between estimated and actual
    const key = `token-budget:${userId}:${this.todayKey()}`;
    const adjustment = actualTokens - estimatedTokens;
    if (adjustment !== 0) {
      await this.redis.incrby(key, adjustment);
    }
  }

  private todayKey(): string {
    return new Date().toISOString().slice(0, 10); // "2026-04-10"
  }

  private endOfDay(): Date {
    const d = new Date();
    d.setHours(23, 59, 59, 999);
    return d;
  }
}

// Layer 3: Global budget (emergency stop)
class GlobalBudgetLimiter {
  constructor(
    private redis: Redis,
    private maxDailySpendCents: number = 10000 // $100/day
  ) {}

  async checkBudget(): Promise<{ allowed: boolean; spentCents: number }> {
    const key = `global-budget:${new Date().toISOString().slice(0, 10)}`;
    const spent = (await this.redis.get<number>(key)) || 0;

    return {
      allowed: spent < this.maxDailySpendCents,
      spentCents: spent,
    };
  }

  async recordSpend(tokens: number, costPer1kTokens: number) {
    const key = `global-budget:${new Date().toISOString().slice(0, 10)}`;
    const costCents = Math.ceil((tokens / 1000) * costPer1kTokens * 100);
    await this.redis.incrby(key, costCents);
    await this.redis.expire(key, 86400 * 2); // TTL: 2 days
  }
}

export const rateLimiter = {
  requests: requestLimiter,
  tokenBudget: new TokenBudgetLimiter(redis, 100_000),
  globalBudget: new GlobalBudgetLimiter(redis, 10000),
};
```

### 2.3 Applying Rate Limits in a Route

```typescript
// app/api/chat/route.ts — rate-limited AI route
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { rateLimiter } from "@/lib/rate-limiter";
import { estimateTokens } from "@/lib/token-utils";

export const maxDuration = 60;

export async function POST(req: Request) {
  const userId = req.headers.get("x-user-id") || "anonymous";
  const { messages } = await req.json();

  // Layer 1: Request rate limit
  const { success: requestAllowed } = await rateLimiter.requests.limit(userId);
  if (!requestAllowed) {
    return Response.json(
      { error: "Too many requests. Please wait a moment." },
      { status: 429, headers: { "Retry-After": "60" } }
    );
  }

  // Layer 2: Token budget
  const estimatedTokens = estimateTokens(messages);
  const { allowed: budgetAllowed, remaining } =
    await rateLimiter.tokenBudget.checkAndConsume(userId, estimatedTokens);
  if (!budgetAllowed) {
    return Response.json(
      {
        error: "Daily token budget exceeded.",
        remaining,
        resetsAt: new Date().setHours(24, 0, 0, 0),
      },
      { status: 429 }
    );
  }

  // Layer 3: Global budget
  const { allowed: globalAllowed } = await rateLimiter.globalBudget.checkBudget();
  if (!globalAllowed) {
    // This is an emergency — alert the team
    console.error("GLOBAL BUDGET EXCEEDED — all AI requests blocked");
    return Response.json(
      { error: "Service temporarily unavailable." },
      { status: 503 }
    );
  }

  // All checks passed — make the LLM call
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    messages,
  });

  return result.toDataStreamResponse();
}
```

### 2.4 Token Estimation

You need to estimate tokens before the LLM call (to check budgets) and record actual tokens after (to adjust).

```typescript
// lib/token-utils.ts — estimate token counts without calling the API
export function estimateTokens(messages: Array<{ role: string; content: string }>): number {
  // Rule of thumb: 1 token ≈ 4 characters in English
  // This is an estimate — actual tokenization varies by model
  let totalChars = 0;
  for (const msg of messages) {
    totalChars += msg.content.length;
    totalChars += 10; // Overhead for role, message framing
  }
  // Add 20% buffer for safety
  return Math.ceil((totalChars / 4) * 1.2);
}

export function estimateCostCents(
  inputTokens: number,
  outputTokens: number,
  model: string
): number {
  // Pricing per 1M tokens (approximate, check provider pricing pages)
  const pricing: Record<string, { input: number; output: number }> = {
    "claude-sonnet-4-20250514": { input: 3.0, output: 15.0 },
    "claude-haiku-3-5": { input: 0.25, output: 1.25 },
    "gpt-4o": { input: 2.5, output: 10.0 },
    "gpt-4o-mini": { input: 0.15, output: 0.6 },
  };

  const prices = pricing[model] || pricing["gpt-4o"];
  const inputCost = (inputTokens / 1_000_000) * prices.input;
  const outputCost = (outputTokens / 1_000_000) * prices.output;

  return Math.ceil((inputCost + outputCost) * 100); // cents
}
```

---

## 3. Retry Strategies

### 3.1 Why Retries Matter for AI

LLM providers have outages. Not "once a year" outages -- partial degradation happens weekly. A robust retry strategy is the difference between "our AI feature was down for 45 minutes" and "some requests took a couple extra seconds."

### 3.2 Exponential Backoff with Jitter

```typescript
// lib/retry.ts — retry with exponential backoff and jitter
interface RetryConfig {
  maxRetries: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryableStatuses: number[];
}

const DEFAULT_CONFIG: RetryConfig = {
  maxRetries: 3,
  baseDelayMs: 1000,       // Start with 1 second
  maxDelayMs: 30000,       // Cap at 30 seconds
  retryableStatuses: [
    429,  // Rate limited
    500,  // Internal server error
    502,  // Bad gateway
    503,  // Service unavailable
    504,  // Gateway timeout
  ],
};

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  config: Partial<RetryConfig> = {}
): Promise<T> {
  const { maxRetries, baseDelayMs, maxDelayMs, retryableStatuses } = {
    ...DEFAULT_CONFIG,
    ...config,
  };

  let lastError: Error | null = null;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      lastError = error;

      // Don't retry if we've exhausted attempts
      if (attempt === maxRetries) break;

      // Don't retry non-retryable errors
      const status = error?.status || error?.statusCode;
      if (status && !retryableStatuses.includes(status)) {
        throw error;
      }

      // Check for provider-suggested retry delay
      const retryAfter = error?.headers?.["retry-after"];
      let delay: number;

      if (retryAfter) {
        delay = parseInt(retryAfter) * 1000;
      } else {
        // Exponential backoff with full jitter
        // delay = random(0, min(maxDelay, baseDelay * 2^attempt))
        const exponentialDelay = baseDelayMs * Math.pow(2, attempt);
        const cappedDelay = Math.min(maxDelayMs, exponentialDelay);
        delay = Math.random() * cappedDelay;
      }

      console.warn(
        `AI request failed (attempt ${attempt + 1}/${maxRetries + 1}). ` +
        `Status: ${status}. Retrying in ${Math.round(delay)}ms...`
      );

      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}
```

### 3.3 Using Retry with the AI SDK

```typescript
// lib/ai-client.ts — AI client with built-in retry
import { generateText, streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { retryWithBackoff } from "./retry";

export async function generateWithRetry(
  params: Parameters<typeof generateText>[0]
) {
  return retryWithBackoff(() => generateText(params), {
    maxRetries: 3,
    baseDelayMs: 1000,
  });
}

export function streamWithRetry(
  params: Parameters<typeof streamText>[0]
) {
  // For streaming, retry logic needs to happen before the stream starts.
  // If the initial connection fails, retry. Once streaming begins,
  // a failure mid-stream cannot be retried transparently.
  return retryWithBackoff(() => {
    const result = streamText(params);
    // Return a promise that resolves once the stream is established
    return Promise.resolve(result);
  }, {
    maxRetries: 2,
    baseDelayMs: 500,
  });
}
```

### 3.4 Retry Patterns to Avoid

```typescript
// BAD: Retrying non-idempotent operations
// If the LLM call that creates a database record fails AFTER the record
// was created but BEFORE you got the response, retrying will create a duplicate.
await retryWithBackoff(async () => {
  const result = await generateText({ ... });
  await db.insert(result); // This might have already happened!
  return result;
});

// GOOD: Separate the LLM call from the side effect
const result = await retryWithBackoff(() => generateText({ ... }));
// Only run the side effect once, after the LLM call succeeds
await db.insert(result);

// BAD: Infinite retries
// If the provider is down for an extended period, you'll queue up
// thousands of retries that all fire when it comes back.
{ maxRetries: Infinity }

// GOOD: Bounded retries with circuit breaker (see Section 8)
{ maxRetries: 3 }
```

---

## 4. Provider Failover

### 4.1 The Failover Pattern

When your primary provider goes down, switch to a backup. This requires that your prompts work across providers (they usually do with minor adjustments).

```typescript
// lib/provider-failover.ts — automatic provider failover
import { generateText, streamText, LanguageModel } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { openai } from "@ai-sdk/openai";

interface ProviderConfig {
  name: string;
  model: LanguageModel;
  priority: number;        // Lower = higher priority
  healthy: boolean;
  lastFailure: Date | null;
  consecutiveFailures: number;
}

class ProviderManager {
  private providers: ProviderConfig[];
  private healthCheckInterval: NodeJS.Timeout | null = null;

  constructor() {
    this.providers = [
      {
        name: "anthropic",
        model: anthropic("claude-sonnet-4-20250514"),
        priority: 1,
        healthy: true,
        lastFailure: null,
        consecutiveFailures: 0,
      },
      {
        name: "openai",
        model: openai("gpt-4o"),
        priority: 2,
        healthy: true,
        lastFailure: null,
        consecutiveFailures: 0,
      },
    ];
  }

  getModel(): LanguageModel {
    // Return the highest-priority healthy provider
    const available = this.providers
      .filter((p) => p.healthy)
      .sort((a, b) => a.priority - b.priority);

    if (available.length === 0) {
      // All providers down — try the primary anyway
      console.error("ALL PROVIDERS UNHEALTHY — falling back to primary");
      return this.providers[0].model;
    }

    return available[0].model;
  }

  markFailure(providerName: string) {
    const provider = this.providers.find((p) => p.name === providerName);
    if (!provider) return;

    provider.consecutiveFailures++;
    provider.lastFailure = new Date();

    // Mark unhealthy after 3 consecutive failures
    if (provider.consecutiveFailures >= 3) {
      provider.healthy = false;
      console.warn(
        `Provider ${providerName} marked unhealthy after ` +
        `${provider.consecutiveFailures} consecutive failures`
      );

      // Auto-recover after 60 seconds
      setTimeout(() => {
        provider.healthy = true;
        provider.consecutiveFailures = 0;
        console.info(`Provider ${providerName} marked healthy (auto-recovery)`);
      }, 60_000);
    }
  }

  markSuccess(providerName: string) {
    const provider = this.providers.find((p) => p.name === providerName);
    if (!provider) return;

    provider.consecutiveFailures = 0;
    provider.healthy = true;
  }

  getProviderName(model: LanguageModel): string {
    const provider = this.providers.find((p) => p.model === model);
    return provider?.name || "unknown";
  }
}

export const providerManager = new ProviderManager();
```

### 4.2 Failover in Practice

```typescript
// lib/ai-with-failover.ts — complete AI call with failover and retry
import { generateText } from "ai";
import { providerManager } from "./provider-failover";
import { retryWithBackoff } from "./retry";

export async function generateWithFailover(
  params: Omit<Parameters<typeof generateText>[0], "model">
) {
  const maxProviderAttempts = 2; // Try up to 2 different providers

  for (let providerAttempt = 0; providerAttempt < maxProviderAttempts; providerAttempt++) {
    const model = providerManager.getModel();
    const providerName = providerManager.getProviderName(model);

    try {
      const result = await retryWithBackoff(
        () => generateText({ ...params, model }),
        { maxRetries: 2 }
      );

      providerManager.markSuccess(providerName);
      return result;
    } catch (error: any) {
      console.error(
        `Provider ${providerName} failed:`,
        error.message || error
      );
      providerManager.markFailure(providerName);

      // If this was the last provider attempt, throw
      if (providerAttempt === maxProviderAttempts - 1) {
        throw error;
      }

      // Otherwise, loop will try the next provider
      console.info("Attempting failover to next provider...");
    }
  }

  throw new Error("All providers exhausted");
}
```

---

## 5. The Vercel AI Gateway

### 5.1 What It Does

The Vercel AI Gateway is a managed gateway that sits between your application and LLM providers. It provides:

- **Unified API**: One interface for multiple providers
- **Automatic failover**: Configure primary and fallback models
- **Cost tracking**: Per-request cost visibility without custom instrumentation
- **Zero data retention**: Requests are not stored by the gateway
- **OIDC authentication**: No API keys in your environment variables

### 5.2 Setting Up the AI Gateway

The Vercel AI Gateway works through the AI SDK provider packages. When deployed on Vercel, the SDK automatically routes requests through the gateway.

```typescript
// app/api/chat/route.ts — using the Vercel AI Gateway
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

export const maxDuration = 60;

export async function POST(req: Request) {
  const { messages } = await req.json();

  // When deployed on Vercel, this automatically routes through
  // the AI Gateway. No API keys needed in environment variables —
  // the gateway handles authentication via OIDC.
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    messages,
  });

  return result.toDataStreamResponse();
}
```

### 5.3 Multi-Provider Routing with the Gateway

```typescript
// lib/model-router.ts — route to different models based on the request
import { anthropic } from "@ai-sdk/anthropic";
import { openai } from "@ai-sdk/openai";
import { LanguageModel } from "ai";

type ModelTier = "fast" | "balanced" | "powerful";

export function selectModel(tier: ModelTier): LanguageModel {
  switch (tier) {
    case "fast":
      // Low-latency, low-cost tasks: classification, extraction
      return openai("gpt-4o-mini");
    case "balanced":
      // Most tasks: chat, Q&A, analysis
      return anthropic("claude-sonnet-4-20250514");
    case "powerful":
      // Complex reasoning, long documents, code generation
      return anthropic("claude-opus-4-20250514");
  }
}

// Usage in routes
export async function POST(req: Request) {
  const { messages, tier = "balanced" } = await req.json();

  const result = streamText({
    model: selectModel(tier as ModelTier),
    messages,
  });

  return result.toDataStreamResponse();
}
```

### 5.4 Gateway Fallback Configuration

```typescript
// lib/gateway-config.ts — fallback chain configuration
// The Vercel AI Gateway supports automatic fallbacks.
// If the primary model fails, it tries the next one.

import { anthropic } from "@ai-sdk/anthropic";
import { openai } from "@ai-sdk/openai";
import { streamText } from "ai";

// Approach 1: Application-level fallback
export async function chatWithFallback(messages: any[]) {
  try {
    // Primary: Anthropic Claude
    return await streamText({
      model: anthropic("claude-sonnet-4-20250514"),
      messages,
    });
  } catch (error: any) {
    if (error.status === 429 || error.status >= 500) {
      console.warn("Primary provider failed, falling back to OpenAI");
      // Fallback: OpenAI
      return await streamText({
        model: openai("gpt-4o"),
        messages,
      });
    }
    throw error;
  }
}
```

---

## 6. Request Queuing for Burst Traffic

### 6.1 The Problem

Traffic spikes happen. A viral tweet, a product launch, a demo -- suddenly you have 10x your normal traffic. LLM providers have rate limits. If you send all those requests simultaneously, most will get 429 errors.

### 6.2 Queue-Based Architecture

```typescript
// lib/request-queue.ts — queue AI requests to smooth out bursts
import { Redis } from "@upstash/redis";

interface QueuedRequest {
  id: string;
  messages: any[];
  userId: string;
  model: string;
  priority: number; // Lower = higher priority
  timestamp: number;
  ttl: number; // Max time to wait in queue (ms)
}

class AIRequestQueue {
  private redis: Redis;
  private processing = false;
  private concurrentLimit: number;
  private activeRequests = 0;

  constructor(redis: Redis, concurrentLimit = 50) {
    this.redis = redis;
    this.concurrentLimit = concurrentLimit;
  }

  async enqueue(request: Omit<QueuedRequest, "id" | "timestamp">): Promise<string> {
    const id = crypto.randomUUID();
    const item: QueuedRequest = {
      ...request,
      id,
      timestamp: Date.now(),
    };

    // Add to sorted set (score = priority * 1e13 + timestamp for FIFO within priority)
    const score = request.priority * 1e13 + Date.now();
    await this.redis.zadd("ai-request-queue", {
      score,
      member: JSON.stringify(item),
    });

    // Trigger processing
    this.processQueue();

    return id;
  }

  private async processQueue() {
    if (this.processing) return;
    this.processing = true;

    try {
      while (this.activeRequests < this.concurrentLimit) {
        // Get the highest-priority item
        const items = await this.redis.zpopmin<string[]>("ai-request-queue", 1);
        if (!items || items.length === 0) break;

        const request: QueuedRequest = JSON.parse(items[0]);

        // Check if the request has expired
        if (Date.now() - request.timestamp > request.ttl) {
          console.warn(`Request ${request.id} expired in queue`);
          await this.redis.set(
            `ai-result:${request.id}`,
            JSON.stringify({ error: "Request timed out in queue" }),
            { ex: 300 }
          );
          continue;
        }

        // Process the request
        this.activeRequests++;
        this.processRequest(request)
          .catch((error) => {
            console.error(`Request ${request.id} failed:`, error);
          })
          .finally(() => {
            this.activeRequests--;
            this.processQueue(); // Process next item
          });
      }
    } finally {
      this.processing = false;
    }
  }

  private async processRequest(request: QueuedRequest) {
    // This is where you make the actual LLM call
    // Results are stored in Redis for the client to poll
    const { generateText } = await import("ai");
    const { anthropic } = await import("@ai-sdk/anthropic");

    const result = await generateText({
      model: anthropic("claude-sonnet-4-20250514"),
      messages: request.messages,
    });

    await this.redis.set(
      `ai-result:${request.id}`,
      JSON.stringify({ text: result.text, usage: result.usage }),
      { ex: 300 } // Results expire after 5 minutes
    );
  }

  async getResult(id: string): Promise<any | null> {
    const result = await this.redis.get(`ai-result:${id}`);
    return result ? JSON.parse(result as string) : null;
  }
}

export const aiQueue = new AIRequestQueue(
  new Redis({
    url: process.env.UPSTASH_REDIS_REST_URL!,
    token: process.env.UPSTASH_REDIS_REST_TOKEN!,
  }),
  50 // Max 50 concurrent LLM requests
);
```

### 6.3 Queue API Endpoint

```typescript
// app/api/ai/queue/route.ts — enqueue an AI request
import { aiQueue } from "@/lib/request-queue";

export async function POST(req: Request) {
  const { messages, userId, priority = 5 } = await req.json();

  const requestId = await aiQueue.enqueue({
    messages,
    userId,
    model: "claude-sonnet-4-20250514",
    priority,
    ttl: 30_000, // 30 second timeout
  });

  return Response.json({ requestId, status: "queued" });
}

// app/api/ai/queue/[id]/route.ts — poll for result
import { aiQueue } from "@/lib/request-queue";

export async function GET(
  req: Request,
  { params }: { params: { id: string } }
) {
  const result = await aiQueue.getResult(params.id);

  if (!result) {
    return Response.json({ status: "processing" }, { status: 202 });
  }

  return Response.json({ status: "complete", result });
}
```

---

## 7. Caching LLM Responses

### 7.1 When Caching Is Safe

Caching LLM responses can save significant cost and latency. But it is only safe for certain types of requests.

| Request Type | Safe to Cache? | Why |
|-------------|---------------|-----|
| Classification | Yes | Same input always wants the same classification |
| Entity extraction | Yes | Extracting names/dates from the same text is deterministic |
| Summarization | Mostly | Same document should produce similar summaries |
| Translation | Mostly | Same text in same languages should match |
| Chat conversation | No | Context-dependent, personalized |
| Creative writing | No | Users expect variety |
| Agent tool calls | No | Actions should not be replayed from cache |

### 7.2 Semantic Caching

Simple exact-match caching rarely helps because users phrase the same question differently. Semantic caching uses embeddings to find similar past questions.

```typescript
// lib/semantic-cache.ts — cache LLM responses by semantic similarity
import { embed } from "ai";
import { openai } from "@ai-sdk/openai";

interface CachedResponse {
  query: string;
  embedding: number[];
  response: string;
  model: string;
  cachedAt: number;
  ttl: number;
}

class SemanticCache {
  private cache: CachedResponse[] = []; // In production, use a vector DB
  private similarityThreshold = 0.95; // High threshold to avoid wrong hits

  async get(query: string): Promise<string | null> {
    const { embedding: queryEmbedding } = await embed({
      model: openai.embedding("text-embedding-3-small"),
      value: query,
    });

    let bestMatch: CachedResponse | null = null;
    let bestSimilarity = 0;

    for (const entry of this.cache) {
      // Check TTL
      if (Date.now() - entry.cachedAt > entry.ttl) continue;

      const similarity = cosineSimilarity(queryEmbedding, entry.embedding);
      if (similarity > this.similarityThreshold && similarity > bestSimilarity) {
        bestMatch = entry;
        bestSimilarity = similarity;
      }
    }

    if (bestMatch) {
      console.info(
        `Cache hit (similarity: ${bestSimilarity.toFixed(3)}): "${query}" ≈ "${bestMatch.query}"`
      );
      return bestMatch.response;
    }

    return null;
  }

  async set(query: string, response: string, model: string, ttlMs = 3600_000) {
    const { embedding } = await embed({
      model: openai.embedding("text-embedding-3-small"),
      value: query,
    });

    this.cache.push({
      query,
      embedding,
      response,
      model,
      cachedAt: Date.now(),
      ttl: ttlMs,
    });

    // Evict old entries
    this.cache = this.cache.filter(
      (entry) => Date.now() - entry.cachedAt < entry.ttl
    );
  }
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;
  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}

export const semanticCache = new SemanticCache();
```

### 7.3 Using the Cache in Routes

```typescript
// app/api/faq/route.ts — cached FAQ endpoint
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { semanticCache } from "@/lib/semantic-cache";

export async function POST(req: Request) {
  const { question } = await req.json();

  // Check cache first
  const cached = await semanticCache.get(question);
  if (cached) {
    return Response.json({
      answer: cached,
      cached: true,
    });
  }

  // Cache miss — call the LLM
  const result = await generateText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: "You are a customer support assistant. Answer based on the FAQ database.",
    prompt: question,
  });

  // Store in cache for 1 hour
  await semanticCache.set(question, result.text, "claude-sonnet-4-20250514", 3600_000);

  return Response.json({
    answer: result.text,
    cached: false,
  });
}
```

### 7.4 Caching Dangers

```typescript
// DANGER: caching personalized responses
// If User A asks "What's my account balance?" and the response is cached,
// User B asking the same question would see User A's balance.

// DANGER: caching tool call results
// If the LLM decided to "create_ticket" for a previous request,
// replaying that from cache would create a duplicate ticket.

// DANGER: stale cache for time-sensitive data
// "What's the stock price of AAPL?" cached for 1 hour will be wrong.

// SAFE: cache key should include user ID for personalized content
const cacheKey = `${userId}:${question}`; // Per-user cache

// SAFE: only cache read-only, non-personalized requests
const CACHEABLE_ROUTES = ["/api/faq", "/api/summarize", "/api/classify"];
```

---

## 8. Circuit Breakers

### 8.1 What Is a Circuit Breaker?

A circuit breaker prevents your application from repeatedly calling a failing service. When failures exceed a threshold, the circuit "opens" and all requests fail immediately without calling the provider. After a cooldown period, the circuit enters "half-open" state and allows a test request through.

```typescript
// lib/circuit-breaker.ts — circuit breaker for LLM providers
type CircuitState = "closed" | "open" | "half-open";

class CircuitBreaker {
  private state: CircuitState = "closed";
  private failureCount = 0;
  private lastFailureTime: number = 0;
  private successCount = 0;

  constructor(
    private name: string,
    private failureThreshold: number = 5,
    private resetTimeoutMs: number = 60_000,
    private halfOpenMaxRequests: number = 3
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      // Check if reset timeout has elapsed
      if (Date.now() - this.lastFailureTime > this.resetTimeoutMs) {
        this.state = "half-open";
        this.successCount = 0;
        console.info(`Circuit ${this.name}: half-open (testing)`);
      } else {
        throw new CircuitOpenError(
          `Circuit ${this.name} is OPEN. Try again in ` +
          `${Math.ceil((this.resetTimeoutMs - (Date.now() - this.lastFailureTime)) / 1000)}s`
        );
      }
    }

    try {
      const result = await fn();

      // Success
      if (this.state === "half-open") {
        this.successCount++;
        if (this.successCount >= this.halfOpenMaxRequests) {
          this.state = "closed";
          this.failureCount = 0;
          console.info(`Circuit ${this.name}: closed (recovered)`);
        }
      } else {
        this.failureCount = 0;
      }

      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.failureThreshold || this.state === "half-open") {
        this.state = "open";
        console.error(
          `Circuit ${this.name}: OPEN after ${this.failureCount} failures`
        );
      }

      throw error;
    }
  }

  getState(): { state: CircuitState; failures: number } {
    return { state: this.state, failures: this.failureCount };
  }
}

class CircuitOpenError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "CircuitOpenError";
  }
}

// One circuit breaker per provider
export const circuits = {
  anthropic: new CircuitBreaker("anthropic", 5, 60_000),
  openai: new CircuitBreaker("openai", 5, 60_000),
};
```

### 8.2 Circuit Breaker + Failover

```typescript
// lib/resilient-ai.ts — combining circuit breaker, retry, and failover
import { generateText, LanguageModel } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { openai } from "@ai-sdk/openai";
import { circuits, CircuitOpenError } from "./circuit-breaker";
import { retryWithBackoff } from "./retry";

interface ProviderOption {
  name: string;
  model: LanguageModel;
  circuit: (typeof circuits)[keyof typeof circuits];
}

const providers: ProviderOption[] = [
  { name: "anthropic", model: anthropic("claude-sonnet-4-20250514"), circuit: circuits.anthropic },
  { name: "openai", model: openai("gpt-4o"), circuit: circuits.openai },
];

export async function resilientGenerate(
  params: Omit<Parameters<typeof generateText>[0], "model">
) {
  for (const provider of providers) {
    try {
      const result = await provider.circuit.execute(() =>
        retryWithBackoff(() => generateText({ ...params, model: provider.model }), {
          maxRetries: 2,
        })
      );
      return { ...result, provider: provider.name };
    } catch (error) {
      if (error instanceof CircuitOpenError) {
        console.info(`Skipping ${provider.name} (circuit open)`);
        continue;
      }
      console.error(`${provider.name} failed after retries:`, error);
      continue;
    }
  }

  throw new Error("All AI providers exhausted. Service unavailable.");
}
```

---

## 9. API Key Rotation and Secrets Management

### 9.1 The Rotation Problem

API keys need to be rotated regularly. If a key is compromised, you need to rotate immediately. If your provider requires rotation, you need to do it without downtime.

### 9.2 Zero-Downtime Key Rotation

```typescript
// lib/key-manager.ts — zero-downtime API key rotation
// The pattern: support two active keys and phase out the old one

interface KeyConfig {
  primary: string;
  secondary: string | null;  // Set during rotation
  rotationStarted: Date | null;
}

class APIKeyManager {
  private keys: Map<string, KeyConfig> = new Map();

  constructor() {
    this.keys.set("anthropic", {
      primary: process.env.ANTHROPIC_API_KEY!,
      secondary: process.env.ANTHROPIC_API_KEY_SECONDARY || null,
      rotationStarted: null,
    });
  }

  getKey(provider: string): string {
    const config = this.keys.get(provider);
    if (!config) throw new Error(`Unknown provider: ${provider}`);

    // During rotation, randomly use primary or secondary
    // to verify the new key works before cutting over
    if (config.secondary && config.rotationStarted) {
      const rotationAge = Date.now() - config.rotationStarted.getTime();
      const secondaryWeight = Math.min(rotationAge / 3600_000, 1); // Ramp up over 1 hour

      if (Math.random() < secondaryWeight) {
        return config.secondary;
      }
    }

    return config.primary;
  }

  startRotation(provider: string, newKey: string) {
    const config = this.keys.get(provider);
    if (!config) throw new Error(`Unknown provider: ${provider}`);

    config.secondary = newKey;
    config.rotationStarted = new Date();
    console.info(`Key rotation started for ${provider}`);
  }

  completeRotation(provider: string) {
    const config = this.keys.get(provider);
    if (!config || !config.secondary) return;

    config.primary = config.secondary;
    config.secondary = null;
    config.rotationStarted = null;
    console.info(`Key rotation completed for ${provider}`);
  }
}

export const keyManager = new APIKeyManager();
```

### 9.3 Secrets Management with AWS Secrets Manager

```python
# infra/secrets_manager.py — rotate API keys with AWS Secrets Manager
import boto3
import json
from datetime import datetime


def get_api_key(secret_name: str, region: str = "us-east-1") -> str:
    """Retrieve an API key from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name=region)

    response = client.get_secret_value(SecretId=secret_name)
    secret = json.loads(response["SecretString"])
    return secret["api_key"]


def rotate_api_key(
    secret_name: str,
    new_key: str,
    region: str = "us-east-1"
) -> None:
    """Rotate an API key in AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name=region)

    # Store the new key as pending
    client.put_secret_value(
        SecretId=secret_name,
        SecretString=json.dumps({
            "api_key": new_key,
            "rotated_at": datetime.utcnow().isoformat(),
        }),
        VersionStages=["AWSCURRENT"],
    )

    print(f"Secret {secret_name} rotated successfully")
```

```hcl
# infra/secrets.tf — Terraform for secrets management
resource "aws_secretsmanager_secret" "anthropic_api_key" {
  name        = "ai-service/anthropic-api-key"
  description = "Anthropic API key for AI service"

  tags = {
    Service     = "ai-service"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_rotation" "anthropic_api_key" {
  secret_id           = aws_secretsmanager_secret.anthropic_api_key.id
  rotation_lambda_arn = aws_lambda_function.secret_rotator.arn

  rotation_rules {
    automatically_after_days = 90
  }
}

# IAM policy: only the AI service can read the secret
resource "aws_iam_policy" "read_ai_secrets" {
  name = "read-ai-secrets-${var.environment}"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
        ]
        Resource = [
          aws_secretsmanager_secret.anthropic_api_key.arn,
        ]
      }
    ]
  })
}
```

---

## 10. Cost Controls at the Gateway

### 10.1 Token Budgets

```typescript
// lib/cost-controls.ts — enforce cost limits at the gateway level
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

interface CostAlert {
  level: "warning" | "critical" | "emergency";
  threshold: number; // cents
  action: "alert" | "throttle" | "block";
}

const COST_ALERTS: CostAlert[] = [
  { level: "warning", threshold: 5000, action: "alert" },      // $50: alert
  { level: "critical", threshold: 10000, action: "throttle" },  // $100: throttle
  { level: "emergency", threshold: 25000, action: "block" },    // $250: block
];

export async function checkCostControls(): Promise<{
  allowed: boolean;
  action: string;
  spentCents: number;
}> {
  const today = new Date().toISOString().slice(0, 10);
  const spentCents = (await redis.get<number>(`daily-cost:${today}`)) || 0;

  for (const alert of COST_ALERTS.reverse()) {
    if (spentCents >= alert.threshold) {
      if (alert.action === "block") {
        return { allowed: false, action: "blocked", spentCents };
      }
      if (alert.action === "throttle") {
        // Allow only 10% of requests through
        const allowed = Math.random() < 0.1;
        return { allowed, action: "throttled", spentCents };
      }
      if (alert.action === "alert") {
        // Send alert but allow through
        await sendCostAlert(alert.level, spentCents);
        return { allowed: true, action: "alerted", spentCents };
      }
    }
  }

  return { allowed: true, action: "normal", spentCents };
}

async function sendCostAlert(level: string, spentCents: number) {
  const alertKey = `cost-alert:${level}:${new Date().toISOString().slice(0, 10)}`;
  const alreadySent = await redis.get(alertKey);
  if (alreadySent) return; // Only alert once per day per level

  // Send to your alerting system (Slack, PagerDuty, etc.)
  console.warn(`[COST ALERT] ${level}: $${(spentCents / 100).toFixed(2)} spent today`);

  await redis.set(alertKey, "sent", { ex: 86400 });
}

export async function recordCost(inputTokens: number, outputTokens: number, model: string) {
  const costCents = calculateCostCents(inputTokens, outputTokens, model);
  const today = new Date().toISOString().slice(0, 10);

  await redis.incrby(`daily-cost:${today}`, costCents);
  await redis.expire(`daily-cost:${today}`, 86400 * 2);
}

function calculateCostCents(
  inputTokens: number,
  outputTokens: number,
  model: string
): number {
  const pricing: Record<string, { input: number; output: number }> = {
    "claude-sonnet-4-20250514": { input: 3.0, output: 15.0 },
    "gpt-4o": { input: 2.5, output: 10.0 },
  };
  const p = pricing[model] || { input: 5, output: 15 };
  return Math.ceil(
    ((inputTokens / 1_000_000) * p.input +
      (outputTokens / 1_000_000) * p.output) * 100
  );
}
```

### 10.2 Per-User Cost Visibility

```typescript
// app/api/usage/route.ts — expose usage to users
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export async function GET(req: Request) {
  const userId = req.headers.get("x-user-id");
  if (!userId) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const today = new Date().toISOString().slice(0, 10);
  const tokensUsed = (await redis.get<number>(`token-budget:${userId}:${today}`)) || 0;
  const dailyLimit = 100_000;

  return Response.json({
    tokensUsed,
    dailyLimit,
    remaining: Math.max(0, dailyLimit - tokensUsed),
    percentUsed: Math.round((tokensUsed / dailyLimit) * 100),
    resetsAt: `${today}T23:59:59Z`,
  });
}
```

---

## 11. Putting It All Together: The Gateway Middleware

### 11.1 Complete Gateway Middleware

```typescript
// middleware/ai-gateway.ts — complete AI gateway middleware
import { NextRequest, NextResponse } from "next/server";
import { rateLimiter } from "@/lib/rate-limiter";
import { checkCostControls, recordCost } from "@/lib/cost-controls";
import { semanticCache } from "@/lib/semantic-cache";
import { circuits } from "@/lib/circuit-breaker";

export async function aiGatewayMiddleware(req: NextRequest) {
  const userId = req.headers.get("x-user-id") || "anonymous";
  const startTime = Date.now();

  // 1. Rate limit check
  const { success: requestAllowed } = await rateLimiter.requests.limit(userId);
  if (!requestAllowed) {
    return NextResponse.json(
      { error: "Rate limit exceeded" },
      { status: 429, headers: { "Retry-After": "60" } }
    );
  }

  // 2. Cost control check
  const { allowed: costAllowed, action } = await checkCostControls();
  if (!costAllowed) {
    return NextResponse.json(
      { error: "Service temporarily limited due to high usage" },
      { status: 503 }
    );
  }

  // 3. Circuit breaker status
  const circuitStatus = {
    anthropic: circuits.anthropic.getState(),
    openai: circuits.openai.getState(),
  };

  // Add gateway headers for downstream route handlers
  const headers = new Headers(req.headers);
  headers.set("x-gateway-user-id", userId);
  headers.set("x-gateway-timestamp", startTime.toString());
  headers.set("x-gateway-circuit-status", JSON.stringify(circuitStatus));
  headers.set("x-gateway-cost-action", action);

  return NextResponse.next({
    request: {
      headers,
    },
  });
}
```

### 11.2 Applying the Middleware

```typescript
// middleware.ts — Next.js middleware configuration
import { aiGatewayMiddleware } from "./middleware/ai-gateway";
import { NextRequest } from "next/server";

export async function middleware(req: NextRequest) {
  // Only apply AI gateway to AI-related routes
  if (req.nextUrl.pathname.startsWith("/api/chat") ||
      req.nextUrl.pathname.startsWith("/api/ai")) {
    return aiGatewayMiddleware(req);
  }
}

export const config = {
  matcher: ["/api/chat/:path*", "/api/ai/:path*"],
};
```

---

## 12. Monitoring the Gateway

### 12.1 Gateway Metrics

```typescript
// lib/gateway-metrics.ts — track key gateway metrics
interface GatewayMetrics {
  requestCount: number;
  cacheHits: number;
  cacheMisses: number;
  providerErrors: Record<string, number>;
  rateLimitHits: number;
  avgLatencyMs: number;
  tokenUsage: {
    input: number;
    output: number;
  };
  costCents: number;
}

class GatewayMetricsCollector {
  private metrics: GatewayMetrics = {
    requestCount: 0,
    cacheHits: 0,
    cacheMisses: 0,
    providerErrors: {},
    rateLimitHits: 0,
    avgLatencyMs: 0,
    tokenUsage: { input: 0, output: 0 },
    costCents: 0,
  };

  private latencies: number[] = [];

  recordRequest(provider: string, latencyMs: number, tokens: { input: number; output: number }) {
    this.metrics.requestCount++;
    this.latencies.push(latencyMs);
    this.metrics.tokenUsage.input += tokens.input;
    this.metrics.tokenUsage.output += tokens.output;

    // Keep rolling window of latencies
    if (this.latencies.length > 1000) {
      this.latencies = this.latencies.slice(-1000);
    }

    this.metrics.avgLatencyMs =
      this.latencies.reduce((a, b) => a + b, 0) / this.latencies.length;
  }

  recordCacheHit() { this.metrics.cacheHits++; }
  recordCacheMiss() { this.metrics.cacheMisses++; }

  recordError(provider: string) {
    this.metrics.providerErrors[provider] =
      (this.metrics.providerErrors[provider] || 0) + 1;
  }

  recordRateLimitHit() { this.metrics.rateLimitHits++; }

  getMetrics(): GatewayMetrics & { cacheHitRate: number } {
    const total = this.metrics.cacheHits + this.metrics.cacheMisses;
    return {
      ...this.metrics,
      cacheHitRate: total > 0 ? this.metrics.cacheHits / total : 0,
    };
  }
}

export const gatewayMetrics = new GatewayMetricsCollector();
```

### 12.2 Metrics Dashboard Endpoint

```typescript
// app/api/admin/gateway-metrics/route.ts
import { gatewayMetrics } from "@/lib/gateway-metrics";

export async function GET(req: Request) {
  // Protect this endpoint — admin only
  const authHeader = req.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.ADMIN_API_KEY}`) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  return Response.json(gatewayMetrics.getMetrics());
}
```

---

## 13. What's Next

You now have a gateway between your application and your LLM providers. It handles rate limiting, retry with backoff, provider failover, caching, and cost controls. Your AI application is resilient to provider outages and protected from cost overruns.

But there is a class of AI workload that needs more than a gateway: agents with tools. An agent that can search the web, read files, and call APIs is fundamentally dangerous. It can access production data it should not see. It can make network requests to systems it should not reach. It can be manipulated by prompt injection into taking actions you never intended.

In Chapter 55, we will go deep on **sandboxing and isolating agents**. You will learn the OpenClaw pattern -- how the Nelo engineering team deployed an agent with zero access to production systems, zero inbound internet access, and egress restricted to an explicit allowlist of domains. This is the gold standard for agent isolation, and it requires infrastructure-level controls that no amount of application code can replace.

---

## Key Takeaways

1. **An API gateway is not optional for production AI.** Direct provider access means no rate limiting, no failover, no cost controls, and no audit trail.
2. **Rate limit by tokens, not just requests.** One AI request can cost 1,000x more than another.
3. **Retry with exponential backoff and jitter.** Provider outages are frequent. Your retry strategy is the difference between "some requests were slower" and "the feature was down."
4. **Provider failover requires bounded failure detection.** Use circuit breakers to prevent cascading failures when a provider is down.
5. **Cache selectively.** Classification and extraction are safe to cache. Chat conversations and tool calls are not.
6. **Cost controls must be enforced at the infrastructure level.** Application-level budgets can be bypassed. Gateway-level controls cannot.

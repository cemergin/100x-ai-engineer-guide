<!--
  CHAPTER: 22
  TITLE: Telemetry & Tracing
  PART: 4 — Evals & Quality
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 21 (Eval-Driven Development)
  KEY_TOPICS: OpenTelemetry, Laminar, Datadog LLM observability, token usage monitoring, cost per feature, cost per user, instrumentation, tracing, spans, latency tracking
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 22: Telemetry & Tracing

> Part 4: Evals & Quality · Phase 1: Get Dangerous · Prerequisites: Ch 21 · Difficulty: Intermediate · Language: TypeScript

You've built assessments. You know how to measure quality offline. But what happens in production? Users don't file eval reports. They just leave. Or they send angry emails. Or they silently use the feature less and less.

Telemetry and tracing are how you know what's happening in production. How long is each LLM call taking? How many tokens are you burning per request? Which features cost the most? Is quality degrading over time? Which users are hitting edge cases?

This chapter instruments your AI system so you can see everything, catch problems early, and make data-driven decisions about cost, quality, and architecture.

### In This Chapter

1. Why telemetry for LLM applications
2. OpenTelemetry fundamentals
3. Laminar for LLM tracing
4. Datadog LLM observability
5. Token usage and cost monitoring
6. Setting up instrumentation that doesn't slow you down

### Related Chapters

- **Ch 21 (Eval-Driven Development)** — Tracing supports the eval loop.
- **Ch 51 (Production Eval Pipelines)** — Production assessment pipelines built on telemetry.
- **Ch 52 (AI Observability & Incidents)** — Full observability stack for AI.

---

## 1. Why Telemetry for LLM Applications

### 1.1 The Unique Challenges

LLM applications have telemetry needs that traditional web apps don't.

```typescript
// why-telemetry.ts

// Traditional web app monitoring:
// - Request latency: 50-200ms
// - Cost per request: negligible
// - Output: deterministic
// - Errors: exceptions and status codes

// LLM application monitoring:
// - Request latency: 500ms-30s (10-100x more variable)
// - Cost per request: $0.001-$1.00 (real money)
// - Output: non-deterministic (same input, different output)
// - Errors: hallucinations, wrong tools, bad format (no exception thrown)
// - Multi-step: agent loops have many LLM calls per user request

interface LLMTelemetryNeeds {
  latency: {
    perCall: "How long does each LLM call take?";
    perStep: "How long does each agent step take?";
    endToEnd: "How long from user input to final response?";
    streaming: "Time to first token vs total response time";
  };
  cost: {
    promptTokens: "How many input tokens per call?";
    completionTokens: "How many output tokens per call?";
    totalCost: "Dollar cost per request, per feature, per user";
    modelBreakdown: "Cost by model (GPT-4o vs GPT-4o-mini)";
  };
  quality: {
    toolAccuracy: "Did the agent pick the right tool?";
    formatCompliance: "Did the response match the expected schema?";
    userFeedback: "Thumbs up/down, ratings";
    assessmentScores: "Automated quality scores on sampled traffic";
  };
  debugging: {
    fullTrace: "Every step of the agent loop with inputs/outputs";
    promptHistory: "What system prompt was used?";
    contextUsed: "What RAG context was retrieved?";
    errorChains: "Which step failed and why?";
  };
}
```

---

## 2. OpenTelemetry Fundamentals

### 2.1 Setting Up OpenTelemetry

OpenTelemetry (OTel) is the standard for application observability. It works with every backend (Datadog, Jaeger, Grafana, etc.).

```typescript
// otel-setup.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { Resource } from "@opentelemetry/resources";
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from "@opentelemetry/semantic-conventions";

// Initialize OpenTelemetry
const sdk = new NodeSDK({
  resource: new Resource({
    [ATTR_SERVICE_NAME]: "ai-support-bot",
    [ATTR_SERVICE_VERSION]: "1.2.0",
    "deployment.environment": process.env.NODE_ENV ?? "development",
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/traces",
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/metrics",
    }),
    exportIntervalMillis: 30_000, // Export every 30 seconds
  }),
});

sdk.start();

// Graceful shutdown
process.on("SIGTERM", () => sdk.shutdown());
process.on("SIGINT", () => sdk.shutdown());

export { sdk };
```

### 2.2 Tracing LLM Calls

```typescript
// otel-llm-tracing.ts
import { trace, context, SpanKind, SpanStatusCode } from "@opentelemetry/api";
import { metrics } from "@opentelemetry/api";
import OpenAI from "openai";

const tracer = trace.getTracer("ai-support-bot", "1.0.0");
const meter = metrics.getMeter("ai-support-bot", "1.0.0");

// Define metrics
const llmCallDuration = meter.createHistogram("llm.call.duration", {
  description: "Duration of LLM API calls in milliseconds",
  unit: "ms",
});

const llmTokensUsed = meter.createCounter("llm.tokens.total", {
  description: "Total tokens used across all LLM calls",
});

const llmCallErrors = meter.createCounter("llm.call.errors", {
  description: "Number of failed LLM calls",
});

const llmCost = meter.createCounter("llm.cost.dollars", {
  description: "Estimated cost of LLM calls in dollars",
});

// Traced LLM call wrapper
async function tracedLLMCall(
  openai: OpenAI,
  params: OpenAI.ChatCompletionCreateParamsNonStreaming,
  metadata: {
    feature?: string;
    userId?: string;
    requestId?: string;
  } = {}
): Promise<OpenAI.ChatCompletion> {
  return tracer.startActiveSpan(
    "llm.chat_completion",
    {
      kind: SpanKind.CLIENT,
      attributes: {
        "llm.model": params.model,
        "llm.temperature": params.temperature ?? 1,
        "llm.max_tokens": params.max_tokens ?? -1,
        "llm.message_count": params.messages.length,
        "app.feature": metadata.feature ?? "unknown",
        "app.user_id": metadata.userId ?? "anonymous",
        "app.request_id": metadata.requestId ?? "none",
      },
    },
    async (span) => {
      const startTime = Date.now();

      try {
        const response = await openai.chat.completions.create(params);
        const duration = Date.now() - startTime;

        // Record token usage
        const promptTokens = response.usage?.prompt_tokens ?? 0;
        const completionTokens = response.usage?.completion_tokens ?? 0;
        const totalTokens = promptTokens + completionTokens;

        // Set span attributes
        span.setAttributes({
          "llm.prompt_tokens": promptTokens,
          "llm.completion_tokens": completionTokens,
          "llm.total_tokens": totalTokens,
          "llm.duration_ms": duration,
          "llm.finish_reason": response.choices[0]?.finish_reason ?? "unknown",
        });

        // Record metrics
        llmCallDuration.record(duration, {
          model: params.model,
          feature: metadata.feature ?? "unknown",
        });

        llmTokensUsed.add(totalTokens, {
          model: params.model,
          token_type: "total",
          feature: metadata.feature ?? "unknown",
        });

        // Estimate cost
        const cost = estimateCost(params.model, promptTokens, completionTokens);
        llmCost.add(cost, {
          model: params.model,
          feature: metadata.feature ?? "unknown",
        });

        span.setStatus({ code: SpanStatusCode.OK });
        return response;
      } catch (error) {
        const duration = Date.now() - startTime;

        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error instanceof Error ? error.message : "Unknown error",
        });
        span.recordException(error as Error);

        llmCallErrors.add(1, {
          model: params.model,
          error_type: error instanceof Error ? error.constructor.name : "unknown",
        });
        llmCallDuration.record(duration, {
          model: params.model,
          feature: metadata.feature ?? "unknown",
          error: "true",
        });

        throw error;
      } finally {
        span.end();
      }
    }
  );
}

function estimateCost(
  model: string,
  promptTokens: number,
  completionTokens: number
): number {
  // Pricing per 1M tokens (approximate, check current pricing)
  const pricing: Record<string, { input: number; output: number }> = {
    "gpt-4o": { input: 2.50, output: 10.00 },
    "gpt-4o-mini": { input: 0.15, output: 0.60 },
    "gpt-4-turbo": { input: 10.00, output: 30.00 },
  };

  const modelPricing = pricing[model] ?? pricing["gpt-4o-mini"];
  return (
    (promptTokens / 1_000_000) * modelPricing.input +
    (completionTokens / 1_000_000) * modelPricing.output
  );
}
```

### 2.3 Tracing Agent Loops

Agent loops involve multiple LLM calls. Create parent spans that contain child spans for each step.

```typescript
// otel-agent-tracing.ts
import { trace, SpanKind, SpanStatusCode } from "@opentelemetry/api";
import OpenAI from "openai";

const tracer = trace.getTracer("ai-agent", "1.0.0");
const openai = new OpenAI();

interface AgentStep {
  type: "llm_call" | "tool_execution" | "retrieval";
  name: string;
  durationMs: number;
  tokens?: number;
  metadata: Record<string, unknown>;
}

async function tracedAgentLoop(
  userMessage: string,
  systemPrompt: string,
  metadata: { feature?: string; userId?: string } = {}
): Promise<{ response: string; steps: AgentStep[] }> {
  return tracer.startActiveSpan(
    "agent.loop",
    {
      kind: SpanKind.SERVER,
      attributes: {
        "agent.user_message": userMessage.slice(0, 200),
        "app.feature": metadata.feature ?? "unknown",
        "app.user_id": metadata.userId ?? "anonymous",
      },
    },
    async (parentSpan) => {
      const steps: AgentStep[] = [];
      const messages: OpenAI.ChatCompletionMessageParam[] = [
        { role: "system", content: systemPrompt },
        { role: "user", content: userMessage },
      ];

      let maxIterations = 10;
      let response = "";

      while (maxIterations-- > 0) {
        // Step: LLM call
        const llmStep = await tracer.startActiveSpan(
          "agent.step.llm",
          { attributes: { "agent.iteration": 10 - maxIterations } },
          async (span) => {
            const start = Date.now();

            const completion = await openai.chat.completions.create({
              model: "gpt-4o-mini",
              messages,
            });

            const duration = Date.now() - start;
            const tokens = completion.usage?.total_tokens ?? 0;

            span.setAttributes({
              "llm.tokens": tokens,
              "llm.duration_ms": duration,
              "llm.finish_reason": completion.choices[0]?.finish_reason ?? "unknown",
            });
            span.end();

            steps.push({
              type: "llm_call",
              name: "chat_completion",
              durationMs: duration,
              tokens,
              metadata: { model: "gpt-4o-mini" },
            });

            return completion;
          }
        );

        const choice = llmStep.choices[0];

        // Check if the agent is done
        if (choice.finish_reason === "stop" || !choice.message.tool_calls) {
          response = choice.message.content ?? "";
          break;
        }

        // Step: Tool execution
        for (const toolCall of choice.message.tool_calls ?? []) {
          await tracer.startActiveSpan(
            `agent.step.tool.${toolCall.function.name}`,
            {
              attributes: {
                "tool.name": toolCall.function.name,
                "tool.args": toolCall.function.arguments.slice(0, 500),
              },
            },
            async (toolSpan) => {
              const start = Date.now();

              try {
                // Execute the tool (mock for this example)
                const result = `Tool result for ${toolCall.function.name}`;
                const duration = Date.now() - start;

                toolSpan.setAttributes({
                  "tool.duration_ms": duration,
                  "tool.success": true,
                });

                steps.push({
                  type: "tool_execution",
                  name: toolCall.function.name,
                  durationMs: duration,
                  metadata: { args: toolCall.function.arguments },
                });

                // Add tool result to messages
                messages.push({
                  role: "tool",
                  content: result,
                  tool_call_id: toolCall.id,
                });
              } catch (error) {
                toolSpan.setStatus({
                  code: SpanStatusCode.ERROR,
                  message: (error as Error).message,
                });
              } finally {
                toolSpan.end();
              }
            }
          );
        }

        messages.push(choice.message);
      }

      parentSpan.setAttributes({
        "agent.total_steps": steps.length,
        "agent.total_tokens": steps.reduce((sum, s) => sum + (s.tokens ?? 0), 0),
        "agent.response_length": response.length,
      });
      parentSpan.end();

      return { response, steps };
    }
  );
}
```

---

## 3. Laminar for LLM Tracing

### 3.1 What Is Laminar?

Laminar is a purpose-built observability platform for LLM applications. It provides tracing, logging, and assessment capabilities specifically designed for AI workloads.

```typescript
// laminar-setup.ts
import { Laminar, observe } from "@lmnr-ai/lmnr";

// Initialize Laminar
Laminar.initialize({
  projectApiKey: process.env.LAMINAR_API_KEY!,
});

// The observe decorator automatically traces function calls
const tracedFunction = observe(
  { name: "customer-support-query" },
  async (query: string, context: string) => {
    const openai = new (await import("openai")).default();

    const response = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [
        { role: "system", content: `Answer based on context: ${context}` },
        { role: "user", content: query },
      ],
    });

    return response.choices[0].message.content ?? "";
  }
);

// Nested tracing — each sub-function gets its own span
const tracedRAGPipeline = observe(
  { name: "rag-pipeline" },
  async (query: string) => {
    // Step 1: Retrieve
    const chunks = await observe(
      { name: "retrieve" },
      async () => {
        // Your retrieval logic here
        return [{ content: "relevant chunk", score: 0.9 }];
      }
    )();

    // Step 2: Generate
    const answer = await observe(
      { name: "generate" },
      async () => {
        const openai = new (await import("openai")).default();
        const response = await openai.chat.completions.create({
          model: "gpt-4o-mini",
          messages: [
            {
              role: "system",
              content: `Context: ${chunks.map(c => c.content).join("\n")}`,
            },
            { role: "user", content: query },
          ],
        });
        return response.choices[0].message.content ?? "";
      }
    )();

    return { answer, sources: chunks };
  }
);
```

---

## 4. Datadog LLM Observability

### 4.1 Setting Up Datadog for LLM Monitoring

```typescript
// datadog-llm-setup.ts

// Datadog provides LLM observability through its APM and Logs products.
// The ddtrace library instruments OpenAI calls automatically.

// Step 1: Install and configure
// npm install dd-trace @datadog/datadog-api-client

// Step 2: Initialize tracing (do this before importing OpenAI)
import tracer from "dd-trace";

tracer.init({
  service: "ai-support-bot",
  env: process.env.DD_ENV ?? "development",
  version: "1.2.0",
  // Enable LLM observability
  plugins: true,
});

// The OpenAI integration is automatic with dd-trace
// It captures:
// - Request/response timing
// - Token usage
// - Model used
// - Error rates
// - Prompt/completion text (configurable)

import OpenAI from "openai";

const openai = new OpenAI();

// All OpenAI calls are now automatically traced
async function handleQuery(query: string): Promise<string> {
  // This call is automatically instrumented by dd-trace
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: query }],
  });

  return response.choices[0].message.content ?? "";
}
```

### 4.2 Custom Metrics for Datadog

```typescript
// datadog-custom-metrics.ts
import { StatsD } from "hot-shots";

const dogstatsd = new StatsD({
  host: process.env.DD_AGENT_HOST ?? "localhost",
  port: 8125,
  prefix: "ai_bot.",
  globalTags: [`env:${process.env.DD_ENV ?? "development"}`],
});

// Track LLM-specific metrics
function trackLLMCall(params: {
  model: string;
  feature: string;
  promptTokens: number;
  completionTokens: number;
  latencyMs: number;
  success: boolean;
  userId?: string;
}) {
  const tags = [
    `model:${params.model}`,
    `feature:${params.feature}`,
    `success:${params.success}`,
  ];

  // Latency
  dogstatsd.histogram("llm.latency", params.latencyMs, tags);

  // Tokens
  dogstatsd.increment("llm.tokens.prompt", params.promptTokens, tags);
  dogstatsd.increment("llm.tokens.completion", params.completionTokens, tags);
  dogstatsd.increment("llm.tokens.total", params.promptTokens + params.completionTokens, tags);

  // Cost (estimated)
  const cost = estimateCost(params.model, params.promptTokens, params.completionTokens);
  dogstatsd.increment("llm.cost.microdollars", Math.round(cost * 1_000_000), tags);

  // Per-user tracking
  if (params.userId) {
    dogstatsd.increment("llm.calls_per_user", 1, [...tags, `user:${params.userId}`]);
  }

  // Error tracking
  if (!params.success) {
    dogstatsd.increment("llm.errors", 1, tags);
  }
}

function estimateCost(model: string, prompt: number, completion: number): number {
  const rates: Record<string, { i: number; o: number }> = {
    "gpt-4o": { i: 2.5, o: 10 },
    "gpt-4o-mini": { i: 0.15, o: 0.6 },
  };
  const r = rates[model] ?? rates["gpt-4o-mini"];
  return (prompt / 1e6) * r.i + (completion / 1e6) * r.o;
}

// Track quality metrics
function trackQualityScore(params: {
  feature: string;
  metric: string;
  score: number;
  userId?: string;
}) {
  dogstatsd.histogram(`quality.${params.metric}`, params.score, [
    `feature:${params.feature}`,
    ...(params.userId ? [`user:${params.userId}`] : []),
  ]);
}

// Track user feedback
function trackFeedback(params: {
  feature: string;
  type: "thumbs_up" | "thumbs_down" | "rating";
  value: number;
  userId: string;
}) {
  dogstatsd.increment(`feedback.${params.type}`, 1, [
    `feature:${params.feature}`,
    `user:${params.userId}`,
  ]);

  if (params.type === "rating") {
    dogstatsd.histogram("feedback.rating_value", params.value, [
      `feature:${params.feature}`,
    ]);
  }
}
```

---

## 5. Token Usage and Cost Monitoring

### 5.1 Cost Tracking Per Feature

```typescript
// cost-tracker.ts

interface CostEntry {
  timestamp: Date;
  feature: string;
  model: string;
  promptTokens: number;
  completionTokens: number;
  cost: number;
  userId?: string;
  requestId: string;
}

class CostTracker {
  private entries: CostEntry[] = [];
  private budgets: Map<string, { daily: number; monthly: number }> = new Map();

  record(entry: CostEntry) {
    this.entries.push(entry);
    this.checkBudget(entry.feature);
  }

  // Set budget alerts
  setBudget(feature: string, daily: number, monthly: number) {
    this.budgets.set(feature, { daily, monthly });
  }

  private checkBudget(feature: string) {
    const budget = this.budgets.get(feature);
    if (!budget) return;

    const daily = this.getCost(feature, "day");
    const monthly = this.getCost(feature, "month");

    if (daily > budget.daily * 0.8) {
      console.warn(`BUDGET ALERT: ${feature} daily cost $${daily.toFixed(4)} is at ${((daily / budget.daily) * 100).toFixed(0)}% of $${budget.daily} budget`);
    }
    if (monthly > budget.monthly * 0.8) {
      console.warn(`BUDGET ALERT: ${feature} monthly cost $${monthly.toFixed(2)} is at ${((monthly / budget.monthly) * 100).toFixed(0)}% of $${budget.monthly} budget`);
    }
  }

  getCost(feature: string, period: "day" | "week" | "month"): number {
    const now = new Date();
    const cutoff = new Date();

    if (period === "day") cutoff.setDate(now.getDate() - 1);
    else if (period === "week") cutoff.setDate(now.getDate() - 7);
    else cutoff.setMonth(now.getMonth() - 1);

    return this.entries
      .filter(e => e.feature === feature && e.timestamp >= cutoff)
      .reduce((sum, e) => sum + e.cost, 0);
  }

  // Cost breakdown by feature
  getCostBreakdown(period: "day" | "week" | "month"): Record<string, number> {
    const now = new Date();
    const cutoff = new Date();

    if (period === "day") cutoff.setDate(now.getDate() - 1);
    else if (period === "week") cutoff.setDate(now.getDate() - 7);
    else cutoff.setMonth(now.getMonth() - 1);

    const filtered = this.entries.filter(e => e.timestamp >= cutoff);
    const byFeature: Record<string, number> = {};

    for (const entry of filtered) {
      byFeature[entry.feature] = (byFeature[entry.feature] ?? 0) + entry.cost;
    }

    return byFeature;
  }

  // Cost per user
  getCostByUser(period: "day" | "week" | "month"): Record<string, number> {
    const now = new Date();
    const cutoff = new Date();

    if (period === "day") cutoff.setDate(now.getDate() - 1);
    else if (period === "week") cutoff.setDate(now.getDate() - 7);
    else cutoff.setMonth(now.getMonth() - 1);

    const filtered = this.entries.filter(e => e.timestamp >= cutoff && e.userId);
    const byUser: Record<string, number> = {};

    for (const entry of filtered) {
      const userId = entry.userId ?? "anonymous";
      byUser[userId] = (byUser[userId] ?? 0) + entry.cost;
    }

    return byUser;
  }

  // Print a cost report
  printReport(period: "day" | "week" | "month" = "day") {
    const byFeature = this.getCostBreakdown(period);
    const totalCost = Object.values(byFeature).reduce((s, c) => s + c, 0);

    console.log(`\n=== Cost Report (${period}) ===`);
    console.log(`Total: $${totalCost.toFixed(4)}`);
    console.log("\nBy feature:");
    for (const [feature, cost] of Object.entries(byFeature).sort((a, b) => b[1] - a[1])) {
      const pct = ((cost / totalCost) * 100).toFixed(1);
      console.log(`  ${feature}: $${cost.toFixed(4)} (${pct}%)`);
    }
  }
}

// Usage
const costTracker = new CostTracker();
costTracker.setBudget("support-bot", 5, 100);   // $5/day, $100/month
costTracker.setBudget("doc-search", 10, 200);    // $10/day, $200/month
```

### 5.2 Wrapping It All Together

```typescript
// instrumented-llm-client.ts
import OpenAI from "openai";
import crypto from "crypto";

interface InstrumentedClientConfig {
  serviceName: string;
  defaultFeature: string;
  costTracker: CostTracker;
  onCall?: (params: {
    model: string;
    feature: string;
    latencyMs: number;
    tokens: number;
    cost: number;
    userId?: string;
  }) => void;
}

class InstrumentedOpenAI {
  private openai: OpenAI;
  private config: InstrumentedClientConfig;

  constructor(config: InstrumentedClientConfig) {
    this.openai = new OpenAI();
    this.config = config;
  }

  async chatCompletion(
    params: OpenAI.ChatCompletionCreateParamsNonStreaming,
    metadata: {
      feature?: string;
      userId?: string;
      requestId?: string;
    } = {}
  ): Promise<OpenAI.ChatCompletion> {
    const feature = metadata.feature ?? this.config.defaultFeature;
    const requestId = metadata.requestId ?? crypto.randomUUID();
    const start = Date.now();

    try {
      const response = await this.openai.chat.completions.create(params);
      const latencyMs = Date.now() - start;
      const promptTokens = response.usage?.prompt_tokens ?? 0;
      const completionTokens = response.usage?.completion_tokens ?? 0;
      const cost = estimateCostForModel(params.model, promptTokens, completionTokens);

      // Record to cost tracker
      this.config.costTracker.record({
        timestamp: new Date(),
        feature,
        model: params.model,
        promptTokens,
        completionTokens,
        cost,
        userId: metadata.userId,
        requestId,
      });

      // Callback for custom metrics
      this.config.onCall?.({
        model: params.model,
        feature,
        latencyMs,
        tokens: promptTokens + completionTokens,
        cost,
        userId: metadata.userId,
      });

      return response;
    } catch (error) {
      const latencyMs = Date.now() - start;
      console.error(
        `LLM call failed: ${feature} ${params.model} ${latencyMs}ms`,
        error
      );
      throw error;
    }
  }

  async embed(
    texts: string[],
    metadata: { feature?: string; userId?: string } = {}
  ): Promise<number[][]> {
    const feature = metadata.feature ?? this.config.defaultFeature;
    const start = Date.now();

    const response = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: texts,
    });

    const latencyMs = Date.now() - start;
    const totalTokens = response.usage?.total_tokens ?? 0;
    const cost = (totalTokens / 1_000_000) * 0.02;

    this.config.costTracker.record({
      timestamp: new Date(),
      feature,
      model: "text-embedding-3-small",
      promptTokens: totalTokens,
      completionTokens: 0,
      cost,
      userId: metadata.userId,
      requestId: crypto.randomUUID(),
    });

    return response.data.map(d => d.embedding);
  }
}

function estimateCostForModel(model: string, prompt: number, completion: number): number {
  const rates: Record<string, { i: number; o: number }> = {
    "gpt-4o": { i: 2.5, o: 10 },
    "gpt-4o-mini": { i: 0.15, o: 0.6 },
  };
  const r = rates[model] ?? rates["gpt-4o-mini"];
  return (prompt / 1e6) * r.i + (completion / 1e6) * r.o;
}

// --- Usage ---

async function main() {
  const tracker = new CostTracker();
  tracker.setBudget("support", 5, 100);

  const client = new InstrumentedOpenAI({
    serviceName: "support-bot",
    defaultFeature: "support",
    costTracker: tracker,
    onCall: (params) => {
      console.log(
        `[${params.feature}] ${params.model} — ${params.latencyMs}ms, ` +
        `${params.tokens} tokens, $${params.cost.toFixed(6)}`
      );
    },
  });

  // Use the instrumented client
  const response = await client.chatCompletion(
    {
      model: "gpt-4o-mini",
      messages: [
        { role: "system", content: "You are a support agent." },
        { role: "user", content: "How much is Pro?" },
      ],
    },
    { feature: "support", userId: "user-123" }
  );

  console.log(`\nResponse: ${response.choices[0].message.content}`);

  // Print cost report
  tracker.printReport("day");
}

main();
```

---

## 6. Instrumentation Without Slowing Down

### 6.1 Async Telemetry

Never let telemetry block your response path.

```typescript
// async-telemetry.ts

class AsyncTelemetryBuffer {
  private buffer: Record<string, unknown>[] = [];
  private flushInterval: ReturnType<typeof setInterval>;
  private maxBufferSize: number;
  private flushFn: (entries: Record<string, unknown>[]) => Promise<void>;

  constructor(
    flushFn: (entries: Record<string, unknown>[]) => Promise<void>,
    options: { flushIntervalMs?: number; maxBufferSize?: number } = {}
  ) {
    this.flushFn = flushFn;
    this.maxBufferSize = options.maxBufferSize ?? 100;

    this.flushInterval = setInterval(() => {
      this.flush().catch(console.error);
    }, options.flushIntervalMs ?? 10_000);
  }

  record(entry: Record<string, unknown>) {
    this.buffer.push({ ...entry, timestamp: new Date().toISOString() });

    if (this.buffer.length >= this.maxBufferSize) {
      this.flush().catch(console.error);
    }
  }

  private async flush() {
    if (this.buffer.length === 0) return;

    const entries = [...this.buffer];
    this.buffer = [];

    try {
      await this.flushFn(entries);
    } catch (error) {
      // Don't lose data — put it back
      this.buffer.unshift(...entries);
      console.error("Telemetry flush failed:", error);
    }
  }

  async shutdown() {
    clearInterval(this.flushInterval);
    await this.flush();
  }
}

// Usage: telemetry that doesn't add latency to the hot path
const telemetry = new AsyncTelemetryBuffer(
  async (entries) => {
    // Send to your backend (Datadog, custom API, etc.)
    // This happens in the background
    console.log(`Flushing ${entries.length} telemetry entries`);
  },
  { flushIntervalMs: 5000, maxBufferSize: 50 }
);

// In your request handler:
async function handleRequest(query: string): Promise<string> {
  const start = Date.now();

  // Do the actual work
  const result = await processQuery(query);

  // Record telemetry asynchronously — does NOT block the response
  telemetry.record({
    type: "llm_request",
    query: query.slice(0, 200),
    latencyMs: Date.now() - start,
    responseLength: result.length,
  });

  return result;
}

async function processQuery(query: string): Promise<string> {
  return "response"; // Placeholder
}

// Cleanup on shutdown
process.on("SIGTERM", () => telemetry.shutdown());
```

### 6.2 Sampling for High-Traffic Systems

```typescript
// sampled-telemetry.ts

class SampledTelemetry {
  private sampleRate: number;
  private detailedSampleRate: number;

  constructor(options: {
    sampleRate?: number;         // % of requests to record basic telemetry (1.0 = 100%)
    detailedSampleRate?: number; // % of sampled requests to record full details
  } = {}) {
    this.sampleRate = options.sampleRate ?? 1.0;       // Default: record everything
    this.detailedSampleRate = options.detailedSampleRate ?? 0.1; // Default: 10% get full details
  }

  shouldRecord(): boolean {
    return Math.random() < this.sampleRate;
  }

  shouldRecordDetailed(): boolean {
    return Math.random() < this.detailedSampleRate;
  }

  // Wrap an LLM call with sampled telemetry
  async tracedCall<T>(
    name: string,
    fn: () => Promise<T>,
    metadata: Record<string, unknown> = {}
  ): Promise<T> {
    if (!this.shouldRecord()) {
      return fn(); // Skip telemetry entirely
    }

    const start = Date.now();
    const detailed = this.shouldRecordDetailed();

    try {
      const result = await fn();
      const duration = Date.now() - start;

      // Always record basic metrics
      console.log(`[METRIC] ${name}: ${duration}ms`);

      // Only record detailed traces for sampled requests
      if (detailed) {
        console.log(`[TRACE] ${name}: ${JSON.stringify(metadata).slice(0, 500)}`);
      }

      return result;
    } catch (error) {
      // Always record errors (don't sample errors)
      console.error(`[ERROR] ${name}: ${(error as Error).message}`);
      throw error;
    }
  }
}

// For 1000 req/s: basic telemetry on 100%, detailed on 1%
const highTrafficTelemetry = new SampledTelemetry({
  sampleRate: 1.0,
  detailedSampleRate: 0.01,
});

// For 10 req/s: basic and detailed on everything
const lowTrafficTelemetry = new SampledTelemetry({
  sampleRate: 1.0,
  detailedSampleRate: 1.0,
});
```

---

## Summary

You now have a complete telemetry stack for LLM applications:

1. **OpenTelemetry setup** — Standard observability with traces and metrics
2. **LLM call tracing** — Every API call tracked with tokens, latency, and cost
3. **Agent loop tracing** — Parent-child spans for multi-step workflows
4. **Laminar integration** — Purpose-built LLM tracing and assessment
5. **Datadog LLM observability** — Production monitoring with alerts
6. **Cost tracking** — Per feature, per user, per model with budget alerts
7. **Non-blocking instrumentation** — Async buffers and sampling for high traffic

This telemetry is the foundation for production assessment pipelines (Ch 51) and full AI observability (Ch 52) in Phase 2. But even at Phase 1 depth, you can now see what your AI system is doing, how much it costs, and where it's struggling.

---

*Previous: [Ch 21 — Eval-Driven Development](./21-eval-driven-dev.md)* | *Next: [Ch 23 — Claude Code Mastery](../part-5-harness-engineering/23-claude-code-mastery.md)*

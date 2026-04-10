<!--
  CHAPTER: 52
  TITLE: AI Observability & Incidents
  PART: 10 — Production AI Systems
  PHASE: 2 — Become an Expert
  PREREQS: Ch 22 (telemetry & tracing), Ch 51 (production evals)
  KEY_TOPICS: LLM observability, Datadog LLM monitoring, hallucination detection, degradation alerts, cost anomaly detection, debugging non-deterministic systems, incident response, postmortem patterns
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 52: AI Observability & Incidents

> **Part 10 — Production AI Systems** | Phase 2: Become an Expert | Prerequisites: Ch 22, Ch 51 | Difficulty: Advanced | Language: TypeScript + Python

It's 2 AM and your AI customer support system just told a user they can return a product for a full refund — a product you haven't sold in three years, with a return policy that doesn't exist. The user screenshotted it and posted it on social media. Now it's going viral.

How did this happen? When did it start? Is it still happening? How many users were affected? Can you reproduce it? How do you fix it?

These are the questions AI observability answers. Traditional observability (logs, metrics, traces) tells you *that* something went wrong. AI observability tells you *why* the model said what it said — which is fundamentally harder when your system is non-deterministic and its "reasoning" is a black box.

### In This Chapter
- The LLM observability stack: what to monitor and why
- Datadog LLM observability: traces, spans, cost tracking
- Hallucination detection in production
- Degradation alerts: when quality drops silently
- Cost anomaly detection: catching unexpected spending
- Debugging LLM failures: the challenge of non-deterministic systems
- Incident response for AI: a runbook template
- Postmortem patterns: what to document after an AI incident

### Related Chapters
- **Ch 22 (Telemetry & Tracing)** — spirals back: basic telemetry to full observability
- **Ch 51 (Production Eval Pipelines)** — spirals back: evals trigger alerts
- **Ch 56 (CI/CD for AI Applications)** — spirals forward: deployment monitoring
- **Ch 62 (AI Adoption & Enablement)** — spirals forward: org-level observability

---

## 1. The LLM Observability Stack

### 1.1 What Makes LLM Observability Different

Traditional observability tracks request/response patterns: latency, error rates, throughput. LLM observability adds dimensions that don't exist in traditional systems:

| Dimension | Traditional | LLM-Specific |
|-----------|-------------|--------------|
| Errors | HTTP 500, exceptions | Hallucinations, wrong tool calls, off-topic responses |
| Latency | Response time | Time to first token, tokens per second, total generation time |
| Cost | Compute, bandwidth | Input tokens, output tokens, cached tokens, per-model pricing |
| Quality | N/A (correct or not) | Relevance score, accuracy score, helpfulness score (all continuous) |
| Capacity | CPU, memory | Context window utilization, rate limit proximity |
| Dependencies | Service health | Model provider availability, model version changes |

### 1.2 The Five Pillars of LLM Observability

```
┌────────────────────────────────────────────────────────────┐
│                  LLM Observability Stack                    │
├──────────┬──────────┬──────────┬──────────┬───────────────┤
│ Traces   │ Metrics  │ Quality  │ Cost     │ Alerts        │
│          │          │          │          │               │
│ Full     │ Latency  │ Eval     │ Token    │ Degradation   │
│ request  │ Token    │ scores   │ usage    │ Cost spikes   │
│ traces   │ counts   │ Halluci- │ Per-     │ Safety        │
│ with     │ Error    │ nation   │ feature  │ violations    │
│ prompts  │ rates    │ rates    │ Per-     │ Model         │
│ and      │ Cache    │ User     │ user     │ failures      │
│ responses│ hit rate │ feedback │ Per-     │               │
│          │          │          │ model    │               │
└──────────┴──────────┴──────────┴──────────┴───────────────┘
```

---

## 2. Implementing LLM Traces

### 2.1 Trace Structure

An LLM trace captures the full lifecycle of a request, including the prompt sent, the response received, any tool calls made, and quality scores.

```typescript
// TypeScript: LLM trace data model

interface LLMTrace {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  
  // Request
  timestamp: Date;
  feature: string;
  userId: string;
  model: string;
  
  // Prompt
  systemPrompt: string;
  messages: Array<{ role: string; content: string }>;
  tools?: Array<{ name: string; description: string }>;
  
  // Response
  response: string;
  toolCalls?: Array<{
    toolName: string;
    arguments: Record<string, unknown>;
    result: string;
  }>;
  stopReason: string;
  
  // Metrics
  inputTokens: number;
  outputTokens: number;
  cachedInputTokens: number;
  latencyMs: number;
  timeToFirstTokenMs: number;
  cost: number;
  
  // Quality
  evalScores?: Record<string, number>;
  userFeedback?: { rating: number; comment?: string };
  
  // Error
  error?: { type: string; message: string; stack?: string };
  
  // Metadata
  promptVersion: string;
  modelVersion: string;
  abTestVariant?: string;
  tags: Record<string, string>;
}
```

### 2.2 Tracing Middleware

```typescript
// TypeScript: Automatic tracing for LLM calls

import { v4 as uuidv4 } from "uuid";
import Anthropic from "@anthropic-ai/sdk";

class TracedLLMClient {
  private client: Anthropic;
  private traceStore: TraceStore;

  constructor(traceStore: TraceStore) {
    this.client = new Anthropic();
    this.traceStore = traceStore;
  }

  async createMessage(
    params: Anthropic.Messages.MessageCreateParams,
    context: {
      feature: string;
      userId: string;
      promptVersion: string;
      parentTraceId?: string;
      tags?: Record<string, string>;
    },
  ): Promise<{ response: Anthropic.Messages.Message; trace: LLMTrace }> {
    const traceId = uuidv4();
    const spanId = uuidv4();
    const startTime = Date.now();

    let response: Anthropic.Messages.Message;
    let error: LLMTrace["error"];

    try {
      response = await this.client.messages.create(params);
    } catch (err: any) {
      error = {
        type: err.constructor.name,
        message: err.message,
        stack: err.stack,
      };
      throw err;
    } finally {
      const endTime = Date.now();
      const latencyMs = endTime - startTime;

      const trace: LLMTrace = {
        traceId,
        spanId,
        parentSpanId: context.parentTraceId,
        timestamp: new Date(startTime),
        feature: context.feature,
        userId: context.userId,
        model: params.model,
        systemPrompt: typeof params.system === "string"
          ? params.system
          : JSON.stringify(params.system),
        messages: params.messages.map((m) => ({
          role: m.role,
          content: typeof m.content === "string" ? m.content : JSON.stringify(m.content),
        })),
        response: response!
          ? response.content.map((c) => (c.type === "text" ? c.text : JSON.stringify(c))).join("")
          : "",
        toolCalls: response!
          ? response.content
              .filter((c) => c.type === "tool_use")
              .map((c: any) => ({
                toolName: c.name,
                arguments: c.input,
                result: "",  // Filled in later
              }))
          : undefined,
        stopReason: response!?.stop_reason || "error",
        inputTokens: response!?.usage?.input_tokens || 0,
        outputTokens: response!?.usage?.output_tokens || 0,
        cachedInputTokens: (response!?.usage as any)?.cache_read_input_tokens || 0,
        latencyMs,
        timeToFirstTokenMs: latencyMs,  // Non-streaming; for streaming, measure separately
        cost: calculateCost(
          params.model,
          response!?.usage?.input_tokens || 0,
          response!?.usage?.output_tokens || 0,
          (response!?.usage as any)?.cache_read_input_tokens || 0,
        ),
        error,
        promptVersion: context.promptVersion,
        modelVersion: params.model,
        tags: context.tags || {},
      };

      // Store trace asynchronously (don't block the response)
      this.traceStore.store(trace).catch(console.error);
    }

    return { response: response!, trace: {} as LLMTrace };
  }
}
```

```python
# Python: Automatic tracing for LLM calls

import uuid
import time
from dataclasses import dataclass, field, asdict
from typing import Any
import anthropic

@dataclass
class LLMTrace:
    trace_id: str
    timestamp: float
    feature: str
    user_id: str
    model: str
    system_prompt: str
    input_messages: list[dict]
    response_text: str
    stop_reason: str
    input_tokens: int
    output_tokens: int
    cached_tokens: int
    latency_ms: float
    cost: float
    prompt_version: str
    error: dict | None = None
    eval_scores: dict | None = None
    tool_calls: list[dict] = field(default_factory=list)
    tags: dict = field(default_factory=dict)

class TracedClient:
    def __init__(self, trace_store):
        self.client = anthropic.Anthropic()
        self.trace_store = trace_store

    def create_message(
        self,
        feature: str,
        user_id: str,
        prompt_version: str,
        **kwargs,
    ) -> tuple[anthropic.types.Message, LLMTrace]:
        trace_id = str(uuid.uuid4())
        start = time.time()
        error = None

        try:
            response = self.client.messages.create(**kwargs)
        except Exception as e:
            error = {"type": type(e).__name__, "message": str(e)}
            raise
        finally:
            elapsed_ms = (time.time() - start) * 1000

            trace = LLMTrace(
                trace_id=trace_id,
                timestamp=start,
                feature=feature,
                user_id=user_id,
                model=kwargs.get("model", "unknown"),
                system_prompt=str(kwargs.get("system", "")),
                input_messages=kwargs.get("messages", []),
                response_text=response.content[0].text if response and response.content else "",
                stop_reason=response.stop_reason if response else "error",
                input_tokens=response.usage.input_tokens if response else 0,
                output_tokens=response.usage.output_tokens if response else 0,
                cached_tokens=getattr(response.usage, "cache_read_input_tokens", 0) if response else 0,
                latency_ms=elapsed_ms,
                cost=calculate_cost(kwargs.get("model"), response.usage if response else None),
                prompt_version=prompt_version,
                error=error,
            )

            # Store async
            self.trace_store.store(trace)

        return response, trace
```

---

## 3. Datadog LLM Observability

### 3.1 Setting Up Datadog LLM Monitoring

Datadog has built first-class LLM observability features. Here's how to integrate them.

```python
# Python: Datadog LLM observability integration

from ddtrace.llmobs import LLMObs
from ddtrace.llmobs.decorators import llm, workflow, tool

# Initialize LLM Observability
LLMObs.enable(
    ml_app="my-ai-app",
    api_key="your-dd-api-key",
    site="datadoghq.com",
    agentless_enabled=True,
)

@workflow
def handle_customer_query(user_id: str, query: str) -> str:
    """Top-level workflow — creates a trace in Datadog."""
    
    # Retrieve relevant documents
    docs = retrieve_documents(query)
    
    # Generate response
    response = generate_response(query, docs)
    
    return response

@tool
def retrieve_documents(query: str) -> list[str]:
    """RAG retrieval — tracked as a tool span."""
    results = vector_store.search(query, top_k=5)
    return [r.content for r in results]

@llm(model_name="claude-sonnet-4", model_provider="anthropic")
def generate_response(query: str, context_docs: list[str]) -> str:
    """LLM call — tracked as an LLM span with full prompt/response."""
    context = "\n\n".join(context_docs)
    
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system="You are a helpful customer support agent.",
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {query}",
        }],
    )
    
    # Annotate the span with token usage
    LLMObs.annotate(
        input_data=[{"role": "user", "content": query}],
        output_data=response.content[0].text,
        metrics={
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "total_tokens": response.usage.input_tokens + response.usage.output_tokens,
        },
        tags={
            "model_version": "claude-sonnet-4-20250514",
            "prompt_version": "v3.2",
        },
    )
    
    return response.content[0].text
```

```typescript
// TypeScript: Datadog LLM tracing with dd-trace

import tracer from "dd-trace";

// Initialize Datadog tracing with LLM observability
tracer.init({
  service: "my-ai-app",
  env: process.env.NODE_ENV,
  llmobs: {
    enabled: true,
    mlApp: "my-ai-app",
    agentlessEnabled: true,
  },
});

import { llmobs } from "dd-trace";

async function handleRequest(userId: string, query: string): Promise<string> {
  // Create a workflow span
  return llmobs.trace({ kind: "workflow", name: "customer_support" }, async (span) => {
    span.setTag("user_id", userId);

    // RAG retrieval (tool span)
    const docs = await llmobs.trace({ kind: "tool", name: "rag_retrieval" }, async () => {
      return await retrieveDocuments(query);
    });

    // LLM call (llm span)
    const response = await llmobs.trace(
      { kind: "llm", name: "generate_response", modelName: "claude-sonnet-4", modelProvider: "anthropic" },
      async (llmSpan) => {
        const result = await client.messages.create({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1024,
          system: "Customer support agent.",
          messages: [{ role: "user", content: query }],
        });

        // Annotate with LLM-specific data
        llmSpan.setTag("input_tokens", result.usage.input_tokens);
        llmSpan.setTag("output_tokens", result.usage.output_tokens);
        llmSpan.setTag("prompt_version", "v3.2");

        return result.content[0].type === "text" ? result.content[0].text : "";
      }
    );

    return response;
  });
}
```

### 3.2 What Datadog Shows You

With LLM observability configured, Datadog provides:

```
┌─────────────────────────────────────────────────────────────┐
│ Datadog LLM Observability Dashboard                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Traces View:                                                 │
│ ┌──────────────────────────────────────────────────┐        │
│ │ workflow: customer_support  [342ms]                │        │
│ │  ├─ tool: rag_retrieval    [45ms]                  │        │
│ │  │   └─ 5 documents retrieved, relevance: 0.82     │        │
│ │  └─ llm: generate_response [289ms]                 │        │
│ │      ├─ model: claude-sonnet-4                     │        │
│ │      ├─ input_tokens: 2,847                        │        │
│ │      ├─ output_tokens: 312                         │        │
│ │      ├─ cost: $0.013                               │        │
│ │      └─ prompt: [click to expand]                  │        │
│ └──────────────────────────────────────────────────┘        │
│                                                              │
│ Aggregated Metrics (Last 24h):                               │
│  Requests:     12,847                                        │
│  Avg Latency:  387ms                                         │
│  P99 Latency:  1,240ms                                       │
│  Error Rate:   0.3%                                          │
│  Avg Cost:     $0.014/request                                │
│  Total Cost:   $179.86                                       │
│  Total Tokens: 41.2M input, 3.8M output                     │
│                                                              │
│ Top Errors:                                                  │
│  Rate Limit (429):    12 occurrences                         │
│  Timeout:             3 occurrences                          │
│  Context Overflow:    1 occurrence                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Hallucination Detection

### 4.1 Types of Hallucinations

1. **Factual hallucination** — states something false as fact ("Our store is open 24/7" when it closes at 9 PM)
2. **Source hallucination** — claims to reference a source that doesn't exist
3. **Identity hallucination** — claims to be something it's not or claims capabilities it doesn't have
4. **Temporal hallucination** — confuses timeframes, references outdated information as current
5. **Attribution hallucination** — attributes quotes or actions to the wrong person/entity

### 4.2 Automated Hallucination Detection

```typescript
// TypeScript: Multi-layer hallucination detection

interface HallucinationCheck {
  detected: boolean;
  type: string;
  severity: "low" | "medium" | "high";
  details: string;
}

async function detectHallucinations(
  response: string,
  sourceDocuments: string[],
  feature: string,
): Promise<HallucinationCheck[]> {
  const checks: HallucinationCheck[] = [];

  // Layer 1: Entity consistency check
  // Verify that specific entities mentioned in the response (prices, dates, names)
  // match the source documents
  const entityCheck = await checkEntityConsistency(response, sourceDocuments);
  if (entityCheck.inconsistencies.length > 0) {
    checks.push({
      detected: true,
      type: "factual",
      severity: "high",
      details: `Inconsistent entities: ${entityCheck.inconsistencies.join(", ")}`,
    });
  }

  // Layer 2: Source grounding check
  // Verify claims in the response are supported by source documents
  const groundingCheck = await checkSourceGrounding(response, sourceDocuments);
  if (!groundingCheck.grounded) {
    checks.push({
      detected: true,
      type: "factual",
      severity: "medium",
      details: `Unsupported claims: ${groundingCheck.unsupported.join("; ")}`,
    });
  }

  // Layer 3: Self-contradiction check
  // Check if the response contradicts itself
  const contradictionCheck = await checkSelfContradiction(response);
  if (contradictionCheck.hasContradiction) {
    checks.push({
      detected: true,
      type: "factual",
      severity: "medium",
      details: `Self-contradiction: ${contradictionCheck.details}`,
    });
  }

  // Layer 4: Known-facts check
  // For feature-specific facts (store hours, pricing, policies)
  const factCheck = await checkKnownFacts(response, feature);
  if (factCheck.violations.length > 0) {
    checks.push({
      detected: true,
      type: "factual",
      severity: "high",
      details: `Known fact violations: ${factCheck.violations.join("; ")}`,
    });
  }

  return checks;
}
```

```python
# Python: Entity consistency check for hallucination detection

import re
import anthropic

client = anthropic.Anthropic()

async def check_entity_consistency(
    response: str,
    source_documents: list[str],
) -> dict:
    """
    Extract specific entities (prices, dates, names, numbers) from the response
    and verify they appear in the source documents.
    """
    # Use LLM to extract entities from response
    extraction = client.messages.create(
        model="claude-haiku-4-20250414",
        max_tokens=500,
        system="""Extract all specific factual claims from this text. Focus on:
- Prices and numbers
- Dates and times
- Product names and features
- Policy details (return windows, warranties, etc.)
- Contact information

Respond with JSON: {"claims": [{"text": "the claim", "type": "price|date|policy|contact|other"}]}""",
        messages=[{"role": "user", "content": response}],
    )

    import json
    claims = json.loads(extraction.content[0].text).get("claims", [])

    # Verify each claim against source documents
    source_text = "\n\n".join(source_documents)
    inconsistencies = []

    for claim in claims:
        verification = client.messages.create(
            model="claude-haiku-4-20250414",
            max_tokens=100,
            system="Verify if the claim is supported by the source text. Respond JSON: {\"supported\": true/false, \"reason\": \"...\"}",
            messages=[{
                "role": "user",
                "content": f"Claim: {claim['text']}\n\nSource:\n{source_text[:8000]}",
            }],
        )

        result = json.loads(verification.content[0].text)
        if not result["supported"]:
            inconsistencies.append(f"{claim['text']} — {result['reason']}")

    return {"inconsistencies": inconsistencies}


async def check_known_facts(response: str, feature: str) -> dict:
    """Check response against a database of known facts for this feature."""
    known_facts = load_known_facts(feature)
    # known_facts is a dict like:
    # {"store_hours": "Mon-Fri 9AM-6PM, Sat 10AM-4PM, Closed Sunday",
    #  "return_policy": "30 days with receipt", ...}

    violations = []

    for fact_key, fact_value in known_facts.items():
        # Check if the response mentions this topic and contradicts the fact
        if is_topic_mentioned(response, fact_key):
            consistency = client.messages.create(
                model="claude-haiku-4-20250414",
                max_tokens=100,
                system="Check if the response is consistent with the known fact. Respond JSON: {\"consistent\": true/false, \"issue\": \"...\"}",
                messages=[{
                    "role": "user",
                    "content": f"Known fact ({fact_key}): {fact_value}\n\nResponse excerpt: {response[:2000]}",
                }],
            )

            import json
            result = json.loads(consistency.content[0].text)
            if not result["consistent"]:
                violations.append(f"{fact_key}: {result['issue']}")

    return {"violations": violations}
```

### 4.3 Hallucination Rate Monitoring

```typescript
// TypeScript: Track hallucination rates over time

interface HallucinationMetrics {
  feature: string;
  period: string;  // e.g., "2026-04-10"
  totalResponses: number;
  hallucinationsDetected: number;
  hallucinationRate: number;
  byType: Record<string, number>;
  bySeverity: Record<string, number>;
}

async function aggregateHallucinationMetrics(
  feature: string,
  date: string,
): Promise<HallucinationMetrics> {
  const checks = await getHallucinationChecks(feature, date);

  const total = checks.length;
  const withHallucinations = checks.filter((c) => c.some((h) => h.detected));

  const byType: Record<string, number> = {};
  const bySeverity: Record<string, number> = {};

  for (const checkSet of checks) {
    for (const check of checkSet) {
      if (check.detected) {
        byType[check.type] = (byType[check.type] || 0) + 1;
        bySeverity[check.severity] = (bySeverity[check.severity] || 0) + 1;
      }
    }
  }

  return {
    feature,
    period: date,
    totalResponses: total,
    hallucinationsDetected: withHallucinations.length,
    hallucinationRate: total > 0 ? withHallucinations.length / total : 0,
    byType,
    bySeverity,
  };
}
```

---

## 5. Degradation Alerts

### 5.1 What Degrades and Why

AI systems degrade in ways that are different from traditional software:

| Degradation Type | Traditional Software | AI System |
|-----------------|---------------------|-----------|
| Performance | Server overloaded → slow responses | Model provider rate limits → slow or failed |
| Correctness | Bug → wrong result | Model drift → subtly wrong results |
| Availability | Server down → 500 errors | Provider outage → fallback needed |
| Cost | N/A | Token usage spike → budget exceeded |
| Quality | N/A | Relevance score drops → users notice |

### 5.2 Multi-Signal Degradation Detection

```typescript
// TypeScript: Composite degradation detection

interface DegradationSignal {
  name: string;
  weight: number;
  currentValue: number;
  baselineValue: number;
  threshold: number;  // How much deviation before it's a signal
}

interface DegradationAssessment {
  feature: string;
  overallHealth: number;  // 0-1
  signals: DegradationSignal[];
  degraded: boolean;
  primaryCause: string;
}

async function assessDegradation(feature: string): Promise<DegradationAssessment> {
  const signals: DegradationSignal[] = [
    {
      name: "eval_score",
      weight: 0.3,
      currentValue: await getCurrentMetric(feature, "eval_score"),
      baselineValue: await getBaselineMetric(feature, "eval_score"),
      threshold: 0.05,
    },
    {
      name: "latency_p95",
      weight: 0.15,
      currentValue: await getCurrentMetric(feature, "latency_p95"),
      baselineValue: await getBaselineMetric(feature, "latency_p95"),
      threshold: 0.50,  // 50% latency increase
    },
    {
      name: "error_rate",
      weight: 0.2,
      currentValue: await getCurrentMetric(feature, "error_rate"),
      baselineValue: await getBaselineMetric(feature, "error_rate"),
      threshold: 2.0,  // 2x error rate increase
    },
    {
      name: "hallucination_rate",
      weight: 0.25,
      currentValue: await getCurrentMetric(feature, "hallucination_rate"),
      baselineValue: await getBaselineMetric(feature, "hallucination_rate"),
      threshold: 1.5,  // 1.5x hallucination rate
    },
    {
      name: "user_satisfaction",
      weight: 0.1,
      currentValue: await getCurrentMetric(feature, "user_satisfaction"),
      baselineValue: await getBaselineMetric(feature, "user_satisfaction"),
      threshold: 0.10,  // 10% satisfaction drop
    },
  ];

  // Calculate health score
  let healthScore = 1.0;
  let worstSignal = "";
  let worstDeviation = 0;

  for (const signal of signals) {
    const deviation = Math.abs(signal.currentValue - signal.baselineValue) / signal.baselineValue;

    if (deviation > signal.threshold) {
      const penalty = signal.weight * (deviation / signal.threshold);
      healthScore -= penalty;

      if (deviation > worstDeviation) {
        worstDeviation = deviation;
        worstSignal = signal.name;
      }
    }
  }

  healthScore = Math.max(0, Math.min(1, healthScore));

  return {
    feature,
    overallHealth: healthScore,
    signals,
    degraded: healthScore < 0.7,
    primaryCause: worstSignal || "none",
  };
}
```

### 5.3 Cost Anomaly Detection

```python
# Python: Cost anomaly detection with multiple methods

import numpy as np
from datetime import datetime, timedelta
from dataclasses import dataclass

@dataclass
class CostAnomaly:
    detected: bool
    method: str
    current_cost: float
    expected_cost: float
    deviation: float
    message: str

def detect_cost_anomalies(
    feature: str,
    current_cost: float,
    historical_costs: list[float],
) -> list[CostAnomaly]:
    """Multiple anomaly detection methods for robustness."""
    anomalies = []

    if len(historical_costs) < 7:
        return anomalies

    costs = np.array(historical_costs)

    # Method 1: Z-score (parametric)
    mean = np.mean(costs)
    std = np.std(costs)
    z_score = (current_cost - mean) / std if std > 0 else 0

    if abs(z_score) > 2.5:
        anomalies.append(CostAnomaly(
            detected=True,
            method="z_score",
            current_cost=current_cost,
            expected_cost=mean,
            deviation=z_score,
            message=f"{feature}: ${current_cost:.2f} is {z_score:.1f} std devs from mean ${mean:.2f}",
        ))

    # Method 2: IQR (non-parametric, robust to outliers)
    q1 = np.percentile(costs, 25)
    q3 = np.percentile(costs, 75)
    iqr = q3 - q1
    upper_bound = q3 + 2.0 * iqr
    lower_bound = q1 - 2.0 * iqr

    if current_cost > upper_bound:
        anomalies.append(CostAnomaly(
            detected=True,
            method="iqr",
            current_cost=current_cost,
            expected_cost=float(np.median(costs)),
            deviation=(current_cost - upper_bound) / iqr if iqr > 0 else 0,
            message=f"{feature}: ${current_cost:.2f} exceeds IQR upper bound ${upper_bound:.2f}",
        ))

    # Method 3: Percent change from recent trend
    recent = costs[-3:]  # Last 3 days
    recent_mean = np.mean(recent)
    pct_change = (current_cost - recent_mean) / recent_mean if recent_mean > 0 else 0

    if pct_change > 0.5:  # 50% increase over 3-day trend
        anomalies.append(CostAnomaly(
            detected=True,
            method="trend",
            current_cost=current_cost,
            expected_cost=float(recent_mean),
            deviation=pct_change,
            message=f"{feature}: ${current_cost:.2f} is {pct_change:.0%} above 3-day trend ${recent_mean:.2f}",
        ))

    return anomalies
```

---

## 6. Debugging LLM Failures

### 6.1 The Challenge of Non-Determinism

Traditional software debugging: reproduce the bug, step through the code, find the issue. LLM debugging: the same input might produce a different (correct) output when you re-run it. The "bug" might not reproduce.

This requires a different debugging approach:

```
Traditional Debugging:
  1. Reproduce the bug
  2. Add breakpoints
  3. Step through execution
  4. Find the root cause
  5. Fix the code

LLM Debugging:
  1. Find the trace (you saved it, right?)
  2. Read the full prompt that was sent
  3. Read the full response
  4. Identify the failure mode
  5. Determine if it's:
     a. A prompt issue (change the prompt)
     b. A context issue (wrong/missing documents)
     c. A model issue (model limitation or regression)
     d. A tool issue (wrong tool selected or bad output)
  6. Write an eval that catches this failure
  7. Fix the issue and verify with the eval
```

### 6.2 The LLM Debugging Toolkit

```typescript
// TypeScript: LLM debugging utilities

interface DebugSession {
  traceId: string;
  originalTrace: LLMTrace;
  reproductions: LLMTrace[];
  analysis: DebugAnalysis;
}

interface DebugAnalysis {
  failureMode: "hallucination" | "wrong_tool" | "off_topic" | "refused" | "context_missing" | "unknown";
  rootCause: string;
  promptIssues: string[];
  contextIssues: string[];
  modelIssues: string[];
  recommendation: string;
}

class LLMDebugger {
  private client: Anthropic;
  private traceStore: TraceStore;

  async debug(traceId: string): Promise<DebugSession> {
    // 1. Load the original trace
    const trace = await this.traceStore.get(traceId);

    // 2. Attempt reproduction (multiple times to test consistency)
    const reproductions: LLMTrace[] = [];
    for (let i = 0; i < 3; i++) {
      const repro = await this.reproduce(trace);
      reproductions.push(repro);
    }

    // 3. Analyze the failure
    const analysis = await this.analyze(trace, reproductions);

    return {
      traceId,
      originalTrace: trace,
      reproductions,
      analysis,
    };
  }

  private async reproduce(trace: LLMTrace): Promise<LLMTrace> {
    // Replay the exact same prompt
    const startTime = Date.now();
    const response = await this.client.messages.create({
      model: trace.model,
      max_tokens: 1024,
      system: trace.systemPrompt,
      messages: trace.messages.map((m) => ({
        role: m.role as "user" | "assistant",
        content: m.content,
      })),
    });

    return {
      ...trace,
      traceId: `repro-${Date.now()}`,
      response: response.content[0].type === "text" ? response.content[0].text : "",
      latencyMs: Date.now() - startTime,
    } as LLMTrace;
  }

  private async analyze(
    original: LLMTrace,
    reproductions: LLMTrace[],
  ): Promise<DebugAnalysis> {
    // Check if the issue reproduces
    const reproduces = reproductions.some(
      (r) => r.response.includes(original.response.substring(0, 50))
    );

    // Analyze with a meta-LLM call
    const analysis = await this.client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1000,
      system: `You are an AI debugging expert. Analyze this LLM failure and determine the root cause.`,
      messages: [
        {
          role: "user",
          content: `Original trace:
System prompt: ${original.systemPrompt.substring(0, 2000)}
User message: ${original.messages[original.messages.length - 1]?.content.substring(0, 1000)}
Response (problematic): ${original.response.substring(0, 1000)}

Reproduction attempts:
${reproductions.map((r, i) => `Attempt ${i + 1}: ${r.response.substring(0, 500)}`).join("\n")}

Issue reproduces: ${reproduces}

Analyze the failure. What went wrong? What's the root cause? What should be fixed?

Respond with JSON:
{
  "failure_mode": "hallucination|wrong_tool|off_topic|refused|context_missing|unknown",
  "root_cause": "description",
  "prompt_issues": ["list of prompt problems"],
  "context_issues": ["list of context problems"],
  "model_issues": ["list of model limitations"],
  "recommendation": "what to fix"
}`,
        },
      ],
    });

    const text = analysis.content[0].type === "text" ? analysis.content[0].text : "{}";
    const result = JSON.parse(text);

    return {
      failureMode: result.failure_mode,
      rootCause: result.root_cause,
      promptIssues: result.prompt_issues || [],
      contextIssues: result.context_issues || [],
      modelIssues: result.model_issues || [],
      recommendation: result.recommendation,
    };
  }
}
```

---

## 7. Incident Response for AI

### 7.1 AI Incident Categories

| Category | Severity | Example | Response Time |
|----------|----------|---------|---------------|
| Safety | Critical | AI generates harmful content | Immediate (page on-call) |
| Data Leak | Critical | AI reveals PII or secrets | Immediate |
| Hallucination (high impact) | High | AI gives wrong medical/legal advice | < 30 minutes |
| Quality Regression | Medium | Eval scores drop 10%+ | < 2 hours |
| Cost Anomaly | Medium | Cost 3x above normal | < 4 hours |
| Latency Degradation | Low | P95 latency doubles | < 24 hours |
| Minor Hallucination | Low | AI gets a date slightly wrong | Next business day |

### 7.2 AI Incident Runbook Template

```markdown
# AI Incident Runbook

## Step 1: Assess Severity (< 5 minutes)
- [ ] What went wrong? (hallucination, safety issue, data leak, quality drop)
- [ ] How many users affected? (check logs for similar issues)
- [ ] Is it still happening? (check real-time monitoring)
- [ ] What's the blast radius? (one feature or multiple?)

## Step 2: Immediate Mitigation (< 15 minutes)
- [ ] **If safety/data leak**: Kill switch — disable the feature immediately
- [ ] **If quality regression**: Switch to fallback model or cached responses
- [ ] **If cost anomaly**: Enable rate limiting or throttle the feature
- [ ] **If provider outage**: Activate provider failover (if configured)

## Step 3: Root Cause Investigation (< 2 hours)
- [ ] Pull the traces for affected requests
- [ ] What was the prompt that caused the bad output?
- [ ] What was in the context? (RAG documents, conversation history)
- [ ] Did anything change recently? (prompt update, model version, data update)
- [ ] Can you reproduce the issue?
- [ ] Is this a regression from a known-good state?

## Step 4: Fix
- [ ] **Prompt issue**: Update the system prompt, test with eval suite
- [ ] **Context issue**: Fix RAG pipeline, update knowledge base
- [ ] **Model issue**: Switch models, add output validation
- [ ] **Tool issue**: Fix tool logic, add tool call validation
- [ ] Run eval suite against the fix
- [ ] Deploy fix (if urgent: hotfix; otherwise: normal CI/CD)

## Step 5: Verify
- [ ] Monitor the feature for 1 hour after fix
- [ ] Check eval scores are recovering
- [ ] Confirm no new incidents reported

## Step 6: Postmortem
- [ ] Schedule postmortem within 48 hours
- [ ] Document timeline, root cause, fix, and follow-ups
```

### 7.3 The Kill Switch

Every AI feature in production needs a kill switch — a way to instantly disable it without a deployment.

```typescript
// TypeScript: Feature kill switch implementation

interface KillSwitchConfig {
  feature: string;
  enabled: boolean;
  fallbackResponse: string;
  fallbackModel?: string;  // Downgrade to a safer model
  reason?: string;
  activatedAt?: Date;
  activatedBy?: string;
}

class KillSwitchManager {
  // Use a fast store (Redis, Edge Config, LaunchDarkly)
  private store: KeyValueStore;

  async isFeatureEnabled(feature: string): Promise<boolean> {
    const config = await this.store.get<KillSwitchConfig>(`killswitch:${feature}`);
    return config ? config.enabled : true;  // Default: enabled
  }

  async getFallback(feature: string): Promise<KillSwitchConfig | null> {
    return this.store.get<KillSwitchConfig>(`killswitch:${feature}`);
  }

  async disableFeature(
    feature: string,
    reason: string,
    activatedBy: string,
    fallbackResponse: string = "This feature is temporarily unavailable. Please try again later.",
  ): Promise<void> {
    await this.store.set(`killswitch:${feature}`, {
      feature,
      enabled: false,
      fallbackResponse,
      reason,
      activatedAt: new Date(),
      activatedBy,
    });

    // Alert the team
    await sendAlert("critical", `Kill switch activated for ${feature}`, reason, ["#ai-incidents"]);
  }

  async enableFeature(feature: string, restoredBy: string): Promise<void> {
    await this.store.delete(`killswitch:${feature}`);
    await sendAlert("info", `Kill switch deactivated for ${feature}`, `Restored by ${restoredBy}`, ["#ai-incidents"]);
  }
}

// Usage in the request handler
async function handleAIRequest(feature: string, ...args: any[]): Promise<string> {
  const killSwitch = new KillSwitchManager();

  if (!(await killSwitch.isFeatureEnabled(feature))) {
    const config = await killSwitch.getFallback(feature);
    return config?.fallbackResponse || "This feature is temporarily unavailable.";
  }

  // Normal processing...
  return processRequest(feature, ...args);
}
```

---

## 8. Postmortem Patterns

### 8.1 AI-Specific Postmortem Template

AI incidents have unique characteristics that standard postmortem templates don't capture. Here's an extended template:

```markdown
# AI Incident Postmortem: [Title]

## Summary
- **Date**: 
- **Duration**: 
- **Feature affected**: 
- **Users affected**: 
- **Severity**: Critical / High / Medium / Low

## Timeline
- HH:MM — [What happened]
- HH:MM — [Alert fired / issue detected]
- HH:MM — [Investigation started]
- HH:MM — [Root cause identified]
- HH:MM — [Mitigation applied]
- HH:MM — [Full resolution]

## What Went Wrong
[Describe the incident — what did users see? What output was generated?]

## Root Cause
[Specific cause. Examples:
- "System prompt did not explicitly prohibit [X], and the model generated [X] when asked"
- "RAG pipeline returned an outdated document that contradicted current policy"
- "Model provider updated the model, changing behavior for [specific pattern]"
- "Prompt injection via user-uploaded document caused the model to ignore guardrails"
]

## AI-Specific Analysis

### Was this deterministic or probabilistic?
- [ ] Deterministic (same input always produces the bad output)
- [ ] Probabilistic (bad output occurs X% of the time for this input)
- [ ] Unknown (couldn't reproduce)

### What layer failed?
- [ ] Input validation (attack got through)
- [ ] System prompt (instructions insufficient)
- [ ] Model behavior (model didn't follow instructions)
- [ ] Tool execution (wrong tool called or bad arguments)
- [ ] Output validation (bad output not caught)
- [ ] RAG/context (wrong or missing information)

### Was this in the eval suite?
- [ ] Yes, but the eval didn't catch it (eval gap)
- [ ] No, this scenario wasn't covered (coverage gap)
- [ ] Yes, and the eval is now failing (regression)

## Fix Applied
[What was changed and why]

## Follow-Up Actions
- [ ] Add eval case for this failure mode
- [ ] Update system prompt to address [gap]
- [ ] Add output validation rule for [pattern]
- [ ] Update known-facts database
- [ ] Review similar features for same vulnerability
- [ ] Schedule team review of [broader pattern]

## Lessons Learned
[What can we learn from this for future AI features?]
```

### 8.2 Common Postmortem Patterns

After reviewing dozens of AI incidents, common patterns emerge:

**Pattern 1: "The model changed."**
A provider updates their model and your carefully-tuned prompt stops working. Detection: regression detection (Ch 51). Prevention: pin model versions where possible, run evals on model updates.

**Pattern 2: "The data changed."**
Someone updated a document in your knowledge base, introducing incorrect information. The RAG system dutifully retrieves it. Detection: known-facts checking. Prevention: content review for knowledge base updates.

**Pattern 3: "The edge case."**
A user input that your evals never covered triggers unexpected behavior. Detection: online evals catch it eventually. Prevention: expand eval datasets based on production traffic patterns.

**Pattern 4: "The compound failure."**
Multiple small issues combine: a slightly wrong RAG result meets a slightly ambiguous prompt meets a slightly unusual query. None would fail individually, but together they produce a bad output. Detection: multi-signal degradation monitoring. Prevention: defense in depth.

---

## 9. The Complete Observability Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    User Request                               │
└───────────────────────┬──────────────────────────────────────┘
                        │
                 ┌──────▼──────┐
                 │   Tracing    │────→ Trace Store (Datadog / custom)
                 │   Middleware  │      Full prompts, responses, metrics
                 └──────┬──────┘
                        │
                 ┌──────▼──────┐
                 │   AI Feature │────→ Metrics (latency, tokens, cost)
                 │   Logic      │────→ Error tracking (Sentry / Datadog)
                 └──────┬──────┘
                        │
                 ┌──────▼──────┐
                 │   Online     │────→ Quality scores (sampled)
                 │   Eval       │────→ Hallucination checks
                 └──────┬──────┘
                        │
                 ┌──────▼──────┐
                 │   User       │
                 │   Response   │
                 └──────────────┘

                 Meanwhile, continuously:

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Regression   │  │ Cost Anomaly │  │ Degradation  │
  │ Detector     │  │ Detector     │  │ Monitor      │
  │ (scheduled)  │  │ (scheduled)  │  │ (continuous) │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         └────────────────┬┘─────────────────┘
                          │
                   ┌──────▼──────┐
                   │   Alert     │────→ Slack, PagerDuty, Email
                   │   Manager   │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │   Dashboard │
                   │   (Datadog) │
                   └─────────────┘
```

---

## 10. Key Takeaways

AI observability isn't traditional observability with an LLM label. It requires new tools, new metrics, and new debugging approaches.

**What you need at minimum:**

1. **Traces** — full prompt and response logging for every LLM call. You can't debug what you can't replay.

2. **Cost tracking** — per-request, per-feature, per-user cost tracking with anomaly detection. Know exactly where your money goes.

3. **Quality monitoring** — online evals running on sampled production traffic. Catch quality issues in real time.

4. **Hallucination detection** — multi-layer checks that catch factual errors before they reach users or after they do.

5. **Degradation alerts** — composite signals that detect when your system is getting worse, even if no single metric crosses a threshold.

6. **Kill switches** — instant feature disablement without deployment. The first 15 minutes of an incident are everything.

7. **Runbooks and postmortem templates** — incident response patterns designed for AI's unique failure modes.

The non-negotiable: **every LLM call must be traced and stored.** When the 2 AM incident happens — and it will — the traces are the only thing between you and blind guessing.

> **Spiral note:** This chapter builds on Ch 22 (basic telemetry) and Ch 51 (eval pipelines). Together, they form the complete quality and observability infrastructure for production AI. Ch 56 (CI/CD for AI) will cover how deployment pipelines integrate with this observability stack. Ch 62 (AI Adoption & Enablement) will cover how to roll out observability across an organization.

---

*Previous: [Production Eval Pipelines](./51-production-evals.md)* | *Next: [Part 11 — Deploying LLM Applications](../part-11-deployment-infrastructure/53-deploying-llm-apps.md)*

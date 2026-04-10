<!--
  CHAPTER: 53
  TITLE: Deploying LLM Applications
  PART: 11 — AI Deployment & Infrastructure
  PHASE: 2 — Become an Expert
  PREREQS: Ch 3 (LLM API calls), Ch 6 (chat interface), Ch 9 (agent loop), Ch 48 (security)
  KEY_TOPICS: serverless deployment, Vercel Functions, AWS Lambda, Docker containers, FastAPI, Next.js deployment, environment variables, cold starts, streaming, Fluid Compute
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript, Python, Docker, YAML
  UPDATED: 2026-04-10
-->

# Chapter 53: Deploying LLM Applications

> **Part 11 — AI Deployment & Infrastructure** | Phase 2: Become an Expert | Prerequisites: Ch 3, Ch 6, Ch 9, Ch 48 | Difficulty: Intermediate

You have built AI features. You have called APIs, streamed responses, structured output with Zod, built agent loops, wired up RAG pipelines, and hardened everything with guardrails. All of it ran on your laptop. Now it needs to run somewhere that isn't your laptop.

Deploying LLM applications is not the same as deploying a normal web application. The differences are concrete and consequential. LLM calls take 2-30 seconds to complete (not 50ms). Streaming responses require long-lived connections. API keys need to be managed across environments without leaking into client bundles. Cold starts can add seconds of latency on top of already-slow model responses. Agent workloads may need persistent processes that don't fit into serverless functions.

This chapter covers the deployment landscape for AI applications: serverless functions for simple AI routes, containers for complex agent workloads, and everything in between. You will deploy a Next.js chat application to Vercel, a FastAPI AI service to Docker, and an agent workload to a long-running container. By the end, you will have a decision framework for choosing the right deployment target for any AI workload.

### In This Chapter
- Serverless vs dedicated compute for AI: trade-offs that actually matter
- Vercel Functions for AI routes: streaming, timeout configuration, Fluid Compute
- AWS Lambda for AI: cold start considerations, timeout limits, memory sizing
- Docker containers for agent workloads that need persistent processes
- Deploying a Next.js + Vercel AI SDK chat app to Vercel
- Deploying a FastAPI AI service with Docker
- Environment variable management across dev, staging, and production
- The deployment decision tree: which target for which workload

### Related Chapters
- **Ch 3 (Your First LLM Call)** -- you made the API calls; now deploy them
- **Ch 6 (Building a Chat Interface)** -- the chat UI you built; now deploy the full stack
- **Ch 9 (The Agent Loop)** -- agent loops need special deployment considerations
- **Ch 22 (Telemetry & Tracing)** -- spirals back: deploy with telemetry from day one
- **Ch 48 (AI Security & Guardrails)** -- security requirements shape deployment choices
- **Ch 54 (API Gateway)** -- put a gateway in front of what you deploy here
- **Ch 55 (Sandboxing Agents)** -- isolate the dangerous workloads from this chapter
- **Ch 57 (Scaling & Cost)** -- scale what you deploy here

---

## 1. Why AI Deployments Are Different

### 1.1 The Four Differences That Matter

Regular web applications handle requests in milliseconds. AI applications handle requests in seconds. This single fact changes everything about deployment.

**Difference 1: Latency.** A typical API endpoint returns in 50-200ms. An LLM call takes 2-30 seconds. Streaming helps perceived latency (first token in 200-500ms), but the connection stays open for the full duration. Your infrastructure needs to support long-lived connections.

**Difference 2: Resource consumption.** An LLM call keeps a connection open and a function instance warm for seconds, not milliseconds. In serverless environments, this means you pay for wall-clock time, not CPU time. A function waiting for OpenAI's response is idle but billed.

**Difference 3: Secrets management.** Every AI application has at least one API key (usually several -- OpenAI, Anthropic, vector database, observability). These keys are high-value targets: a leaked OpenAI key can cost you thousands of dollars in minutes. Keys must never reach client bundles.

**Difference 4: Non-determinism.** The same deployment can produce different outputs for the same input. This makes testing, rollbacks, and canary analysis fundamentally different from traditional deployments (more in Ch 56).

### 1.2 The Deployment Spectrum

```
                    Less control                              More control
                    Less ops                                  More ops
                    Lower cost (small scale)                  Lower cost (large scale)
                    ┌──────────────────────────────────────────────────────┐
                    │                                                      │
Serverless ←────────────────────────────────────────────────────────────→ Dedicated
Functions          Containers          Container        VMs / Bare
(Vercel,           (Cloud Run,         Orchestration    Metal
 Lambda)            ECS Fargate)       (ECS, K8s)       (GPU instances)
                    │                                                      │
                    └──────────────────────────────────────────────────────┘
                    ↑                                                    ↑
              Chat apps,                                          Custom model
              simple AI routes                                    serving, agents
                                                                  with GPU needs
```

Most AI applications start on the left and move right only when they have specific reasons.

---

## 2. Serverless: Vercel Functions for AI

### 2.1 Why Serverless Works for Most AI Features

If your AI feature is an API route that receives a request, calls an LLM provider, and returns a response (possibly streaming), serverless functions are the right choice. This covers:

- Chat interfaces (Ch 6)
- Structured output endpoints (Ch 5)
- Simple tool-calling agents with short execution times (Ch 8-9)
- RAG queries (Ch 15-16)
- Embedding generation (Ch 14)

The key constraint: your function must complete within the platform's timeout limit, and your workload must be stateless between requests.

### 2.2 Vercel Functions: The AI-Friendly Defaults

Vercel Functions are designed for AI workloads. They support streaming natively, have configurable timeouts up to 800 seconds (on Enterprise), and integrate directly with the Vercel AI SDK.

Here is a complete AI chat route deployed on Vercel:

```typescript
// app/api/chat/route.ts — Next.js App Router AI route
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

// Configure the Vercel Function for AI workloads
export const maxDuration = 60; // seconds — up from the default 10s
export const runtime = "nodejs"; // 'edge' is also available

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: `You are a helpful assistant for Acme Corp. Answer questions about 
our products. Be concise. If you don't know, say so.`,
    messages,
    maxTokens: 2048,
  });

  // streamText returns a Response-compatible object for streaming
  return result.toDataStreamResponse();
}
```

**Key configuration points:**

```typescript
// maxDuration controls the Vercel Function timeout
// Default: 10s (Hobby), 15s (Pro), configurable up to 800s (Enterprise)
export const maxDuration = 60;

// runtime selects the execution environment
// 'nodejs' — full Node.js, up to 800s timeout, more memory
// 'edge' — V8 isolates, lower latency, 30s max, limited APIs
export const runtime = "nodejs";
```

**When to use Edge vs Node.js runtime:**

| Consideration | Edge Runtime | Node.js Runtime |
|--------------|-------------|-----------------|
| Cold start | ~0ms (V8 isolates) | 250-1000ms |
| Max timeout | 30s | 60-800s |
| Streaming | Yes | Yes |
| Node.js APIs | Limited (no fs, net) | Full |
| Use case | Simple chat, fast responses | Agent loops, long tasks, file processing |

### 2.3 Fluid Compute: Why It Matters for AI

Vercel's Fluid Compute changes how serverless functions handle concurrent requests. Instead of spinning up a new function instance for every request, Fluid Compute allows a single instance to handle multiple requests concurrently.

For AI workloads, this is significant because:

1. **Most of the function's time is spent waiting** -- waiting for the LLM provider to respond, not doing CPU work
2. **While one request waits for OpenAI**, the same instance can start processing another request
3. **This reduces cold starts** -- fewer new instances needed during traffic spikes

```typescript
// app/api/chat/route.ts
// With Fluid Compute (enabled by default on Vercel), this function
// can handle multiple concurrent requests in a single instance.
// While request A is waiting for Anthropic's streaming response,
// request B can begin processing.

export const maxDuration = 60;

export async function POST(req: Request) {
  const { messages } = await req.json();

  // This call takes 2-10 seconds. During that time,
  // Fluid Compute can handle other incoming requests
  // in the same function instance.
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    messages,
  });

  return result.toDataStreamResponse();
}
```

### 2.4 Streaming Architecture on Vercel

Streaming is not optional for AI applications. Without streaming, users stare at a blank screen for 3-10 seconds. With streaming, they see the first token in 200-500ms.

```typescript
// app/api/chat/route.ts — complete streaming example
import { streamText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

export const maxDuration = 60;

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: "You are a helpful assistant with access to a knowledge base.",
    messages,
    tools: {
      searchKnowledgeBase: tool({
        description: "Search the company knowledge base for relevant information",
        parameters: z.object({
          query: z.string().describe("Search query"),
        }),
        execute: async ({ query }) => {
          // In production, this calls your vector store (Ch 14-15)
          const results = await fetch(
            `${process.env.VECTOR_API_URL}/search?q=${encodeURIComponent(query)}`,
            {
              headers: {
                Authorization: `Bearer ${process.env.VECTOR_API_KEY}`,
              },
            }
          );
          return results.json();
        },
      }),
    },
    maxSteps: 3, // Allow up to 3 tool-use rounds
  });

  return result.toDataStreamResponse();
}
```

On the client side:

```typescript
// app/page.tsx — client component consuming the stream
"use client";
import { useChat } from "@ai-sdk/react";

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: "/api/chat",
      // Streaming is handled automatically by useChat
    });

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      <div className="flex-1 overflow-y-auto space-y-4">
        {messages.map((m) => (
          <div
            key={m.id}
            className={`p-3 rounded ${
              m.role === "user" ? "bg-blue-100 ml-auto" : "bg-gray-100"
            }`}
          >
            {m.content}
          </div>
        ))}
      </div>
      <form onSubmit={handleSubmit} className="flex gap-2 pt-4">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask a question..."
          className="flex-1 border rounded p-2"
          disabled={isLoading}
        />
        <button
          type="submit"
          className="bg-blue-500 text-white px-4 py-2 rounded"
          disabled={isLoading}
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

---

## 3. Serverless: AWS Lambda for AI

### 3.1 Lambda for AI: The Constraints

AWS Lambda supports AI workloads, but with important constraints:

| Constraint | Lambda Limit | Impact on AI |
|-----------|-------------|-------------|
| Timeout | 15 minutes (900s) | Generous, but agent loops can exceed it |
| Memory | 128MB - 10,240MB | More memory = more CPU = faster cold starts |
| Payload | 6MB (sync), 256KB (async) | Large context windows can exceed this |
| Cold start | 500ms - 5s (Node.js) | Adds to already-slow LLM latency |
| Concurrency | 1000 default | Burst traffic can hit limits |
| Streaming | Via response streaming | Requires Function URLs or API Gateway v2 |

### 3.2 Lambda AI Function (TypeScript)

```typescript
// lambda/chat-handler.ts — AWS Lambda AI function
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { Readable } from "stream";

// Handler for Lambda Function URL with response streaming
export const handler = awslambda.streamifyResponse(
  async (event: any, responseStream: any) => {
    const body = JSON.parse(event.body || "{}");
    const { messages } = body;

    // Set streaming headers
    const metadata = {
      statusCode: 200,
      headers: {
        "Content-Type": "text/event-stream",
        "Cache-Control": "no-cache",
        Connection: "keep-alive",
      },
    };

    responseStream = awslambda.HttpResponseStream.from(
      responseStream,
      metadata
    );

    const result = streamText({
      model: anthropic("claude-sonnet-4-20250514"),
      messages,
      maxTokens: 2048,
    });

    // Pipe the AI stream to the Lambda response stream
    const reader = result.textStream;
    for await (const chunk of reader) {
      responseStream.write(`data: ${JSON.stringify({ text: chunk })}\n\n`);
    }

    responseStream.end();
  }
);
```

### 3.3 Lambda Configuration for AI (SAM Template)

```yaml
# template.yaml — AWS SAM template for an AI Lambda function
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: AI Chat Lambda Function

Globals:
  Function:
    Timeout: 300          # 5 minutes — generous for most LLM calls
    MemorySize: 1024      # 1GB — more memory = more CPU = faster cold starts
    Runtime: nodejs20.x
    Environment:
      Variables:
        ANTHROPIC_API_KEY: !Ref AnthropicApiKey
        NODE_OPTIONS: "--enable-source-maps"

Parameters:
  AnthropicApiKey:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /ai-chat/anthropic-api-key
    NoEcho: true

Resources:
  ChatFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: dist/
      Handler: chat-handler.handler
      # Function URL for streaming support
      FunctionUrlConfig:
        AuthType: AWS_IAM
        InvokeMode: RESPONSE_STREAM   # Required for streaming responses
        Cors:
          AllowOrigins:
            - "https://your-app.com"
          AllowMethods:
            - POST
          AllowHeaders:
            - Content-Type
            - Authorization
      # Provisioned concurrency to eliminate cold starts
      ProvisionedConcurrencyConfig:
        ProvisionedConcurrentExecutions: 5
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2022"
        Sourcemap: true
        EntryPoints:
          - chat-handler.ts

  # Auto-scaling for provisioned concurrency
  ChatFunctionScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 50
      MinCapacity: 5
      ResourceId: !Sub function:${ChatFunction}:live
      ScalableDimension: lambda:function:ProvisionedConcurrency
      ServiceNamespace: lambda

  ChatFunctionScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: ChatFunctionUtilization
      PolicyType: TargetTrackingScaling
      ScalableTargetId: !Ref ChatFunctionScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 0.7   # Scale up when 70% of provisioned concurrency is used
        PredefinedMetricSpecification:
          PredefinedMetricType: LambdaProvisionedConcurrencyUtilization

Outputs:
  ChatFunctionUrl:
    Description: Chat function streaming URL
    Value: !GetAtt ChatFunctionUrl.FunctionUrl
```

### 3.4 Reducing Cold Starts

Cold starts are the biggest concern for Lambda-based AI applications. The LLM call itself takes seconds; adding another 1-3 seconds of cold start is unacceptable.

**Strategy 1: Provisioned Concurrency** (shown above) -- keeps instances warm. Costs money but eliminates cold starts completely.

**Strategy 2: Smaller bundles** -- tree-shake aggressively, avoid heavy SDKs:

```typescript
// esbuild.config.ts — optimize Lambda bundle size
import { build } from "esbuild";

await build({
  entryPoints: ["./chat-handler.ts"],
  bundle: true,
  minify: true,
  platform: "node",
  target: "node20",
  outdir: "dist",
  external: [
    // AWS SDK v3 is included in the Lambda runtime
    "@aws-sdk/*",
  ],
  // Tree-shake unused code
  treeShaking: true,
});
```

**Strategy 3: Warm-up pings** -- schedule a CloudWatch Event to invoke the function every 5 minutes:

```yaml
# Add to the SAM template
  WarmUpRule:
    Type: AWS::Events::Rule
    Properties:
      ScheduleExpression: rate(5 minutes)
      Targets:
        - Arn: !GetAtt ChatFunction.Arn
          Id: WarmUpTarget
          Input: '{"warmup": true}'
```

```typescript
// In the handler, short-circuit warmup invocations
export const handler = async (event: any) => {
  if (event.warmup) {
    return { statusCode: 200, body: "warm" };
  }
  // ... normal handling
};
```

---

## 4. Docker Containers for Agent Workloads

### 4.1 When You Need Containers

Serverless functions have limits. When your AI workload exceeds them, containers are the answer:

| Need | Why Serverless Fails | Container Solution |
|------|---------------------|-------------------|
| Agent loops > 15 min | Lambda timeout | No timeout limit |
| Persistent connections | Functions are ephemeral | Long-lived WebSocket/SSE |
| Local file processing | No persistent disk | Volume mounts |
| GPU inference | No GPU on Lambda | GPU-enabled containers |
| Background jobs | Pay-per-invocation is expensive | Always-on is cheaper |
| Complex dependencies | Lambda layer limits | Full OS control |

### 4.2 Dockerfile for a TypeScript Agent Service

```dockerfile
# Dockerfile — TypeScript agent service
FROM node:20-slim AS builder

WORKDIR /app

# Install dependencies first for better caching
COPY package.json package-lock.json ./
RUN npm ci --only=production

COPY tsconfig.json ./
COPY src/ ./src/

RUN npm run build

# Production stage — minimal image
FROM node:20-slim AS production

WORKDIR /app

# Security: run as non-root user
RUN addgroup --system agent && adduser --system --group agent

# Copy only what's needed
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/health').then(r => process.exit(r.ok ? 0 : 1))"

USER agent

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### 4.3 The Agent Service (TypeScript)

```typescript
// src/server.ts — long-running agent service
import express from "express";
import { streamText, generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

const app = express();
app.use(express.json({ limit: "10mb" }));

// Health check — critical for container orchestration
app.get("/health", (_req, res) => {
  res.json({
    status: "healthy",
    uptime: process.uptime(),
    memoryUsage: process.memoryUsage().heapUsed / 1024 / 1024,
  });
});

// Streaming chat endpoint
app.post("/api/chat", async (req, res) => {
  const { messages, sessionId } = req.body;

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  try {
    const result = streamText({
      model: anthropic("claude-sonnet-4-20250514"),
      messages,
      maxSteps: 10, // Agents may need many tool-use rounds
      tools: {
        searchDocuments: tool({
          description: "Search internal documents for relevant information",
          parameters: z.object({
            query: z.string().describe("Search query"),
          }),
          execute: async ({ query }) => {
            // Your document search implementation
            return { results: [] };
          },
        }),
        createTicket: tool({
          description: "Create a support ticket in the ticketing system",
          parameters: z.object({
            title: z.string(),
            description: z.string(),
            priority: z.enum(["low", "medium", "high", "critical"]),
          }),
          execute: async ({ title, description, priority }) => {
            // Your ticket creation implementation
            return { ticketId: "TICK-1234", status: "created" };
          },
        }),
      },
    });

    for await (const chunk of result.textStream) {
      res.write(`data: ${JSON.stringify({ text: chunk })}\n\n`);
    }

    res.write(`data: [DONE]\n\n`);
    res.end();
  } catch (error) {
    console.error("Chat error:", error);
    res.write(
      `data: ${JSON.stringify({ error: "Internal server error" })}\n\n`
    );
    res.end();
  }
});

// Long-running agent endpoint — background processing
app.post("/api/agent/run", async (req, res) => {
  const { task, context } = req.body;

  // Return immediately with a job ID
  const jobId = crypto.randomUUID();
  res.json({ jobId, status: "started" });

  // Process in background — this is why we need containers
  processAgentTask(jobId, task, context).catch((error) => {
    console.error(`Job ${jobId} failed:`, error);
  });
});

async function processAgentTask(
  jobId: string,
  task: string,
  context: any
): Promise<void> {
  // This can run for minutes or hours — no timeout
  let iterations = 0;
  const maxIterations = 50;

  while (iterations < maxIterations) {
    const result = await generateText({
      model: anthropic("claude-sonnet-4-20250514"),
      system: `You are an agent processing a task. When the task is complete, 
respond with TASK_COMPLETE. Current iteration: ${iterations + 1}`,
      prompt: `Task: ${task}\nContext: ${JSON.stringify(context)}`,
      tools: {
        // Agent tools here
      },
      maxSteps: 5,
    });

    iterations++;

    // Check for completion
    if (result.text.includes("TASK_COMPLETE")) {
      // Store result somewhere (database, S3, etc.)
      console.log(`Job ${jobId} completed after ${iterations} iterations`);
      return;
    }
  }

  console.log(`Job ${jobId} hit iteration limit`);
}

const port = parseInt(process.env.PORT || "3000");
app.listen(port, () => {
  console.log(`Agent service running on port ${port}`);
});
```

### 4.4 Docker Compose for Local Development

```yaml
# docker-compose.yml — local development with all services
version: "3.8"

services:
  # The AI agent service
  agent:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "3000:3000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - VECTOR_DB_URL=http://qdrant:6333
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=development
    depends_on:
      - qdrant
      - redis
    volumes:
      - agent-workspace:/tmp/workspace  # Ephemeral workspace for file processing
    restart: unless-stopped

  # Vector database for RAG
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage

  # Redis for caching and job queues
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  # Next.js frontend
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
    ports:
      - "3001:3000"
    environment:
      - AGENT_API_URL=http://agent:3000
    depends_on:
      - agent

volumes:
  qdrant-data:
  redis-data:
  agent-workspace:
```

---

## 5. Deploying a Python AI Service with FastAPI

### 5.1 Why Python for AI Services

Python is the natural choice when your AI service needs:
- Direct model inference (Hugging Face, PyTorch) from Ch 40-43
- Heavy data processing (pandas, numpy)
- Integration with ML-specific libraries
- Fine-tuned model serving from Ch 45-47

### 5.2 FastAPI AI Service

```python
# app/main.py — FastAPI AI service
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from anthropic import Anthropic
import os
import json
import asyncio
from typing import AsyncGenerator

app = FastAPI(title="AI Service", version="1.0.0")

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


class ChatRequest(BaseModel):
    messages: list[dict]
    max_tokens: int = 2048
    model: str = "claude-sonnet-4-20250514"


class ChatResponse(BaseModel):
    content: str
    usage: dict


@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "model_ready": True,
    }


@app.post("/api/chat")
async def chat(request: ChatRequest) -> ChatResponse:
    """Non-streaming chat endpoint."""
    try:
        response = client.messages.create(
            model=request.model,
            max_tokens=request.max_tokens,
            messages=request.messages,
        )
        return ChatResponse(
            content=response.content[0].text,
            usage={
                "input_tokens": response.usage.input_tokens,
                "output_tokens": response.usage.output_tokens,
            },
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/api/chat/stream")
async def chat_stream(request: ChatRequest):
    """Streaming chat endpoint using Server-Sent Events."""

    async def generate() -> AsyncGenerator[str, None]:
        try:
            with client.messages.stream(
                model=request.model,
                max_tokens=request.max_tokens,
                messages=request.messages,
            ) as stream:
                for text in stream.text_stream:
                    yield f"data: {json.dumps({'text': text})}\n\n"
                    
            # Send final message with usage
            final = stream.get_final_message()
            yield f"data: {json.dumps({'done': True, 'usage': {'input_tokens': final.usage.input_tokens, 'output_tokens': final.usage.output_tokens}})}\n\n"
        except Exception as e:
            yield f"data: {json.dumps({'error': str(e)})}\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        },
    )


@app.post("/api/embed")
async def embed(request: dict):
    """Generate embeddings using a local model."""
    from sentence_transformers import SentenceTransformer
    
    model = get_embedding_model()  # Cached singleton
    texts = request.get("texts", [])
    embeddings = model.encode(texts).tolist()
    return {"embeddings": embeddings}


# Singleton pattern for expensive models
_embedding_model = None

def get_embedding_model():
    global _embedding_model
    if _embedding_model is None:
        from sentence_transformers import SentenceTransformer
        _embedding_model = SentenceTransformer("all-MiniLM-L6-v2")
    return _embedding_model


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 5.3 Dockerfile for FastAPI

```dockerfile
# Dockerfile.python — FastAPI AI service
FROM python:3.12-slim AS base

WORKDIR /app

# Install system dependencies for ML libraries
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ ./app/

# Security: non-root user
RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 8000

# Uvicorn with production settings
CMD ["uvicorn", "app.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "4", \
     "--limit-concurrency", "100", \
     "--timeout-keep-alive", "65"]
```

```
# requirements.txt
fastapi==0.115.0
uvicorn[standard]==0.30.0
anthropic==0.40.0
pydantic==2.9.0
sentence-transformers==3.0.0
torch==2.4.0 --index-url https://download.pytorch.org/whl/cpu
```

### 5.4 Multi-Service Docker Compose (TypeScript Frontend + Python Backend)

```yaml
# docker-compose.yml — full stack with TS frontend and Python AI backend
version: "3.8"

services:
  # Next.js frontend (TypeScript)
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - AI_SERVICE_URL=http://ai-service:8000
      - NEXT_PUBLIC_APP_URL=http://localhost:3000
    depends_on:
      ai-service:
        condition: service_healthy

  # FastAPI AI service (Python)
  ai-service:
    build:
      context: ./ai-service
      dockerfile: Dockerfile.python
    ports:
      - "8000:8000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s  # ML models take time to load

  # Vector database
  qdrant:
    image: qdrant/qdrant:v1.11.0
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage

volumes:
  qdrant-data:
```

---

## 6. Environment Variable Management

### 6.1 The Problem

AI applications have more secrets than typical web apps. A moderately complex AI feature might need:

```bash
# LLM providers
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...

# Vector database
PINECONE_API_KEY=...
PINECONE_INDEX_URL=...

# Observability
LAMINAR_API_KEY=...
DATADOG_API_KEY=...

# Feature flags
GROWTHBOOK_CLIENT_KEY=...

# Application
DATABASE_URL=postgres://...
REDIS_URL=redis://...
```

Each of these keys has different security requirements, different values per environment, and different blast radii if leaked.

### 6.2 The Three-Environment Pattern

```
                Development              Staging                 Production
                ───────────              ───────                 ──────────
API Keys        Personal keys or         Shared test keys        Production keys
                test org keys            with spending limits    (high limits, monitored)

Models          Cheapest/fastest         Same models as prod     Production models
                (haiku, gpt-4o-mini)     (for parity)           (sonnet, gpt-4o)

Rate Limits     Low                      Medium                  Production limits

Data            Synthetic/test data      Anonymized prod data    Real data

Cost Controls   $10/day budget           $100/day budget         Usage-based alerts
```

### 6.3 Vercel Environment Variables

```bash
# Set environment variables per environment in Vercel
# These are encrypted and injected at build/runtime

# Production only
vercel env add ANTHROPIC_API_KEY production
# Enter value: sk-ant-prod-...

# Preview (staging) only
vercel env add ANTHROPIC_API_KEY preview
# Enter value: sk-ant-staging-...

# Development only
vercel env add ANTHROPIC_API_KEY development
# Enter value: sk-ant-dev-...

# Pull to local .env.local for development
vercel env pull .env.local
```

### 6.4 Environment Validation at Startup

Never let your application start with missing configuration. Fail fast:

```typescript
// lib/env.ts — validate all required environment variables at startup
import { z } from "zod";

const envSchema = z.object({
  // LLM providers — at least one required
  ANTHROPIC_API_KEY: z.string().startsWith("sk-ant-"),
  OPENAI_API_KEY: z.string().startsWith("sk-").optional(),

  // Vector database
  VECTOR_DB_URL: z.string().url(),
  VECTOR_API_KEY: z.string().min(1),

  // Application
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "staging", "production"]),

  // Cost controls
  MAX_DAILY_TOKENS: z.coerce.number().default(1_000_000),
  MAX_REQUEST_TOKENS: z.coerce.number().default(100_000),
});

export type Env = z.infer<typeof envSchema>;

// Parse and validate — throws at startup if invalid
function validateEnv(): Env {
  const result = envSchema.safeParse(process.env);

  if (!result.success) {
    console.error("Environment variable validation failed:");
    for (const error of result.error.errors) {
      console.error(`  ${error.path.join(".")}: ${error.message}`);
    }
    process.exit(1);
  }

  return result.data;
}

export const env = validateEnv();
```

```python
# Python equivalent using Pydantic
# app/config.py
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    anthropic_api_key: str
    openai_api_key: str | None = None
    vector_db_url: str
    vector_api_key: str
    database_url: str
    environment: str = "development"
    max_daily_tokens: int = 1_000_000
    max_request_tokens: int = 100_000

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


# Validates on import — fails fast
settings = Settings()
```

### 6.5 Never Leak Keys to the Client

This is the most common security mistake in AI applications. In Next.js:

```typescript
// WRONG — this key is bundled into the client JavaScript
// Anyone can see it in their browser's DevTools
const NEXT_PUBLIC_ANTHROPIC_KEY = process.env.NEXT_PUBLIC_ANTHROPIC_API_KEY;

// RIGHT — API keys are only accessed in server-side code
// app/api/chat/route.ts (server-side only)
const key = process.env.ANTHROPIC_API_KEY; // No NEXT_PUBLIC_ prefix

// The client calls your API route, not the LLM provider directly
// app/page.tsx (client-side)
const { messages } = useChat({ api: "/api/chat" });
```

**Rule:** If an environment variable name starts with `NEXT_PUBLIC_`, it is included in the client bundle. Never put API keys in `NEXT_PUBLIC_` variables.

---

## 7. Deploying to Cloud Run (GCP)

### 7.1 When to Use Cloud Run

Cloud Run is Google Cloud's container platform. It sits between Lambda and full Kubernetes -- you get containers with auto-scaling and pay-per-use, without managing a cluster.

**Good for AI because:**
- Up to 60-minute request timeout (vs Lambda's 15 minutes)
- Up to 32 GB memory, 8 vCPUs
- Native streaming support
- Scale to zero (pay nothing when idle)
- GPU support (in preview)

### 7.2 Cloud Run Deployment

```yaml
# cloudbuild.yaml — Google Cloud Build configuration
steps:
  - name: "gcr.io/cloud-builders/docker"
    args:
      - "build"
      - "-t"
      - "gcr.io/$PROJECT_ID/ai-service:$COMMIT_SHA"
      - "-f"
      - "Dockerfile.python"
      - "."

  - name: "gcr.io/cloud-builders/docker"
    args: ["push", "gcr.io/$PROJECT_ID/ai-service:$COMMIT_SHA"]

  - name: "gcr.io/cloud-builders/gcloud"
    args:
      - "run"
      - "deploy"
      - "ai-service"
      - "--image=gcr.io/$PROJECT_ID/ai-service:$COMMIT_SHA"
      - "--region=us-central1"
      - "--platform=managed"
      - "--memory=2Gi"
      - "--cpu=2"
      - "--timeout=300"
      - "--concurrency=80"
      - "--min-instances=1"        # Keep one instance warm
      - "--max-instances=50"
      - "--set-secrets=ANTHROPIC_API_KEY=anthropic-key:latest"
      - "--allow-unauthenticated"  # Remove if behind auth
```

---

## 8. The Deployment Decision Tree

### 8.1 Choose Your Target

```
START: What kind of AI workload?
  │
  ├─ Simple chat / structured output / RAG query?
  │   │
  │   ├─ Need GPU? ──→ Serverless GPU (Replicate, Modal) — see Ch 57
  │   │
  │   ├─ Response time > 60 seconds?
  │   │   ├─ Yes ──→ Container (Cloud Run, ECS Fargate)
  │   │   └─ No ──→ Serverless Function (Vercel, Lambda)
  │   │
  │   └─ Already on Vercel? ──→ Vercel Functions (simplest path)
  │
  ├─ Agent with tools that runs for minutes?
  │   │
  │   ├─ Needs internet access? ──→ Sandboxed container (Ch 55)
  │   ├─ Needs file system? ──→ Container with volume mounts
  │   └─ Simple tools, < 5 min? ──→ Lambda with 300s timeout
  │
  ├─ Background job (embedding generation, batch processing)?
  │   │
  │   ├─ Predictable schedule? ──→ ECS Task / Cloud Run Job
  │   └─ Event-driven? ──→ Lambda triggered by SQS/EventBridge
  │
  └─ Custom model serving?
      │
      ├─ Open-source model? ──→ GPU container (Ch 57)
      ├─ Fine-tuned model? ──→ SageMaker / vLLM on GPU (Ch 47, 57)
      └─ Small model (< 1B params)? ──→ CPU container is fine
```

### 8.2 Cost Comparison (Realistic AI Workload)

Assume: 10,000 AI requests/day, average 5 seconds per request, 256MB memory.

| Platform | Monthly Cost | Cold Starts | Max Timeout | Streaming |
|----------|-------------|-------------|-------------|-----------|
| Vercel Functions (Pro) | ~$20/month | Low (Fluid Compute) | 60s | Native |
| AWS Lambda | ~$25/month | Medium | 900s | Function URL |
| Cloud Run | ~$30/month | Low (min 1) | 3600s | Native |
| ECS Fargate (1 task) | ~$80/month | None | Unlimited | Native |
| EC2 (t3.medium) | ~$30/month | None | Unlimited | Native |

**Note:** These costs are for the compute only. The LLM API costs (Ch 49) dwarf the infrastructure costs for most applications. At 10,000 requests/day with Claude Sonnet, you are paying ~$150-300/month in API costs. The infrastructure is a rounding error.

### 8.3 The Real Decision

For most teams deploying AI features:

1. **Start with Vercel Functions** if you are building a Next.js application with AI features. Zero infrastructure management, native streaming, simple deployment.

2. **Use containers** when you hit Vercel's limits: long-running agents, GPU needs, complex dependencies, or workloads that need persistent state.

3. **Use Lambda** when you need event-driven AI processing (process S3 uploads, handle SQS messages) or when you are already deep in the AWS ecosystem.

4. **Use GPU infrastructure** only when you are serving your own models (Ch 57). If you are calling OpenAI or Anthropic APIs, you do not need GPUs.

---

## 9. Production Deployment Checklist

### 9.1 Before You Deploy

```markdown
## AI Application Deployment Checklist

### Secrets
- [ ] All API keys are in environment variables (not in code)
- [ ] No keys use `NEXT_PUBLIC_` prefix
- [ ] Different keys for dev/staging/production
- [ ] Keys have appropriate spending limits set at the provider
- [ ] Key rotation plan documented

### Timeouts
- [ ] Function/container timeout exceeds expected max LLM response time
- [ ] Client-side timeout configured (don't wait forever)
- [ ] Streaming configured (users shouldn't stare at a blank screen)

### Error Handling
- [ ] LLM API errors return graceful fallbacks
- [ ] Rate limit errors trigger retry with backoff (Ch 54)
- [ ] Network errors don't crash the application
- [ ] Partial streaming failures are handled

### Monitoring
- [ ] Request latency tracked (p50, p95, p99)
- [ ] Token usage tracked per endpoint
- [ ] Error rate alerts configured
- [ ] Cost alerts configured at provider level

### Security (Ch 48)
- [ ] Input validation on all user-facing endpoints
- [ ] Output filtering for PII/sensitive data
- [ ] Rate limiting per user (Ch 54)
- [ ] CORS configured correctly
```

### 9.2 Vercel Deployment Example (End-to-End)

```bash
# 1. Create the project
npx create-next-app@latest my-ai-app --typescript --tailwind --app
cd my-ai-app

# 2. Install AI dependencies
npm install ai @ai-sdk/anthropic zod

# 3. Set up environment
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env.local
echo ".env.local" >> .gitignore

# 4. Create the AI route (app/api/chat/route.ts — code from Section 2.2)

# 5. Create the chat UI (app/page.tsx — code from Section 2.4)

# 6. Deploy to Vercel
vercel

# 7. Set production environment variable
vercel env add ANTHROPIC_API_KEY production

# 8. Deploy to production
vercel --prod
```

---

## 10. What's Next

You have an AI application running in production. It is calling LLM providers, streaming responses, and serving users. But it is calling those providers directly -- every request hits OpenAI or Anthropic with no intermediary.

In Chapter 54, we will put an **API gateway** in front of your providers. The gateway handles rate limiting (so one user can't burn your entire token budget), retry with backoff (so provider outages don't crash your app), failover (so you can switch providers automatically), and caching (so repeated questions don't cost you twice). The gateway is the control plane for your AI application's most expensive dependency: the LLM providers themselves.

---

## Key Takeaways

1. **AI deployments are different** because of long response times, streaming requirements, secret management complexity, and non-determinism.
2. **Start serverless** (Vercel Functions) for most AI features. Move to containers only when you hit specific limits.
3. **Streaming is not optional.** Users will not wait 5 seconds staring at a blank screen. Configure streaming from day one.
4. **Validate environment variables at startup.** Fail fast with clear error messages, not cryptic runtime errors.
5. **Never expose API keys to the client.** No `NEXT_PUBLIC_` prefix for any secret.
6. **The compute cost is a rounding error** compared to LLM API costs for most applications. Optimize API usage (Ch 49) before optimizing infrastructure.

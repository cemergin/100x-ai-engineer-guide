<!--
  CHAPTER: 3
  TITLE: Your First LLM Call
  PART: 1 — Building with LLM APIs
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 0, Chapter 2
  KEY_TOPICS: SDK setup, chat completions, system prompts, user messages, assistant messages, streaming, error handling, retries, exponential backoff
  DIFFICULTY: Beginner
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 3: Your First LLM Call

> **Part 1 — Building with LLM APIs** | Phase 1: Get Dangerous | Prerequisites: Ch 0, Ch 2 | Difficulty: Beginner | Language: TypeScript

Time to write code. In this chapter, you'll set up a TypeScript project, install the SDKs, and make your first LLM API call. Then you'll learn the three message roles that make conversations work, stream a response token by token, and build the error handling that production apps need.

This is the chapter where everything from Part 0 becomes real. Tokens? You'll see them reflected in your API bill. Context windows? You'll hit them when conversations get long. Temperature? You'll set it as a parameter. By the end of this chapter, you'll have a working foundation that every later chapter builds on.

One thing before we start: resist the urge to build something fancy. This chapter is about building the *right* foundation. A single, well-structured API call with proper error handling is worth more than a sloppy chatbot. We'll add complexity in the chapters that follow.

### In This Chapter
- Setting up a Node.js/TypeScript project for AI development
- Environment variables and API key management
- Making your first chat completion call
- Message roles: system, user, assistant
- Streaming responses (why and how)
- Error handling: rate limits, timeouts, retries with exponential backoff

### Related Chapters
- **Ch 0 (How LLMs Work)** — spirals back: tokens, context windows, temperature are now API parameters
- **Ch 2 (AI Landscape)** — spirals back: SDKs and providers are now installed and configured
- **Ch 4 (Prompt Engineering)** — spirals forward: refine WHAT you send in these calls
- **Ch 5 (Structured Output)** — spirals forward: get typed data back, not just strings
- **Ch 8 (Tool Calling)** — spirals forward: these LLM calls happen inside tool-calling agent loops

---

## 1. Project Setup

### 1.1 Initialize the Project

Let's create a clean TypeScript project for AI development:

```bash
mkdir ai-engineer-sandbox
cd ai-engineer-sandbox

# Initialize with TypeScript
npm init -y
npm install typescript tsx @types/node --save-dev

# Initialize tsconfig
npx tsc --init --target es2022 --module nodenext --moduleResolution nodenext --outDir dist --strict true
```

### 1.2 Install the AI SDKs

We'll install the Vercel AI SDK (our primary SDK) plus provider adapters:

```bash
# Core SDK
npm install ai

# Provider adapters — install the ones you have API keys for
npm install @ai-sdk/openai
npm install @ai-sdk/anthropic

# Optional: Google provider
npm install @ai-sdk/google

# Utilities
npm install zod dotenv
```

### 1.3 Set Up Environment Variables

Create a `.env.local` file for your API keys:

```bash
# .env.local — NEVER commit this file
OPENAI_API_KEY=sk-proj-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here
# GOOGLE_GENERATIVE_AI_API_KEY=your-key-here
```

Add it to `.gitignore`:

```bash
echo ".env.local" >> .gitignore
echo ".env" >> .gitignore
echo "node_modules" >> .gitignore
echo "dist" >> .gitignore
```

Create a script to load environment variables. Add this to your `package.json`:

```json
{
  "scripts": {
    "dev": "tsx --env-file=.env.local"
  }
}
```

Now you can run any TypeScript file with environment variables loaded:

```bash
npm run dev src/hello.ts
```

### 1.4 Project Structure

Create the initial structure:

```bash
mkdir -p src/ai src/examples
```

```
ai-engineer-sandbox/
├── src/
│   ├── ai/           ← AI client setup, prompts, schemas
│   └── examples/     ← Runnable examples from each chapter
├── .env.local         ← API keys (gitignored)
├── .gitignore
├── package.json
└── tsconfig.json
```

---

## 2. Your First API Call

### 2.1 The Simplest Possible Call

Create `src/examples/01-first-call.ts`:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = await generateText({
    model: openai("gpt-4o-mini"),
    prompt: "What is TypeScript in one sentence?",
  });

  console.log(result.text);
  console.log(`\nTokens used: ${result.usage.totalTokens}`);
}

main();
```

Run it:

```bash
npm run dev src/examples/01-first-call.ts
```

Output:

```
TypeScript is a statically typed superset of JavaScript that compiles
to plain JavaScript, adding optional type annotations and advanced
features for building large-scale applications.

Tokens used: 42
```

That's it. You just called an LLM from TypeScript.

Let's break down what happened:

1. `generateText` sends a request to the model and waits for the full response
2. `openai("gpt-4o-mini")` specifies the model — cheap, fast, good enough for this
3. `prompt` is the simplest input format — just a string
4. `result.text` is the model's response as a string
5. `result.usage` tells you how many tokens were used (for cost tracking)

### 2.2 The Same Call with Anthropic

Swap the provider — everything else stays the same:

```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

async function main() {
  const result = await generateText({
    model: anthropic("claude-3-5-haiku-20241022"),
    prompt: "What is TypeScript in one sentence?",
  });

  console.log(result.text);
  console.log(`\nTokens used: ${result.usage.totalTokens}`);
}

main();
```

Same `generateText` function, same `prompt` parameter, same `result` shape. Only the model changed. This is the power of a provider-agnostic SDK.

### 2.3 Adding Temperature

Remember temperature from Chapter 0? Here's where you use it:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  // Deterministic — same answer every time
  const deterministic = await generateText({
    model: openai("gpt-4o-mini"),
    prompt: "Name one programming language.",
    temperature: 0,
  });
  console.log("Temperature 0:", deterministic.text);

  // Creative — different answer each time
  const creative = await generateText({
    model: openai("gpt-4o-mini"),
    prompt: "Name one programming language.",
    temperature: 1.5,
  });
  console.log("Temperature 1.5:", creative.text);
}

main();
```

Run it several times. Temperature 0 gives you the same language every time (probably "Python"). Temperature 1.5 gives you a different one each run.

---

## 3. Message Roles: The Conversation Structure

### 3.1 The Three Roles

The simple `prompt` parameter is fine for one-off questions. But real applications use the **messages** format with three roles:

**`system`** — Instructions to the model. Sets behavior, personality, constraints. The model treats this as authoritative context. *Not visible to the end user.*

**`user`** — Messages from the human. Questions, instructions, data to process.

**`assistant`** — Messages from the model. Previous responses in a conversation. You include these to maintain context across turns.

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = await generateText({
    model: openai("gpt-4o-mini"),
    messages: [
      {
        role: "system",
        content: `You are a senior TypeScript engineer. You give concise, 
                  practical answers. When showing code, always include types. 
                  Never use 'any'.`,
      },
      {
        role: "user",
        content: "How do I read a file in Node.js?",
      },
    ],
  });

  console.log(result.text);
}

main();
```

The system prompt shapes *how* the model responds. Without it, you get generic answers. With it, you get answers in the style and format you want.

### 3.2 Multi-Turn Conversations

To maintain a conversation, include previous turns as alternating `user` and `assistant` messages:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = await generateText({
    model: openai("gpt-4o-mini"),
    messages: [
      {
        role: "system",
        content: "You are a helpful cooking assistant.",
      },
      {
        role: "user",
        content: "I have chicken, rice, and broccoli. What should I make?",
      },
      {
        role: "assistant",
        content: "You could make a chicken stir-fry with rice and broccoli! Season the chicken with soy sauce, garlic, and ginger.",
      },
      {
        role: "user",
        content: "I don't have soy sauce. What can I substitute?",
      },
    ],
  });

  console.log(result.text);
  // The model "remembers" the conversation because you sent it all
}

main();
```

**Key insight:** The model doesn't actually remember anything. You're sending the *entire conversation* with every request. The model processes all of it and generates the next response. This is why conversations cost more tokens over time — you're paying to re-send the history.

### 3.3 System Prompts: Your Most Important Tool

The system prompt is the single most impactful thing you control. A great system prompt is the difference between a generic chatbot and a useful product.

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

// BAD: vague system prompt
const vagueResult = await generateText({
  model: openai("gpt-4o-mini"),
  messages: [
    { role: "system", content: "You are helpful." },
    { role: "user", content: "How do I handle errors?" },
  ],
});

// GOOD: specific system prompt
const specificResult = await generateText({
  model: openai("gpt-4o-mini"),
  messages: [
    {
      role: "system",
      content: `You are a TypeScript error handling expert. Follow these rules:
1. Always show try/catch with specific error types, never catch(e: any)
2. Include the error type in the catch clause using instanceof checks
3. Show both the happy path and the error path
4. Use real-world examples, not foo/bar
5. Keep code examples under 20 lines
6. End with one practical tip`,
    },
    { role: "user", content: "How do I handle errors?" },
  ],
});
```

The specific system prompt produces dramatically better results. We'll go deep on prompt engineering in Chapter 4.

### 3.4 Max Tokens

Control how long the response can be:

```typescript
const result = await generateText({
  model: openai("gpt-4o-mini"),
  maxTokens: 100,  // Limit response length
  messages: [
    { role: "system", content: "Be concise." },
    { role: "user", content: "Explain quantum computing." },
  ],
});

// result.finishReason tells you why generation stopped:
// "stop" = model finished naturally
// "length" = hit maxTokens limit
console.log("Finish reason:", result.finishReason);
```

**When to set maxTokens:**
- Cost control (cap spending per request)
- Speed (shorter responses are faster)
- Format enforcement (force concise answers)
- Safety (prevent runaway generation)

**When NOT to set it too low:**
- Code generation (might cut off mid-function)
- Structured output (might cut off mid-JSON)
- Detailed explanations (might seem incomplete)

---

## 4. Streaming: Token-by-Token Responses

### 4.1 Why Streaming Matters

When you call `generateText`, you wait for the ENTIRE response before getting anything back. For a 500-token response at ~50 tokens/second, that's 10 seconds of staring at a loading spinner.

**Streaming** sends tokens as they're generated. The user sees the response appear word by word, like watching someone type. This feels dramatically faster, even though the total time is the same.

```
Without streaming:
  User sends message → [10 second wait] → Full response appears

With streaming:
  User sends message → [200ms] → First word appears → words keep appearing → done
```

Time to first token (TTFT) drops from 10 seconds to ~200ms. Users perceive this as nearly instant.

### 4.2 Streaming with the Vercel AI SDK

The Vercel AI SDK makes streaming easy with `streamText`:

```typescript
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = streamText({
    model: openai("gpt-4o-mini"),
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Explain why TypeScript is popular." },
    ],
  });

  // Option 1: Print tokens as they arrive
  for await (const textPart of result.textStream) {
    process.stdout.write(textPart);
  }
  console.log(); // newline at the end

  // After streaming is done, you can access the full result
  const finalResult = await result;
  console.log(`\nTotal tokens: ${finalResult.usage.totalTokens}`);
}

main();
```

Run this and you'll see text appear character by character in your terminal — much more satisfying than waiting.

### 4.3 Stream Processing Patterns

The stream gives you several event types:

```typescript
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = streamText({
    model: openai("gpt-4o-mini"),
    prompt: "Write a haiku about TypeScript.",
  });

  // Full stream with all event types
  for await (const part of result.fullStream) {
    switch (part.type) {
      case "text-delta":
        process.stdout.write(part.textDelta);
        break;
      case "finish":
        console.log("\n\nFinish reason:", part.finishReason);
        console.log("Usage:", part.usage);
        break;
      case "error":
        console.error("Stream error:", part.error);
        break;
    }
  }
}

main();
```

### 4.4 Collecting a Streamed Response

Sometimes you want to stream to the user AND collect the full text:

```typescript
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = streamText({
    model: openai("gpt-4o-mini"),
    prompt: "List 3 TypeScript tips.",
  });

  // Stream to terminal while collecting
  let fullText = "";
  for await (const textPart of result.textStream) {
    process.stdout.write(textPart);
    fullText += textPart;
  }
  console.log();

  // Now you have the full text for logging, storage, etc.
  console.log(`\nCollected ${fullText.length} characters`);

  // Or just use the built-in property
  const finalText = await result.text;
  console.log("Same text:", finalText === fullText); // true
}

main();
```

### 4.5 When to Stream vs When Not To

| Scenario | Stream? | Why |
|---|---|---|
| Chat interface | Yes | Users need immediate feedback |
| API endpoint consumed by another service | Usually no | The consumer wants the full response |
| Data extraction / structured output | No | You need the complete JSON before parsing |
| Long-form generation | Yes | Users shouldn't wait 30+ seconds |
| Background processing (evals, batch) | No | No user is watching |

---

## 5. Error Handling: Production-Grade Patterns

### 5.1 What Can Go Wrong

LLM APIs fail in predictable ways:

| Error | HTTP Code | Cause | Frequency |
|---|---|---|---|
| Rate limit | 429 | Too many requests | Common |
| Server error | 500 | Provider issues | Occasional |
| Timeout | — | Slow response | Occasional |
| Context too long | 400 | Input exceeds model limit | Depends on usage |
| Invalid request | 400 | Bad parameters | During development |
| Authentication | 401 | Bad API key | During setup |
| Insufficient quota | 402/429 | Out of credits | When you forget to add a payment method |

### 5.2 Basic Error Handling

At minimum, catch and handle errors:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function callLLM(prompt: string): Promise<string> {
  try {
    const result = await generateText({
      model: openai("gpt-4o-mini"),
      prompt,
    });
    return result.text;
  } catch (error) {
    if (error instanceof Error) {
      // Check for specific error types
      if (error.message.includes("rate_limit")) {
        console.error("Rate limited — try again in a moment");
      } else if (error.message.includes("context_length")) {
        console.error("Input too long — reduce the prompt or conversation history");
      } else if (error.message.includes("401")) {
        console.error("Authentication failed — check your API key");
      } else {
        console.error("LLM call failed:", error.message);
      }
    }
    throw error;
  }
}
```

### 5.3 Retry with Exponential Backoff

The most important error handling pattern for LLM APIs. When a request fails (especially with 429 or 500), wait and retry — but increase the wait time with each attempt:

```typescript
interface RetryOptions {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
}

const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
};

async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options };
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));

      // Don't retry auth errors or invalid requests
      if (
        lastError.message.includes("401") ||
        lastError.message.includes("invalid_request")
      ) {
        throw lastError;
      }

      if (attempt === opts.maxRetries) {
        break;
      }

      // Calculate delay with exponential backoff + jitter
      const baseDelay = Math.min(
        opts.initialDelayMs * Math.pow(opts.backoffMultiplier, attempt),
        opts.maxDelayMs
      );
      // Add random jitter (0-25% of base delay) to prevent thundering herd
      const jitter = baseDelay * 0.25 * Math.random();
      const delay = baseDelay + jitter;

      console.warn(
        `Attempt ${attempt + 1} failed. Retrying in ${Math.round(delay)}ms...`
      );
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}
```

Usage:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = await withRetry(() =>
    generateText({
      model: openai("gpt-4o-mini"),
      prompt: "Hello world",
    })
  );

  console.log(result.text);
}

main();
```

### 5.4 Timeout Handling

LLM calls can hang, especially under high load. Set timeouts:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function callWithTimeout(
  prompt: string,
  timeoutMs: number = 30000
): Promise<string> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const result = await generateText({
      model: openai("gpt-4o-mini"),
      prompt,
      abortSignal: controller.signal,
    });
    return result.text;
  } finally {
    clearTimeout(timeout);
  }
}

// Usage
try {
  const response = await callWithTimeout("Explain TypeScript", 10000);
  console.log(response);
} catch (error) {
  if (error instanceof Error && error.name === "AbortError") {
    console.error("Request timed out after 10 seconds");
  }
}
```

### 5.5 Combining It All: A Production-Ready LLM Client

Here's a complete, reusable LLM client that you can use throughout your projects:

```typescript
// src/ai/client.ts
import { generateText, streamText, type CoreMessage } from "ai";
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";

type Provider = "openai" | "anthropic";
type ModelId = string;

interface LLMCallOptions {
  model?: ModelId;
  provider?: Provider;
  messages: CoreMessage[];
  temperature?: number;
  maxTokens?: number;
  timeoutMs?: number;
  maxRetries?: number;
}

function getModel(provider: Provider, modelId: string) {
  switch (provider) {
    case "openai":
      return openai(modelId);
    case "anthropic":
      return anthropic(modelId);
    default:
      throw new Error(`Unknown provider: ${provider}`);
  }
}

export async function llm(options: LLMCallOptions): Promise<string> {
  const {
    model: modelId = "gpt-4o-mini",
    provider = "openai",
    messages,
    temperature = 0.7,
    maxTokens = 1024,
    timeoutMs = 30000,
    maxRetries = 3,
  } = options;

  const model = getModel(provider, modelId);
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const result = await generateText({
        model,
        messages,
        temperature,
        maxTokens,
        abortSignal: controller.signal,
      });

      return result.text;
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));

      // Don't retry client errors
      if (
        lastError.message.includes("401") ||
        lastError.message.includes("invalid")
      ) {
        throw lastError;
      }

      if (attempt < maxRetries) {
        const delay = Math.min(1000 * Math.pow(2, attempt), 30000);
        const jitter = delay * 0.25 * Math.random();
        await new Promise((r) => setTimeout(r, delay + jitter));
      }
    } finally {
      clearTimeout(timeout);
    }
  }

  throw lastError;
}

// Usage:
// const response = await llm({
//   messages: [
//     { role: "system", content: "You are a helpful assistant." },
//     { role: "user", content: "Hello!" },
//   ],
// });
```

---

## 6. Working with Responses

### 6.1 The Response Object

`generateText` returns a rich object:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function main() {
  const result = await generateText({
    model: openai("gpt-4o-mini"),
    messages: [
      { role: "system", content: "Be concise." },
      { role: "user", content: "What is TypeScript?" },
    ],
  });

  // The generated text
  console.log("Text:", result.text);

  // Why did generation stop?
  console.log("Finish reason:", result.finishReason);
  // "stop" = model finished naturally
  // "length" = hit maxTokens
  // "tool-calls" = model wants to call a tool (Ch 8)

  // Token usage
  console.log("Prompt tokens:", result.usage.promptTokens);
  console.log("Completion tokens:", result.usage.completionTokens);
  console.log("Total tokens:", result.usage.totalTokens);

  // The full message (useful for building conversation history)
  console.log("Response message:", result.response.messages);
}

main();
```

### 6.2 Building Conversation History

Here's how to build a multi-turn conversation programmatically:

```typescript
import { generateText, type CoreMessage } from "ai";
import { openai } from "@ai-sdk/openai";
import * as readline from "readline";

async function chat() {
  const messages: CoreMessage[] = [
    {
      role: "system",
      content: "You are a friendly assistant. Keep responses under 2 sentences.",
    },
  ];

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const askQuestion = () => {
    rl.question("\nYou: ", async (input) => {
      if (input.toLowerCase() === "quit") {
        console.log("Goodbye!");
        rl.close();
        return;
      }

      // Add user message
      messages.push({ role: "user", content: input });

      try {
        const result = await generateText({
          model: openai("gpt-4o-mini"),
          messages,
        });

        // Add assistant response to history
        messages.push({ role: "assistant", content: result.text });

        console.log(`\nAssistant: ${result.text}`);
        console.log(`  (${result.usage.totalTokens} tokens this turn)`);
      } catch (error) {
        console.error("Error:", error);
      }

      askQuestion();
    });
  };

  console.log('Chat started. Type "quit" to exit.');
  askQuestion();
}

chat();
```

Run it:

```bash
npm run dev src/examples/04-chat.ts
```

Notice how each turn sends ALL previous messages. After 10 turns, you're sending 10 user messages + 10 assistant messages + the system prompt. This is why conversations get expensive and why you'll need context management strategies (Ch 12).

### 6.3 Estimating Costs

Build cost awareness into your code from day one:

```typescript
interface CostEstimate {
  inputTokens: number;
  outputTokens: number;
  inputCost: number;
  outputCost: number;
  totalCost: number;
  model: string;
}

// Prices per 1M tokens (update as prices change)
const PRICING: Record<string, { input: number; output: number }> = {
  "gpt-4o-mini": { input: 0.15, output: 0.60 },
  "gpt-4o": { input: 2.5, output: 10.0 },
  "claude-3-5-haiku-20241022": { input: 0.80, output: 4.0 },
  "claude-sonnet-4-20250514": { input: 3.0, output: 15.0 },
};

function estimateCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): CostEstimate {
  const pricing = PRICING[model] ?? { input: 1.0, output: 3.0 };

  const inputCost = (inputTokens / 1_000_000) * pricing.input;
  const outputCost = (outputTokens / 1_000_000) * pricing.output;

  return {
    inputTokens,
    outputTokens,
    inputCost,
    outputCost,
    totalCost: inputCost + outputCost,
    model,
  };
}

// After an API call:
// const cost = estimateCost("gpt-4o-mini", result.usage.promptTokens, result.usage.completionTokens);
// console.log(`Cost: $${cost.totalCost.toFixed(6)}`);
```

---

## 7. Provider-Specific SDKs (When You Need Them)

### 7.1 OpenAI SDK Directly

Sometimes you need the OpenAI SDK directly — for the batch API, assistants API, or fine-tuning:

```typescript
import OpenAI from "openai";

const client = new OpenAI(); // reads OPENAI_API_KEY from env

async function main() {
  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Hello!" },
    ],
    temperature: 0.7,
    max_tokens: 256,
  });

  const message = response.choices[0].message;
  console.log(message.content);

  // Usage info
  console.log("Tokens:", response.usage);
}

main();
```

### 7.2 Anthropic SDK Directly

The Anthropic SDK has a slightly different API shape:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from env

async function main() {
  const response = await client.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 256,
    system: "You are a helpful assistant.", // system is a top-level param
    messages: [
      { role: "user", content: "Hello!" },
    ],
  });

  // Anthropic returns content blocks
  if (response.content[0].type === "text") {
    console.log(response.content[0].text);
  }

  console.log("Tokens:", response.usage);
}

main();
```

**Key differences from OpenAI:**
- `system` is a top-level parameter, not a message
- `max_tokens` is required (not optional)
- Response is `content` blocks, not `choices[0].message.content`
- Usage field names differ (`input_tokens` vs `prompt_tokens`)

This is exactly why the Vercel AI SDK is valuable — it normalizes these differences.

### 7.3 Streaming with Provider SDKs

Both SDKs support streaming, but with different APIs:

```typescript
// OpenAI streaming
import OpenAI from "openai";

const client = new OpenAI();

const stream = await client.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "Hello!" }],
  stream: true,
});

for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta?.content;
  if (delta) process.stdout.write(delta);
}
```

```typescript
// Anthropic streaming
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const stream = client.messages.stream({
  model: "claude-3-5-haiku-20241022",
  max_tokens: 256,
  messages: [{ role: "user", content: "Hello!" }],
});

stream.on("text", (text) => process.stdout.write(text));
await stream.finalMessage();
```

Compare these to the Vercel AI SDK version:

```typescript
// Vercel AI SDK — same code for both providers
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

const result = streamText({
  model: openai("gpt-4o-mini"),
  prompt: "Hello!",
});

for await (const text of result.textStream) {
  process.stdout.write(text);
}
```

The Vercel AI SDK wins on ergonomics.

---

## 8. Practical Patterns

### 8.1 Logging Every Call

In production, you want to log every LLM call for debugging, cost tracking, and quality monitoring:

```typescript
import { generateText, type CoreMessage } from "ai";
import { openai } from "@ai-sdk/openai";

interface LLMLog {
  timestamp: string;
  model: string;
  messages: CoreMessage[];
  response: string;
  finishReason: string;
  inputTokens: number;
  outputTokens: number;
  durationMs: number;
}

async function loggedGenerateText(
  model: string,
  messages: CoreMessage[]
) {
  const start = Date.now();

  const result = await generateText({
    model: openai(model),
    messages,
  });

  const log: LLMLog = {
    timestamp: new Date().toISOString(),
    model,
    messages,
    response: result.text,
    finishReason: result.finishReason,
    inputTokens: result.usage.promptTokens,
    outputTokens: result.usage.completionTokens,
    durationMs: Date.now() - start,
  };

  // In production, send this to your telemetry system
  console.log(JSON.stringify(log, null, 2));

  return result;
}
```

### 8.2 Provider Fallback

If your primary provider is down, fall back to another:

```typescript
import { generateText, type CoreMessage } from "ai";
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";

const FALLBACK_CHAIN = [
  { provider: openai, model: "gpt-4o-mini" },
  { provider: anthropic, model: "claude-3-5-haiku-20241022" },
];

async function generateWithFallback(
  messages: CoreMessage[]
): Promise<string> {
  for (const { provider, model } of FALLBACK_CHAIN) {
    try {
      const result = await generateText({
        model: provider(model),
        messages,
      });
      return result.text;
    } catch (error) {
      console.warn(`${model} failed, trying next provider...`);
      continue;
    }
  }
  throw new Error("All providers failed");
}
```

### 8.3 Parallel Calls

When you need multiple independent LLM calls, run them in parallel:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

async function analyzeText(text: string) {
  const model = openai("gpt-4o-mini");

  // Three independent analyses in parallel
  const [sentiment, summary, keywords] = await Promise.all([
    generateText({
      model,
      messages: [
        { role: "system", content: "Respond with one word: positive, negative, or neutral." },
        { role: "user", content: text },
      ],
    }),
    generateText({
      model,
      messages: [
        { role: "system", content: "Summarize in one sentence." },
        { role: "user", content: text },
      ],
    }),
    generateText({
      model,
      messages: [
        { role: "system", content: "List 3-5 keywords, comma-separated." },
        { role: "user", content: text },
      ],
    }),
  ]);

  return {
    sentiment: sentiment.text.trim().toLowerCase(),
    summary: summary.text,
    keywords: keywords.text.split(",").map((k) => k.trim()),
  };
}
```

---

## 9. Key Takeaways

1. **Start with the Vercel AI SDK.** `generateText` for full responses, `streamText` for streaming. Swap providers by changing the model.

2. **Use the messages format.** System prompts set behavior. User messages are input. Assistant messages maintain conversation history.

3. **Stream for user-facing features.** Users perceive streaming as 10-50x faster than waiting for full responses.

4. **Build error handling from day one.** Retry with exponential backoff + jitter. Set timeouts. Handle rate limits gracefully.

5. **Log everything.** Model, tokens, duration, finish reason. You'll need this for debugging, cost tracking, and evals.

6. **The conversation isn't stored by the model.** You send the full history with every request. This is why conversations get expensive.

7. **Start with gpt-4o-mini.** It's cheap, fast, and good enough for most tasks. Move up only when quality demands it.

---

## What's Next

You can make LLM calls. Now let's make them better. In **Chapter 4: Prompt Engineering That Works**, you'll learn how to craft system prompts, use few-shot examples, and apply chain-of-thought prompting to get dramatically better results from the same models.

Then in **Chapter 5**, you'll learn to get structured, typed data back from LLMs — not just unstructured text. This is what turns an LLM from a chatbot into a data processing engine.

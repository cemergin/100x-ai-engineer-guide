<!--
  TYPE: appendix
  TITLE: Cheat Sheet
  UPDATED: 2026-04-10
-->

# Appendix: Cheat Sheet

> Quick reference for model comparison, SDK patterns, common architectures, token costs, and decision trees. Print this, bookmark it, keep it open in a tab.

---

## 1. Model Comparison Table

### Frontier Models (April 2026)

| Model | Provider | Context | Input $/1M | Output $/1M | Strengths | Best For |
|-------|----------|---------|-----------|------------|-----------|---------|
| Claude Opus 4 | Anthropic | 200K | $15 | $75 | Deepest reasoning, complex coding, long analysis | Architecture decisions, complex code generation, research |
| Claude Sonnet 4 | Anthropic | 200K | $3 | $15 | Best quality/cost ratio, strong coding, tools | Default for most tasks, agent loops, tool calling |
| Claude Haiku 3.5 | Anthropic | 200K | $0.80 | $4 | Fast, cheap, good enough for simple tasks | Classification, routing, summarization, bulk processing |
| GPT-4o | OpenAI | 128K | $2.50 | $10 | Strong all-around, vision, audio | Multimodal, vision tasks, second opinion |
| GPT-4o mini | OpenAI | 128K | $0.15 | $0.60 | Extremely cheap, fast | High-volume simple tasks, embeddings |
| Gemini 2.5 Pro | Google | 1M | $1.25-$2.50 | $5-$10 | Massive context, strong reasoning | Whole-codebase analysis, long documents |
| Gemini 2.5 Flash | Google | 1M | $0.15 | $0.60 | Fast with huge context | Bulk processing with long context |

### Open-Source Models

| Model | Parameters | Context | VRAM Required | Best For |
|-------|-----------|---------|---------------|---------|
| Llama 3.3 70B | 70B | 128K | 40GB (FP16) / 10GB (4-bit) | Best open-source general purpose |
| Llama 3.2 8B | 8B | 128K | 16GB (FP16) / 4GB (4-bit) | Local development, edge deployment |
| Mistral Large | 123B | 128K | 70GB (FP16) / 16GB (4-bit) | Multilingual, European compliance |
| Mistral 7B | 7B | 32K | 14GB (FP16) / 4GB (4-bit) | Smallest useful model, fast inference |
| Qwen 2.5 72B | 72B | 128K | 40GB (FP16) / 10GB (4-bit) | Strong coding, multilingual |
| Phi-3 Medium | 14B | 128K | 28GB (FP16) / 7GB (4-bit) | Small but capable, reasoning |

### Embedding Models

| Model | Dimensions | Max Tokens | $/1M Tokens | Best For |
|-------|-----------|-----------|------------|---------|
| text-embedding-3-small | 1536 | 8191 | $0.02 | Default API embedding |
| text-embedding-3-large | 3072 | 8191 | $0.13 | Higher quality retrieval |
| all-MiniLM-L6-v2 | 384 | 256 | Free (local) | Local/free, fast |
| BGE-large-en-v1.5 | 1024 | 512 | Free (local) | High quality, MTEB top tier |
| Cohere embed-v3 | 1024 | 512 | $0.10 | Multilingual, reranking compatible |

---

## 2. SDK Quick Reference

### Anthropic SDK (TypeScript)

```typescript
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic(); // Uses ANTHROPIC_API_KEY env var

// Basic message
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
});

// With system prompt
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  system: "You are a helpful assistant.",
  messages: [{ role: "user", content: "Hello" }],
});

// Streaming
const stream = client.messages.stream({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
});
for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}

// Tool calling
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  tools: [{
    name: "get_weather",
    description: "Get current weather for a location",
    input_schema: {
      type: "object",
      properties: {
        location: { type: "string", description: "City name" },
      },
      required: ["location"],
    },
  }],
  messages: [{ role: "user", content: "Weather in NYC?" }],
});
// Check response.content for tool_use blocks
// Execute tool, send tool_result back

// Vision
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{
    role: "user",
    content: [
      { type: "image", source: { type: "base64", media_type: "image/png", data: base64String } },
      { type: "text", text: "What's in this image?" },
    ],
  }],
});
```

### Anthropic SDK (Python)

```python
import anthropic
client = anthropic.Anthropic()  # Uses ANTHROPIC_API_KEY env var

# Basic message
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.content[0].text)

# Streaming
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# Tool calling
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[{
        "name": "get_weather",
        "description": "Get current weather",
        "input_schema": {
            "type": "object",
            "properties": {"location": {"type": "string"}},
            "required": ["location"],
        },
    }],
    messages=[{"role": "user", "content": "Weather in NYC?"}],
)
```

### OpenAI SDK (TypeScript)

```typescript
import OpenAI from "openai";
const client = new OpenAI(); // Uses OPENAI_API_KEY env var

// Basic chat completion
const response = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Hello" },
  ],
});

// Structured output
const response = await client.chat.completions.create({
  model: "gpt-4o",
  response_format: { type: "json_object" },
  messages: [
    { role: "system", content: "Respond in JSON format." },
    { role: "user", content: "List 3 colors" },
  ],
});

// Tool calling
const response = await client.chat.completions.create({
  model: "gpt-4o",
  tools: [{
    type: "function",
    function: {
      name: "get_weather",
      description: "Get current weather",
      parameters: {
        type: "object",
        properties: { location: { type: "string" } },
        required: ["location"],
      },
    },
  }],
  messages: [{ role: "user", content: "Weather in NYC?" }],
});
```

### Vercel AI SDK (TypeScript)

```typescript
import { generateText, streamText, generateObject, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

// Generate text
const { text } = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  prompt: "Hello",
});

// Stream text
const result = streamText({
  model: anthropic("claude-sonnet-4-20250514"),
  prompt: "Tell me a story",
});
for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}

// Structured output with Zod
const { object } = await generateObject({
  model: anthropic("claude-sonnet-4-20250514"),
  schema: z.object({
    name: z.string(),
    age: z.number(),
    hobbies: z.array(z.string()),
  }),
  prompt: "Generate a person profile",
});

// Tool calling
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: {
    weather: tool({
      description: "Get weather for a location",
      parameters: z.object({ location: z.string() }),
      execute: async ({ location }) => {
        return `72F and sunny in ${location}`;
      },
    }),
  },
  prompt: "Weather in NYC?",
});
```

---

## 3. Common Patterns (Code Snippets)

### The Agent Loop

```typescript
// The fundamental agent pattern (Ch 9)
async function agentLoop(task: string, tools: Tool[], maxSteps = 20) {
  const messages: Message[] = [{ role: "user", content: task }];

  for (let i = 0; i < maxSteps; i++) {
    const response = await llm.call({ messages, tools });

    // Check if agent is done
    if (response.stopReason === "end_turn") {
      return response.text;
    }

    // Execute tool calls
    messages.push({ role: "assistant", content: response.content });
    const toolResults = await Promise.all(
      response.toolCalls.map((tc) => executeTool(tc))
    );
    messages.push({ role: "user", content: toolResults });
  }

  return "Max steps reached";
}
```

### RAG Pipeline

```typescript
// The fundamental RAG pattern (Ch 15)
async function rag(query: string) {
  // 1. Embed the query
  const queryEmbedding = await embed(query);

  // 2. Retrieve relevant chunks
  const chunks = await vectorStore.query({
    embedding: queryEmbedding,
    topK: 5,
  });

  // 3. Build context
  const context = chunks.map((c) => c.text).join("\n\n---\n\n");

  // 4. Generate answer grounded in context
  const answer = await llm.call({
    system: "Answer using ONLY the provided context. Cite sources.",
    messages: [
      { role: "user", content: `Context:\n${context}\n\nQuestion: ${query}` },
    ],
  });

  return { answer, sources: chunks };
}
```

### Eval Framework

```typescript
// The fundamental eval pattern (Ch 18-19)
interface EvalCase {
  input: string;
  expected: string;
  scorer: (output: string, expected: string) => number;
}

async function runEval(cases: EvalCase[], systemPrompt: string) {
  const results = await Promise.all(
    cases.map(async (c) => {
      const output = await llm.call({
        system: systemPrompt,
        messages: [{ role: "user", content: c.input }],
      });
      const score = c.scorer(output, c.expected);
      return { input: c.input, output, expected: c.expected, score };
    })
  );

  const avgScore = results.reduce((s, r) => s + r.score, 0) / results.length;
  return { results, avgScore };
}
```

### Tool Definition

```typescript
// Standard tool definition pattern (Ch 8)
const tool = {
  name: "search_database",
  description: "Search the product database by name, category, or ID",
  input_schema: {
    type: "object",
    properties: {
      query: { type: "string", description: "Search query" },
      category: {
        type: "string",
        enum: ["electronics", "clothing", "food"],
        description: "Optional category filter",
      },
      limit: { type: "number", description: "Max results (default 10)" },
    },
    required: ["query"],
  },
};
```

### MCP Server

```typescript
// Minimal MCP server (Ch 24)
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "my-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "my_tool",
      description: "Does something useful",
      inputSchema: {
        type: "object",
        properties: { input: { type: "string" } },
        required: ["input"],
      },
    },
  ],
}));

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  // Handle tool call
  return { content: [{ type: "text", text: "Result" }] };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 4. Token Cost Calculator

### Quick Estimates

| Scenario | Approx. Tokens | Cost (Sonnet) | Cost (Haiku) |
|----------|---------------|--------------|-------------|
| 1 page of text (~500 words) | ~670 input | $0.002 | $0.0005 |
| Short chat (10 turns) | ~3K in + 2K out | $0.039 | $0.010 |
| Long chat (50 turns) | ~25K in + 10K out | $0.225 | $0.056 |
| RAG query (5 chunks + answer) | ~4K in + 1K out | $0.027 | $0.007 |
| Agent task (10 tool calls) | ~15K in + 5K out | $0.120 | $0.030 |
| Code review (500 LOC) | ~5K in + 3K out | $0.060 | $0.015 |
| Full codebase analysis | ~100K in + 10K out | $0.450 | $0.112 |

### Monthly Cost Estimates (Per User)

| Usage Pattern | Sessions/Day | Tokens/Session | Monthly Cost (Sonnet) |
|--------------|-------------|---------------|----------------------|
| Light (L1) | 2 | 5K | ~$9 |
| Regular (L2) | 5 | 10K | ~$45 |
| Heavy (L2-L3) | 10 | 20K | ~$180 |
| Power (L3) | 20 | 30K | ~$540 |

### Cost Optimization Quick Wins

| Technique | Savings | Effort | Ch |
|-----------|--------|--------|-----|
| Use Haiku for simple tasks | 70-80% | Low | 49 |
| Prompt caching (repeated system prompts) | 50-90% | Low | 49 |
| Reduce output tokens (be concise) | 30-50% | Low | 49 |
| Model routing (right model for the task) | 40-60% | Medium | 49 |
| Compaction for long conversations | 30-50% | Medium | 12, 29 |
| Batch processing with concurrent limits | 20-30% | Medium | 58 |

---

## 5. Decision Trees

### RAG vs Fine-Tuning

```
Does the knowledge change frequently?
├── Yes --> RAG
│         (Embedding new docs is cheap; retraining is expensive)
└── No
    │
    Does the model need to learn a new STYLE?
    ├── Yes --> Fine-Tuning
    │         (Style is baked into weights, not retrieved)
    └── No
        │
        Is the knowledge domain-specific and large?
        ├── Yes --> RAG
        │         (Retrieval scales better than context stuffing)
        └── No
            │
            Do you need the knowledge available
            without external dependencies?
            ├── Yes --> Fine-Tuning
            │         (Self-contained, no vector DB needed)
            └── No --> RAG (default choice)
                      (Cheaper, faster to iterate, auditable)
```

### Framework vs From Scratch

```
How complex is your agent?
├── Single tool, single turn --> From scratch (Ch 8-9)
│   (Frameworks add overhead for simple cases)
│
├── Multi-tool, multi-turn --> Consider framework
│   │
│   Do you need memory, RAG, and complex routing?
│   ├── Yes --> Framework (LangChain, Mastra, LlamaIndex)
│   │         (Don't reinvent memory + retrieval + routing)
│   └── No --> From scratch with Anthropic/OpenAI SDK
│             (Less abstraction, more control)
│
└── Multi-agent orchestration --> From scratch (Ch 58)
    (Frameworks for multi-agent are immature;
     build your own coordinator)
```

### API vs Open-Source

```
Do you have strict data sovereignty requirements?
├── Yes --> Open-source (self-hosted)
│         (Data never leaves your infrastructure)
└── No
    │
    Do you need the best possible quality?
    ├── Yes --> API (frontier models are still better)
    └── No
        │
        Do you need to fine-tune?
        ├── Yes --> Open-source (full weight access)
        │         OR API fine-tuning (OpenAI, Anthropic)
        └── No
            │
            Is cost a primary concern at high volume?
            ├── Yes --> Open-source (no per-token cost)
            │         (But factor in infrastructure cost)
            └── No --> API (simplest path to production)
```

### Internal Tool: Build vs Buy

```
Do you have 20+ internal services to integrate?
├── Yes --> Build custom (Glass pattern, Ch 59)
│         (Off-the-shelf tools can't deeply integrate
│          with your internal systems)
└── No
    │
    Is your primary use case coding?
    ├── Yes --> Buy (Claude Code / Cursor)
    │         (Coding agents are mature products)
    └── No
        │
        Do you have 2+ engineers to maintain it?
        ├── Yes --> Build custom or hybrid
        │         (Custom: full control
        │          Hybrid: coding tool + custom for integrations)
        └── No --> Buy + configure
                  (Claude Code + MCP servers for integrations)
```

---

## 6. CLAUDE.md Template

```markdown
# CLAUDE.md

## Project Overview
[One paragraph: what this project is, what stack it uses]

## Architecture
[Key architectural decisions, folder structure, data flow]

## Conventions
- [Code style: linting rules, naming conventions]
- [Git: branch naming, commit format]
- [Testing: framework, coverage expectations]
- [API: error format, auth patterns]

## Key Commands
- `npm run dev` -- start development server
- `npm test` -- run tests
- `npm run lint` -- run linter
- `npm run build` -- production build

## Common Tasks
### Adding a new API endpoint
1. Create handler in `src/routes/`
2. Add schema in `src/schemas/`
3. Add tests in `tests/routes/`
4. Update OpenAPI spec

### Database migrations
[Your migration workflow]

## Things to Watch Out For
- [Known gotchas, edge cases]
- [Deprecated patterns to avoid]
- [Performance-sensitive areas]

## Dependencies
- [Key dependencies and why they were chosen]
- [Pinned versions that should not be upgraded without testing]
```

---

## 7. Quick Reference: Key Formulas

### Cosine Similarity
```
similarity(A, B) = (A . B) / (|A| * |B|)

Range: -1 to 1
> 0.8: Very similar
> 0.6: Similar
> 0.4: Somewhat related
< 0.4: Not related
```

### Token Estimation
```
English text:  1 token ~= 0.75 words  |  1 word ~= 1.3 tokens
Code:          1 token ~= 0.5 words   |  more punctuation = more tokens
JSON:          ~30% more tokens than equivalent plain text
```

### Cost Estimation
```
Cost = (input_tokens / 1,000,000 * input_price) + (output_tokens / 1,000,000 * output_price)

Example: 10K input + 2K output with Sonnet
Cost = (10,000 / 1,000,000 * $3) + (2,000 / 1,000,000 * $15) = $0.03 + $0.03 = $0.06
```

### Chunking Rule of Thumb
```
Chunk size:     500-1000 tokens (for most use cases)
Chunk overlap:  50-100 tokens (10-20% of chunk size)
Smaller chunks: Better precision, worse recall
Larger chunks:  Better recall, worse precision
```

### Context Window Budget
```
Total context = System prompt + Conversation history + Tool definitions + Response

Rule: Keep system prompt + tools under 20% of context window
      Leave 30%+ of context window for response
      Use the remaining 50% for conversation + retrieved context
```

---

*Previous: [Resources](./appendix-resources.md)* | *[Back to Guide](../README.md)*

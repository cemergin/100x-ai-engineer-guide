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

## 8. MCP Server Quick-Start Template

### Minimal MCP Server (TypeScript)

```typescript
// mcp-server.ts — A minimal MCP server that exposes a single tool
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// 1. List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "lookup_user",
      description: "Look up a user by email address. Returns name, role, and team.",
      inputSchema: {
        type: "object" as const,
        properties: {
          email: { type: "string", description: "The user's email address" },
        },
        required: ["email"],
      },
    },
  ],
}));

// 2. Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "lookup_user") {
    const email = args?.email as string;
    // Your business logic here
    const user = await findUserByEmail(email);
    return {
      content: [{ type: "text", text: JSON.stringify(user, null, 2) }],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

// 3. Connect via stdio
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Configure in Claude Code

```json
// .mcp.json (project root)
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["tsx", "./mcp-server.ts"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

---

## 9. Eval Framework Quick-Start

### Minimal Eval Setup (TypeScript)

```typescript
// eval.ts — Run evals against your AI system
import Anthropic from "@anthropic-ai/sdk";

interface EvalCase {
  input: string;
  expected: string;
  tags?: string[];
}

interface EvalResult {
  input: string;
  output: string;
  expected: string;
  score: number;
  pass: boolean;
}

// 1. Define test cases
const cases: EvalCase[] = [
  {
    input: "What is the refund policy?",
    expected: "30-day money-back guarantee",
    tags: ["refund", "policy"],
  },
  {
    input: "How do I cancel my subscription?",
    expected: "Go to Settings > Billing > Cancel",
    tags: ["cancel", "billing"],
  },
];

// 2. Define scorer (LLM-as-judge)
async function scoreWithLLM(
  client: Anthropic,
  input: string,
  output: string,
  expected: string
): Promise<number> {
  const response = await client.messages.create({
    model: "claude-haiku-3-5-20241022",
    max_tokens: 100,
    messages: [{
      role: "user",
      content: `Rate how well the Output answers the Question compared to the Expected answer.
Score 0.0 (completely wrong) to 1.0 (perfect).
Respond with ONLY a number.

Question: ${input}
Expected: ${expected}
Output: ${output}

Score:`,
    }],
  });
  const text = response.content[0].type === "text" ? response.content[0].text : "0";
  return parseFloat(text) || 0;
}

// 3. Run eval
async function runEval(systemPrompt: string): Promise<void> {
  const client = new Anthropic();
  const results: EvalResult[] = [];

  for (const c of cases) {
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 500,
      system: systemPrompt,
      messages: [{ role: "user", content: c.input }],
    });
    const output = response.content[0].type === "text" ? response.content[0].text : "";
    const score = await scoreWithLLM(client, c.input, output, c.expected);

    results.push({
      input: c.input,
      output,
      expected: c.expected,
      score,
      pass: score >= 0.7,
    });
  }

  // 4. Report
  const avgScore = results.reduce((s, r) => s + r.score, 0) / results.length;
  const passRate = results.filter((r) => r.pass).length / results.length;
  console.log(`Average score: ${avgScore.toFixed(2)}`);
  console.log(`Pass rate: ${(passRate * 100).toFixed(0)}%`);
  console.log(`Failures:`, results.filter((r) => !r.pass));
}

runEval("You are a helpful customer support agent for Acme Corp.");
```

---

## 10. Docker Compose Template for Agent Sandboxing

```yaml
# docker-compose.yml — Sandboxed agent environment (OpenClaw pattern)
version: "3.8"

services:
  agent:
    build:
      context: .
      dockerfile: Dockerfile.agent
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - NODE_ENV=production
    volumes:
      # Mount workspace as read-write, but nothing else
      - ./workspace:/app/workspace
    networks:
      - agent-net
    # Resource limits prevent runaway agents
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G
        reservations:
          cpus: "0.5"
          memory: 1G
    # Health check for monitoring
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Egress proxy — controls what the agent can reach
  egress-proxy:
    image: nginx:alpine
    volumes:
      - ./nginx-allowlist.conf:/etc/nginx/nginx.conf:ro
    networks:
      - agent-net
      - external-net

  # Vector database for agent memory
  vector-db:
    image: chromadb/chroma:latest
    volumes:
      - chroma-data:/chroma/chroma
    networks:
      - agent-net

networks:
  agent-net:
    internal: true  # No direct internet access
  external-net:
    # Only the egress proxy has external access

volumes:
  chroma-data:
```

```dockerfile
# Dockerfile.agent
FROM node:20-slim
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --production

# Copy agent code
COPY . .

# Run as non-root user
RUN groupadd -r agent && useradd -r -g agent agent
USER agent

CMD ["node", "dist/agent.js"]
```

---

## 11. GitHub Actions Template for AI CI/CD

```yaml
# .github/workflows/ai-ci.yml — Eval-gated deployments
name: AI CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

jobs:
  # Run evals on every PR
  eval:
    name: Run AI Evals
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Run eval suite
        run: npm run eval
        env:
          EVAL_MODEL: claude-sonnet-4-20250514

      - name: Check eval thresholds
        run: |
          SCORE=$(cat eval-results.json | jq '.avgScore')
          THRESHOLD=0.85
          if (( $(echo "$SCORE < $THRESHOLD" | bc -l) )); then
            echo "Eval score $SCORE is below threshold $THRESHOLD"
            exit 1
          fi
          echo "Eval score $SCORE passes threshold $THRESHOLD"

      - name: Upload eval results
        uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: eval-results.json

  # Deploy only if evals pass
  deploy:
    name: Deploy
    needs: [eval]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: "--prod"
```

---

## 12. Common Error Patterns and Fixes

### Rate Limits

```
Error: 429 Too Many Requests / Rate limit exceeded
```

**Fix:**
```typescript
// Exponential backoff with jitter
async function callWithRetry(fn: () => Promise<any>, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error: any) {
      if (error.status === 429 && i < maxRetries - 1) {
        const baseDelay = Math.pow(2, i) * 1000;
        const jitter = Math.random() * 1000;
        await new Promise((r) => setTimeout(r, baseDelay + jitter));
        continue;
      }
      throw error;
    }
  }
}
```

### Context Window Overflow

```
Error: 400 / prompt is too long: X tokens > Y maximum
```

**Fix:**
```typescript
// Count tokens before sending, truncate if needed
function truncateMessages(messages: Message[], maxTokens: number): Message[] {
  let totalTokens = estimateTokens(messages);
  while (totalTokens > maxTokens && messages.length > 2) {
    // Keep system prompt (first) and latest user message (last)
    // Remove oldest conversation messages
    messages.splice(1, 1);
    totalTokens = estimateTokens(messages);
  }
  return messages;
}

// Or use compaction: summarize old messages
async function compactMessages(messages: Message[]): Promise<Message[]> {
  const oldMessages = messages.slice(0, -4); // Keep last 4 turns
  const summary = await summarize(oldMessages);
  return [
    messages[0], // system prompt
    { role: "user", content: `Previous conversation summary: ${summary}` },
    { role: "assistant", content: "Understood. I have the context." },
    ...messages.slice(-4), // recent turns
  ];
}
```

### Hallucination Detection

```
Problem: Model generates plausible-sounding but incorrect information
```

**Fix:**
```typescript
// 1. Ground responses in retrieved context
const systemPrompt = `Answer using ONLY the provided context.
If the context doesn't contain the answer, say "I don't have enough information."
Always cite which context chunk supports your answer.`;

// 2. Add a hallucination check scorer to your evals
async function checkHallucination(output: string, context: string): Promise<number> {
  const response = await client.messages.create({
    model: "claude-haiku-3-5-20241022",
    max_tokens: 100,
    messages: [{
      role: "user",
      content: `Does the Output contain claims NOT supported by the Context?
Context: ${context}
Output: ${output}
Answer YES or NO, then explain briefly.`,
    }],
  });
  const text = response.content[0].type === "text" ? response.content[0].text : "";
  return text.toLowerCase().startsWith("no") ? 1.0 : 0.0;
}
```

### Tool Call Failures

```
Error: Tool call returned error / Tool not found / Invalid arguments
```

**Fix:**
```typescript
// 1. Validate tool arguments before execution
function executeTool(toolCall: ToolCall): ToolResult {
  const tool = tools.find((t) => t.name === toolCall.name);
  if (!tool) {
    return { error: `Unknown tool: ${toolCall.name}. Available: ${tools.map(t => t.name).join(", ")}` };
  }

  // Validate with Zod
  const parsed = tool.schema.safeParse(toolCall.arguments);
  if (!parsed.success) {
    return { error: `Invalid arguments: ${parsed.error.message}` };
  }

  try {
    return tool.execute(parsed.data);
  } catch (error) {
    // Return error to the LLM so it can self-correct
    return { error: `Tool execution failed: ${error.message}. Try different arguments.` };
  }
}

// 2. Improve tool descriptions (the #1 cause of wrong tool selection)
// BAD:  "search" — too vague
// GOOD: "Search the product database by name, category, or price range.
//        Returns up to 10 matching products with name, price, and description.
//        Use this when the user asks about products, pricing, or availability."
```

### Streaming Errors

```
Error: Stream terminated unexpectedly / Connection reset
```

**Fix:**
```typescript
// Wrap streaming with reconnection logic
async function* streamWithRetry(params: any, maxRetries = 2) {
  let attempt = 0;
  let partialResponse = "";

  while (attempt <= maxRetries) {
    try {
      const stream = client.messages.stream(params);
      for await (const event of stream) {
        if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
          partialResponse += event.delta.text;
          yield event.delta.text;
        }
      }
      return; // Success
    } catch (error) {
      attempt++;
      if (attempt > maxRetries) throw error;
      // On retry, include partial response context
      console.warn(`Stream retry ${attempt}/${maxRetries}`);
    }
  }
}
```

---

## 13. CLAUDE.md Best Practices

### Structure Principles

1. **Put the most important info first** — Claude reads top-down; front-load project context
2. **Be specific, not generic** — "Use `vitest` for tests" beats "Use the project's test framework"
3. **Include commands** — Exact shell commands Claude can run (build, test, lint, deploy)
4. **Document gotchas** — Known issues, deprecated patterns, things that look wrong but are intentional
5. **Keep it under 500 lines** — Too long and it dilutes the signal; too short and it lacks context

### What to Include

```
MUST HAVE:
- Project overview (1 paragraph)
- Tech stack and key dependencies
- Build/test/lint commands
- File structure conventions
- API patterns and error handling conventions

NICE TO HAVE:
- Common task walkthroughs (add endpoint, write migration, etc.)
- Performance-sensitive areas
- Security considerations
- Deployment process

AVOID:
- Duplicating README content
- Lengthy history or changelog
- Information that changes weekly (put in code comments instead)
- Generic advice ("write clean code")
```

### Nested CLAUDE.md Files

```
project/
├── CLAUDE.md              ← Project-wide context
├── src/
│   ├── api/
│   │   └── CLAUDE.md      ← API-specific conventions
│   ├── components/
│   │   └── CLAUDE.md      ← Component patterns
│   └── db/
│       └── CLAUDE.md      ← Database and migration patterns
└── tests/
    └── CLAUDE.md          ← Testing conventions
```

Claude Code automatically loads CLAUDE.md files from the current directory and parent directories, giving agents context-specific instructions as they navigate the codebase.

---

*Previous: [Resources](./appendix-resources.md)* | *[Back to Guide](../README.md)*

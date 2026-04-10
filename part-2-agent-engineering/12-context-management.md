<!--
  CHAPTER: 12
  TITLE: Context Window Management
  PART: II — Agent Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 0 (context windows, tokens), Ch 9 (agent loop), Ch 10 (memory vs context)
  KEY_TOPICS: token counting, context window budget, compaction, sliding window, sub-agent delegation, summarization
  DIFFICULTY: Intermediate → Advanced
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 12: Context Window Management

> **Part II — Agent Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 0, Ch 9, Ch 10 | Difficulty: Intermediate → Advanced

Here's a truth that bites every agent builder eventually: your agent works beautifully for three tool calls, then falls apart on the twentieth. The responses get vague, the tool calls get repetitive, and eventually you hit an error: context window exceeded. Or worse, the agent quietly degrades -- it "forgets" what it read earlier because the important information is buried under pages of tool results.

The context window is the agent's bottleneck. Every message, every tool call, every tool result, every system prompt instruction -- it all competes for the same finite space. Claude gives you 200K tokens. GPT-4 gives you 128K. That sounds like a lot until your agent reads five source files (50K tokens), makes ten tool calls (30K tokens of round-trip overhead), and you realize the system prompt and conversation history are fighting for the remaining space.

This chapter is about managing that budget. Token counting, compaction strategies, sliding windows, and the most powerful pattern of all: sub-agent delegation, where you spawn a focused child agent for a specific task and bring back only the result. These techniques are what let production agents like Claude Code run for hundreds of iterations without drowning in their own history.

### In This Chapter
- Token counting: how to count tokens programmatically
- The context window budget: how to allocate space
- Compaction: summarizing old messages to free space
- Sliding window: keeping the N most recent messages
- Sub-agent delegation: spawning child agents for focused tasks
- Building a compaction system that triggers automatically
- Monitoring and alerting on context usage

### Related Chapters
- **Ch 0 (How LLMs Work)** -- context windows, tokens, and why they cost money
- **Ch 9 (The Agent Loop)** -- the loop fills the context window with each iteration
- **Ch 10 (Agent Memory)** -- memory vs context: the fundamental trade-off
- **Ch 29 (Context Window Internals)** -- how Claude Code manages context at scale
- **Ch 49 (Cost Engineering)** -- context management as a cost optimization lever
- **Ch 50 (Advanced Context Strategies)** -- recursive compaction, tiered memory

---

## 1. Token Counting

### 1.1 Why Count Tokens?

You can't manage what you can't measure. Before you can implement any context management strategy, you need to know how many tokens your messages consume.

**Token counting answers:**
- How full is the context window right now?
- When should I trigger compaction?
- How much room is left for the LLM's response?
- How much will this tool result cost?

### 1.2 Counting Tokens with tiktoken

The `tiktoken` library (originally from OpenAI) provides tokenizers for various models. For Anthropic models, the token count is approximate but close enough for budgeting.

```typescript
import { encoding_for_model } from "tiktoken";

// Create an encoder (cl100k_base works for most modern models)
const encoder = encoding_for_model("gpt-4o");

function countTokens(text: string): number {
  const tokens = encoder.encode(text);
  return tokens.length;
}

// Usage
console.log(countTokens("Hello, world!")); // ~4 tokens
console.log(countTokens("The quick brown fox")); // ~4 tokens

// Count tokens in a full message array
function countMessageTokens(messages: Array<{ role: string; content: string }>): number {
  let total = 0;
  for (const msg of messages) {
    // Each message has overhead: role tokens, separator tokens
    total += 4; // Approximate per-message overhead
    total += countTokens(msg.content);
  }
  return total;
}
```

### 1.3 Counting Tokens in Anthropic Messages

Anthropic messages are more complex because content can be an array of blocks (text, tool_use, tool_result). Here's a practical counter:

```typescript
import Anthropic from "@anthropic-ai/sdk";

function countAnthropicTokens(messages: Anthropic.MessageParam[]): number {
  let total = 0;

  for (const msg of messages) {
    total += 4; // Per-message overhead

    if (typeof msg.content === "string") {
      total += countTokens(msg.content);
    } else if (Array.isArray(msg.content)) {
      for (const block of msg.content) {
        if ("text" in block && typeof block.text === "string") {
          total += countTokens(block.text);
        } else if ("content" in block && typeof block.content === "string") {
          // tool_result blocks
          total += countTokens(block.content);
        } else {
          // tool_use blocks, images, etc.
          total += countTokens(JSON.stringify(block));
        }
      }
    }
  }

  return total;
}

// Even simpler: just serialize and count
function quickTokenCount(messages: Anthropic.MessageParam[]): number {
  const serialized = JSON.stringify(messages);
  return Math.ceil(serialized.length / 4); // Rough: 4 chars per token
}
```

### 1.4 The 4-Characters Heuristic

For quick estimates when you don't want the overhead of a tokenizer library:

```typescript
function estimateTokens(text: string): number {
  // English text averages ~4 characters per token
  // Code averages ~3.5 characters per token (more symbols)
  // JSON averages ~3 characters per token (lots of punctuation)
  return Math.ceil(text.length / 4);
}
```

This is good enough for budgeting decisions. Don't use it for precise limit enforcement -- use the actual tokenizer for that.

### 1.5 Using the Anthropic API's Token Count

The Anthropic API returns token usage in every response:

```typescript
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 4096,
  messages: [{ role: "user", content: "Hello" }],
});

console.log(response.usage);
// {
//   input_tokens: 12,    // Tokens sent to the model
//   output_tokens: 48,   // Tokens generated by the model
// }
```

Track this across iterations to monitor context growth:

```typescript
interface TokenTracker {
  iterations: Array<{
    inputTokens: number;
    outputTokens: number;
    cumulativeInput: number;
  }>;
  totalInputTokens: number;
  totalOutputTokens: number;
}

function createTokenTracker(): TokenTracker {
  return {
    iterations: [],
    totalInputTokens: 0,
    totalOutputTokens: 0,
  };
}

function recordIteration(
  tracker: TokenTracker,
  usage: { input_tokens: number; output_tokens: number }
): void {
  tracker.totalInputTokens += usage.input_tokens;
  tracker.totalOutputTokens += usage.output_tokens;
  tracker.iterations.push({
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cumulativeInput: usage.input_tokens, // This is what the model saw this turn
  });
}
```

---

## 2. The Context Window Budget

### 2.1 How the Budget Works

The context window has a fixed size. Everything competes for space:

```
Context Window (e.g., 200K tokens)
├── System prompt:          ~500-2000 tokens
├── Tool definitions:       ~200-500 tokens per tool (10 tools = 2K-5K)
├── Memory (from Ch 10):    ~2K-10K tokens
├── Conversation history:   grows with each iteration
├── Current tool results:   varies wildly (file contents can be huge)
└── Reserved for output:    max_tokens (e.g., 4096)
```

### 2.2 Implementing a Budget Manager

```typescript
interface ContextBudget {
  windowSize: number;       // Total context window (e.g., 200000)
  maxOutputTokens: number;  // Reserved for generation (e.g., 4096)
  systemPrompt: number;     // Measured once
  toolDefinitions: number;  // Measured once
  memory: number;           // Measured each iteration

  // Calculated
  available: number;        // What's left for conversation + tool results
  used: number;             // Current usage
  remaining: number;        // available - used
  utilizationPct: number;   // used / available * 100
}

function calculateBudget(
  windowSize: number,
  maxOutputTokens: number,
  systemPromptTokens: number,
  toolDefinitionTokens: number,
  memoryTokens: number,
  conversationTokens: number
): ContextBudget {
  const fixed = systemPromptTokens + toolDefinitionTokens + memoryTokens + maxOutputTokens;
  const available = windowSize - fixed;
  const used = conversationTokens;
  const remaining = available - used;

  return {
    windowSize,
    maxOutputTokens,
    systemPrompt: systemPromptTokens,
    toolDefinitions: toolDefinitionTokens,
    memory: memoryTokens,
    available,
    used,
    remaining,
    utilizationPct: (used / available) * 100,
  };
}

// Usage in the agent loop:
function shouldCompact(budget: ContextBudget): boolean {
  // Trigger compaction at 70% utilization
  return budget.utilizationPct > 70;
}

function isNearLimit(budget: ContextBudget): boolean {
  // Emergency: less than 10% remaining
  return budget.utilizationPct > 90;
}
```

### 2.3 Visualizing the Budget

A helpful debug tool -- print the budget on each iteration:

```typescript
function printBudget(budget: ContextBudget, iteration: number): void {
  const bar = (pct: number): string => {
    const filled = Math.round(pct / 2);
    const empty = 50 - filled;
    return "[" + "#".repeat(filled) + ".".repeat(empty) + "]";
  };

  console.log(`\n--- Context Budget (iteration ${iteration}) ---`);
  console.log(`  Window:  ${budget.windowSize.toLocaleString()} tokens`);
  console.log(`  System:  ${budget.systemPrompt.toLocaleString()}`);
  console.log(`  Tools:   ${budget.toolDefinitions.toLocaleString()}`);
  console.log(`  Memory:  ${budget.memory.toLocaleString()}`);
  console.log(`  Output:  ${budget.maxOutputTokens.toLocaleString()} (reserved)`);
  console.log(`  Conv:    ${budget.used.toLocaleString()} / ${budget.available.toLocaleString()}`);
  console.log(`  ${bar(budget.utilizationPct)} ${budget.utilizationPct.toFixed(1)}%`);

  if (budget.utilizationPct > 90) {
    console.log(`  *** WARNING: Context nearly full ***`);
  } else if (budget.utilizationPct > 70) {
    console.log(`  * Compaction recommended *`);
  }
}
```

---

## 3. Compaction: Summarizing to Free Space

### 3.1 What Compaction Is

Compaction replaces older messages with a shorter summary. You lose detail but preserve the essential context. It's the most important context management strategy.

```
Before compaction (40K tokens):
  [user] Read all config files
  [assistant] [tool_use: readFile("config/db.json")]
  [user] [tool_result: 2000 lines of JSON]
  [assistant] [tool_use: readFile("config/app.json")]
  [user] [tool_result: 500 lines of JSON]
  [assistant] [tool_use: readFile("config/auth.json")]
  [user] [tool_result: 300 lines of JSON]
  [assistant] The configs show that...
  [user] Now update the database host
  ...20 more messages...

After compaction (5K tokens):
  [user] [Summary: User asked to read config files. I read db.json, app.json,
          and auth.json. Key findings: DB host is localhost:5432, app runs on
          port 3000, auth uses JWT with RS256. User then asked to update the
          database host. I modified db.json to point to prod-db.company.com:5432.]
  [assistant] I understand the context. Continuing from where we left off.
  ...recent messages preserved...
```

### 3.2 Building a Compaction Service

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface CompactionResult {
  summary: string;
  preservedMessages: Anthropic.MessageParam[];
  tokensBefore: number;
  tokensAfter: number;
  tokensSaved: number;
}

async function compactMessages(
  messages: Anthropic.MessageParam[],
  keepRecentCount: number = 6 // Keep the last N messages intact
): Promise<CompactionResult> {
  const tokensBefore = quickTokenCount(messages);

  if (messages.length <= keepRecentCount) {
    return {
      summary: "",
      preservedMessages: messages,
      tokensBefore,
      tokensAfter: tokensBefore,
      tokensSaved: 0,
    };
  }

  const olderMessages = messages.slice(0, -keepRecentCount);
  const recentMessages = messages.slice(-keepRecentCount);

  // Summarize the older messages
  const summaryResponse = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: `You are summarizing a conversation between a user and an AI assistant for context continuity. Create a concise summary that preserves:

1. KEY DECISIONS: What was decided and why
2. ACTIONS TAKEN: What files were read/written, what commands were run
3. CURRENT STATE: What has been accomplished, what remains
4. IMPORTANT DETAILS: Specific values, paths, configurations that might be needed

Be factual and specific. Include file paths, variable names, and concrete values. Do NOT include pleasantries or meta-commentary. Keep under 1000 words.`,
    messages: [
      {
        role: "user",
        content: `Summarize this conversation for context continuity:\n\n${serializeMessages(olderMessages)}`,
      },
    ],
  });

  const summary = summaryResponse.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("\n");

  // Build the compacted message array
  const compactedMessages: Anthropic.MessageParam[] = [
    {
      role: "user",
      content: `[CONVERSATION SUMMARY - Earlier messages have been compacted to save context space]\n\n${summary}`,
    },
    {
      role: "assistant",
      content:
        "I understand the context from the summary. I'll continue from where we left off, keeping in mind the decisions made and actions taken.",
    },
    ...recentMessages,
  ];

  const tokensAfter = quickTokenCount(compactedMessages);

  return {
    summary,
    preservedMessages: compactedMessages,
    tokensBefore,
    tokensAfter,
    tokensSaved: tokensBefore - tokensAfter,
  };
}

function serializeMessages(messages: Anthropic.MessageParam[]): string {
  return messages
    .map((msg) => {
      const role = msg.role.toUpperCase();
      if (typeof msg.content === "string") {
        return `[${role}]: ${msg.content}`;
      }
      // For complex content blocks, create a readable summary
      const blocks = msg.content as Array<Record<string, unknown>>;
      const parts = blocks.map((block) => {
        if (block.type === "text") return block.text;
        if (block.type === "tool_use") return `[Tool call: ${block.name}(${JSON.stringify(block.input).slice(0, 200)})]`;
        if (block.type === "tool_result") {
          const content = typeof block.content === "string" ? block.content : JSON.stringify(block.content);
          return `[Tool result: ${(content as string).slice(0, 500)}${(content as string).length > 500 ? "..." : ""}]`;
        }
        return JSON.stringify(block).slice(0, 200);
      });
      return `[${role}]: ${parts.join(" ")}`;
    })
    .join("\n\n");
}

function quickTokenCount(messages: Anthropic.MessageParam[]): number {
  return Math.ceil(JSON.stringify(messages).length / 4);
}
```

### 3.3 Auto-Compaction in the Agent Loop

Integrate compaction so it triggers automatically when the context gets full:

```typescript
async function agentLoopWithCompaction(
  userMessage: string,
  options: {
    systemPrompt: string;
    tools: Anthropic.Tool[];
    maxIterations: number;
    contextWindowSize: number; // e.g., 200000
    compactionThreshold: number; // e.g., 0.7 (70% full)
  }
): Promise<string> {
  let messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  const systemTokens = estimateTokens(options.systemPrompt);
  const toolTokens = estimateTokens(JSON.stringify(options.tools));
  const maxOutput = 4096;
  const fixedOverhead = systemTokens + toolTokens + maxOutput;
  const availableForConversation = options.contextWindowSize - fixedOverhead;
  const compactionTrigger = availableForConversation * options.compactionThreshold;

  for (let i = 0; i < options.maxIterations; i++) {
    // Check if compaction is needed BEFORE calling the LLM
    const currentTokens = quickTokenCount(messages);
    if (currentTokens > compactionTrigger) {
      console.log(
        `\n  [Compacting: ${currentTokens} tokens > ${compactionTrigger} threshold]`
      );

      const result = await compactMessages(messages, 6);
      messages = result.preservedMessages;

      console.log(
        `  [Compacted: ${result.tokensBefore} → ${result.tokensAfter} tokens (saved ${result.tokensSaved})]`
      );
    }

    // Normal agent loop continues
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: maxOutput,
      system: options.systemPrompt,
      tools: options.tools,
      messages,
    });

    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      return response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");
    }

    messages.push({ role: "assistant", content: response.content });

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const toolUse of toolUseBlocks) {
      const result = await executeTool(
        toolUse.name,
        toolUse.input as Record<string, unknown>
      );

      // Truncate large tool results
      const maxResultTokens = 5000; // ~20KB
      const resultTokens = estimateTokens(result);
      const finalResult =
        resultTokens > maxResultTokens
          ? result.slice(0, maxResultTokens * 4) +
            `\n\n[Truncated: result was ${resultTokens} tokens. Use more specific queries.]`
          : result;

      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: finalResult,
      });
    }

    messages.push({ role: "user", content: toolResults });
  }

  return "[Max iterations reached]";
}
```

---

## 4. Sliding Window: Keeping Only Recent Messages

### 4.1 The Simple Approach

Instead of summarizing, just drop old messages. This is simpler but loses more information:

```typescript
function slidingWindow(
  messages: Anthropic.MessageParam[],
  maxMessages: number
): Anthropic.MessageParam[] {
  if (messages.length <= maxMessages) return messages;

  // Always keep the first message (original user request)
  const first = messages[0];
  const recent = messages.slice(-maxMessages + 1);

  return [first, ...recent];
}
```

### 4.2 Smart Sliding Window

A better version that keeps important messages (like the original request and key decisions) while dropping less important ones (like intermediate tool results):

```typescript
interface TaggedMessage extends Anthropic.MessageParam {
  importance?: "high" | "medium" | "low";
  timestamp?: number;
}

function smartSlidingWindow(
  messages: TaggedMessage[],
  maxTokens: number
): Anthropic.MessageParam[] {
  // Priority queue: high importance first, then recency
  const sorted = [...messages].sort((a, b) => {
    const importanceOrder = { high: 3, medium: 2, low: 1 };
    const aImportance = importanceOrder[a.importance || "medium"];
    const bImportance = importanceOrder[b.importance || "medium"];

    if (aImportance !== bImportance) return bImportance - aImportance;
    return (b.timestamp || 0) - (a.timestamp || 0); // More recent first
  });

  // Add messages until we hit the token budget
  const kept: TaggedMessage[] = [];
  let tokenCount = 0;

  for (const msg of sorted) {
    const msgTokens = quickTokenCount([msg]);
    if (tokenCount + msgTokens > maxTokens) break;
    kept.push(msg);
    tokenCount += msgTokens;
  }

  // Re-sort by original order (messages must be in chronological order)
  const originalIndices = new Map(messages.map((m, i) => [m, i]));
  kept.sort((a, b) => (originalIndices.get(a) || 0) - (originalIndices.get(b) || 0));

  return kept;
}
```

### 4.3 Compaction vs Sliding Window

| Strategy | Pros | Cons |
|----------|------|------|
| **Compaction** | Preserves semantic content; LLM understands what happened | Costs an extra LLM call; summary might miss details |
| **Sliding Window** | Zero cost; instant; simple | Loses all context from dropped messages |
| **Smart Sliding Window** | Preserves important messages | Requires importance tagging; gaps in conversation |
| **Compaction + Sliding** | Best of both: summarize old, keep recent | Most complex; two strategies to maintain |

**Recommendation:** Use compaction for production agents. Use sliding window for prototypes or when cost is a primary concern.

---

## 5. Sub-Agent Delegation

### 5.1 The Most Powerful Pattern

When the agent needs to do a focused subtask (like reading and analyzing a large file), spawn a separate agent with its own context window. This child agent has the full context budget for its specific task, and reports back a concise result.

```
Main Agent:  "Read and analyze all files in src/. Tell me the architecture."
                ↓
            Spawns sub-agent with task: "Analyze src/ directory architecture"
                ↓
Sub-Agent:   Reads 20 files, explores imports, maps dependencies
             (Uses its own 200K context window for this task)
                ↓
Sub-Agent:   Returns: "3-layer architecture: routes -> services -> db.
             Main entry: index.ts. 12 route files, 8 service files, 
             4 database modules. Uses Drizzle ORM with PostgreSQL."
                ↓
Main Agent:  Receives ~200 tokens instead of 50K tokens of file contents
             Continues with its task using this concise summary
```

### 5.2 Building Sub-Agent Delegation

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface SubAgentTask {
  task: string;          // What the sub-agent should do
  tools: Anthropic.Tool[]; // Tools available to the sub-agent
  maxIterations: number;   // Iteration limit for the sub-agent
  maxOutputTokens: number; // How concise the response should be
}

interface SubAgentResult {
  summary: string;       // The sub-agent's findings
  toolCallsMade: number; // How many tool calls the sub-agent made
  iterationsUsed: number;
  tokensUsed: number;
}

async function runSubAgent(task: SubAgentTask): Promise<SubAgentResult> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: task.task },
  ];

  let toolCallsMade = 0;
  let totalTokens = 0;
  let iterations = 0;

  for (let i = 0; i < task.maxIterations; i++) {
    iterations++;

    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: task.maxOutputTokens,
      system: `You are a focused sub-agent. Complete the assigned task using the available tools, then provide a concise summary of your findings. Be specific and factual. Include key details like file paths, configurations, and numbers.`,
      tools: task.tools,
      messages,
    });

    totalTokens += response.usage.input_tokens + response.usage.output_tokens;

    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      const summary = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");

      return {
        summary,
        toolCallsMade,
        iterationsUsed: iterations,
        tokensUsed: totalTokens,
      };
    }

    messages.push({ role: "assistant", content: response.content });

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const toolUse of toolUseBlocks) {
      toolCallsMade++;
      const result = await executeTool(
        toolUse.name,
        toolUse.input as Record<string, unknown>
      );
      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "user", content: toolResults });
  }

  return {
    summary: "[Sub-agent reached max iterations without completing]",
    toolCallsMade,
    iterationsUsed: iterations,
    tokensUsed: totalTokens,
  };
}
```

### 5.3 Using Sub-Agents as a Tool

The cleanest integration: make sub-agent delegation a tool that the main agent can call:

```typescript
const delegateTaskTool: Anthropic.Tool = {
  name: "delegateTask",
  description: `Delegate a focused subtask to a sub-agent. The sub-agent gets its own context window and tools. Use this for:
- Reading and analyzing large files or directories
- Research tasks that require reading many files
- Any task that would consume too much of your context window
The sub-agent returns a concise summary of its findings.`,
  input_schema: {
    type: "object",
    properties: {
      task: {
        type: "string",
        description:
          "Clear description of what the sub-agent should do. Be specific about what information you need back.",
      },
      scope: {
        type: "string",
        enum: ["small", "medium", "large"],
        description:
          "Expected scope. small=1-3 tool calls, medium=3-10, large=10+",
      },
    },
    required: ["task"],
  },
};

async function executeDelegateTask(
  input: Record<string, unknown>
): Promise<string> {
  const scope = input.scope as string || "medium";
  const maxIterations = { small: 5, medium: 10, large: 20 }[scope] || 10;

  console.log(`\n  [Delegating to sub-agent: "${(input.task as string).slice(0, 60)}..."]`);

  const result = await runSubAgent({
    task: input.task as string,
    tools: fileSystemTools, // Give the sub-agent filesystem access
    maxIterations,
    maxOutputTokens: 2048, // Keep the summary concise
  });

  console.log(
    `  [Sub-agent completed: ${result.iterationsUsed} iterations, ${result.toolCallsMade} tool calls, ${result.tokensUsed} tokens]`
  );

  return JSON.stringify({
    summary: result.summary,
    metadata: {
      toolCallsMade: result.toolCallsMade,
      iterationsUsed: result.iterationsUsed,
    },
  });
}
```

### 5.4 When to Use Sub-Agents

**Use sub-agents when:**
- The subtask involves reading many files (code analysis, dependency mapping)
- The subtask generates lots of intermediate data you don't need
- You want to keep the main agent's context focused on the high-level task
- The subtask is independent and well-defined

**Don't use sub-agents when:**
- The subtask depends heavily on the main conversation context
- The subtask is simple (1-2 tool calls)
- You need the full details, not a summary
- Latency matters more than context efficiency

---

## 6. Tool Result Management

### 6.1 Truncating Large Results

The biggest context hog is tool results. A single `readFile` on a large file can consume 10K+ tokens. Always truncate:

```typescript
interface TruncationConfig {
  maxTokens: number;     // Maximum tokens for this result
  strategy: "head" | "tail" | "smart"; // Where to cut
}

function truncateToolResult(
  result: string,
  config: TruncationConfig
): string {
  const currentTokens = estimateTokens(result);

  if (currentTokens <= config.maxTokens) return result;

  const maxChars = config.maxTokens * 4; // Rough conversion

  switch (config.strategy) {
    case "head":
      return (
        result.slice(0, maxChars) +
        `\n\n[TRUNCATED: Showing first ${config.maxTokens} tokens of ${currentTokens} total. Use more specific queries or line ranges to see the rest.]`
      );

    case "tail":
      return (
        `[TRUNCATED: Showing last ${config.maxTokens} tokens of ${currentTokens} total.]\n\n` +
        result.slice(-maxChars)
      );

    case "smart": {
      // Keep the beginning and end, truncate the middle
      const halfChars = Math.floor(maxChars / 2);
      const head = result.slice(0, halfChars);
      const tail = result.slice(-halfChars);
      return (
        head +
        `\n\n[... ${currentTokens - config.maxTokens} tokens omitted ...]\n\n` +
        tail
      );
    }
  }
}

// Default truncation configs per tool
const toolTruncationDefaults: Record<string, TruncationConfig> = {
  readFile: { maxTokens: 3000, strategy: "smart" },
  listDirectory: { maxTokens: 2000, strategy: "head" },
  webSearch: { maxTokens: 2000, strategy: "head" },
  runCommand: { maxTokens: 1500, strategy: "tail" }, // Errors are usually at the end
};
```

### 6.2 Result Caching

If the agent reads the same file twice, don't use context space for both:

```typescript
class ToolResultCache {
  private cache: Map<string, { result: string; timestamp: number }> = new Map();
  private ttl: number; // Cache TTL in milliseconds

  constructor(ttlMs: number = 60000) {
    this.ttl = ttlMs;
  }

  getCacheKey(toolName: string, args: Record<string, unknown>): string {
    return `${toolName}:${JSON.stringify(args)}`;
  }

  get(toolName: string, args: Record<string, unknown>): string | null {
    const key = this.getCacheKey(toolName, args);
    const entry = this.cache.get(key);

    if (!entry) return null;
    if (Date.now() - entry.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }

    return entry.result;
  }

  set(
    toolName: string,
    args: Record<string, unknown>,
    result: string
  ): void {
    // Only cache read-only tools
    const cacheable = ["readFile", "listDirectory", "webSearch"];
    if (!cacheable.includes(toolName)) return;

    const key = this.getCacheKey(toolName, args);
    this.cache.set(key, { result, timestamp: Date.now() });
  }
}

// In the agent loop:
const cache = new ToolResultCache(60000); // 1 minute TTL

async function executeToolWithCache(
  name: string,
  args: Record<string, unknown>
): Promise<string> {
  const cached = cache.get(name, args);
  if (cached) {
    return `[Cached result]\n${cached}`;
  }

  const result = await executeTool(name, args);
  cache.set(name, args, result);
  return result;
}
```

---

## 7. Putting It All Together

### 7.1 A Production-Ready Context-Managed Agent Loop

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface ManagedAgentConfig {
  systemPrompt: string;
  tools: Anthropic.Tool[];
  maxIterations: number;
  contextWindowSize: number;
  compactionThreshold: number; // 0-1, e.g., 0.7
  maxToolResultTokens: number;
}

async function managedAgentLoop(
  userMessage: string,
  config: ManagedAgentConfig
): Promise<{ response: string; stats: AgentStats }> {
  let messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  const stats: AgentStats = {
    iterations: 0,
    toolCalls: 0,
    compactions: 0,
    totalInputTokens: 0,
    totalOutputTokens: 0,
    subAgentsSpawned: 0,
  };

  const cache = new ToolResultCache();
  const maxOutput = 4096;

  // Calculate fixed overhead
  const fixedTokens =
    estimateTokens(config.systemPrompt) +
    estimateTokens(JSON.stringify(config.tools)) +
    maxOutput;

  const availableTokens = config.contextWindowSize - fixedTokens;
  const compactionTrigger = availableTokens * config.compactionThreshold;

  for (let i = 0; i < config.maxIterations; i++) {
    stats.iterations++;

    // --- Context Management: Check and compact ---
    const currentTokens = quickTokenCount(messages);

    if (currentTokens > compactionTrigger) {
      console.log(`  [Context: ${currentTokens}/${availableTokens} tokens - compacting]`);

      const compacted = await compactMessages(messages, 6);
      messages = compacted.preservedMessages;
      stats.compactions++;

      console.log(`  [Saved ${compacted.tokensSaved} tokens]`);
    }

    // --- Call LLM ---
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: maxOutput,
      system: config.systemPrompt,
      tools: config.tools,
      messages,
    });

    stats.totalInputTokens += response.usage.input_tokens;
    stats.totalOutputTokens += response.usage.output_tokens;

    // --- Check for completion ---
    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");

      return { response: text, stats };
    }

    // --- Execute tools with context management ---
    messages.push({ role: "assistant", content: response.content });
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const toolUse of toolUseBlocks) {
      stats.toolCalls++;
      const args = toolUse.input as Record<string, unknown>;

      let result: string;

      // Check for sub-agent delegation
      if (toolUse.name === "delegateTask") {
        result = await executeDelegateTask(args);
        stats.subAgentsSpawned++;
      } else {
        result = await executeToolWithCache(toolUse.name, args);
      }

      // Truncate large results
      const truncConfig = toolTruncationDefaults[toolUse.name] || {
        maxTokens: config.maxToolResultTokens,
        strategy: "smart" as const,
      };
      result = truncateToolResult(result, truncConfig);

      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "user", content: toolResults });
  }

  return {
    response: "[Max iterations reached]",
    stats,
  };
}

interface AgentStats {
  iterations: number;
  toolCalls: number;
  compactions: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  subAgentsSpawned: number;
}

function printStats(stats: AgentStats): void {
  console.log("\n--- Agent Stats ---");
  console.log(`  Iterations:     ${stats.iterations}`);
  console.log(`  Tool calls:     ${stats.toolCalls}`);
  console.log(`  Compactions:    ${stats.compactions}`);
  console.log(`  Sub-agents:     ${stats.subAgentsSpawned}`);
  console.log(`  Input tokens:   ${stats.totalInputTokens.toLocaleString()}`);
  console.log(`  Output tokens:  ${stats.totalOutputTokens.toLocaleString()}`);
  console.log(
    `  Total tokens:   ${(stats.totalInputTokens + stats.totalOutputTokens).toLocaleString()}`
  );
}
```

---

## 8. Cost Implications

### 8.1 Context Management IS Cost Management

Every time the agent loops, it sends the *entire conversation* back to the LLM. Without context management:

```
Iteration 1:  1,000 input tokens  × $0.003/1K = $0.003
Iteration 2:  3,000 input tokens  × $0.003/1K = $0.009
Iteration 3:  8,000 input tokens  × $0.003/1K = $0.024
Iteration 4:  15,000 input tokens × $0.003/1K = $0.045
...
Iteration 20: 80,000 input tokens × $0.003/1K = $0.240

Total for 20 iterations without management: ~$1.50
```

With compaction at 70% threshold:

```
Iterations 1-8:  Normal growth
Compaction:      80,000 → 8,000 tokens
Iterations 9-16: Resume from 8,000
Compaction:      60,000 → 8,000 tokens
Iterations 17-20: Final stretch

Total for 20 iterations with management: ~$0.60 (60% savings)
```

The savings compound because you're not paying to re-send old, already-processed information.

### 8.2 The Compaction Cost Trade-off

Compaction itself costs tokens (an LLM call to summarize). But it's a one-time cost that saves on every subsequent iteration:

```
Cost of compaction: ~2K input + ~1K output = ~$0.009
Savings per iteration after compaction: ~$0.03-0.10
Break-even: 1 iteration after compaction
```

Compaction almost always pays for itself immediately.

---

## 9. What's Next

You've now built an agent that can call tools (Ch 8), loop autonomously (Ch 9), remember what it's learned (Ch 10), ask for permission (Ch 11), and manage its own context window (Ch 12). You've built an agent from scratch.

Chapter 13 steps back and looks at the landscape. Now that you understand what an agent is made of, let's see how frameworks like LangChain, Mastra, Vercel AI SDK, and the Claude Agent SDK package these same concepts. You'll see that everything in Chapters 8-12 maps directly to framework abstractions -- and you'll be able to make an informed decision about when to use a framework vs when to build from scratch.

---

## Key Takeaways

1. **The context window is a fixed budget.** System prompt, tools, memory, conversation, tool results, and output all compete for the same space.
2. **Count tokens** to know when you're running out. Use the API's `usage` field or a tokenizer library.
3. **Compaction** (summarizing old messages) is the most important strategy. Trigger it at 70% context utilization.
4. **Truncate tool results** before they enter the context. A 10K-line file doesn't belong in the conversation.
5. **Sub-agent delegation** is the most powerful pattern: spawn a child agent, get back a concise summary, save thousands of tokens in the parent's context.
6. **Context management IS cost management.** You pay per token per iteration. Keeping the context lean saves real money.
7. **Compaction pays for itself** after one iteration. The cost of summarization is dwarfed by the savings on subsequent LLM calls.

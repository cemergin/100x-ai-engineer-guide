<!--
  CHAPTER: 10
  TITLE: Agent Memory & State
  PART: 2 — Agent Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 6 (conversation history), Ch 9 (the agent loop)
  KEY_TOPICS: short-term memory, working memory, long-term memory, scratchpad patterns, memory retrieval, memory-context trade-off
  DIFFICULTY: Intermediate → Advanced
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 10: Agent Memory & State

> **Part 2 — Agent Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 6, Ch 9 | Difficulty: Intermediate → Advanced | Language: TypeScript

In Chapter 9, you built an agent loop. The messages array is the agent's brain -- it remembers everything from the current conversation. But close the program and it's gone. Start a new conversation and the agent has no idea what happened five minutes ago. It can't learn from past mistakes. It can't remember your preferences. It can't pick up where it left off.

That's not how useful agents work. Claude Code remembers your project conventions in CLAUDE.md. ChatGPT remembers your name and preferences across sessions. Coding agents remember which approaches failed so they don't retry them. Memory is what turns a stateless tool into an assistant that gets better over time.

Memory in agent systems comes in three layers, and they mirror how human memory works. **Short-term memory** is the current conversation -- what we've been talking about for the last few minutes. **Working memory** is the agent's scratchpad -- notes it keeps updated as the conversation evolves. **Long-term memory** is persistent storage -- facts, preferences, and learnings that survive across sessions. Understanding these three layers and when to use each is what separates a toy agent from a production one.

### In This Chapter
- Short-term memory: the messages array as working context
- Working memory: scratchpad patterns and dynamic system prompts
- Long-term memory: persistent storage across sessions
- Memory retrieval: when and how to look up past interactions
- The memory-context trade-off: more memory = more tokens = more cost
- Building a memory system for the CLI agent from Chapter 9

### Related Chapters
- **Ch 6 (Chat Interface)** — conversation history is the foundation of short-term memory
- **Ch 9 (The Agent Loop)** — the loop's messages array IS short-term memory
- **Ch 15 (RAG Pipeline)** — RAG is external memory for knowledge retrieval
- **Ch 28 (Memory Systems: KAIROS & Beyond)** — Claude Code's 3-layer memory system in detail
- **Ch 50 (Advanced Context Strategies)** — advanced memory and context strategies at scale

---

## 1. The Three Layers of Agent Memory

### 1.1 Mental Model

Think about how you work:

- **Short-term memory**: You're reading this paragraph. You remember the previous paragraph. You're holding the current chapter's context in your head.
- **Working memory**: You have a mental model of the project you're working on -- the architecture, the current task, the problems you've identified. This updates as you work.
- **Long-term memory**: You know TypeScript syntax, your company's coding conventions, what happened in last week's sprint retro.

Agent memory maps to the same three layers:

```
Layer           | Scope              | Stored Where           | Lifetime
----------------|-------------------|------------------------|------------------
Short-term      | Current turn      | messages[]             | Current conversation
Working memory  | Current task      | System prompt / state  | Current session
Long-term       | All time          | Files / DB / vector DB | Persistent
```

### 1.2 Why All Three Matter

You could try to shove everything into one layer. Here's why that doesn't work:

- **Short-term only**: Agent forgets everything between sessions. Can't learn. Can't adapt.
- **Everything in the messages array**: Context window fills up. You hit token limits. Costs explode. (Chapter 12 dives deep into this.)
- **Everything in long-term storage**: Agent has to search for basic context every turn. Slow and expensive.

The art of agent memory is putting the right information in the right layer at the right time.

---

## 2. Short-Term Memory: The Messages Array

### 2.1 What It Is

Short-term memory is the conversation history -- the `messages[]` array from Chapter 9. Every user message, assistant response, tool call, and tool result lives here. The LLM sees all of it on every turn.

```typescript
// The messages array IS short-term memory
const messages: Message[] = [
  { role: "system", content: "You are a helpful assistant." },
  { role: "user", content: "Read the config file" },
  { role: "assistant", content: [/* tool_use: readFile */] },
  { role: "user", content: [/* tool_result: file contents */] },
  { role: "assistant", content: "The config file shows..." },
  { role: "user", content: "Now update the port to 8080" },
  // The LLM sees ALL of this on the next call
];
```

### 2.2 Strengths and Limits

**Strengths:**
- Perfect recall of everything in the current conversation
- No retrieval needed -- it's all right there in context
- The LLM can reference any prior message
- No infrastructure required

**Limits:**
- Bounded by the context window (200K tokens for Claude, 128K for GPT-4)
- Every token costs money on every LLM call
- Lost when the conversation ends
- Gets noisy as conversations grow long

### 2.3 Practical Pattern: Message Summarization

When the conversation gets long, you can summarize older messages to preserve the important bits while freeing tokens:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface ConversationManager {
  messages: Anthropic.MessageParam[];
  addUserMessage(content: string): void;
  addAssistantMessage(content: Anthropic.ContentBlock[]): void;
  addToolResult(toolResults: Anthropic.ToolResultBlockParam[]): void;
  getMessages(): Anthropic.MessageParam[];
  summarizeOlderMessages(keepRecent: number): Promise<void>;
}

function createConversationManager(): ConversationManager {
  let messages: Anthropic.MessageParam[] = [];
  let summary: string | null = null;

  return {
    get messages() {
      return messages;
    },

    addUserMessage(content: string) {
      messages.push({ role: "user", content });
    },

    addAssistantMessage(content: Anthropic.ContentBlock[]) {
      messages.push({ role: "assistant", content });
    },

    addToolResult(toolResults: Anthropic.ToolResultBlockParam[]) {
      messages.push({ role: "user", content: toolResults });
    },

    getMessages(): Anthropic.MessageParam[] {
      if (summary) {
        // Inject summary as the first message, then recent messages
        return [
          {
            role: "user",
            content: `[Previous conversation summary: ${summary}]`,
          },
          { role: "assistant", content: "I understand the context. Continuing..." },
          ...messages,
        ];
      }
      return messages;
    },

    async summarizeOlderMessages(keepRecent: number) {
      if (messages.length <= keepRecent) return;

      const olderMessages = messages.slice(0, -keepRecent);
      const recentMessages = messages.slice(-keepRecent);

      // Use the LLM to summarize the older conversation
      const summarization = await client.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1024,
        system:
          "Summarize the following conversation concisely. Focus on: key decisions made, files modified, current task state, and any important context. Keep it under 500 words.",
        messages: [
          {
            role: "user",
            content: `Summarize this conversation:\n${JSON.stringify(olderMessages)}`,
          },
        ],
      });

      const summaryText = summarization.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("\n");

      // Combine with existing summary if there is one
      summary = summary
        ? `${summary}\n\nMore recent context: ${summaryText}`
        : summaryText;

      // Keep only recent messages
      messages = recentMessages;
    },
  };
}
```

This is a preview of Chapter 12's compaction strategy. The key idea: compress old context rather than losing it entirely.

---

## 3. Working Memory: The Scratchpad

### 3.1 What It Is

Working memory is a scratchpad that the agent maintains *during* a session. Unlike the messages array (which records the full conversation), working memory tracks the agent's *current understanding* -- what it knows, what it's working on, what it's figured out.

The most common implementation: a dynamic system prompt that gets updated as the agent works.

### 3.2 The Updatable System Prompt Pattern

```typescript
interface WorkingMemory {
  currentTask: string;
  discoveries: string[];
  plan: string[];
  completedSteps: string[];
  openQuestions: string[];
}

function buildSystemPrompt(base: string, memory: WorkingMemory): string {
  let prompt = base;

  prompt += `\n\n## Current Working Memory\n`;

  if (memory.currentTask) {
    prompt += `\n### Current Task\n${memory.currentTask}\n`;
  }

  if (memory.discoveries.length > 0) {
    prompt += `\n### What I've Discovered\n`;
    memory.discoveries.forEach((d) => (prompt += `- ${d}\n`));
  }

  if (memory.plan.length > 0) {
    prompt += `\n### My Plan\n`;
    memory.plan.forEach((step, i) => {
      const status = memory.completedSteps.includes(step) ? "[done]" : "[todo]";
      prompt += `${i + 1}. ${status} ${step}\n`;
    });
  }

  if (memory.openQuestions.length > 0) {
    prompt += `\n### Open Questions\n`;
    memory.openQuestions.forEach((q) => (prompt += `- ${q}\n`));
  }

  return prompt;
}

// Usage in the agent loop:
const basePrompt = "You are a coding assistant. Use tools to help the user.";

let workingMemory: WorkingMemory = {
  currentTask: "",
  discoveries: [],
  plan: [],
  completedSteps: [],
  openQuestions: [],
};

// On each iteration, rebuild the system prompt with current working memory
const systemPrompt = buildSystemPrompt(basePrompt, workingMemory);
```

### 3.3 Agent-Managed Working Memory

The powerful version: let the agent update its own working memory. Add a tool for it:

```typescript
import { z } from "zod";

// The working memory state
let workingMemory: WorkingMemory = {
  currentTask: "",
  discoveries: [],
  plan: [],
  completedSteps: [],
  openQuestions: [],
};

const updateMemoryTool: Anthropic.Tool = {
  name: "updateWorkingMemory",
  description:
    "Update your working memory / scratchpad. Use this to track your current understanding, plan, discoveries, and open questions. This helps you stay organized across multiple tool calls.",
  input_schema: {
    type: "object",
    properties: {
      currentTask: {
        type: "string",
        description: "What you are currently working on",
      },
      addDiscovery: {
        type: "string",
        description: "A new fact or finding to remember",
      },
      setPlan: {
        type: "array",
        items: { type: "string" },
        description: "Set the full plan (list of steps)",
      },
      completeStep: {
        type: "string",
        description: "Mark a plan step as completed",
      },
      addQuestion: {
        type: "string",
        description: "An open question to investigate",
      },
      resolveQuestion: {
        type: "string",
        description: "Remove a question that has been answered",
      },
    },
    required: [],
  },
};

function executeUpdateMemory(input: Record<string, unknown>): string {
  if (input.currentTask) {
    workingMemory.currentTask = input.currentTask as string;
  }
  if (input.addDiscovery) {
    workingMemory.discoveries.push(input.addDiscovery as string);
  }
  if (input.setPlan) {
    workingMemory.plan = input.setPlan as string[];
    workingMemory.completedSteps = [];
  }
  if (input.completeStep) {
    workingMemory.completedSteps.push(input.completeStep as string);
  }
  if (input.addQuestion) {
    workingMemory.openQuestions.push(input.addQuestion as string);
  }
  if (input.resolveQuestion) {
    workingMemory.openQuestions = workingMemory.openQuestions.filter(
      (q) => q !== input.resolveQuestion
    );
  }

  return JSON.stringify({
    success: true,
    currentMemory: workingMemory,
  });
}
```

Now the agent loop includes working memory in the system prompt:

```typescript
async function agentLoopWithMemory(
  userMessage: string,
  conversationHistory: Anthropic.MessageParam[]
): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    ...conversationHistory,
    { role: "user", content: userMessage },
  ];

  for (let i = 0; i < 20; i++) {
    // Rebuild system prompt with current working memory on EVERY iteration
    const systemPrompt = buildSystemPrompt(
      "You are a helpful coding assistant. Update your working memory as you learn things.",
      workingMemory
    );

    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: systemPrompt,
      tools: [...otherTools, updateMemoryTool],
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
      let result: string;
      if (toolUse.name === "updateWorkingMemory") {
        result = executeUpdateMemory(toolUse.input as Record<string, unknown>);
      } else {
        result = await executeTool(toolUse.name, toolUse.input as Record<string, unknown>);
      }
      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "user", content: toolResults });
  }

  return "[Max iterations reached]";
}
```

### 3.4 Why Working Memory Matters

Without working memory, the agent has to re-derive its understanding from the entire conversation history on every turn. With working memory, the system prompt contains a concise summary of the agent's current state. This:

1. **Reduces re-reading**: The LLM doesn't have to parse 50 messages to remember what it's doing
2. **Focuses attention**: The working memory highlights what matters right now
3. **Survives context compaction**: When you summarize old messages (Chapter 12), the working memory persists because it's in the system prompt
4. **Makes behavior more predictable**: The agent's plan is explicit, not implicit in the conversation flow

---

## 4. Long-Term Memory: Persistent Storage

### 4.1 What It Is

Long-term memory persists across conversations. When the user comes back tomorrow, the agent remembers relevant context from previous sessions.

There are three common approaches:
1. **File-based**: Simple, works everywhere
2. **Database-based**: Structured, queryable
3. **Vector-based**: Semantic search over memories (connects to Ch 15)

### 4.2 File-Based Memory (The CLAUDE.md Pattern)

The simplest long-term memory: a file that the agent reads at startup and can update. This is how Claude Code's CLAUDE.md works -- it's a plain text file that gives the agent persistent context about the project.

```typescript
import { readFile, writeFile, mkdir } from "fs/promises";
import { resolve, dirname } from "path";

interface MemoryStore {
  load(): Promise<string>;
  save(content: string): Promise<void>;
  append(entry: string): Promise<void>;
  search(query: string): Promise<string[]>;
}

function createFileMemory(filePath: string): MemoryStore {
  const absPath = resolve(filePath);

  return {
    async load(): Promise<string> {
      try {
        return await readFile(absPath, "utf-8");
      } catch {
        return ""; // No memory file yet
      }
    },

    async save(content: string): Promise<void> {
      await mkdir(dirname(absPath), { recursive: true });
      await writeFile(absPath, content, "utf-8");
    },

    async append(entry: string): Promise<void> {
      const existing = await this.load();
      const timestamp = new Date().toISOString();
      const newContent = existing
        ? `${existing}\n\n---\n[${timestamp}]\n${entry}`
        : `[${timestamp}]\n${entry}`;
      await this.save(newContent);
    },

    async search(query: string): Promise<string[]> {
      const content = await this.load();
      if (!content) return [];

      // Simple keyword search -- for semantic search, see Chapter 15
      const entries = content.split("\n---\n");
      const queryWords = query.toLowerCase().split(/\s+/);

      return entries.filter((entry) => {
        const lower = entry.toLowerCase();
        return queryWords.some((word) => lower.includes(word));
      });
    },
  };
}
```

### 4.3 Integrating File Memory into the Agent

```typescript
const memory = createFileMemory("./.agent-memory.md");

async function agentWithLongTermMemory(userMessage: string) {
  // Load long-term memory at the start of each conversation
  const longTermContext = await memory.load();

  const systemPrompt = `You are a helpful coding assistant.

${longTermContext ? `## What I Remember From Previous Sessions\n${longTermContext}\n` : ""}

## Tools
You have access to filesystem tools and a memory tool.
When you learn something important about the project or user preferences, save it to memory.`;

  // Run the agent loop with the enhanced system prompt
  // ... (same loop as before)
}

// Tool to save to long-term memory
const saveMemoryTool: Anthropic.Tool = {
  name: "saveToMemory",
  description:
    "Save important information to long-term memory. Use for: user preferences, project conventions, key decisions, lessons learned. This persists across conversations.",
  input_schema: {
    type: "object",
    properties: {
      category: {
        type: "string",
        enum: ["preference", "convention", "decision", "lesson", "fact"],
        description: "Category of the memory",
      },
      content: {
        type: "string",
        description: "What to remember",
      },
    },
    required: ["category", "content"],
  },
};

async function executeSaveMemory(
  input: Record<string, unknown>
): Promise<string> {
  const category = input.category as string;
  const content = input.content as string;
  await memory.append(`[${category}] ${content}`);
  return JSON.stringify({
    success: true,
    saved: `[${category}] ${content}`,
  });
}
```

### 4.4 Structured Database Memory

For more complex agents, a structured database gives you queryable memory:

```typescript
import Database from "better-sqlite3";

interface MemoryEntry {
  id: number;
  timestamp: string;
  category: string;
  content: string;
  metadata: Record<string, unknown>;
}

class DatabaseMemory {
  private db: Database.Database;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS memories (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        timestamp TEXT NOT NULL,
        category TEXT NOT NULL,
        content TEXT NOT NULL,
        metadata TEXT DEFAULT '{}',
        session_id TEXT,
        relevance_score REAL DEFAULT 1.0
      )
    `);
    this.db.exec(`
      CREATE INDEX IF NOT EXISTS idx_category ON memories(category)
    `);
    this.db.exec(`
      CREATE INDEX IF NOT EXISTS idx_timestamp ON memories(timestamp)
    `);
  }

  save(
    category: string,
    content: string,
    metadata: Record<string, unknown> = {},
    sessionId?: string
  ): void {
    const stmt = this.db.prepare(`
      INSERT INTO memories (timestamp, category, content, metadata, session_id)
      VALUES (?, ?, ?, ?, ?)
    `);
    stmt.run(
      new Date().toISOString(),
      category,
      content,
      JSON.stringify(metadata),
      sessionId || null
    );
  }

  getRecent(limit: number = 20): MemoryEntry[] {
    const stmt = this.db.prepare(`
      SELECT * FROM memories ORDER BY timestamp DESC LIMIT ?
    `);
    return stmt.all(limit) as MemoryEntry[];
  }

  getByCategory(category: string, limit: number = 10): MemoryEntry[] {
    const stmt = this.db.prepare(`
      SELECT * FROM memories WHERE category = ? ORDER BY timestamp DESC LIMIT ?
    `);
    return stmt.all(category, limit) as MemoryEntry[];
  }

  search(query: string, limit: number = 10): MemoryEntry[] {
    // Simple LIKE search -- for semantic search, use vector embeddings (Ch 15)
    const stmt = this.db.prepare(`
      SELECT * FROM memories
      WHERE content LIKE ?
      ORDER BY timestamp DESC
      LIMIT ?
    `);
    return stmt.all(`%${query}%`, limit) as MemoryEntry[];
  }

  getSessionHistory(sessionId: string): MemoryEntry[] {
    const stmt = this.db.prepare(`
      SELECT * FROM memories WHERE session_id = ? ORDER BY timestamp ASC
    `);
    return stmt.all(sessionId) as MemoryEntry[];
  }

  prune(maxAgeMs: number): number {
    const cutoff = new Date(Date.now() - maxAgeMs).toISOString();
    const stmt = this.db.prepare(`
      DELETE FROM memories WHERE timestamp < ? AND relevance_score < 0.5
    `);
    return stmt.run(cutoff).changes;
  }
}
```

### 4.5 When to Use Each Approach

| Approach | Best For | Trade-offs |
|----------|---------|------------|
| **File-based** | Simple agents, project context (CLAUDE.md style) | No structure, hard to query, grows linearly |
| **SQLite** | Local agents that need queryable history | Requires schema design, no semantic search |
| **Vector DB** | Agents that need to find relevant past context | Complex setup, requires embeddings (Ch 15) |
| **Combined** | Production agents | Most flexible but most complex |

For most agents starting out, file-based memory is enough. Graduate to a database when you need to query by category, time range, or session. Graduate to vectors when you need semantic retrieval.

---

## 5. Memory Retrieval: Getting the Right Context

### 5.1 The Retrieval Problem

Long-term memory is useless if you can't find the right memories at the right time. Loading ALL memories into the context window is wasteful and expensive. You need a retrieval strategy.

### 5.2 Retrieval Strategies

**Strategy 1: Always Load Everything (Simple, Limited)**

```typescript
async function getMemoryContext(memory: MemoryStore): Promise<string> {
  const all = await memory.load();
  return all; // Just dump it all into the system prompt
}
```

Works when memory is small (< 2000 tokens). Fails when memory grows.

**Strategy 2: Category-Based Loading**

```typescript
async function getMemoryContext(
  db: DatabaseMemory,
  userMessage: string
): Promise<string> {
  // Always load preferences and conventions
  const preferences = db.getByCategory("preference", 10);
  const conventions = db.getByCategory("convention", 10);

  // Load recent memories (last 5 sessions)
  const recent = db.getRecent(20);

  // Search for relevant memories based on the user's message
  const relevant = db.search(userMessage, 5);

  // Combine and deduplicate
  const all = [...preferences, ...conventions, ...recent, ...relevant];
  const unique = [...new Map(all.map((m) => [m.id, m])).values()];

  // Format for the system prompt
  return unique
    .map((m) => `[${m.category}] ${m.content}`)
    .join("\n");
}
```

**Strategy 3: LLM-Guided Retrieval**

Let the LLM decide what to remember by giving it a "recall" tool:

```typescript
const recallMemoryTool: Anthropic.Tool = {
  name: "recallMemory",
  description:
    "Search your long-term memory for relevant information. Use when you need to remember something from a previous conversation -- preferences, past decisions, project conventions, or lessons learned.",
  input_schema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "What to search for in memory",
      },
      category: {
        type: "string",
        enum: ["preference", "convention", "decision", "lesson", "fact", "all"],
        description: "Category to search within, or 'all'",
      },
    },
    required: ["query"],
  },
};

async function executeRecallMemory(
  input: Record<string, unknown>,
  db: DatabaseMemory
): Promise<string> {
  const query = input.query as string;
  const category = input.category as string;

  let results: MemoryEntry[];
  if (category && category !== "all") {
    results = db.getByCategory(category, 10);
    // Further filter by query
    results = results.filter((r) =>
      r.content.toLowerCase().includes(query.toLowerCase())
    );
  } else {
    results = db.search(query, 10);
  }

  if (results.length === 0) {
    return JSON.stringify({
      found: false,
      message: "No memories found matching that query.",
    });
  }

  return JSON.stringify({
    found: true,
    count: results.length,
    memories: results.map((r) => ({
      category: r.category,
      content: r.content,
      when: r.timestamp,
    })),
  });
}
```

This is the most flexible approach. The agent asks for memories when it needs them, rather than having everything loaded upfront.

---

## 6. The Memory-Context Trade-off

### 6.1 The Fundamental Tension

Every piece of memory you inject into the context window:
- **Costs tokens** (money on every LLM call)
- **Takes space** from the tool results and conversation
- **Adds noise** that might distract the LLM from the current task

But not having enough context means the agent:
- **Repeats mistakes** it already learned from
- **Asks redundant questions**
- **Ignores user preferences** it should know

### 6.2 A Token Budget for Memory

```typescript
interface MemoryBudget {
  maxTokens: number;          // Total token budget for memory
  systemPromptBase: number;   // Tokens used by base system prompt
  workingMemory: number;      // Tokens allocated to working memory
  longTermMemory: number;     // Tokens allocated to long-term memory
  reserved: number;           // Buffer for safety
}

function calculateMemoryBudget(
  contextWindowSize: number,    // e.g., 200000 for Claude
  maxOutputTokens: number,      // e.g., 4096
  currentConversationTokens: number
): MemoryBudget {
  const available = contextWindowSize - maxOutputTokens - currentConversationTokens;

  return {
    maxTokens: available,
    systemPromptBase: 500,              // ~500 tokens for base instructions
    workingMemory: Math.min(2000, available * 0.1),  // 10% of available, max 2000
    longTermMemory: Math.min(4000, available * 0.15), // 15% of available, max 4000
    reserved: 1000,                     // Safety buffer
  };
}

// Use the budget to decide how much memory to load
async function loadMemoryWithinBudget(
  budget: MemoryBudget,
  db: DatabaseMemory,
  workingMem: WorkingMemory
): Promise<{ systemPrompt: string; tokensUsed: number }> {
  const parts: string[] = [];
  let estimatedTokens = budget.systemPromptBase;

  // Working memory (highest priority)
  const workingMemStr = formatWorkingMemory(workingMem);
  const workingMemTokens = estimateTokens(workingMemStr);
  if (workingMemTokens <= budget.workingMemory) {
    parts.push(workingMemStr);
    estimatedTokens += workingMemTokens;
  }

  // Long-term memory (load until budget is reached)
  const memories = db.getRecent(50);
  let longTermStr = "";
  for (const mem of memories) {
    const entry = `[${mem.category}] ${mem.content}\n`;
    const entryTokens = estimateTokens(entry);
    if (estimatedTokens + entryTokens > budget.maxTokens - budget.reserved) {
      break; // Budget exhausted
    }
    longTermStr += entry;
    estimatedTokens += entryTokens;
  }
  if (longTermStr) {
    parts.push(`## Long-Term Memory\n${longTermStr}`);
  }

  return {
    systemPrompt: parts.join("\n\n"),
    tokensUsed: estimatedTokens,
  };
}

// Rough token estimation (4 chars per token is a common heuristic)
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

function formatWorkingMemory(mem: WorkingMemory): string {
  const parts = ["## Working Memory"];
  if (mem.currentTask) parts.push(`Task: ${mem.currentTask}`);
  if (mem.discoveries.length) {
    parts.push(`Discoveries:\n${mem.discoveries.map((d) => `- ${d}`).join("\n")}`);
  }
  if (mem.plan.length) {
    parts.push(`Plan:\n${mem.plan.map((s, i) => `${i + 1}. ${s}`).join("\n")}`);
  }
  return parts.join("\n");
}
```

### 6.3 The Rule of Thumb

A practical heuristic for memory budgets:

```
Context Window Distribution:
  System prompt + memory:  10-20% of context window
  Conversation history:    40-60% of context window
  Tool results:            20-30% of context window
  Output generation:       10% reserved for model output
```

For a 200K context window:
- System prompt + memory: 20K-40K tokens
- Conversation: 80K-120K tokens
- Tool results: 40K-60K tokens
- Output: 20K tokens

---

## 7. Putting It All Together: A Memory-Enhanced Agent

Let's build the full picture -- an agent with all three memory layers:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { readFile, writeFile, readdir, mkdir } from "fs/promises";
import { resolve, dirname } from "path";

const client = new Anthropic();

// --- Memory Layers ---

// Layer 1: Short-term (the messages array -- handled by the loop)

// Layer 2: Working memory
let workingMemory: WorkingMemory = {
  currentTask: "",
  discoveries: [],
  plan: [],
  completedSteps: [],
  openQuestions: [],
};

// Layer 3: Long-term memory (file-based for simplicity)
const longTermMemory = createFileMemory("./.agent-memory.md");

// --- Tools ---
const allTools: Anthropic.Tool[] = [
  // Filesystem tools (same as Chapter 9)
  {
    name: "readFile",
    description: "Read a file from the filesystem.",
    input_schema: {
      type: "object",
      properties: { path: { type: "string", description: "File path" } },
      required: ["path"],
    },
  },
  {
    name: "listDirectory",
    description: "List files in a directory.",
    input_schema: {
      type: "object",
      properties: { path: { type: "string", description: "Directory path" } },
      required: [],
    },
  },

  // Working memory tool
  {
    name: "updateWorkingMemory",
    description:
      "Update your scratchpad. Track current task, discoveries, plan, and questions.",
    input_schema: {
      type: "object",
      properties: {
        currentTask: { type: "string" },
        addDiscovery: { type: "string" },
        setPlan: { type: "array", items: { type: "string" } },
        completeStep: { type: "string" },
        addQuestion: { type: "string" },
      },
      required: [],
    },
  },

  // Long-term memory tools
  {
    name: "saveToMemory",
    description:
      "Save to long-term memory (persists across conversations). Use for preferences, conventions, decisions, lessons.",
    input_schema: {
      type: "object",
      properties: {
        category: {
          type: "string",
          enum: ["preference", "convention", "decision", "lesson", "fact"],
        },
        content: { type: "string" },
      },
      required: ["category", "content"],
    },
  },
  {
    name: "recallMemory",
    description:
      "Search long-term memory. Use when you need context from previous conversations.",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "What to search for" },
      },
      required: ["query"],
    },
  },
];

// --- Tool Executor ---
async function executeMemoryAwareTool(
  name: string,
  input: Record<string, unknown>
): Promise<string> {
  switch (name) {
    case "readFile": {
      try {
        return await readFile(resolve(input.path as string), "utf-8");
      } catch (err) {
        return JSON.stringify({ error: (err as Error).message });
      }
    }
    case "listDirectory": {
      try {
        const entries = await readdir(resolve((input.path as string) || "."), {
          withFileTypes: true,
        });
        return entries
          .map((e) => `${e.isDirectory() ? "[dir]" : "[file]"} ${e.name}`)
          .join("\n");
      } catch (err) {
        return JSON.stringify({ error: (err as Error).message });
      }
    }
    case "updateWorkingMemory": {
      return executeUpdateMemory(input);
    }
    case "saveToMemory": {
      await longTermMemory.append(
        `[${input.category}] ${input.content}`
      );
      return JSON.stringify({ saved: true });
    }
    case "recallMemory": {
      const results = await longTermMemory.search(input.query as string);
      return results.length > 0
        ? results.join("\n---\n")
        : "No relevant memories found.";
    }
    default:
      return JSON.stringify({ error: `Unknown tool: ${name}` });
  }
}

// --- The Memory-Enhanced Agent Loop ---
async function memoryAgent(
  userMessage: string,
  conversationHistory: Anthropic.MessageParam[]
): Promise<{
  response: string;
  history: Anthropic.MessageParam[];
}> {
  // Load long-term memory for the system prompt
  const longTermContext = await longTermMemory.load();

  const messages: Anthropic.MessageParam[] = [
    ...conversationHistory,
    { role: "user", content: userMessage },
  ];

  for (let i = 0; i < 20; i++) {
    // Rebuild system prompt each iteration with current working memory
    const systemPrompt = buildMemorySystemPrompt(
      workingMemory,
      longTermContext
    );

    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: systemPrompt,
      tools: allTools,
      messages,
    });

    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");

      messages.push({ role: "assistant", content: response.content });
      return { response: text, history: messages };
    }

    messages.push({ role: "assistant", content: response.content });

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const toolUse of toolUseBlocks) {
      const result = await executeMemoryAwareTool(
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

  return { response: "[Max iterations]", history: messages };
}

function buildMemorySystemPrompt(
  working: WorkingMemory,
  longTerm: string
): string {
  let prompt = `You are a helpful coding assistant with memory.

You have three types of memory:
1. SHORT-TERM: The conversation history (you can see this already)
2. WORKING: Your scratchpad (updated via updateWorkingMemory tool)
3. LONG-TERM: Persistent storage (via saveToMemory / recallMemory tools)

Use working memory to track your current plan and discoveries.
Use long-term memory to save things worth remembering across conversations.
Use recallMemory when you need context from previous sessions.`;

  // Inject working memory
  if (working.currentTask || working.discoveries.length > 0) {
    prompt += `\n\n## Current Working Memory`;
    if (working.currentTask) prompt += `\nTask: ${working.currentTask}`;
    if (working.discoveries.length > 0) {
      prompt += `\nDiscoveries:`;
      working.discoveries.forEach((d) => (prompt += `\n- ${d}`));
    }
    if (working.plan.length > 0) {
      prompt += `\nPlan:`;
      working.plan.forEach((step, i) => {
        const done = working.completedSteps.includes(step);
        prompt += `\n${i + 1}. ${done ? "[DONE]" : "[TODO]"} ${step}`;
      });
    }
  }

  // Inject long-term memory (truncated to budget)
  if (longTerm) {
    const maxLongTermChars = 8000; // ~2000 tokens
    const truncated =
      longTerm.length > maxLongTermChars
        ? longTerm.slice(-maxLongTermChars) + "\n[...older memories truncated]"
        : longTerm;
    prompt += `\n\n## Long-Term Memory (from previous sessions)\n${truncated}`;
  }

  return prompt;
}
```

---

## 8. Real-World Memory Patterns

### 8.1 The CLAUDE.md Pattern

Claude Code uses a file called `CLAUDE.md` as long-term project memory. It contains:
- Project description and conventions
- Build/test/lint commands
- Architecture decisions
- Code style preferences

```markdown
# CLAUDE.md

## Project
TypeScript monorepo using Turborepo. Node 20. pnpm.

## Commands
- Build: `pnpm build`
- Test: `pnpm test`
- Lint: `pnpm lint`

## Conventions
- Use `type` not `interface` for object types
- Error handling: return Result<T, E>, never throw
- All API routes use Zod validation

## Architecture
- `/apps/web` - Next.js frontend
- `/apps/api` - Express backend
- `/packages/shared` - Shared types and utils
```

This is pure long-term memory: the agent loads it at startup and uses it throughout the session. The agent can also update it -- "add to CLAUDE.md that we decided to use Drizzle for the ORM."

### 8.2 The Session Summary Pattern

At the end of each conversation, have the agent write a summary to long-term memory:

```typescript
async function endSession(
  messages: Anthropic.MessageParam[],
  memory: MemoryStore
): Promise<void> {
  // Ask the LLM to summarize the session
  const summary = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system:
      "Summarize this conversation. Focus on: what was accomplished, what decisions were made, what remains to do. Be concise.",
    messages: [
      {
        role: "user",
        content: `Summarize this session:\n${JSON.stringify(messages.slice(-20))}`,
      },
    ],
  });

  const summaryText = summary.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");

  await memory.append(`SESSION SUMMARY:\n${summaryText}`);
}
```

### 8.3 The Preference Learning Pattern

Track user corrections and preferences over time:

```typescript
// When the user corrects the agent, save the correction
async function learnFromCorrection(
  userFeedback: string,
  agentAction: string,
  memory: MemoryStore
): Promise<void> {
  await memory.append(
    `[lesson] When asked to "${agentAction}", the user prefers: ${userFeedback}`
  );
}

// Example: the agent uses tabs, user says "use spaces"
// Memory saves: [lesson] When formatting code, the user prefers:
//   2-space indentation, not tabs
```

---

## 9. Memory Anti-Patterns

### 9.1 Remembering Everything

```typescript
// BAD: save every single interaction
await memory.append(`User said: ${userMessage}`);
await memory.append(`I responded: ${response}`);
// This makes long-term memory a duplicate of conversation history
```

Only save things that are *worth remembering* across sessions: preferences, decisions, conventions, lessons.

### 9.2 Never Forgetting

Long-term memory should have a decay mechanism. Old, irrelevant memories waste context space.

```typescript
// Prune old memories periodically
async function pruneMemory(db: DatabaseMemory): Promise<void> {
  const thirtyDaysMs = 30 * 24 * 60 * 60 * 1000;
  const deleted = db.prune(thirtyDaysMs);
  console.log(`Pruned ${deleted} old memories`);
}
```

### 9.3 Loading All Memory Into Context

```typescript
// BAD: dump all 500 memories into the system prompt
const allMemories = db.getRecent(500);
systemPrompt += allMemories.map((m) => m.content).join("\n");

// GOOD: load selectively based on the current conversation
const relevant = db.search(userMessage, 10);
const prefs = db.getByCategory("preference", 5);
```

### 9.4 Not Versioning Memory

When the agent updates a preference, it should replace the old one, not add a conflicting entry:

```typescript
// BAD: contradictory memories
// [preference] Use tabs for indentation
// [preference] Use 2-space indentation  <-- added later, but old one still exists

// GOOD: update in place
async function updatePreference(
  db: DatabaseMemory,
  key: string,
  value: string
): Promise<void> {
  // Delete old preference with this key
  const stmt = db["db"].prepare(
    `DELETE FROM memories WHERE category = 'preference' AND content LIKE ?`
  );
  stmt.run(`%[preference:${key}]%`);
  // Save new one
  db.save("preference", `[preference:${key}] ${value}`);
}
```

---

## 10. What's Next

You now have an agent with three layers of memory: short-term (conversation), working (scratchpad), and long-term (persistent). But there's a problem we've been sidestepping: the agent can do *anything*. Read any file. Write any file. Call any API.

In Chapter 11, we add human-in-the-loop: approval gates that let humans review and approve dangerous actions before the agent executes them. Because an agent with tools and memory but no guardrails is a liability, not an asset.

---

## Key Takeaways

1. **Three memory layers**: short-term (messages array), working (scratchpad), long-term (persistent storage).
2. **Short-term memory is free but bounded** by the context window. Everything the agent has said and done is there.
3. **Working memory is the agent's scratchpad** -- a dynamic system prompt that tracks current task state. Update it on every iteration.
4. **Long-term memory persists across sessions** -- file-based for simple agents, database for complex ones, vector DB for semantic retrieval.
5. **Memory retrieval matters** as much as storage. Load selectively, not exhaustively.
6. **Budget your memory tokens.** 10-20% of context for system prompt + memory is a good starting point.
7. **The CLAUDE.md pattern** (project-level context file) is the simplest effective long-term memory.

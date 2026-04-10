<!--
  CHAPTER: 9
  TITLE: The Agent Loop
  PART: 2 — Agent Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 8 (tool calling), Ch 6 (conversation history), Ch 3 (streaming)
  KEY_TOPICS: agent loop pattern, prompt-llm-tool-execute-append cycle, stop conditions, streaming in loops, parallel tool calls, CLI agent
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 9: The Agent Loop

> **Part 2 — Agent Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 8, Ch 6, Ch 3 | Difficulty: Intermediate | Language: TypeScript

In Chapter 8, you gave the LLM tools. It can call a function, get a result, and incorporate that result into its response. But that's a one-shot interaction. The LLM calls a tool, sees the result, and responds. What if the result tells the LLM it needs to call *another* tool? What if the task requires five tools in sequence, where each step depends on the previous result?

That's what the agent loop is. It's the simplest and most powerful pattern in agent engineering: send a prompt to the LLM, check if it wants to call a tool, execute the tool, append the result to the conversation, and loop back to the LLM. Repeat until the LLM says "I'm done" or you hit a safety limit.

This chapter is the most important in the entire guide. Every agent -- Claude Code, Cursor, Devin, custom coding agents -- is, at its core, a loop. The implementations differ in memory management (Chapter 10), approval gates (Chapter 11), and context compaction (Chapter 12), but the fundamental pattern is always the same: prompt, think, act, observe, repeat. Once you can build this loop from scratch, you understand how every agent works.

### In This Chapter
- The core pattern: prompt -> LLM -> tool call -> execute -> append -> repeat
- Building the loop from scratch with the Anthropic SDK
- Building the loop with the Vercel AI SDK
- Stop conditions: maxIterations, no more tool calls, explicit done signal
- Streaming tokens while the loop runs
- Handling multiple parallel tool calls in a single response
- Building a real CLI agent that can do useful tasks
- Debugging agent loops: when things go wrong

### Related Chapters
- **Ch 8 (Tool Calling)** — tools are what run inside the loop
- **Ch 6 (Chat Interface)** — conversation history IS the loop's state
- **Ch 10 (Agent Memory)** — how memory extends and shapes the loop
- **Ch 20 (Multi-Turn Evals)** — evaluate agent loop behavior
- **Ch 23 (Claude Code Mastery)** — Claude Code IS an agent loop
- **Ch 27 (Harness Architecture)** — production agent loop architectures

---

## 1. The Core Pattern

### 1.1 The Loop in Five Steps

Every agent loop does the same five things:

```
1. PROMPT    → Send messages to the LLM (includes conversation history)
2. THINK     → LLM processes and decides: "I need to call a tool" or "I'm done"
3. ACT       → If tool call: execute the tool, get the result
4. OBSERVE   → Append the tool result to the conversation history
5. REPEAT    → Go back to step 1 with the updated history

Exit when:
  - LLM responds with text (no tool calls) → task is done
  - Maximum iterations reached → safety limit
  - Explicit stop signal → the agent decides to stop
```

In pseudocode:

```
messages = [system_prompt, user_message]

loop:
  response = llm.call(messages)
  
  if response has tool_calls:
    for each tool_call in response.tool_calls:
      result = execute(tool_call)
      messages.append(assistant_message_with_tool_call)
      messages.append(tool_result)
    continue loop
  
  else:
    return response.text  // Done!
```

That's it. That's the agent. Everything else is optimization.

### 1.2 Why This Works

The magic is that the LLM sees the *entire conversation history* on every iteration. When the loop appends a tool result and sends everything back, the LLM can:

1. See what it already tried
2. See what the tool returned
3. Decide whether it needs more information
4. Plan its next action based on everything so far

This is the same pattern as a human problem-solver: try something, observe the result, decide what to do next. The conversation history is the agent's "working memory" -- we'll formalize this in Chapter 10.

---

## 2. Building the Loop from Scratch (Anthropic SDK)

Let's build the agent loop directly with the Anthropic SDK. No frameworks, no abstractions -- just the raw loop. Understanding this is essential before using any framework.

### 2.1 Defining Tools in Anthropic's Format

The Anthropic SDK expects tools in a specific format. Here's how our tools from Chapter 8 look in native Anthropic format:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const tools: Anthropic.Tool[] = [
  {
    name: "getCurrentDateTime",
    description:
      "Get the current date and time. Use when the user asks about today's date, current time, or needs temporal context.",
    input_schema: {
      type: "object" as const,
      properties: {
        timezone: {
          type: "string",
          description: "IANA timezone like 'America/New_York'. Defaults to UTC.",
        },
      },
      required: [],
    },
  },
  {
    name: "readFile",
    description:
      "Read a file from the local filesystem. Returns file content as text. Use for code, configs, docs.",
    input_schema: {
      type: "object" as const,
      properties: {
        path: {
          type: "string",
          description: "File path to read",
        },
      },
      required: ["path"],
    },
  },
  {
    name: "listDirectory",
    description:
      "List files and directories at a given path. Use to explore the filesystem structure.",
    input_schema: {
      type: "object" as const,
      properties: {
        path: {
          type: "string",
          description: "Directory path to list. Defaults to current directory.",
        },
      },
      required: [],
    },
  },
  {
    name: "writeFile",
    description:
      "Write content to a file. Creates the file if it doesn't exist, overwrites if it does.",
    input_schema: {
      type: "object" as const,
      properties: {
        path: {
          type: "string",
          description: "File path to write to",
        },
        content: {
          type: "string",
          description: "Content to write to the file",
        },
      },
      required: ["path", "content"],
    },
  },
];
```

### 2.2 The Tool Execution Registry

We need a function that takes a tool name and input, and executes the right function:

```typescript
import { readFile, writeFile, readdir, stat } from "fs/promises";
import { resolve, join } from "path";

type ToolInput = Record<string, unknown>;

async function executeTool(
  name: string,
  input: ToolInput
): Promise<string> {
  switch (name) {
    case "getCurrentDateTime": {
      const timezone = (input.timezone as string) || "UTC";
      const now = new Date();
      return JSON.stringify({
        datetime: now.toLocaleString("en-US", {
          timeZone: timezone,
          dateStyle: "full",
          timeStyle: "long",
        }),
        timezone,
        iso: now.toISOString(),
      });
    }

    case "readFile": {
      try {
        const filePath = resolve(input.path as string);
        const content = await readFile(filePath, "utf-8");
        return JSON.stringify({
          content,
          path: filePath,
          lines: content.split("\n").length,
        });
      } catch (err) {
        return JSON.stringify({
          error: `Failed to read file: ${(err as Error).message}`,
        });
      }
    }

    case "listDirectory": {
      try {
        const dirPath = resolve((input.path as string) || ".");
        const entries = await readdir(dirPath, { withFileTypes: true });
        const items = entries.map((e) => ({
          name: e.name,
          type: e.isDirectory() ? "directory" : "file",
        }));
        return JSON.stringify({ path: dirPath, entries: items });
      } catch (err) {
        return JSON.stringify({
          error: `Failed to list directory: ${(err as Error).message}`,
        });
      }
    }

    case "writeFile": {
      try {
        const filePath = resolve(input.path as string);
        await writeFile(filePath, input.content as string, "utf-8");
        return JSON.stringify({
          success: true,
          path: filePath,
          bytesWritten: (input.content as string).length,
        });
      } catch (err) {
        return JSON.stringify({
          error: `Failed to write file: ${(err as Error).message}`,
        });
      }
    }

    default:
      return JSON.stringify({ error: `Unknown tool: ${name}` });
  }
}
```

### 2.3 The Agent Loop

Now the core loop. This is the heart of every agent:

```typescript
const client = new Anthropic();

interface AgentOptions {
  systemPrompt: string;
  maxIterations: number;
  onToolCall?: (name: string, input: ToolInput) => void;
  onToolResult?: (name: string, result: string) => void;
  onText?: (text: string) => void;
}

async function agentLoop(
  userMessage: string,
  options: AgentOptions
): Promise<string> {
  const {
    systemPrompt,
    maxIterations,
    onToolCall,
    onToolResult,
    onText,
  } = options;

  // The messages array IS the agent's state
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  for (let iteration = 0; iteration < maxIterations; iteration++) {
    // Step 1: PROMPT — send messages to the LLM
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: systemPrompt,
      tools,
      messages,
    });

    // Step 2: THINK — check what the LLM decided
    // Separate text blocks from tool_use blocks
    const textBlocks = response.content.filter(
      (block): block is Anthropic.TextBlock => block.type === "text"
    );
    const toolUseBlocks = response.content.filter(
      (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
    );

    // If no tool calls, we're done — the LLM has its final answer
    if (toolUseBlocks.length === 0) {
      const finalText = textBlocks.map((b) => b.text).join("\n");
      onText?.(finalText);
      return finalText;
    }

    // Step 3: ACT — execute each tool call
    // First, add the assistant's response (with tool_use blocks) to history
    messages.push({ role: "assistant", content: response.content });

    // Execute all tool calls and collect results
    const toolResults: Anthropic.ToolResultBlockParam[] = [];

    for (const toolUse of toolUseBlocks) {
      onToolCall?.(toolUse.name, toolUse.input as ToolInput);

      const result = await executeTool(
        toolUse.name,
        toolUse.input as ToolInput
      );

      onToolResult?.(toolUse.name, result);

      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    // Step 4: OBSERVE — append tool results to conversation
    messages.push({ role: "user", content: toolResults });

    // Step 5: REPEAT — the loop continues
    // If any text was generated alongside tool calls, log it
    if (textBlocks.length > 0) {
      onText?.(textBlocks.map((b) => b.text).join("\n"));
    }
  }

  // Safety: hit max iterations
  return "[Agent stopped: maximum iterations reached]";
}
```

### 2.4 Using the Loop

```typescript
async function main() {
  const response = await agentLoop(
    "Look at the files in the current directory, read the package.json if it exists, and tell me what this project does.",
    {
      systemPrompt:
        "You are a helpful coding assistant. Use the available tools to explore the filesystem and answer questions about code. Be thorough but concise.",
      maxIterations: 10,
      onToolCall: (name, input) => {
        console.log(`  → Calling ${name}(${JSON.stringify(input)})`);
      },
      onToolResult: (name, result) => {
        const parsed = JSON.parse(result);
        if (parsed.error) {
          console.log(`  ← ${name}: ERROR - ${parsed.error}`);
        } else {
          console.log(`  ← ${name}: OK`);
        }
      },
      onText: (text) => {
        // Text generated alongside tool calls (thinking out loud)
      },
    }
  );

  console.log("\n--- Agent Response ---");
  console.log(response);
}

main();
```

**What happens:**
```
  → Calling listDirectory({"path":"."})
  ← listDirectory: OK
  → Calling readFile({"path":"./package.json"})
  ← readFile: OK

--- Agent Response ---
This is a TypeScript project that...
```

The agent called `listDirectory` first, saw that `package.json` exists, then called `readFile` to examine it. Two loop iterations, fully autonomous.

---

## 3. Stop Conditions

An agent loop without stop conditions is an infinite loop with an API bill. Here are the three ways agents stop.

### 3.1 No More Tool Calls (Natural Stop)

The most common stop. The LLM decides it has enough information and responds with text instead of a tool call.

```typescript
// In the loop:
if (toolUseBlocks.length === 0) {
  // LLM is done -- return its final text
  return textBlocks.map((b) => b.text).join("\n");
}
```

This is the ideal stop -- the agent completed its task and is reporting results.

### 3.2 Maximum Iterations (Safety Stop)

Always set a maximum. LLMs can get stuck in loops, calling the same tool repeatedly or chasing a dead end.

```typescript
for (let iteration = 0; iteration < maxIterations; iteration++) {
  // ... loop body ...
}

// If we get here, the loop exhausted its budget
return "[Agent stopped: maximum iterations reached]";
```

**How to choose `maxIterations`:**
- Simple lookup tasks: 3-5
- File exploration: 5-10
- Code modification tasks: 10-20
- Complex multi-step tasks: 20-50
- Claude Code-style agents: 100+ (with compaction, see Chapter 12)

### 3.3 Explicit Stop Signal

Sometimes you want the agent to be able to signal "I'm done" even in the middle of tool execution. You can add a `done` tool:

```typescript
const doneSignalTool: Anthropic.Tool = {
  name: "taskComplete",
  description:
    "Signal that the task is complete. Call this when you have finished all requested work and want to provide your final summary.",
  input_schema: {
    type: "object",
    properties: {
      summary: {
        type: "string",
        description: "Final summary of what was accomplished",
      },
    },
    required: ["summary"],
  },
};

// In the loop, check for the stop signal:
for (const toolUse of toolUseBlocks) {
  if (toolUse.name === "taskComplete") {
    const input = toolUse.input as { summary: string };
    return input.summary;
  }
  // ... execute other tools normally
}
```

This is useful when the task involves writing files or making changes, and the "final answer" is a summary of what was done, not a natural text response.

---

## 4. Building the Loop with the Vercel AI SDK

The Vercel AI SDK provides `generateText` with `maxSteps` which is essentially a built-in agent loop. But for full control, you can also build the loop yourself using the SDK's tool abstractions.

### 4.1 The Simple Way: maxSteps

```typescript
import { generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  system: "You are a helpful coding assistant.",
  tools: {
    listDirectory: tool({
      description: "List files in a directory",
      parameters: z.object({
        path: z.string().default(".").describe("Directory path"),
      }),
      execute: async ({ path: dirPath }) => {
        const entries = await readdir(resolve(dirPath), { withFileTypes: true });
        return entries.map((e) => ({
          name: e.name,
          type: e.isDirectory() ? "dir" : "file",
        }));
      },
    }),
    readFile: tool({
      description: "Read a file's contents",
      parameters: z.object({
        path: z.string().describe("File path"),
      }),
      execute: async ({ path: filePath }) => {
        return await readFile(resolve(filePath), "utf-8");
      },
    }),
  },
  maxSteps: 10,
  prompt: "Read the package.json and tell me what this project does",
});

console.log(result.text);
console.log(`Completed in ${result.steps.length} steps`);
```

`maxSteps` is the Vercel AI SDK's agent loop. Under the hood, it does exactly what our manual loop does: call the LLM, execute tools, append results, repeat.

### 4.2 Step Callbacks: Observing the Loop

```typescript
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { /* ... */ },
  maxSteps: 10,
  prompt: "Explore the project structure",

  // Called after each step (LLM call + tool execution)
  onStepFinish: ({ text, toolCalls, toolResults, finishReason, usage }) => {
    if (toolCalls.length > 0) {
      for (const tc of toolCalls) {
        console.log(`Step: called ${tc.toolName}(${JSON.stringify(tc.args)})`);
      }
    }
    if (text) {
      console.log(`Step: generated text (${text.length} chars)`);
    }
    console.log(`  Tokens used: ${usage.totalTokens}`);
  },
});
```

### 4.3 When to Use maxSteps vs a Manual Loop

| Use `maxSteps` when... | Build a manual loop when... |
|---|---|
| Simple tool use chains | You need approval gates (Ch 11) |
| You trust all tools | You need memory management (Ch 10) |
| No special stop conditions | You need context compaction (Ch 12) |
| Straightforward tasks | You need custom error recovery |
| Prototype/exploration | Production agents |

For the rest of this chapter, we'll continue with the manual loop because understanding the raw mechanics makes everything else clearer.

---

## 5. Streaming in the Agent Loop

Users hate staring at a blank screen. When the agent is thinking or calling tools, the user should see what's happening. Let's add streaming.

### 5.1 Streaming Text While Tools Execute

```typescript
async function streamingAgentLoop(
  userMessage: string,
  options: AgentOptions
): Promise<string> {
  const { systemPrompt, maxIterations } = options;
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  let fullResponse = "";

  for (let iteration = 0; iteration < maxIterations; iteration++) {
    // Use streaming instead of a blocking call
    const stream = client.messages.stream({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: systemPrompt,
      tools,
      messages,
    });

    // Collect the full response while streaming text
    const response = await (async () => {
      let currentText = "";
      const toolCalls: Array<{
        id: string;
        name: string;
        input: ToolInput;
      }> = [];

      // Stream text tokens as they arrive
      stream.on("text", (text) => {
        process.stdout.write(text); // Real-time output
        currentText += text;
      });

      // Collect tool calls
      stream.on("contentBlock", (block) => {
        if (block.type === "tool_use") {
          // Tool call detected -- show it to the user
          process.stdout.write(
            `\n[Calling ${block.name}...]\n`
          );
        }
      });

      // Wait for the complete response
      const finalMessage = await stream.finalMessage();
      return finalMessage;
    })();

    // Check for tool calls (same as before)
    const toolUseBlocks = response.content.filter(
      (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      const text = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("\n");
      fullResponse += text;
      return fullResponse;
    }

    // Execute tools and continue (same as before)
    messages.push({ role: "assistant", content: response.content });

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const toolUse of toolUseBlocks) {
      const result = await executeTool(
        toolUse.name,
        toolUse.input as ToolInput
      );
      process.stdout.write(
        `[${toolUse.name} completed]\n`
      );
      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "user", content: toolResults });
  }

  return fullResponse || "[Agent stopped: maximum iterations reached]";
}
```

### 5.2 The Vercel AI SDK Streaming Loop

```typescript
import { streamText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

async function streamingAgentWithSDK(userMessage: string) {
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: "You are a helpful coding assistant.",
    tools: {
      readFile: tool({
        description: "Read a file",
        parameters: z.object({ path: z.string() }),
        execute: async ({ path: p }) => readFile(resolve(p), "utf-8"),
      }),
    },
    maxSteps: 10,
    prompt: userMessage,
  });

  // Stream the full response including tool events
  for await (const event of result.fullStream) {
    switch (event.type) {
      case "text-delta":
        process.stdout.write(event.textDelta);
        break;
      case "tool-call":
        console.log(`\n[Calling ${event.toolName}...]`);
        break;
      case "tool-result":
        console.log(`[Tool completed]`);
        break;
      case "step-finish":
        // A step (LLM call) completed
        break;
      case "finish":
        console.log(`\n[Done in ${event.usage?.totalTokens} tokens]`);
        break;
    }
  }
}
```

---

## 6. Handling Parallel Tool Calls

LLMs can request multiple tool calls in a single response. This typically happens when the tools are independent -- "read these three files" or "search for X and also get the current time."

### 6.1 Sequential vs Parallel Execution

```typescript
// Sequential: one at a time (simpler, slower)
for (const toolUse of toolUseBlocks) {
  const result = await executeTool(toolUse.name, toolUse.input as ToolInput);
  toolResults.push({
    type: "tool_result",
    tool_use_id: toolUse.id,
    content: result,
  });
}

// Parallel: all at once (faster, better for independent operations)
const toolResults = await Promise.all(
  toolUseBlocks.map(async (toolUse) => {
    const result = await executeTool(toolUse.name, toolUse.input as ToolInput);
    return {
      type: "tool_result" as const,
      tool_use_id: toolUse.id,
      content: result,
    };
  })
);
```

**When to parallelize:**
- Reading multiple files: yes
- Multiple web searches: yes (if your API allows concurrent requests)
- Writing to the same file: NO (race condition)
- Operations that depend on each other: NO

A safe default is to parallelize reads and sequentialize writes:

```typescript
const toolResults: Anthropic.ToolResultBlockParam[] = [];

// Separate reads from writes
const reads = toolUseBlocks.filter((t) =>
  ["readFile", "listDirectory", "webSearch", "getCurrentDateTime"].includes(t.name)
);
const writes = toolUseBlocks.filter((t) =>
  ["writeFile", "runCommand"].includes(t.name)
);

// Execute reads in parallel
const readResults = await Promise.all(
  reads.map(async (toolUse) => ({
    type: "tool_result" as const,
    tool_use_id: toolUse.id,
    content: await executeTool(toolUse.name, toolUse.input as ToolInput),
  }))
);
toolResults.push(...readResults);

// Execute writes sequentially
for (const toolUse of writes) {
  const result = await executeTool(toolUse.name, toolUse.input as ToolInput);
  toolResults.push({
    type: "tool_result",
    tool_use_id: toolUse.id,
    content: result,
  });
}
```

---

## 7. Building a Real CLI Agent

Let's put everything together into a CLI agent that accepts user input, loops autonomously, and streams output. This is a simplified version of what Claude Code does.

```typescript
import Anthropic from "@anthropic-ai/sdk";
import * as readline from "readline";
import { readFile, writeFile, readdir, stat, mkdir } from "fs/promises";
import { resolve, dirname } from "path";

// --- Configuration ---
const client = new Anthropic();
const MODEL = "claude-sonnet-4-20250514";
const MAX_ITERATIONS = 20;

const SYSTEM_PROMPT = `You are a helpful CLI assistant with access to the local filesystem. You can read files, list directories, write files, and run simple tasks.

Guidelines:
- Always explore the filesystem before making assumptions about project structure
- When writing files, show the user what you plan to write before doing it
- If a task requires multiple steps, explain your plan before starting
- Be concise in your explanations but thorough in your work`;

// --- Tools ---
const agentTools: Anthropic.Tool[] = [
  {
    name: "readFile",
    description:
      "Read a file's contents. Use for code, config, docs. Returns content as string.",
    input_schema: {
      type: "object",
      properties: {
        path: { type: "string", description: "File path to read" },
      },
      required: ["path"],
    },
  },
  {
    name: "writeFile",
    description:
      "Write content to a file. Creates parent directories if needed.",
    input_schema: {
      type: "object",
      properties: {
        path: { type: "string", description: "File path to write" },
        content: { type: "string", description: "Content to write" },
      },
      required: ["path", "content"],
    },
  },
  {
    name: "listDirectory",
    description:
      "List files and directories at a path. Shows names and types (file/dir).",
    input_schema: {
      type: "object",
      properties: {
        path: {
          type: "string",
          description: "Directory path. Defaults to current directory.",
        },
      },
      required: [],
    },
  },
  {
    name: "getCurrentDateTime",
    description: "Get current date and time.",
    input_schema: {
      type: "object",
      properties: {},
      required: [],
    },
  },
];

// --- Tool Execution ---
async function executeAgentTool(
  name: string,
  input: Record<string, unknown>
): Promise<string> {
  switch (name) {
    case "readFile": {
      try {
        const content = await readFile(resolve(input.path as string), "utf-8");
        return content;
      } catch (err) {
        return `Error: ${(err as Error).message}`;
      }
    }
    case "writeFile": {
      try {
        const filePath = resolve(input.path as string);
        await mkdir(dirname(filePath), { recursive: true });
        await writeFile(filePath, input.content as string, "utf-8");
        return `Successfully wrote ${(input.content as string).length} bytes to ${filePath}`;
      } catch (err) {
        return `Error: ${(err as Error).message}`;
      }
    }
    case "listDirectory": {
      try {
        const dirPath = resolve((input.path as string) || ".");
        const entries = await readdir(dirPath, { withFileTypes: true });
        return entries
          .map((e) => `${e.isDirectory() ? "[dir]" : "[file]"} ${e.name}`)
          .join("\n");
      } catch (err) {
        return `Error: ${(err as Error).message}`;
      }
    }
    case "getCurrentDateTime": {
      return new Date().toISOString();
    }
    default:
      return `Unknown tool: ${name}`;
  }
}

// --- The Agent Loop ---
async function runAgent(
  userMessage: string,
  conversationHistory: Anthropic.MessageParam[]
): Promise<{
  response: string;
  updatedHistory: Anthropic.MessageParam[];
}> {
  // Add user message to history
  const messages: Anthropic.MessageParam[] = [
    ...conversationHistory,
    { role: "user", content: userMessage },
  ];

  let iterations = 0;

  while (iterations < MAX_ITERATIONS) {
    iterations++;

    // Call the LLM with streaming
    const stream = client.messages.stream({
      model: MODEL,
      max_tokens: 4096,
      system: SYSTEM_PROMPT,
      tools: agentTools,
      messages,
    });

    // Stream text output
    stream.on("text", (text) => {
      process.stdout.write(text);
    });

    const response = await stream.finalMessage();

    // Check for tool calls
    const toolUseBlocks = response.content.filter(
      (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
    );

    if (toolUseBlocks.length === 0) {
      // Done -- extract final text
      const finalText = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === "text")
        .map((b) => b.text)
        .join("");

      // Update history with this exchange
      messages.push({ role: "assistant", content: response.content });

      return { response: finalText, updatedHistory: messages };
    }

    // Has tool calls -- execute them
    messages.push({ role: "assistant", content: response.content });

    const toolResults: Anthropic.ToolResultBlockParam[] = [];
    for (const toolUse of toolUseBlocks) {
      console.log(
        `\n  [${toolUse.name}(${JSON.stringify(toolUse.input).slice(0, 100)})]`
      );

      const result = await executeAgentTool(
        toolUse.name,
        toolUse.input as Record<string, unknown>
      );

      // Show a brief result preview
      const preview =
        result.length > 200 ? result.slice(0, 200) + "..." : result;
      console.log(`  → ${preview.split("\n")[0]}`);

      toolResults.push({
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: result,
      });
    }

    messages.push({ role: "user", content: toolResults });
    console.log(""); // Newline before next iteration's output
  }

  // Hit max iterations
  const stopMessage = `\n[Stopped after ${MAX_ITERATIONS} iterations]`;
  console.log(stopMessage);
  return { response: stopMessage, updatedHistory: messages };
}

// --- Interactive CLI ---
async function main() {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  console.log("CLI Agent (type 'exit' to quit)");
  console.log("================================\n");

  let conversationHistory: Anthropic.MessageParam[] = [];

  const prompt = () => {
    rl.question("You: ", async (input) => {
      const trimmed = input.trim();
      if (trimmed.toLowerCase() === "exit") {
        console.log("Goodbye!");
        rl.close();
        return;
      }

      if (!trimmed) {
        prompt();
        return;
      }

      console.log(""); // Spacing
      process.stdout.write("Agent: ");

      try {
        const { updatedHistory } = await runAgent(
          trimmed,
          conversationHistory
        );
        conversationHistory = updatedHistory;
      } catch (err) {
        console.error(`\nError: ${(err as Error).message}`);
      }

      console.log("\n"); // Spacing after response
      prompt();
    });
  };

  prompt();
}

main();
```

### 7.1 What This Agent Can Do

Run it and try:

```
You: What files are in this project?
Agent: [listDirectory({})]
  → [dir] src
     [dir] node_modules
     [file] package.json
     [file] tsconfig.json
     ...
Let me explore the structure...
[listDirectory({"path":"./src"})]
  → [file] index.ts
     [file] utils.ts
     ...

This project contains...

You: Read the package.json and explain the dependencies
Agent: [readFile({"path":"./package.json"})]
  → {"name": "my-project"...

This project uses the following dependencies:
- @anthropic-ai/sdk: For calling Claude's API
- zod: For schema validation
...

You: Create a .env.example file with the required environment variables
Agent: Based on the dependencies, I'll create a .env.example file:

[writeFile({"path":"./.env.example","content":"# Anthropic API Key\nANTHROPIC_API_KEY=\n\n# Optional\nMODEL=claude-sonnet-4-20250514\n"})]
  → Successfully wrote 67 bytes to /path/to/.env.example

I created `.env.example` with the following variables:
- ANTHROPIC_API_KEY (required)
- MODEL (optional, defaults to claude-sonnet-4-20250514)
```

This is a real agent. It explores, reasons, acts, and reports. It's ~150 lines of core logic. Everything you add from here -- memory, approvals, compaction -- builds on this foundation.

---

## 8. Conversation History: The Loop's State

A critical detail: **the messages array IS the agent's state**. Every tool call and tool result is appended to the messages. This means:

### 8.1 The Message Flow

```
Initial:
  messages = [
    { role: "user", content: "Read package.json" }
  ]

After iteration 1 (LLM calls readFile):
  messages = [
    { role: "user", content: "Read package.json" },
    { role: "assistant", content: [tool_use block for readFile] },
    { role: "user", content: [tool_result with file contents] },
  ]

After iteration 2 (LLM responds with summary):
  messages = [
    { role: "user", content: "Read package.json" },
    { role: "assistant", content: [tool_use block for readFile] },
    { role: "user", content: [tool_result with file contents] },
    { role: "assistant", content: [text block with summary] },
  ]
```

### 8.2 Why Tool Results Are "user" Messages

Notice that tool results go in as `role: "user"` messages. This is Anthropic's convention. The flow is:
- **assistant** generates a tool_use block ("I want to call readFile")
- **user** provides the tool_result ("here's what readFile returned")

This mirrors the real protocol: the assistant requests, the system (acting as user) provides the result. OpenAI uses a dedicated `tool` role instead, but the concept is the same.

### 8.3 The Growing Context Problem

Every iteration adds messages. A 10-iteration agent loop might have 25+ messages. Each message contains the full tool input AND output. For file-reading agents, this adds up fast.

This is why Chapter 12 exists. But for now, be aware: long agent loops can fill the context window. We'll solve this properly later.

---

## 9. Debugging Agent Loops

When agent loops go wrong, they go wrong in consistent ways. Here's how to diagnose them.

### 9.1 The Infinite Loop

**Symptom:** The agent keeps calling the same tool with the same arguments.

**Cause:** Usually the tool returns an error the LLM doesn't understand, or the LLM keeps trying a strategy that doesn't work.

**Fix:** Add iteration tracking and detection:

```typescript
const toolCallHistory: string[] = [];

for (let iteration = 0; iteration < maxIterations; iteration++) {
  // ... get response ...

  for (const toolUse of toolUseBlocks) {
    const callSignature = `${toolUse.name}:${JSON.stringify(toolUse.input)}`;

    // Detect repeated calls
    const repeatCount = toolCallHistory.filter(
      (h) => h === callSignature
    ).length;

    if (repeatCount >= 3) {
      console.warn(
        `Agent called ${toolUse.name} with same args ${repeatCount} times. Injecting guidance.`
      );
      // Inject a hint into the conversation
      messages.push({
        role: "user",
        content:
          "You've called the same tool with the same arguments multiple times. Try a different approach or explain what's going wrong.",
      });
      break;
    }

    toolCallHistory.push(callSignature);
  }
}
```

### 9.2 The Wrong Tool Problem

**Symptom:** The LLM calls `webSearch` when it should call `readFile`, or vice versa.

**Cause:** Ambiguous tool descriptions. See Chapter 8, Section 5.

**Fix:** Improve descriptions. Add negative guidance: "Do NOT use this for local file operations."

### 9.3 The Token Explosion

**Symptom:** Costs spike, requests slow down, eventually the LLM returns an error about exceeding context limits.

**Cause:** Tool results are too large (e.g., reading a 10,000-line file), or too many iterations accumulate.

**Fix:** Truncate tool results, implement context management (Chapter 12):

```typescript
async function executeTool(name: string, input: ToolInput): Promise<string> {
  const result = await executeToolRaw(name, input);

  // Safety: truncate large results
  const MAX_RESULT_LENGTH = 10_000; // ~2500 tokens
  if (result.length > MAX_RESULT_LENGTH) {
    return (
      result.slice(0, MAX_RESULT_LENGTH) +
      `\n\n[Truncated: result was ${result.length} characters. Use more specific queries to get focused results.]`
    );
  }

  return result;
}
```

### 9.4 Logging for Debugging

The most useful thing you can do for debugging agent loops is structured logging:

```typescript
interface LoopLog {
  iteration: number;
  timestamp: number;
  toolCalls: Array<{ name: string; input: unknown }>;
  toolResults: Array<{ name: string; resultLength: number; error?: string }>;
  textGenerated: number;
  totalTokens: number;
  stopReason: string;
}

const logs: LoopLog[] = [];

// In the loop:
logs.push({
  iteration,
  timestamp: Date.now(),
  toolCalls: toolUseBlocks.map((t) => ({
    name: t.name,
    input: t.input,
  })),
  toolResults: toolResults.map((r) => ({
    name: r.tool_use_id,
    resultLength: r.content?.toString().length ?? 0,
  })),
  textGenerated: textBlocks.reduce((sum, b) => sum + b.text.length, 0),
  totalTokens: response.usage.input_tokens + response.usage.output_tokens,
  stopReason: response.stop_reason,
});

// After the loop, dump logs:
console.log(JSON.stringify(logs, null, 2));
```

---

## 10. The Agent Loop vs Vercel AI SDK's maxSteps

Let's be precise about the relationship. The Vercel AI SDK's `maxSteps` IS an agent loop. When you write:

```typescript
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { readFile, listDir },
  maxSteps: 10,
  prompt: "Explore this project",
});
```

The SDK is doing exactly what our manual loop does:

```
Step 1: Call LLM → gets tool_use for listDir → executes → appends result
Step 2: Call LLM → gets tool_use for readFile → executes → appends result
Step 3: Call LLM → generates text → done
```

The difference is that our manual loop gives us hooks for:
- Custom stop conditions
- Approval gates before tool execution (Chapter 11)
- Memory injection between iterations (Chapter 10)
- Context compaction when messages grow too large (Chapter 12)
- Custom error recovery
- Logging and observability

`maxSteps` is perfect for simple agents. The manual loop is for production agents.

---

## 11. What's Next

You have the loop. The LLM can think, act, and repeat until the job is done. But two problems are already visible:

1. **The messages array grows forever.** Every tool call and result accumulates. After 20 iterations, you might have 50+ messages consuming most of your context window. Chapter 12 solves this with compaction and sliding windows.

2. **The agent has no memory across sessions.** Close the program, and all context is lost. The next conversation starts from scratch. Chapter 10 adds short-term, working, and long-term memory.

But first -- Chapter 10. Because the question isn't just "how does the loop run?" but "what does the loop remember?"

---

## Key Takeaways

1. **The agent loop is: prompt -> think -> act -> observe -> repeat.** Every agent, from Claude Code to custom bots, uses this pattern.
2. **The messages array IS the agent's state.** Each iteration adds to it, giving the LLM more context for its next decision.
3. **Three stop conditions:** no more tool calls (natural), max iterations (safety), explicit stop signal (design choice).
4. **Stream the output.** Users need feedback while the agent works.
5. **Handle parallel tool calls.** Parallelize reads, sequentialize writes.
6. **Log everything.** Agent loops are harder to debug than linear code because behavior depends on the LLM's decisions.
7. **`maxSteps` is a simple agent loop.** Use it for prototypes, build manually for production.

<!--
  CHAPTER: 8
  TITLE: Tool Calling
  PART: 2 — Agent Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 3 (LLM API calls), Ch 5 (Zod schemas, structured output)
  KEY_TOPICS: tool definitions, function calling, Zod parameter schemas, execute pattern, description engineering, multi-tool selection
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 8: Tool Calling

> **Part 2 — Agent Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 3, Ch 5 | Difficulty: Intermediate

An LLM that can only generate text is like a brain in a jar. It can think, reason, even argue -- but it can't *do* anything. It can't look up today's date. It can't read a file. It can't search the web. It's stuck generating plausible-sounding text based on training data that stopped months or years ago.

Tool calling changes everything. When you give an LLM access to tools, you give it hands. The LLM doesn't execute the tools directly -- it *chooses* which tool to call and *what arguments to pass*, and then your code executes the tool and feeds the result back. This is the single most important concept in agent engineering. Everything in the rest of this Part builds on it.

If you've built structured output with Zod in Chapter 5, you already know half of the pattern. A tool definition is a Zod schema (the parameters the tool accepts) plus a description (so the LLM knows when to use it) plus an execute function (what actually runs). That's it. The mental leap is that the LLM is no longer just generating data -- it's making *decisions* about what actions to take.

### In This Chapter
- What tool calling is and how it differs from structured output
- Defining tools with name, description, and Zod parameter schemas
- The execute function pattern
- How LLMs decide which tool to call (description quality matters enormously)
- Building real tools: `getCurrentDateTime`, `webSearch`, `fileRead`
- Multiple tools: letting the LLM choose between them
- Error handling and tool call validation
- Common pitfalls and best practices

### Related Chapters
- **Ch 5 (Structured Output)** — same Zod schema pattern, now used for tool parameters
- **Ch 3 (Your First LLM Call)** — the LLM chooses which function to invoke
- **Ch 9 (The Agent Loop)** — tools run inside the loop; this chapter defines them
- **Ch 19 (Single-Turn Evals)** — evaluate whether the LLM picks the right tool
- **Ch 24 (MCP Servers)** — MCP servers ARE tools, exposed over a protocol
- **Ch 30 (Tool Execution Internals)** — how Claude Code manages tool execution at scale

---

## 1. What Tool Calling Actually Is

### 1.1 The Core Idea

**What it is:** Tool calling (sometimes called "function calling") is a protocol where the LLM can request that your application execute a function on its behalf. The LLM doesn't run the function. It generates a structured request -- "call this function with these arguments" -- and your code handles the rest.

**The flow looks like this:**

```
You send a message: "What time is it in Tokyo?"
    ↓
LLM thinks: "I need the getCurrentDateTime tool with timezone=Asia/Tokyo"
    ↓
LLM responds with: tool_call { name: "getCurrentDateTime", args: { timezone: "Asia/Tokyo" } }
    ↓
Your code executes getCurrentDateTime({ timezone: "Asia/Tokyo" })
    ↓
Your code sends the result back: "2026-04-10T22:30:00+09:00"
    ↓
LLM generates final answer: "It's 10:30 PM in Tokyo right now."
```

**What it is NOT:**

- It's NOT the LLM running code on a server
- It's NOT just structured output (though it uses the same schema mechanism)
- It's NOT a hack or prompt trick -- it's a first-class protocol supported by all major providers

### 1.2 Tool Calling vs Structured Output

In Chapter 5, you used Zod schemas to force the LLM to respond in a specific JSON format. Tool calling uses the exact same schema mechanism, but the intent is different:

| | Structured Output (Ch 5) | Tool Calling (Ch 8) |
|--|-------------------------|---------------------|
| **Purpose** | Format the response | Take an action |
| **Who decides?** | You decide the format | The LLM decides which action |
| **Schema describes** | The output shape | The function parameters |
| **What happens next** | You parse the response | You execute the function |
| **LLM generates** | The data | The function call |

The schema is the same. The meaning is different. In structured output, you say "respond in this format." In tool calling, you say "here are functions you *can* call -- decide if you need any of them."

### 1.3 How Providers Implement It

Every major provider supports tool calling, but the wire format differs slightly. Here's the good news: if you use the Vercel AI SDK (which we will), the interface is unified. But it helps to know what's underneath.

```typescript
// Anthropic's native format (what goes over the wire)
// The LLM responds with a content block of type "tool_use":
{
  type: "tool_use",
  id: "toolu_01A09q90qw90lq917835lq9",
  name: "getCurrentDateTime",
  input: { timezone: "Asia/Tokyo" }
}

// OpenAI's native format
// The LLM responds with a tool_calls array:
{
  tool_calls: [{
    id: "call_abc123",
    type: "function",
    function: {
      name: "getCurrentDateTime",
      arguments: '{"timezone": "Asia/Tokyo"}'  // Note: stringified JSON
    }
  }]
}
```

Notice that OpenAI stringifies the arguments (you have to `JSON.parse` them), while Anthropic gives you a parsed object. These differences are why using an SDK that normalizes them is valuable.

---

## 2. Defining Tools

### 2.1 The Three Parts of a Tool

Every tool has exactly three components:

1. **Description** -- tells the LLM what the tool does and when to use it
2. **Parameters** -- a Zod schema defining what arguments the tool accepts
3. **Execute** -- the function that runs when the tool is called

```typescript
import { z } from "zod";
import { tool } from "ai"; // Vercel AI SDK

const getCurrentDateTime = tool({
  description: "Get the current date and time in a specific timezone",
  parameters: z.object({
    timezone: z
      .string()
      .describe("IANA timezone identifier, e.g. 'America/New_York', 'Europe/London'"),
  }),
  execute: async ({ timezone }) => {
    const now = new Date();
    const formatted = now.toLocaleString("en-US", { timeZone: timezone });
    return { datetime: formatted, timezone };
  },
});
```

That's the complete pattern. Let's break each part down.

### 2.2 Descriptions: The Most Underestimated Part

The description is how the LLM decides whether to call your tool. A bad description means the LLM will call your tool at the wrong time, or fail to call it when it should. This is not a documentation string for humans -- it's an instruction for an LLM.

**Bad descriptions:**

```typescript
// Too vague -- when would the LLM use this?
description: "Searches things"

// Too technical -- the LLM doesn't care about implementation
description: "Executes a PostgreSQL query against the search_index table using ts_vector"

// Missing context about WHEN to use it
description: "Returns search results"
```

**Good descriptions:**

```typescript
// Clear what it does AND when to use it
description: "Search the web for current information. Use this when the user asks about recent events, current prices, live data, or anything that might have changed after your training data cutoff."

// Specific about what it returns
description: "Read a file from the local filesystem and return its contents. Use for reading code, config files, or documents. Returns the file content as a string. Fails if the file does not exist."

// Explains trade-offs between similar tools
description: "Search the company knowledge base for internal documentation. Use this instead of web search when the user asks about company policies, internal processes, or proprietary systems."
```

**The rule of thumb:** Write the description as if you're explaining to a smart colleague when they should use this tool vs other available tools.

### 2.3 Parameters: Zod Schemas with `.describe()`

This is where Chapter 5 pays off directly. Tool parameters are Zod schemas -- the same schemas you used for structured output. The critical addition is `.describe()` on each field, because the LLM needs to know what values to pass.

```typescript
const searchWeb = tool({
  description:
    "Search the web for current information. Use when the user needs up-to-date facts, prices, news, or data not available in your training data.",
  parameters: z.object({
    query: z
      .string()
      .describe("The search query. Be specific and include relevant keywords."),
    maxResults: z
      .number()
      .min(1)
      .max(10)
      .default(5)
      .describe("Number of results to return. Default is 5."),
    recency: z
      .enum(["day", "week", "month", "year", "any"])
      .default("any")
      .describe(
        "Filter results by recency. Use 'day' for breaking news, 'week' for recent events."
      ),
  }),
  execute: async ({ query, maxResults, recency }) => {
    // We'll implement this for real in a moment
    return { results: [] };
  },
});
```

**Key patterns for parameter schemas:**

```typescript
// Optional parameters with defaults
z.number().default(5).describe("How many results to return")

// Enums to constrain choices
z.enum(["high", "medium", "low"]).describe("Priority level")

// Nested objects for complex inputs
z.object({
  lat: z.number().describe("Latitude"),
  lng: z.number().describe("Longitude"),
}).describe("Geographic coordinates")

// Arrays
z.array(z.string()).describe("List of tags to filter by")
```

**What NOT to put in parameters:**

```typescript
// Don't make internal implementation details a parameter
z.object({
  apiKey: z.string(), // NO -- the LLM shouldn't be passing API keys
  databaseUrl: z.string(), // NO -- infrastructure config
  retryCount: z.number(), // NO -- internal retry logic
})
```

Parameters should represent the *user's intent*, not your system's internals. API keys, database URLs, and configuration belong in the execute function's closure, not in the schema.

### 2.4 The Execute Function

The execute function is where the actual work happens. It receives the validated parameters and returns a result that gets sent back to the LLM.

```typescript
execute: async ({ timezone }) => {
  // This runs in YOUR code, on YOUR server
  // You have access to your database, APIs, filesystem, etc.
  const now = new Date();
  const formatted = now.toLocaleString("en-US", { timeZone: timezone });
  return { datetime: formatted, timezone, utcOffset: getUtcOffset(timezone) };
}
```

**Important principles:**

1. **Always return structured data**, not prose. The LLM will format it for the user.
2. **Handle errors gracefully** -- return error info, don't throw (we'll cover this in detail).
3. **Keep it focused** -- one tool, one job. Don't make a tool that does 5 things.
4. **Don't return too much data** -- if you're searching a database, limit results. Every byte goes back into the context window.

---

## 3. Building Real Tools

Let's build three real tools that an agent would actually use.

### 3.1 Tool: getCurrentDateTime

The simplest useful tool. LLMs don't know the current date -- this fixes that.

```typescript
import { z } from "zod";
import { tool } from "ai";

const getCurrentDateTime = tool({
  description:
    "Get the current date and time. Use this whenever the user asks about today's date, the current time, or needs temporal context for their request.",
  parameters: z.object({
    timezone: z
      .string()
      .default("UTC")
      .describe(
        "IANA timezone identifier like 'America/New_York' or 'Europe/London'. Defaults to UTC."
      ),
    format: z
      .enum(["iso", "human", "date-only", "time-only"])
      .default("human")
      .describe("Output format. 'human' for readable, 'iso' for ISO 8601."),
  }),
  execute: async ({ timezone, format }) => {
    const now = new Date();

    switch (format) {
      case "iso":
        return {
          datetime: now.toISOString(),
          timezone: "UTC",
        };
      case "human":
        return {
          datetime: now.toLocaleString("en-US", {
            timeZone: timezone,
            weekday: "long",
            year: "numeric",
            month: "long",
            day: "numeric",
            hour: "2-digit",
            minute: "2-digit",
            second: "2-digit",
          }),
          timezone,
        };
      case "date-only":
        return {
          date: now.toLocaleDateString("en-US", {
            timeZone: timezone,
            weekday: "long",
            year: "numeric",
            month: "long",
            day: "numeric",
          }),
          timezone,
        };
      case "time-only":
        return {
          time: now.toLocaleTimeString("en-US", {
            timeZone: timezone,
            hour: "2-digit",
            minute: "2-digit",
            second: "2-digit",
          }),
          timezone,
        };
    }
  },
});
```

### 3.2 Tool: fileRead

Read files from the filesystem. This is what makes coding agents possible.

```typescript
import { readFile, stat } from "fs/promises";
import { extname, resolve } from "path";

const fileRead = tool({
  description:
    "Read the contents of a file from the local filesystem. Use this when you need to examine code, configuration files, documentation, or any text file. Returns the file contents as a string. Will fail if the file does not exist or is not a text file.",
  parameters: z.object({
    path: z
      .string()
      .describe(
        "Absolute or relative file path to read. Examples: './src/index.ts', '/etc/config.json'"
      ),
    maxLines: z
      .number()
      .optional()
      .describe(
        "Maximum number of lines to return. Omit to read the entire file. Use for large files to avoid overwhelming context."
      ),
  }),
  execute: async ({ path: filePath, maxLines }) => {
    try {
      const absolutePath = resolve(filePath);
      const fileStat = await stat(absolutePath);

      // Safety check: don't read huge files
      const MAX_FILE_SIZE = 1024 * 1024; // 1MB
      if (fileStat.size > MAX_FILE_SIZE) {
        return {
          error: `File is too large (${(fileStat.size / 1024).toFixed(0)}KB). Maximum is 1MB. Use maxLines to read a portion.`,
          path: absolutePath,
        };
      }

      const content = await readFile(absolutePath, "utf-8");
      const extension = extname(absolutePath);

      if (maxLines) {
        const lines = content.split("\n");
        const truncated = lines.slice(0, maxLines).join("\n");
        return {
          content: truncated,
          path: absolutePath,
          extension,
          totalLines: lines.length,
          returnedLines: Math.min(maxLines, lines.length),
          truncated: lines.length > maxLines,
        };
      }

      return {
        content,
        path: absolutePath,
        extension,
        totalLines: content.split("\n").length,
      };
    } catch (err) {
      const error = err as NodeJS.ErrnoException;
      if (error.code === "ENOENT") {
        return { error: `File not found: ${filePath}`, path: filePath };
      }
      if (error.code === "EACCES") {
        return { error: `Permission denied: ${filePath}`, path: filePath };
      }
      return { error: `Failed to read file: ${error.message}`, path: filePath };
    }
  },
});
```

Notice the error handling pattern: we return error information as data rather than throwing exceptions. The LLM can read the error and decide what to do -- maybe try a different path, or tell the user the file doesn't exist. If we threw, the agent loop would have to catch it and figure out how to present it. Returning errors as data keeps the LLM in the driver's seat.

### 3.3 Tool: webSearch

A practical web search tool using a search API.

```typescript
const webSearch = tool({
  description:
    "Search the web for current information. Use this when the user asks about recent events, current data, live prices, news, or anything that might have changed after your training cutoff. Also use when you need to verify facts or find specific URLs.",
  parameters: z.object({
    query: z
      .string()
      .describe(
        "Search query. Be specific -- include names, dates, and context for better results."
      ),
    maxResults: z
      .number()
      .min(1)
      .max(10)
      .default(5)
      .describe("Number of results to return."),
  }),
  execute: async ({ query, maxResults }) => {
    try {
      // Using Brave Search API as an example
      // In production, you'd use whichever search provider you prefer
      const response = await fetch(
        `https://api.search.brave.com/res/v1/web/search?q=${encodeURIComponent(query)}&count=${maxResults}`,
        {
          headers: {
            "X-Subscription-Token": process.env.BRAVE_SEARCH_API_KEY!,
            Accept: "application/json",
          },
        }
      );

      if (!response.ok) {
        return {
          error: `Search failed with status ${response.status}`,
          query,
        };
      }

      const data = await response.json();

      // Extract just what the LLM needs -- don't dump the entire API response
      const results = data.web?.results?.map(
        (r: { title: string; url: string; description: string }) => ({
          title: r.title,
          url: r.url,
          snippet: r.description,
        })
      ) ?? [];

      return {
        query,
        resultCount: results.length,
        results,
      };
    } catch (err) {
      return {
        error: `Search failed: ${(err as Error).message}`,
        query,
      };
    }
  },
});
```

**Key design decision:** We return `title`, `url`, and `snippet` -- not the entire API response. Every token returned from a tool goes into the context window. If you dump raw API responses with metadata, tracking IDs, and pagination tokens, you're wasting context budget on data the LLM doesn't need.

---

## 4. Using Tools with the Vercel AI SDK

### 4.1 The `generateText` Approach

The simplest way to use tools. You pass them to `generateText` and the SDK handles the tool call protocol.

```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

async function askWithTools(userMessage: string) {
  const result = await generateText({
    model: anthropic("claude-sonnet-4-20250514"),
    tools: {
      getCurrentDateTime,
      fileRead,
      webSearch,
    },
    prompt: userMessage,
  });

  // The result contains the final text AND any tool calls that were made
  console.log("Response:", result.text);
  console.log("Tool calls:", result.toolCalls);
  console.log("Tool results:", result.toolResults);

  return result;
}

// Usage
await askWithTools("What time is it in Tokyo?");
// LLM will call getCurrentDateTime({ timezone: "Asia/Tokyo" })
// Then generate a response using the result

await askWithTools("Read the package.json file");
// LLM will call fileRead({ path: "./package.json" })
// Then summarize the contents

await askWithTools("What's the weather like today?");
// Might call webSearch({ query: "weather today" })
// Or might just say "I don't have a weather tool"
```

### 4.2 maxSteps: Automatic Multi-Step Tool Use

By default, `generateText` makes one LLM call. If the LLM wants to call a tool, you get back the tool call -- but not the LLM's response *after* seeing the tool result. The `maxSteps` option fixes this by allowing the SDK to loop automatically:

```typescript
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime, fileRead, webSearch },
  maxSteps: 5, // Allow up to 5 LLM calls in sequence
  prompt: "What files are in the src directory, and what time were they last modified?",
});

// With maxSteps, the flow is:
// Step 1: LLM calls fileRead({ path: "./src" }) -- oops, that's a directory
// Step 2: LLM gets error, tries a different approach
// Step 3: LLM calls webSearch or adjusts strategy
// ...until it has a final answer or hits maxSteps
```

**When to use `maxSteps`:**

- Simple tool use (1-2 steps): `maxSteps: 3`
- Multi-tool tasks: `maxSteps: 5`
- Complex reasoning chains: `maxSteps: 10`

**What `maxSteps` is NOT:** It's not a full agent loop. It doesn't have memory management, context compaction, or human approval gates. For that, you need Chapter 9. But for simple "call a tool and respond" workflows, `maxSteps` is all you need.

### 4.3 The `streamText` Approach

Same idea, but streaming. The LLM streams its text response, and tool calls appear as events in the stream.

```typescript
import { streamText } from "ai";

async function streamWithTools(userMessage: string) {
  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    tools: { getCurrentDateTime, fileRead, webSearch },
    maxSteps: 5,
    prompt: userMessage,
  });

  // Stream text as it arrives
  for await (const chunk of result.textStream) {
    process.stdout.write(chunk);
  }

  // Or use the full stream for tool call events
  for await (const event of result.fullStream) {
    switch (event.type) {
      case "text-delta":
        process.stdout.write(event.textDelta);
        break;
      case "tool-call":
        console.log(`\n[Calling ${event.toolName}(${JSON.stringify(event.args)})]`);
        break;
      case "tool-result":
        console.log(`[Got result from ${event.toolName}]`);
        break;
    }
  }
}
```

### 4.4 Tool Choice: Controlling When Tools Are Used

Sometimes you want to force the LLM to use a specific tool, or prevent it from using tools at all.

```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

// Auto (default): LLM decides whether to use tools
const auto = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime, webSearch },
  toolChoice: "auto",
  prompt: "What time is it?",
});

// Required: LLM MUST use at least one tool
const required = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime, webSearch },
  toolChoice: "required",
  prompt: "Tell me something interesting",
});

// None: LLM cannot use tools (useful for follow-up messages)
const none = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime, webSearch },
  toolChoice: "none",
  prompt: "Based on the search results above, summarize the findings",
});

// Specific tool: force a particular tool
const specific = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime, webSearch },
  toolChoice: { type: "tool", toolName: "webSearch" },
  prompt: "Anthropic",
});
```

**When to use each:**
- `auto` -- most of the time. Let the LLM decide.
- `required` -- when you know the user's request needs an action (e.g., "search for X").
- `none` -- when you want the LLM to reason over existing tool results without calling more tools.
- Specific tool -- when you've already determined which tool is needed and just need the LLM to figure out the arguments.

---

## 5. How LLMs Decide Which Tool to Call

This is the most important section of this chapter. Understanding how the LLM selects tools lets you debug when it picks wrong.

### 5.1 The Selection Process

When you provide tools, the LLM receives them as part of its system context. It sees:
1. The tool name
2. The description
3. The parameter schema (including `.describe()` text on each field)

The LLM then decides: "Given the user's message and the available tools, should I call a tool? If so, which one? With what arguments?"

This is a *classification* task followed by a *generation* task. The LLM is classifying the user's intent against available tools, then generating the arguments.

### 5.2 Description Quality Directly Affects Selection

```typescript
// EXPERIMENT: Same tool, different descriptions

// Description A (vague)
const searchA = tool({
  description: "Search for things",
  parameters: z.object({ query: z.string() }),
  execute: async ({ query }) => ({ results: [] }),
});

// Description B (specific, with usage guidance)
const searchB = tool({
  description:
    "Search the company's internal knowledge base for documentation, policies, runbooks, and SOPs. Use this when the user asks about company-specific processes, internal tools, or organizational information. Do NOT use for general web searches.",
  parameters: z.object({
    query: z.string().describe("Search query focused on internal documentation topics"),
  }),
  execute: async ({ query }) => ({ results: [] }),
});
```

With Description A, the LLM might call `search` for any question. With Description B, it knows exactly when to use it and when not to. The description is your strongest lever for controlling tool selection.

### 5.3 When the LLM Gets It Wrong

Common failure modes and fixes:

**Problem: LLM calls a tool when it shouldn't.**
```typescript
// User: "Tell me a joke about databases"
// LLM calls: webSearch({ query: "jokes about databases" })
// Fix: Add negative guidance to the description
description: "Search the web for factual, current information. Do NOT use for creative tasks like jokes, stories, or brainstorming -- handle those from your own knowledge."
```

**Problem: LLM doesn't call a tool when it should.**
```typescript
// User: "What's the current Bitcoin price?"
// LLM responds with outdated training data
// Fix: Emphasize recency in the description
description: "Search the web for current, real-time information. ALWAYS use this for: current prices, live scores, today's news, recent events, anything time-sensitive."
```

**Problem: LLM calls the wrong tool when multiple are available.**
```typescript
// Fix: Differentiate tools clearly in their descriptions
const webSearch = tool({
  description: "Search the PUBLIC web. Use for general knowledge, news, external information.",
  // ...
});

const internalSearch = tool({
  description: "Search INTERNAL company docs. Use for policies, procedures, internal tools. NOT for general knowledge.",
  // ...
});
```

### 5.4 The Description Engineering Checklist

For every tool you write, check:

1. **What does it do?** (one sentence)
2. **When should the LLM use it?** (positive examples)
3. **When should the LLM NOT use it?** (negative examples, especially if you have similar tools)
4. **What does it return?** (so the LLM knows what to expect)
5. **What are the failure modes?** (so the LLM can handle errors)

---

## 6. Multiple Tool Calls in a Single Response

Modern LLMs can request multiple tool calls in a single response. This is important for efficiency -- instead of calling tools one at a time, the LLM can "batch" independent calls.

### 6.1 How Parallel Tool Calls Work

```typescript
// User: "What time is it in Tokyo and New York?"
// The LLM might generate TWO tool calls in one response:
// 1. getCurrentDateTime({ timezone: "Asia/Tokyo" })
// 2. getCurrentDateTime({ timezone: "America/New_York" })

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { getCurrentDateTime },
  maxSteps: 3,
  prompt: "What time is it in Tokyo and New York?",
});

// result.toolCalls might contain both calls
// The SDK executes them in parallel and sends both results back
console.log(result.toolCalls);
// [
//   { toolName: "getCurrentDateTime", args: { timezone: "Asia/Tokyo" } },
//   { toolName: "getCurrentDateTime", args: { timezone: "America/New_York" } }
// ]
```

### 6.2 Handling Multiple Tool Results

When using `maxSteps`, the SDK handles this automatically. But when building a manual agent loop (Chapter 9), you need to handle it yourself:

```typescript
// The manual way (preview of Chapter 9's agent loop)
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function handleToolCalls(response: Anthropic.Message) {
  const toolUseBlocks = response.content.filter(
    (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
  );

  // Execute ALL tool calls (they might be parallelizable)
  const toolResults = await Promise.all(
    toolUseBlocks.map(async (toolCall) => {
      const result = await executeToolByName(toolCall.name, toolCall.input);
      return {
        type: "tool_result" as const,
        tool_use_id: toolCall.id,
        content: JSON.stringify(result),
      };
    })
  );

  return toolResults;
}

async function executeToolByName(name: string, input: unknown): Promise<unknown> {
  switch (name) {
    case "getCurrentDateTime":
      return getCurrentDateTime.execute(input as { timezone: string; format: string });
    case "fileRead":
      return fileRead.execute(input as { path: string; maxLines?: number });
    case "webSearch":
      return webSearch.execute(input as { query: string; maxResults: number });
    default:
      return { error: `Unknown tool: ${name}` };
  }
}
```

---

## 7. Error Handling in Tools

Tools fail. APIs go down, files don't exist, queries return no results. How you handle errors determines whether your agent is robust or fragile.

### 7.1 The "Errors as Data" Pattern

**Don't throw exceptions from tools. Return errors as structured data.**

```typescript
const fileRead = tool({
  description: "Read a file from the filesystem",
  parameters: z.object({
    path: z.string().describe("File path to read"),
  }),
  execute: async ({ path: filePath }) => {
    try {
      const content = await readFile(resolve(filePath), "utf-8");
      return { success: true, content, path: filePath };
    } catch (err) {
      const error = err as NodeJS.ErrnoException;
      // Return the error as data -- the LLM can decide what to do
      return {
        success: false,
        error: error.code === "ENOENT"
          ? `File not found: ${filePath}. Check the path and try again.`
          : `Could not read file: ${error.message}`,
        path: filePath,
        suggestion: error.code === "ENOENT"
          ? "Try listing the directory first to find the correct filename."
          : undefined,
      };
    }
  },
});
```

When the LLM sees `{ success: false, error: "File not found", suggestion: "Try listing the directory" }`, it can intelligently retry, adjust its approach, or explain the problem to the user. If you threw an exception, the LLM wouldn't get any of that context.

### 7.2 Timeout Protection

Tools that call external APIs should have timeouts:

```typescript
const webSearch = tool({
  description: "Search the web",
  parameters: z.object({ query: z.string() }),
  execute: async ({ query }) => {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10_000); // 10s timeout

    try {
      const response = await fetch(
        `https://api.search.brave.com/res/v1/web/search?q=${encodeURIComponent(query)}`,
        {
          headers: { "X-Subscription-Token": process.env.BRAVE_SEARCH_API_KEY! },
          signal: controller.signal,
        }
      );
      clearTimeout(timeout);

      if (!response.ok) {
        return { error: `Search returned ${response.status}`, query };
      }

      const data = await response.json();
      return { results: data.web?.results?.slice(0, 5) ?? [] };
    } catch (err) {
      clearTimeout(timeout);
      if ((err as Error).name === "AbortError") {
        return { error: "Search timed out after 10 seconds", query };
      }
      return { error: `Search failed: ${(err as Error).message}`, query };
    }
  },
});
```

### 7.3 Input Validation Beyond Zod

Zod validates the *shape* of the input, but you might need additional business logic validation:

```typescript
const fileRead = tool({
  description: "Read a file from the filesystem",
  parameters: z.object({
    path: z.string().describe("File path to read"),
  }),
  execute: async ({ path: filePath }) => {
    // Security: prevent reading outside allowed directories
    const resolved = resolve(filePath);
    const allowedRoot = resolve("./workspace");

    if (!resolved.startsWith(allowedRoot)) {
      return {
        error: "Access denied: can only read files within the workspace directory",
        path: filePath,
      };
    }

    // Proceed with read...
    const content = await readFile(resolved, "utf-8");
    return { content, path: resolved };
  },
});
```

---

## 8. A Complete Example: Multi-Tool Assistant

Let's put it all together. A complete assistant with three tools that handles a real conversation.

```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";
import { tool } from "ai";
import { readFile, stat } from "fs/promises";
import { resolve, extname } from "path";

// --- Tool Definitions ---

const getCurrentDateTime = tool({
  description:
    "Get the current date and time. Use when the user asks about today's date, current time, or needs temporal context.",
  parameters: z.object({
    timezone: z
      .string()
      .default("UTC")
      .describe("IANA timezone like 'America/New_York'. Defaults to UTC."),
  }),
  execute: async ({ timezone }) => {
    const now = new Date();
    return {
      datetime: now.toLocaleString("en-US", {
        timeZone: timezone,
        dateStyle: "full",
        timeStyle: "long",
      }),
      timezone,
      iso: now.toISOString(),
    };
  },
});

const readLocalFile = tool({
  description:
    "Read a file from the local filesystem. Use for examining code, configs, docs, or any text file. Returns content as string.",
  parameters: z.object({
    path: z.string().describe("File path (relative or absolute)"),
    maxLines: z
      .number()
      .optional()
      .describe("Max lines to return. Omit for full file."),
  }),
  execute: async ({ path: filePath, maxLines }) => {
    try {
      const abs = resolve(filePath);
      const content = await readFile(abs, "utf-8");
      const lines = content.split("\n");

      if (maxLines && lines.length > maxLines) {
        return {
          content: lines.slice(0, maxLines).join("\n"),
          totalLines: lines.length,
          truncated: true,
          path: abs,
        };
      }

      return { content, totalLines: lines.length, path: abs };
    } catch (err) {
      return { error: `Failed to read: ${(err as Error).message}`, path: filePath };
    }
  },
});

const calculate = tool({
  description:
    "Evaluate a mathematical expression. Use for arithmetic, unit conversions, percentages, or any calculation the user needs. Supports standard math operations.",
  parameters: z.object({
    expression: z
      .string()
      .describe(
        "Math expression to evaluate, e.g. '(15 * 1.08) + 5' or '1024 / 8'"
      ),
  }),
  execute: async ({ expression }) => {
    try {
      // Simple safe math evaluation (in production, use a proper math library)
      // This is intentionally restrictive -- only allows numbers and math operators
      const sanitized = expression.replace(/[^0-9+\-*/().%\s]/g, "");
      if (sanitized !== expression) {
        return { error: "Expression contains invalid characters", expression };
      }
      const result = Function(`"use strict"; return (${sanitized})`)();
      return {
        expression,
        result: Number(result),
        formatted: Number(result).toLocaleString(),
      };
    } catch {
      return { error: "Could not evaluate expression", expression };
    }
  },
});

// --- The Assistant ---

async function chat(userMessage: string) {
  const result = await generateText({
    model: anthropic("claude-sonnet-4-20250514"),
    system:
      "You are a helpful assistant with access to tools. Use them when needed. Be concise but thorough.",
    tools: {
      getCurrentDateTime,
      readLocalFile,
      calculate,
    },
    maxSteps: 5,
    prompt: userMessage,
  });

  return {
    response: result.text,
    toolsUsed: result.toolCalls.map((tc) => tc.toolName),
    steps: result.steps.length,
  };
}

// --- Example Usage ---

async function main() {
  // Example 1: Uses getCurrentDateTime
  const r1 = await chat("What's today's date?");
  console.log(r1.response);
  // "Today is Thursday, April 10, 2026."

  // Example 2: Uses calculate
  const r2 = await chat("If I have 15 items at $8.50 each with 8% tax, what's the total?");
  console.log(r2.response);
  // LLM calls calculate({ expression: "(15 * 8.50) * 1.08" })
  // "The total would be $137.70"

  // Example 3: Uses readLocalFile
  const r3 = await chat("What dependencies does this project use?");
  console.log(r3.response);
  // LLM calls readLocalFile({ path: "./package.json" })
  // Then summarizes the dependencies

  // Example 4: No tool needed
  const r4 = await chat("Explain what recursion is");
  console.log(r4.response);
  // LLM just answers from its own knowledge -- no tool call
}

main();
```

---

## 9. Common Pitfalls

### 9.1 Too Many Tools

If you give the LLM 50 tools, it gets confused. Each tool definition takes up context space, and the classification task gets harder with more options.

**Rule of thumb:**
- 1-5 tools: works great
- 5-15 tools: works well if descriptions are clear and differentiated
- 15-30 tools: need very precise descriptions, consider grouping
- 30+: consider routing to sub-agents with focused tool sets (Chapter 12)

### 9.2 Tools That Return Too Much Data

```typescript
// BAD: dumps entire database response
execute: async ({ query }) => {
  const results = await db.query(query);
  return results; // Could be 100KB of JSON
}

// GOOD: returns only what the LLM needs
execute: async ({ query }) => {
  const results = await db.query(query);
  return {
    count: results.length,
    results: results.slice(0, 10).map(r => ({
      id: r.id,
      title: r.title,
      summary: r.summary?.slice(0, 200),
    })),
    hasMore: results.length > 10,
  };
}
```

Every byte of tool output goes into the context window. If your tool returns 50KB of JSON, that's thousands of tokens consumed.

### 9.3 Non-Deterministic Tool Behavior

Tools should be as predictable as possible. If the same input produces wildly different output each time, the LLM can't reason effectively about the results.

```typescript
// BAD: returns random results
execute: async ({ query }) => {
  return randomSample(allResults, 5); // Different every time
}

// GOOD: deterministic ordering
execute: async ({ query }) => {
  return allResults
    .sort((a, b) => b.relevanceScore - a.relevanceScore)
    .slice(0, 5);
}
```

### 9.4 Missing `.describe()` on Parameters

```typescript
// BAD: the LLM has to guess what "q" means
parameters: z.object({
  q: z.string(),
  n: z.number(),
})

// GOOD: clear intent
parameters: z.object({
  query: z.string().describe("Search query with relevant keywords"),
  maxResults: z.number().describe("Number of results to return (1-10)"),
})
```

Both the field name and the `.describe()` text help the LLM figure out what to pass. Don't make it guess.

---

## 10. Tools Without Execute: Client-Side Tools

Sometimes you want the LLM to decide to call a tool, but you want to execute it on the *client* side (like in a browser). You can define tools without an `execute` function:

```typescript
const showWeatherWidget = tool({
  description:
    "Display a weather widget in the UI for a given location. Use when the user asks about weather.",
  parameters: z.object({
    location: z.string().describe("City name or coordinates"),
    unit: z.enum(["celsius", "fahrenheit"]).default("fahrenheit"),
  }),
  // No execute function -- the client handles this
});

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  tools: { showWeatherWidget },
  prompt: "What's the weather in Paris?",
});

// result.toolCalls contains the call, but it wasn't executed
// Your frontend code picks it up and renders a weather widget
for (const call of result.toolCalls) {
  if (call.toolName === "showWeatherWidget") {
    renderWeatherWidget(call.args); // Client-side rendering
  }
}
```

This pattern is how AI chatbots render rich UI components -- the LLM decides *what* to show, and the client decides *how* to show it.

---

## 11. What's Next

You now know how to give an LLM hands. It can read files, search the web, check the time, and calculate. But right now it can only do one round of tool use -- call a tool, get the result, respond.

In Chapter 9, we'll build the **agent loop**: the pattern where the LLM can call tools *repeatedly*, using each result to decide what to do next, until the task is complete. The tools you defined here become the building blocks of autonomous behavior.

The leap from "an LLM that can call one tool" to "an LLM that loops until the job is done" is the leap from a chatbot to an agent. Let's make that leap.

---

## Key Takeaways

1. **Tool calling lets LLMs take actions**, not just generate text. The LLM chooses the tool and arguments; your code executes.
2. **Every tool has three parts**: description, parameters (Zod schema), and execute function.
3. **Description quality is the #1 factor** in whether the LLM picks the right tool. Write descriptions for the LLM, not for humans.
4. **Return errors as data**, not exceptions. Let the LLM decide how to handle failures.
5. **Keep tool output lean.** Every byte goes into the context window.
6. **Use `maxSteps`** for simple multi-step tool use. For full agent behavior, build the loop (Chapter 9).

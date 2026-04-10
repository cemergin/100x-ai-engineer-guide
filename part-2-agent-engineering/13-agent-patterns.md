<!--
  CHAPTER: 13
  TITLE: Agent Patterns & Frameworks
  PART: 2 — Agent Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 8-12 (entire agent engineering sequence)
  KEY_TOPICS: ReAct pattern, Plan-and-Execute, Reflexion, framework comparison, LangChain, Mastra, Vercel AI SDK, Claude Agent SDK, OpenAI Agents SDK, when to use frameworks
  DIFFICULTY: Intermediate → Advanced
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 13: Agent Patterns & Frameworks

> **Part 2 — Agent Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 8-12 | Difficulty: Intermediate → Advanced

You just built an agent from scratch. Over five chapters, you implemented tool calling (Ch 8), the agent loop (Ch 9), three layers of memory (Ch 10), human-in-the-loop approvals (Ch 11), and context window management (Ch 12). You wrote all of it yourself, in TypeScript, with no frameworks.

Now let's zoom out. There are named patterns for what you built -- ReAct, Plan-and-Execute, Reflexion -- and there are frameworks that package these patterns into libraries. Some of these frameworks are excellent. Some will slow you down. The goal of this chapter is to give you enough understanding of both the patterns and the frameworks to make a clear-eyed decision: should you use a framework, or should you build on the primitives you already have?

Here's my opinion up front: **start with the Vercel AI SDK or the Anthropic SDK directly**, because they give you the right level of abstraction -- they handle the API protocol (tool call formats, streaming, message types) without imposing an agent architecture. Use higher-level frameworks when you need specific features they provide (like LangChain's document loaders or Mastra's workflow engine) and you understand what they're doing under the hood. The worst position is using a framework you don't understand, because when it breaks -- and it will -- you won't know where to look.

### In This Chapter
- Named agent patterns: ReAct, Plan-and-Execute, Reflexion
- How Chapters 8-12 map to each pattern
- Framework landscape: LangChain, Mastra, Vercel AI SDK, Claude Agent SDK, OpenAI Agents SDK
- What each framework gives you for free
- The framework decision tree
- When to build from scratch vs use a framework
- Composing patterns: real agents use multiple patterns

### Related Chapters
- **Ch 8-12** — you built these concepts; now see how frameworks implement them
- **Ch 26 (AI-Augmented Development)** — coding agents use these patterns at scale
- **Ch 27 (Harness Architecture)** — production agent architectures beyond frameworks
- **Ch 32 (Multi-Agent Coordination)** — when one agent isn't enough
- **Ch 58 (Multi-Agent Orchestration)** — enterprise-scale agent coordination

---

## 1. Named Agent Patterns

Before frameworks, there are patterns. These are the recurring architectural approaches that show up in agent systems. You've already built pieces of each.

### 1.1 ReAct: Reason + Act

**What it is:** The agent alternates between reasoning (thinking about what to do) and acting (calling tools). After each action, it observes the result and reasons about what to do next.

```
Think → Act → Observe → Think → Act → Observe → ... → Answer
```

**This is what you built in Chapter 9.** The agent loop IS the ReAct pattern:

```typescript
// ReAct in its purest form:
while (true) {
  // REASON: LLM decides what to do (this happens inside the model)
  const response = await llm.call(messages);

  // ACT: Execute the tool call
  if (response.hasToolCall) {
    const result = await executeTool(response.toolCall);

    // OBSERVE: Add the result to context
    messages.push(assistantMessage(response));
    messages.push(toolResultMessage(result));

    // Loop back to REASON
    continue;
  }

  // DONE: No more tool calls, return the answer
  return response.text;
}
```

**When to use ReAct:**
- General-purpose agents that need to explore and respond
- Tasks where the approach isn't known upfront
- Interactive agents where the human might redirect mid-task

**Limitations:**
- No explicit planning -- the agent figures it out one step at a time
- Can get stuck in loops (tries the same failing approach)
- Each step only considers what to do *next*, not the overall strategy

### 1.2 Plan-and-Execute

**What it is:** The agent first creates a plan (a list of steps), then executes each step. The plan can be revised if a step fails.

```
Plan → Execute Step 1 → Execute Step 2 → ... → Revise Plan (if needed) → ... → Done
```

**You built the planning piece in Chapter 10** with working memory. The agent's `setPlan` tool creates an explicit plan:

```typescript
// Plan-and-Execute pattern:
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function planAndExecute(task: string): Promise<string> {
  // Phase 1: PLAN
  const planResponse = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: `You are a planning agent. Given a task, create a step-by-step plan.
Output ONLY a JSON array of steps, no explanation.
Example: ["Read package.json", "Identify dependencies", "Check for vulnerabilities"]`,
    messages: [{ role: "user", content: `Plan this task: ${task}` }],
  });

  const planText = planResponse.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");

  let plan: string[];
  try {
    plan = JSON.parse(planText);
  } catch {
    plan = [task]; // Fallback: single-step plan
  }

  console.log("Plan:", plan);

  // Phase 2: EXECUTE each step
  const results: string[] = [];

  for (let i = 0; i < plan.length; i++) {
    console.log(`\nStep ${i + 1}/${plan.length}: ${plan[i]}`);

    const stepResult = await executeStep(plan[i], results);
    results.push(stepResult);

    // Phase 3: REVISE plan if needed
    if (stepResult.includes("ERROR") || stepResult.includes("BLOCKED")) {
      console.log("  Step failed. Revising plan...");
      const revisedPlan = await revisePlan(plan, i, results);
      plan.splice(i + 1, plan.length, ...revisedPlan);
      console.log("  Revised remaining steps:", revisedPlan);
    }
  }

  // Phase 4: SYNTHESIZE results
  return synthesizeResults(task, plan, results);
}

async function executeStep(
  step: string,
  previousResults: string[]
): Promise<string> {
  const context = previousResults.length > 0
    ? `\nPrevious step results:\n${previousResults.map((r, i) => `Step ${i + 1}: ${r}`).join("\n")}`
    : "";

  const result = await agentLoop(
    `Execute this step: ${step}${context}`,
    {
      systemPrompt: "You are executing a single step of a plan. Be focused and thorough.",
      maxIterations: 5,
    }
  );

  return result;
}

async function revisePlan(
  originalPlan: string[],
  failedStep: number,
  results: string[]
): Promise<string[]> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: "Revise the remaining plan steps based on what happened. Output a JSON array of new steps.",
    messages: [
      {
        role: "user",
        content: `Original plan: ${JSON.stringify(originalPlan)}
Failed at step ${failedStep + 1}: ${originalPlan[failedStep]}
Results so far: ${JSON.stringify(results)}
What should the remaining steps be?`,
      },
    ],
  });

  const text = response.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");

  try {
    return JSON.parse(text);
  } catch {
    return originalPlan.slice(failedStep + 1); // Keep original if revision fails
  }
}

async function synthesizeResults(
  task: string,
  plan: string[],
  results: string[]
): Promise<string> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    messages: [
      {
        role: "user",
        content: `Task: ${task}\nPlan: ${JSON.stringify(plan)}\nResults: ${JSON.stringify(results)}\n\nSynthesize a final answer.`,
      },
    ],
  });

  return response.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");
}
```

**When to use Plan-and-Execute:**
- Complex tasks with multiple clear phases
- Tasks where you can plan upfront (e.g., "migrate this codebase from JavaScript to TypeScript")
- Tasks where the user benefits from seeing a plan before execution starts

**Limitations:**
- Planning takes time and tokens (the plan itself is an LLM call)
- Plans can be wrong -- the planner doesn't have full information yet
- Rigid structure can't easily handle unexpected discoveries mid-execution

### 1.3 Reflexion: Self-Correcting Agents

**What it is:** After completing a task (or failing), the agent reflects on what went wrong and tries again with the lesson learned.

```
Attempt 1 → Evaluate → Reflect on mistakes → Attempt 2 (with reflection) → ... → Success
```

```typescript
interface ReflexionResult {
  success: boolean;
  output: string;
  reflection?: string;
  attempts: number;
}

async function reflexionAgent(
  task: string,
  evaluator: (output: string) => Promise<{ passed: boolean; feedback: string }>,
  maxAttempts: number = 3
): Promise<ReflexionResult> {
  let reflections: string[] = [];

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    console.log(`\nAttempt ${attempt}/${maxAttempts}`);

    // Build context with past reflections
    const reflectionContext = reflections.length > 0
      ? `\n\nLessons from previous attempts:\n${reflections.map((r, i) => `Attempt ${i + 1}: ${r}`).join("\n")}`
      : "";

    // Execute the task
    const output = await agentLoop(
      `${task}${reflectionContext}`,
      {
        systemPrompt: `You are an agent that learns from its mistakes. ${
          reflections.length > 0
            ? "Review the lessons from previous attempts and avoid the same mistakes."
            : ""
        }`,
        maxIterations: 10,
      }
    );

    // Evaluate the output
    const evaluation = await evaluator(output);

    if (evaluation.passed) {
      return {
        success: true,
        output,
        attempts: attempt,
      };
    }

    // Reflect on the failure
    console.log(`  Evaluation: FAILED - ${evaluation.feedback}`);

    const reflectionResponse = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 512,
      messages: [
        {
          role: "user",
          content: `You attempted a task and failed.
Task: ${task}
Your output: ${output}
Evaluation feedback: ${evaluation.feedback}

Reflect on what went wrong and what you should do differently next time. Be specific and actionable.`,
        },
      ],
    });

    const reflection = reflectionResponse.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");

    reflections.push(reflection);
    console.log(`  Reflection: ${reflection.slice(0, 200)}...`);
  }

  return {
    success: false,
    output: "Failed after maximum attempts",
    reflection: reflections[reflections.length - 1],
    attempts: maxAttempts,
  };
}

// Usage:
const result = await reflexionAgent(
  "Write a function that validates email addresses using regex",
  async (output) => {
    // Run the code through tests
    const testCases = [
      { input: "user@example.com", expected: true },
      { input: "invalid", expected: false },
      { input: "user@.com", expected: false },
      { input: "a@b.co", expected: true },
    ];

    // Simplified eval -- in practice, actually run the code
    const allPassed = testCases.every((tc) =>
      output.includes(tc.input) || output.includes("regex")
    );

    return {
      passed: allPassed,
      feedback: allPassed
        ? "All test cases pass"
        : "Some test cases fail. Check edge cases.",
    };
  }
);
```

**When to use Reflexion:**
- Code generation (run tests, reflect on failures, retry)
- Tasks with clear success criteria (evals!)
- When the first attempt is likely imperfect but improvable

**Limitations:**
- Expensive (multiple full attempts)
- Needs a good evaluator (if you can't evaluate, you can't reflect)
- Diminishing returns after 2-3 attempts

### 1.4 Pattern Comparison

| Pattern | Planning | Iteration | Self-Correction | Best For |
|---------|----------|-----------|-----------------|----------|
| **ReAct** | Implicit | Step-by-step | No | General tasks, exploration |
| **Plan-and-Execute** | Explicit upfront | Plan-guided | Revises plan | Multi-step known workflows |
| **Reflexion** | Per-attempt | Full retry | Yes (reflection) | Tasks with clear eval criteria |

### 1.5 Composing Patterns

Real agents often combine patterns:

```
Plan-and-Execute at the top level:
  Step 1: Analyze codebase (ReAct for exploration)
  Step 2: Write migration (ReAct for implementation)
  Step 3: Run tests (Reflexion if tests fail)
  Step 4: Revise plan if Step 3 failed too many times
```

Don't pick one pattern and stick to it. Use the right pattern for each phase of the task.

---

## 2. The Framework Landscape

Now that you know the patterns, let's look at the frameworks that implement them. Each makes different trade-offs about abstraction level, flexibility, and ecosystem.

### 2.1 Vercel AI SDK

**What it is:** A TypeScript-first library for building AI applications. Provides primitives for text generation, streaming, tool calling, and agent loops.

**Abstraction level:** Low-to-medium. Handles the API protocol, gives you building blocks, doesn't impose an architecture.

**What you get for free:**
- Unified interface across providers (Anthropic, OpenAI, Google, etc.)
- Streaming with `streamText` and `streamObject`
- Tool calling with Zod schemas
- Multi-step agents via `maxSteps` (or `stopWhen` in newer versions)
- React hooks for chat UIs (`useChat`, `useCompletion`)

```typescript
import { generateText, tool, streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

// The Vercel AI SDK version of everything in Ch 8-9:
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  system: "You are a helpful coding assistant.",
  tools: {
    readFile: tool({
      description: "Read a file from the filesystem",
      parameters: z.object({
        path: z.string().describe("File path to read"),
      }),
      execute: async ({ path }) => {
        const content = await readFile(resolve(path), "utf-8");
        return content;
      },
    }),
    listDir: tool({
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
  },
  maxSteps: 10, // Built-in agent loop
  prompt: "What files are in this project?",

  // Step callbacks (replaces our manual logging)
  onStepFinish: ({ text, toolCalls, toolResults, usage }) => {
    if (toolCalls.length > 0) {
      toolCalls.forEach((tc) =>
        console.log(`  [${tc.toolName}(${JSON.stringify(tc.args)})]`)
      );
    }
    console.log(`  Tokens: ${usage.totalTokens}`);
  },
});

console.log(result.text);
console.log(`Done in ${result.steps.length} steps`);
```

**When to use Vercel AI SDK:**
- You're building a web application (it has React hooks)
- You want provider flexibility (swap models easily)
- You want the thinnest useful abstraction
- You're already in the Vercel ecosystem

**When NOT to use it:**
- You need complex agent orchestration (multi-agent, approval flows)
- You need built-in memory management
- You're building a Python application

### 2.2 Anthropic SDK (Direct)

**What it is:** The official Anthropic TypeScript/Python SDK. Direct access to the Claude API with no abstraction layer.

**Abstraction level:** Lowest. You manage everything yourself.

**What you get for free:**
- Type-safe API access
- Streaming support
- Tool calling protocol
- Message batching
- Token counting (via API usage response)

```typescript
import Anthropic from "@anthropic-ai/sdk";

// This is what you built in Ch 8-9, using the raw SDK:
const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 4096,
  system: "You are a helpful assistant.",
  tools: [
    {
      name: "readFile",
      description: "Read a file",
      input_schema: {
        type: "object",
        properties: {
          path: { type: "string", description: "File path" },
        },
        required: ["path"],
      },
    },
  ],
  messages: [{ role: "user", content: "Read package.json" }],
});
```

**When to use the Anthropic SDK directly:**
- You only use Claude (no need for provider abstraction)
- You want maximum control over every API call
- You're building a custom agent loop with specific requirements
- You need features not yet in other SDKs

### 2.3 Claude Agent SDK

**What it is:** Anthropic's higher-level SDK specifically for building agents. Built on top of the Anthropic SDK with agent-specific abstractions.

**Abstraction level:** Medium. Provides agent loop, tool management, and session handling.

```typescript
// Claude Agent SDK approach (conceptual -- check docs for current API)
import { Agent, Tool } from "@anthropic-ai/agent-sdk";

const agent = new Agent({
  model: "claude-sonnet-4-20250514",
  systemPrompt: "You are a helpful coding assistant.",
  tools: [
    new Tool({
      name: "readFile",
      description: "Read a file",
      schema: z.object({ path: z.string() }),
      handler: async ({ path }) => readFile(resolve(path), "utf-8"),
    }),
  ],
  maxTurns: 20,
});

// The agent loop is built in
const result = await agent.run("Read the package.json and explain it");
console.log(result.output);
```

**How it differs from the Messages API:** The raw Anthropic SDK (Section 2.2) gives you `client.messages.create()` -- a single LLM call. You build the loop, manage state, route tool calls. The Agent SDK builds on top of that and gives you:
- **The agent loop as a primitive.** `agent.run()` handles the think-act-observe cycle internally. You define tools and the SDK manages the iteration.
- **Session/state management.** The SDK tracks conversation history, tool results, and turn counts across the session. You don't manually build the messages array.
- **Structured tool registration.** Tools are first-class objects with schemas, handlers, and metadata -- not raw JSON input_schema definitions.
- **Built-in guardrails.** Max turns, token limits, and error recovery are configuration, not code you write.

Here is a more complete example showing tool registration, state, and the run loop:

```typescript
import { Agent, Tool } from "@anthropic-ai/agent-sdk";
import { z } from "zod";

// Tools are registered as objects with schema + handler
const searchCodeTool = new Tool({
  name: "searchCode",
  description: "Search the codebase for files matching a pattern or content",
  schema: z.object({
    query: z.string().describe("Search query (regex or text)"),
    filePattern: z.string().optional().describe("Glob pattern like '*.ts'"),
  }),
  handler: async ({ query, filePattern }) => {
    const results = await codeSearch(query, filePattern);
    return results.map((r) => `${r.file}:${r.line}: ${r.content}`).join("\n");
  },
});

const editFileTool = new Tool({
  name: "editFile",
  description: "Edit a file by replacing old content with new content",
  schema: z.object({
    path: z.string(),
    oldContent: z.string(),
    newContent: z.string(),
  }),
  handler: async ({ path, oldContent, newContent }) => {
    const file = await readFile(path, "utf-8");
    if (!file.includes(oldContent)) {
      return "ERROR: old content not found in file";
    }
    await writeFile(path, file.replace(oldContent, newContent));
    return `Updated ${path}`;
  },
});

// The agent manages the loop, state, and tool dispatch
const agent = new Agent({
  model: "claude-sonnet-4-20250514",
  systemPrompt: `You are a coding agent. You can search code and edit files.
Always search before editing to understand the context.`,
  tools: [searchCodeTool, editFileTool],
  maxTurns: 20,          // Stop after 20 tool-use rounds
  maxTokens: 4096,       // Per-response token limit
});

// Run returns the final result after the full loop
const result = await agent.run(
  "Find the authentication middleware and add rate limiting to the /api/login endpoint"
);

console.log(result.output);         // The agent's final text response
console.log(result.turns);          // Number of tool-use rounds executed
console.log(result.tokenUsage);     // Total tokens used across all rounds
```

The Agent SDK is what Ramp used to build Glass (Ch 59). Its strength is that it gives you just enough structure for production agents without hiding the Claude API behind opaque abstractions. You still think in terms of tools, messages, and turns -- the SDK just handles the plumbing.

**When to use Claude Agent SDK:**
- You're building a Claude-only agent and want a managed loop
- You want structured tool registration with built-in best practices
- You need session management without building it yourself
- You're building a production internal tool (the Glass pattern, Ch 59)

### 2.4 OpenAI Agents SDK

**What it is:** OpenAI's framework for building agents with GPT models. Includes agents, tools, handoffs, and guardrails.

**Abstraction level:** Medium-high. Opinionated about agent structure.

```typescript
// OpenAI Agents SDK approach (conceptual)
import { Agent, Runner, FunctionTool } from "@openai/agents";

const readFileTool = new FunctionTool({
  name: "readFile",
  description: "Read a file from the filesystem",
  params_json_schema: {
    type: "object",
    properties: { path: { type: "string" } },
    required: ["path"],
  },
  on_invoke_tool: async ({ input }) => {
    return await readFile(resolve(input.path), "utf-8");
  },
});

const agent = new Agent({
  name: "Coder",
  instructions: "You are a helpful coding assistant.",
  model: "gpt-4o",
  tools: [readFileTool],
});

const result = await Runner.run(agent, "Read package.json");
console.log(result.final_output);
```

**Handoff between agents** — the SDK's most distinctive feature. You define specialized agents and they delegate to each other:

```typescript
import { Agent, Runner } from "@openai/agents";

const billingAgent = new Agent({
  name: "Billing",
  instructions: "You handle billing and payment questions. If the user asks about technical issues, hand off to the TechSupport agent.",
  model: "gpt-4o",
  tools: [getBillingInfo, createRefund],
});

const techAgent = new Agent({
  name: "TechSupport",
  instructions: "You handle technical issues. If the user asks about billing, hand off to the Billing agent.",
  model: "gpt-4o",
  tools: [checkSystemStatus, lookupErrorCode],
});

// Each agent knows about the other — handoff is a first-class primitive
billingAgent.handoffs = [techAgent];
techAgent.handoffs = [billingAgent];

// The Runner manages the conversation across agent boundaries
const result = await Runner.run(billingAgent, "I was charged twice and also my app is crashing");
// The runner may start with Billing, hand off to TechSupport, and return
console.log(result.final_output);
```

**Guardrails** — validate inputs before the agent processes them and outputs before they reach the user:

```typescript
import { Agent, InputGuardrail, OutputGuardrail } from "@openai/agents";

const contentFilter: InputGuardrail = {
  name: "content_filter",
  execute: async (input) => {
    // Check for prompt injection, PII, or policy violations
    const check = await moderationAPI.check(input);
    if (check.flagged) {
      return { tripwire: true, reason: "Input flagged by content filter" };
    }
    return { tripwire: false };
  },
};

const outputValidator: OutputGuardrail = {
  name: "output_validator",
  execute: async (output) => {
    // Ensure the agent didn't hallucinate or leak sensitive data
    if (output.includes("INTERNAL_SECRET") || output.includes("sk-")) {
      return { tripwire: true, reason: "Output contains sensitive data" };
    }
    return { tripwire: false };
  },
};

const agent = new Agent({
  name: "SafeAgent",
  instructions: "You are a helpful assistant.",
  model: "gpt-4o",
  input_guardrails: [contentFilter],
  output_guardrails: [outputValidator],
});
```

**Tracing** — every agent run is traced automatically. The SDK records each LLM call, tool invocation, handoff, and guardrail check:

```typescript
import { Runner, setTracingExportEndpoint } from "@openai/agents";

// Send traces to OpenAI's trace viewer (or your own backend)
setTracingExportEndpoint("https://your-tracing-backend.com/v1/traces");

const result = await Runner.run(agent, "Analyze this data", {
  runConfig: {
    traceId: "user-session-123", // Correlate with your own IDs
    tracingDisabled: false,
  },
});

// Each step is visible: LLM call → tool call → guardrail check → handoff → LLM call → ...
```

**When to use OpenAI Agents SDK:**
- You're in the OpenAI ecosystem
- You need built-in multi-agent handoffs (its strongest feature)
- You want input/output guardrails without building them yourself
- You want automatic tracing of agent runs

### 2.5 LangChain

**What it is:** The largest and most feature-rich AI framework. Started in Python, has a TypeScript version (LangChain.js). Provides everything from document loaders to agent executors to vector stores.

**Abstraction level:** High. Many abstractions, many concepts to learn.

```typescript
// LangChain.js approach
import { ChatAnthropic } from "@langchain/anthropic";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const readFileTool = tool(
  async ({ path }) => {
    return await readFile(resolve(path), "utf-8");
  },
  {
    name: "readFile",
    description: "Read a file from the filesystem",
    schema: z.object({ path: z.string() }),
  }
);

const model = new ChatAnthropic({ model: "claude-sonnet-4-20250514" });
const agent = createReactAgent({ llm: model, tools: [readFileTool] });

const result = await agent.invoke({
  messages: [{ role: "user", content: "Read package.json" }],
});
```

**What you get for free:**
- 100+ document loaders (PDF, CSV, web, databases)
- Vector store integrations (Pinecone, Weaviate, Chroma, pgvector)
- Pre-built chains and agents
- LangGraph for complex agent workflows
- LangSmith for tracing and evaluation
- Huge community and ecosystem

**When to use LangChain:**
- You need document loaders and vector stores (RAG, Ch 15)
- You want a large ecosystem of pre-built integrations
- You're building a RAG pipeline with many data sources

**When NOT to use LangChain:**
- Simple agents (the abstraction overhead isn't worth it)
- You want to understand exactly what's happening (many layers of abstraction)
- You need predictable, debuggable behavior
- You're sensitive to dependency weight

### 2.5.1 LangGraph: When You Need Stateful Graph Execution

LangChain gives you chains (linear sequences) and a basic ReAct agent. **LangGraph** is its sibling library for when you need something more: stateful, multi-step agents with branching logic, cycles, and persistent state.

The core idea: your agent is a **graph**. Nodes are functions (LLM calls, tool executions, conditional logic). Edges define the flow. State is passed between nodes and persisted automatically.

```typescript
import { StateGraph, Annotation, END } from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";
import { ToolNode } from "@langchain/langgraph/prebuilt";

// 1. Define the state that flows through the graph
const AgentState = Annotation.Root({
  messages: Annotation({
    reducer: (prev, next) => [...prev, ...next],
    default: () => [],
  }),
});

// 2. Define the nodes
const model = new ChatAnthropic({ model: "claude-sonnet-4-20250514" }).bindTools(tools);

async function callModel(state: typeof AgentState.State) {
  const response = await model.invoke(state.messages);
  return { messages: [response] };
}

function shouldContinue(state: typeof AgentState.State) {
  const lastMessage = state.messages[state.messages.length - 1];
  // If the model made tool calls, route to the tool node
  if (lastMessage.tool_calls?.length > 0) {
    return "tools";
  }
  // Otherwise, we're done
  return END;
}

// 3. Build the graph
const workflow = new StateGraph(AgentState)
  .addNode("agent", callModel)
  .addNode("tools", new ToolNode(tools))
  .addEdge("__start__", "agent")
  .addConditionalEdges("agent", shouldContinue)
  .addEdge("tools", "agent"); // After tools, go back to the agent

// 4. Compile and run
const app = workflow.compile({
  checkpointer: new MemorySaver(), // Persist state across invocations
});

const result = await app.invoke(
  { messages: [{ role: "user", content: "Research this topic and write a report" }] },
  { configurable: { thread_id: "session-123" } } // Resume from previous state
);
```

**Why LangGraph over a plain loop?** Three reasons:
1. **Persistent state.** The checkpointer saves state after every node. If the process crashes, it resumes from the last checkpoint, not from the beginning.
2. **Branching and cycles.** Conditional edges let you build complex flows: "if the research is insufficient, go back to the search node; if complete, go to the writing node."
3. **Human-in-the-loop.** You can add `interrupt_before` or `interrupt_after` to any node, pausing the graph for human approval before continuing.

### 2.5.2 LangSmith: Tracing and Evaluation

LangSmith is LangChain's observability and evaluation platform. It works with LangChain, LangGraph, or any LLM application (even those not using LangChain):

```typescript
// Enable tracing — just set environment variables
// LANGCHAIN_TRACING_V2=true
// LANGCHAIN_API_KEY=ls-...
// LANGCHAIN_PROJECT=my-agent

// Every LLM call, tool invocation, and chain step is automatically traced.
// You see: input → LLM call → tool call → LLM call → output
// With latency, token counts, and cost for each step.

// LangSmith also supports evaluation datasets:
import { Client } from "langsmith";

const client = new Client();

// Create a dataset of test cases
await client.createDataset("agent-evals", {
  description: "Test cases for our coding agent",
});

// Add examples
await client.createExamples({
  inputs: [
    { query: "What files are in the project?" },
    { query: "Fix the bug in auth.ts" },
  ],
  outputs: [
    { expected: "Should list files using listDir tool" },
    { expected: "Should read auth.ts, identify the bug, and write a fix" },
  ],
  datasetName: "agent-evals",
});
```

### 2.5.3 The Trade-Off: Convenience vs Lock-In vs Abstraction Overhead

| Factor | LangChain | LangGraph | Direct SDK |
|--------|-----------|-----------|------------|
| **Time to prototype** | Fast — many pre-built components | Medium — graph design takes thought | Slower — you build everything |
| **Debuggability** | Hard — many abstraction layers | Medium — graph structure is explicit | Easy — you see every call |
| **Lock-in** | High — LangChain types throughout your code | High — graph structure is LangGraph-specific | None — standard API calls |
| **Ecosystem** | Huge — 100+ integrations | Growing — LangChain integrations work | Whatever you build |
| **Abstraction overhead** | Significant — wrapper on wrapper | Moderate — graph primitives are clear | Zero |
| **Best for** | RAG pipelines, document processing | Complex stateful workflows, multi-agent | Simple agents, maximum control |

**The practical guideline:** Use LangChain when you need its ecosystem (document loaders, vector stores, pre-built retrievers for RAG). Use LangGraph when you need persistent state and complex flow control. Use neither when a simple loop does the job — which is more often than you think.

### 2.6 Mastra

**What it is:** A TypeScript-first framework for building AI applications with a focus on workflows, integrations, and type safety.

**Abstraction level:** Medium. More opinionated than Vercel AI SDK, less sprawling than LangChain.

```typescript
// Mastra approach (conceptual)
import { Agent } from "@mastra/core/agent";
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

const readFileTool = createTool({
  id: "readFile",
  description: "Read a file from the filesystem",
  inputSchema: z.object({ path: z.string() }),
  execute: async ({ context }) => {
    return await readFile(resolve(context.path), "utf-8");
  },
});

const agent = new Agent({
  name: "Coder",
  instructions: "You are a helpful coding assistant.",
  model: {
    provider: "ANTHROPIC",
    name: "claude-sonnet-4-20250514",
  },
  tools: { readFile: readFileTool },
});

const response = await agent.generate("Read package.json");
```

**When to use Mastra:**
- You want TypeScript-first with good type inference
- You need workflow orchestration (Mastra Workflows)
- You want built-in integrations with third-party services
- You're building agents that need structured workflows

---

## 3. Framework Mapping: What You Built vs What Frameworks Provide

Here's the critical comparison. Everything in the left column is something you built in Chapters 8-12. The right column shows which framework provides it:

```
What You Built              | Vercel AI | Anthropic | LangChain | Mastra | OpenAI Agents
---------------------------------------------------------------------------------------------
Tool definitions (Ch 8)     | tool()    | input_schema | tool()  | createTool | FunctionTool
Tool calling protocol       | Built-in  | Built-in  | Built-in  | Built-in   | Built-in
Agent loop (Ch 9)           | maxSteps  | Manual    | agents    | agent.generate | Runner.run
Streaming (Ch 9)            | streamText| .stream() | .stream() | .stream()  | Runner.stream
Working memory (Ch 10)      | Manual    | Manual    | LangGraph state | Manual | Manual
Long-term memory (Ch 10)    | Manual    | Manual    | Memory classes | Manual | Manual
Approval gates (Ch 11)      | Manual    | Manual    | Human-in-loop | Manual | Guardrails (partial)
Context management (Ch 12)  | Manual    | Manual    | Manual    | Manual     | Manual
Sub-agent delegation (Ch12) | Manual    | Manual    | LangGraph | Manual     | Handoffs
```

**Key observation:** No framework handles everything. They all handle the API protocol and basic tool calling. Most provide a basic agent loop. But memory, approvals, and context management are almost always your responsibility.

---

## 4. The Framework Decision Tree

```
Do you need provider flexibility?
  YES -> Vercel AI SDK (supports Anthropic, OpenAI, Google, etc.)
  NO  -> Which provider?
    Claude  -> Anthropic SDK or Claude Agent SDK
    OpenAI  -> OpenAI Agents SDK
    
Do you need document loaders and vector stores?
  YES -> LangChain (unmatched ecosystem for RAG)
  NO  -> Keep it lighter

Do you need complex stateful workflows?
  YES -> LangGraph (part of LangChain) or Mastra Workflows
  NO  -> Simple agent loop is fine

Do you need multi-agent coordination?
  YES -> LangGraph, OpenAI Agents SDK (handoffs), or build custom
  NO  -> Single agent loop

Do you need maximum control and debuggability?
  YES -> Anthropic SDK directly + your own loop (Ch 8-12)
  NO  -> Any framework that fits your other criteria

Are you building a web app with React?
  YES -> Vercel AI SDK (has useChat, useCompletion hooks)
  NO  -> Doesn't matter
```

### 4.1 My Recommendations

**For learning:** Build from scratch with the Anthropic SDK (what you did in Ch 8-12). Nothing teaches you better than building it yourself.

**For most production agents:** Vercel AI SDK + custom code for memory and approvals. It gives you the right level of abstraction without locking you in.

**For RAG-heavy applications:** LangChain for document loading and vector stores, but consider extracting the relevant pieces rather than adopting the whole framework.

**For complex multi-step workflows:** Mastra or LangGraph, depending on your ecosystem preference.

**For Claude-only agents in production:** Anthropic SDK or Claude Agent SDK. The least abstraction is the most predictable.

---

## 5. Framework Anti-Patterns

### 5.1 The Everything Framework

```typescript
// BAD: Using LangChain to make a simple API call
import { ChatAnthropic } from "@langchain/anthropic";
import { HumanMessage } from "@langchain/core/messages";

const model = new ChatAnthropic({ model: "claude-sonnet-4-20250514" });
const response = await model.invoke([new HumanMessage("Hello")]);

// GOOD: Just use the SDK
const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
});
```

Don't add a framework dependency for something the SDK does in three lines.

### 5.2 The Abstraction Trap

```typescript
// BAD: Can't debug because you don't know what the framework does
const agent = createAgent(config); // What's in here?
const result = await agent.run(task); // How many LLM calls? What tools? What happened?

// GOOD: Explicit loop where you can see and log everything
for (let i = 0; i < maxIterations; i++) {
  const response = await client.messages.create({/* ... */});
  // You see every call, every response, every tool execution
}
```

When an abstracted agent misbehaves, you need to read the framework source code to debug it. With a manual loop, the bug is in *your* code.

### 5.3 Framework Lock-In

```typescript
// BAD: Your entire business logic is inside framework abstractions
class MyAgent extends LangChainAgent {
  // All your logic is coupled to LangChain types
  async handleTool(tool: LangChainTool): Promise<LangChainResponse> {
    // ...
  }
}

// GOOD: Business logic is independent, framework is a thin adapter
async function myBusinessLogic(input: MyInput): Promise<MyOutput> {
  // Pure business logic, no framework dependencies
}

// Thin adapter
const tool = new LangChainTool({
  handler: async (input) => myBusinessLogic(input),
});
```

Keep your business logic framework-independent. Use the framework as an integration layer, not a foundation.

---

## 6. Building Your Own Micro-Framework

After Chapters 8-12, you have all the pieces. Here's how to package them into a reusable mini-framework that you control:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { z, ZodObject, ZodRawShape } from "zod";
import { readFile, readdir } from "fs/promises";
import { resolve } from "path";

// --- Tool Definition ---
interface ToolDef<T extends ZodRawShape> {
  name: string;
  description: string;
  parameters: ZodObject<T>;
  execute: (args: z.infer<ZodObject<T>>) => Promise<unknown>;
  risk?: "none" | "low" | "medium" | "high" | "critical";
}

function defineTool<T extends ZodRawShape>(def: ToolDef<T>): ToolDef<T> {
  return def;
}

// --- Agent Configuration ---
interface AgentConfig {
  model: string;
  systemPrompt: string;
  tools: ToolDef<any>[];
  maxIterations: number;
  contextWindowSize?: number;
  compactionThreshold?: number;
  requireApproval?: (toolName: string, args: unknown) => boolean | Promise<boolean>;
  onToolCall?: (name: string, args: unknown) => void;
  onToolResult?: (name: string, result: unknown) => void;
  onCompaction?: (before: number, after: number) => void;
}

// --- The Agent ---
class SimpleAgent {
  private client: Anthropic;
  private config: AgentConfig;
  private anthropicTools: Anthropic.Tool[];

  constructor(config: AgentConfig) {
    this.client = new Anthropic();
    this.config = config;
    this.anthropicTools = config.tools.map((t) =>
      this.toolDefToAnthropic(t)
    );
  }

  private toolDefToAnthropic(toolDef: ToolDef<any>): Anthropic.Tool {
    // Convert Zod schema to JSON Schema for Anthropic
    // In production, use the zod-to-json-schema library
    const shape = toolDef.parameters.shape;
    const properties: Record<string, { type: string; description: string }> = {};
    const required: string[] = [];

    for (const [key, value] of Object.entries(shape)) {
      const zodField = value as z.ZodType;
      properties[key] = { type: "string", description: key };
      if (!zodField.isOptional()) {
        required.push(key);
      }
    }

    return {
      name: toolDef.name,
      description: toolDef.description,
      input_schema: {
        type: "object" as const,
        properties,
        required,
      },
    };
  }

  async run(userMessage: string): Promise<string> {
    const messages: Anthropic.MessageParam[] = [
      { role: "user", content: userMessage },
    ];

    for (let i = 0; i < this.config.maxIterations; i++) {
      // Context management (simplified)
      if (this.config.compactionThreshold) {
        const tokens = Math.ceil(JSON.stringify(messages).length / 4);
        const threshold =
          (this.config.contextWindowSize || 200000) * this.config.compactionThreshold;
        if (tokens > threshold) {
          const before = tokens;
          // Simple sliding window -- replace with full compaction from Ch 12
          messages.splice(0, messages.length - 6);
          const after = Math.ceil(JSON.stringify(messages).length / 4);
          this.config.onCompaction?.(before, after);
        }
      }

      const response = await this.client.messages.create({
        model: this.config.model,
        max_tokens: 4096,
        system: this.config.systemPrompt,
        tools: this.anthropicTools,
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
        const args = toolUse.input;

        // Approval check
        if (this.config.requireApproval) {
          const needsApproval = await this.config.requireApproval(
            toolUse.name,
            args
          );
          if (needsApproval) {
            toolResults.push({
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: JSON.stringify({ rejected: true, message: "Approval required" }),
              is_error: true,
            });
            continue;
          }
        }

        this.config.onToolCall?.(toolUse.name, args);

        // Execute the tool
        const toolDef = this.config.tools.find((t) => t.name === toolUse.name);
        if (!toolDef) {
          toolResults.push({
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: JSON.stringify({ error: `Unknown tool: ${toolUse.name}` }),
            is_error: true,
          });
          continue;
        }

        try {
          const result = await toolDef.execute(args as any);
          const resultStr = typeof result === "string" ? result : JSON.stringify(result);

          this.config.onToolResult?.(toolUse.name, result);

          toolResults.push({
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: resultStr,
          });
        } catch (err) {
          toolResults.push({
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: JSON.stringify({ error: (err as Error).message }),
            is_error: true,
          });
        }
      }

      messages.push({ role: "user", content: toolResults });
    }

    return "[Max iterations reached]";
  }
}

// --- Usage ---
const agent = new SimpleAgent({
  model: "claude-sonnet-4-20250514",
  systemPrompt: "You are a helpful coding assistant.",
  maxIterations: 15,
  compactionThreshold: 0.7,
  contextWindowSize: 200000,
  tools: [
    defineTool({
      name: "readFile",
      description: "Read a file from the filesystem",
      parameters: z.object({
        path: z.string().describe("File path to read"),
      }),
      execute: async ({ path }) => {
        return await readFile(resolve(path), "utf-8");
      },
      risk: "low",
    }),
    defineTool({
      name: "listDir",
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
      risk: "none",
    }),
  ],
  onToolCall: (name, args) => console.log(`  [${name}(${JSON.stringify(args)})]`),
  onToolResult: (name) => console.log(`  [${name} done]`),
});

const response = await agent.run("Read the package.json and explain it");
console.log(response);
```

This is ~150 lines. It handles tools, the agent loop, basic context management, and approval hooks. You understand every line because you wrote it.

---

## 7. What's Next

You've completed Part 2. You can now:
- Define tools with descriptions, Zod schemas, and execute functions (Ch 8)
- Build an agent loop from scratch (Ch 9)
- Implement three layers of memory (Ch 10)
- Add human-in-the-loop approval gates (Ch 11)
- Manage the context window with compaction and delegation (Ch 12)
- Recognize agent patterns and choose between frameworks (Ch 13)

**You can build autonomous AI agents.**

Part 3 takes a different direction: RAG (Retrieval-Augmented Generation). Instead of giving the agent tools to take actions, you give it tools to *access knowledge*. How do you search a vector database? How do you chunk and embed documents? How do you build a system that answers questions from your company's documentation?

The tools you built in this Part become the foundation. A RAG pipeline is just a tool that the agent calls -- "search the knowledge base for X." The agent loop runs the retrieval. The context management ensures the retrieved documents don't blow the token budget.

Everything spirals.

---

## Key Takeaways

1. **ReAct** (think-act-observe) is the most common pattern. It's what you built in Chapter 9.
2. **Plan-and-Execute** adds explicit upfront planning. Use for complex multi-step tasks.
3. **Reflexion** adds self-correction. Use when you have clear success criteria (evals).
4. **Real agents compose patterns** -- plan at the top level, ReAct for execution, Reflexion when things fail.
5. **Frameworks handle the protocol** (tool calling, streaming, message formats). They rarely handle memory, approvals, or context management.
6. **Vercel AI SDK** is the sweet spot for most TypeScript agents: thin abstraction, provider flexibility.
7. **LangChain** is best for RAG (document loaders, vector stores). **LangGraph** is for stateful multi-step workflows with persistent state and branching. Don't use either just for simple agent loops.
8. **OpenAI Agents SDK** shines for multi-agent handoffs and built-in guardrails. **Claude Agent SDK** shines for managed agent loops in Claude-only systems.
9. **Build from scratch first** (Ch 8-12), then adopt frameworks for specific features you need.
10. **Keep business logic framework-independent.** Use frameworks as integration layers, not foundations.
11. **The best framework is the one you understand.** When it breaks, you need to know where to look.

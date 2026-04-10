<!--
  CHAPTER: 58
  TITLE: Multi-Agent Orchestration
  PART: XII — AI Platform Engineering
  PHASE: 2 — Become an Expert
  PREREQS: Ch 9 (agent loop), Ch 13 (agent patterns/frameworks), Ch 32 (multi-agent coordination internals)
  KEY_TOPICS: multi-agent systems, parent-child spawning, peer-to-peer agents, coordinator pattern, council pattern, parallel execution, sequential pipelines, error handling, token budgets, shared memory, message passing
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 58: Multi-Agent Orchestration

> **Part XII — AI Platform Engineering** | Phase 2: Become an Expert | Prerequisites: Ch 9, Ch 13, Ch 32 | Difficulty: Advanced | Language: TypeScript + Python

In Chapter 9 you built a single agent loop: prompt, call LLM, execute tool, append result, repeat. In Chapter 32 you studied how Claude Code coordinates multiple agents internally -- coordinator mode, tick loops, background workers. Now you build multi-agent systems yourself. Not toy examples. Production orchestration where agents spawn agents, where a council of models debates a decision, where a fleet of workers processes tasks in parallel, and where the whole thing handles failures gracefully.

The jump from one agent to many agents is not a linear increase in complexity -- it is a qualitative shift. A single agent manages one conversation thread. A multi-agent system manages *coordination*: who knows what, who talks to whom, who waits for whom, who fails and what happens when they do, who pays for tokens and how much they are allowed to spend. These are distributed systems problems, and they require distributed systems thinking.

This chapter gives you the patterns, the code, and the failure modes. By the end you will have a working multi-agent orchestration framework in TypeScript that supports parent-child spawning, parallel execution, sequential pipelines, and the council pattern.

### In This Chapter
- Why multi-agent: when one agent is not enough
- Agent spawning patterns: parent-child, peer-to-peer, coordinator
- The council pattern: multiple models with adversarial stances cross-reviewing each other
- Parallel execution: running independent tasks simultaneously
- Sequential pipelines: chaining agents where output feeds input
- Error handling: what happens when one agent in a fleet fails
- Communication patterns: shared memory, message passing, event queues
- Resource management: token budgets across agent fleets, cost allocation
- Building a complete multi-agent system in TypeScript
- Production considerations: timeouts, retries, observability

### Related Chapters
- **Ch 9 (The Agent Loop)** -- single agent loop; this chapter runs many in parallel
- **Ch 13 (Agent Patterns & Frameworks)** -- ReAct, Plan-and-Execute; this chapter adds orchestration patterns
- **Ch 32 (Multi-Agent Coordination)** -- how Claude Code does it internally; this chapter builds your own
- **Ch 26 (AI-Augmented Development)** -- Stripe Minions as multi-agent delegation; this chapter generalizes the pattern
- **Ch 49 (Cost Engineering)** -- token budgets; this chapter allocates them across fleets
- **Ch 59 (Building Internal AI Tools)** -- the orchestration from this chapter becomes the engine inside platform tools

---

## 1. Why Multi-Agent Systems

### 1.1 When One Agent Is Not Enough

A single agent loop works beautifully for focused tasks: answer a question, generate code, process a document. But real-world workflows frequently involve:

**Scope beyond a single context window.** You need to analyze 50 files, each requiring its own reasoning chain. One agent cannot hold all of that in context simultaneously. Five agents each handling 10 files can.

**Tasks that require different expertise.** A code review needs a security expert, a performance expert, and a style expert. One agent trying to be all three produces shallow analysis. Three specialized agents produce deep analysis.

**Tasks that benefit from disagreement.** For critical decisions, you want models to argue with each other. A single model talking to itself cannot generate genuine adversarial review. Multiple models with different system prompts can.

**Tasks where parallelism matters.** Processing 100 customer support tickets sequentially takes 100x as long as processing them in parallel with 100 agents. For batch workloads, multi-agent is a throughput multiplier.

**Tasks that require human-like delegation.** A project manager does not write all the code, run all the tests, and deploy the app. They delegate to specialists. Multi-agent systems mirror this natural division of labor.

### 1.2 The Three Fundamental Patterns

Every multi-agent system is a variation on three patterns:

```
PARENT-CHILD (hierarchical)
┌─────────────┐
│  Coordinator │
│   (parent)   │
└──┬───┬───┬──┘
   │   │   │
   ▼   ▼   ▼
  ┌─┐ ┌─┐ ┌─┐
  │A│ │B│ │C│   ← child agents
  └─┘ └─┘ └─┘

PEER-TO-PEER (collaborative)
  ┌─┐   ┌─┐   ┌─┐
  │A│◄─►│B│◄─►│C│   ← agents communicate directly
  └─┘   └─┘   └─┘

PIPELINE (sequential)
  ┌─┐   ┌─┐   ┌─┐
  │A│──►│B│──►│C│   ← output flows forward
  └─┘   └─┘   └─┘
```

Most production systems use **parent-child** because it is easiest to reason about. The coordinator knows the full plan, delegates tasks to children, collects results, and synthesizes a final output. The children do not need to know about each other.

**Peer-to-peer** is powerful but complex. Agents communicate directly, which means you need a protocol for message passing and a strategy for preventing infinite loops. Use this when agents genuinely need to negotiate or debate (the council pattern).

**Pipeline** is the simplest. Agent A produces output, Agent B refines it, Agent C validates it. Use this when your workflow has clear sequential stages.

### 1.3 The Coordination Tax

Multi-agent systems pay a coordination tax that single agents do not. Every message between agents costs tokens. Every context switch means re-establishing state. Every failure in one agent can cascade.

**Rule of thumb:** If a task can be done well by a single agent with the right tools, do not use multiple agents. Multi-agent orchestration is for when the task genuinely exceeds the capacity or specialization of one agent.

---

## 2. Agent Spawning Patterns

### 2.1 Parent-Child: The Coordinator Pattern

The most common production pattern. One agent acts as coordinator: it receives the task, decomposes it into subtasks, spawns child agents for each subtask, collects results, and synthesizes the final output.

```typescript
// types.ts — Core types for multi-agent orchestration
import Anthropic from "@anthropic-ai/sdk";

interface AgentConfig {
  id: string;
  model: string;
  systemPrompt: string;
  tools?: Anthropic.Tool[];
  maxTokens?: number;
  tokenBudget?: number; // max tokens this agent can consume total
}

interface AgentResult {
  agentId: string;
  status: "success" | "error" | "timeout" | "budget_exceeded";
  output: string;
  tokensUsed: { input: number; output: number };
  durationMs: number;
  error?: string;
}

interface OrchestratorConfig {
  coordinator: AgentConfig;
  workers: AgentConfig[];
  strategy: "parallel" | "sequential" | "adaptive";
  totalTokenBudget: number;
  timeoutMs: number;
}
```

```typescript
// agent-runner.ts — Execute a single agent loop
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface TokenTracker {
  input: number;
  output: number;
}

async function runAgent(
  config: AgentConfig,
  task: string,
  context?: string
): Promise<AgentResult> {
  const startTime = Date.now();
  const tokens: TokenTracker = { input: 0, output: 0 };
  const messages: Anthropic.MessageParam[] = [];

  // Build the initial message with task and optional context
  let userMessage = task;
  if (context) {
    userMessage = `## Context from coordinator\n${context}\n\n## Your task\n${task}`;
  }
  messages.push({ role: "user", content: userMessage });

  const maxIterations = 20;

  for (let i = 0; i < maxIterations; i++) {
    // Check token budget before calling
    if (config.tokenBudget && tokens.input + tokens.output > config.tokenBudget) {
      return {
        agentId: config.id,
        status: "budget_exceeded",
        output: `Agent ${config.id} exceeded token budget of ${config.tokenBudget}`,
        tokensUsed: tokens,
        durationMs: Date.now() - startTime,
      };
    }

    try {
      const response = await client.messages.create({
        model: config.model,
        max_tokens: config.maxTokens || 4096,
        system: config.systemPrompt,
        tools: config.tools || [],
        messages,
      });

      // Track tokens
      tokens.input += response.usage.input_tokens;
      tokens.output += response.usage.output_tokens;

      // Check for tool use
      const toolUseBlocks = response.content.filter(
        (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
      );

      if (response.stop_reason === "end_turn" || toolUseBlocks.length === 0) {
        // Agent is done — extract text response
        const textBlocks = response.content.filter(
          (block): block is Anthropic.TextBlock => block.type === "text"
        );
        const output = textBlocks.map((b) => b.text).join("\n");

        return {
          agentId: config.id,
          status: "success",
          output,
          tokensUsed: tokens,
          durationMs: Date.now() - startTime,
        };
      }

      // Execute tools and continue the loop
      messages.push({ role: "assistant", content: response.content });

      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const toolUse of toolUseBlocks) {
        const result = await executeToolCall(toolUse.name, toolUse.input);
        toolResults.push({
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: result,
        });
      }

      messages.push({ role: "user", content: toolResults });
    } catch (error) {
      return {
        agentId: config.id,
        status: "error",
        output: "",
        tokensUsed: tokens,
        durationMs: Date.now() - startTime,
        error: error instanceof Error ? error.message : String(error),
      };
    }
  }

  // Exceeded max iterations
  return {
    agentId: config.id,
    status: "error",
    output: "Max iterations exceeded",
    tokensUsed: tokens,
    durationMs: Date.now() - startTime,
    error: "Agent did not complete within iteration limit",
  };
}

async function executeToolCall(
  toolName: string,
  input: unknown
): Promise<string> {
  // Your tool implementations here — delegated to a tool registry
  // This is the same pattern from Ch 8, now inside a multi-agent framework
  const registry = getToolRegistry();
  const tool = registry.get(toolName);
  if (!tool) return `Error: Unknown tool ${toolName}`;

  try {
    return await tool.execute(input);
  } catch (error) {
    return `Error executing ${toolName}: ${error}`;
  }
}
```

### 2.2 The Coordinator Agent

The coordinator is the brain. It receives the top-level task, breaks it down, dispatches to workers, and synthesizes results. The key insight: the coordinator itself is an agent that uses *spawning workers* as its primary tool.

```typescript
// coordinator.ts — The parent agent that orchestrates workers
import Anthropic from "@anthropic-ai/sdk";

interface TaskDecomposition {
  subtasks: {
    id: string;
    description: string;
    assignTo: string; // worker agent ID
    dependencies: string[]; // IDs of subtasks that must complete first
    context?: string; // additional context for the worker
  }[];
  executionStrategy: "parallel" | "sequential" | "dependency_graph";
}

const COORDINATOR_SYSTEM_PROMPT = `You are a task coordinator. Your job is to:
1. Analyze the incoming task
2. Break it into subtasks that can be handled by specialist workers
3. Assign each subtask to the right worker
4. Specify dependencies between subtasks
5. Choose an execution strategy

Available workers:
{{WORKER_DESCRIPTIONS}}

Respond with a JSON task decomposition. Be specific in your subtask descriptions.
Each subtask should be self-contained enough that a worker can complete it
without needing to ask clarifying questions.

IMPORTANT: Do not create subtasks for things you can answer directly.
Only delegate when specialized knowledge or parallel processing adds value.`;

async function coordinatorDecompose(
  task: string,
  workers: AgentConfig[]
): Promise<TaskDecomposition> {
  const client = new Anthropic();

  const workerDescriptions = workers
    .map((w) => `- ${w.id}: ${w.systemPrompt.slice(0, 200)}`)
    .join("\n");

  const systemPrompt = COORDINATOR_SYSTEM_PROMPT.replace(
    "{{WORKER_DESCRIPTIONS}}",
    workerDescriptions
  );

  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: systemPrompt,
    messages: [
      {
        role: "user",
        content: `Decompose this task into subtasks:\n\n${task}`,
      },
    ],
  });

  const text = response.content
    .filter((b): b is Anthropic.TextBlock => b.type === "text")
    .map((b) => b.text)
    .join("");

  // Parse JSON from the response
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (!jsonMatch) {
    throw new Error("Coordinator did not produce valid task decomposition");
  }

  return JSON.parse(jsonMatch[0]) as TaskDecomposition;
}
```

### 2.3 Spawning and Managing Workers

```typescript
// orchestrator.ts — Run the full multi-agent pipeline

async function orchestrate(
  task: string,
  config: OrchestratorConfig
): Promise<{
  finalOutput: string;
  results: AgentResult[];
  totalTokens: { input: number; output: number };
  totalDurationMs: number;
}> {
  const startTime = Date.now();
  const allResults: AgentResult[] = [];
  let totalTokens = { input: 0, output: 0 };

  // Step 1: Coordinator decomposes the task
  console.log(`[orchestrator] Decomposing task with coordinator...`);
  const decomposition = await coordinatorDecompose(task, config.workers);
  console.log(
    `[orchestrator] Got ${decomposition.subtasks.length} subtasks, ` +
    `strategy: ${decomposition.executionStrategy}`
  );

  // Step 2: Execute subtasks according to strategy
  if (decomposition.executionStrategy === "parallel") {
    const results = await executeParallel(
      decomposition.subtasks,
      config.workers,
      config.totalTokenBudget,
      config.timeoutMs
    );
    allResults.push(...results);
  } else if (decomposition.executionStrategy === "sequential") {
    const results = await executeSequential(
      decomposition.subtasks,
      config.workers,
      config.totalTokenBudget,
      config.timeoutMs
    );
    allResults.push(...results);
  } else {
    // dependency_graph — topological sort then parallel within each level
    const results = await executeDependencyGraph(
      decomposition.subtasks,
      config.workers,
      config.totalTokenBudget,
      config.timeoutMs
    );
    allResults.push(...results);
  }

  // Step 3: Synthesize results
  totalTokens = allResults.reduce(
    (acc, r) => ({
      input: acc.input + r.tokensUsed.input,
      output: acc.output + r.tokensUsed.output,
    }),
    { input: 0, output: 0 }
  );

  const finalOutput = await synthesizeResults(task, allResults, config.coordinator);

  return {
    finalOutput,
    results: allResults,
    totalTokens,
    totalDurationMs: Date.now() - startTime,
  };
}
```

---

## 3. Parallel Execution

### 3.1 The Simplest Case: Independent Tasks

When subtasks have no dependencies, you can run them all at once. This is the most common multi-agent pattern in production: take a batch of work, fan out to workers, fan in results.

```typescript
// parallel.ts — Execute independent subtasks in parallel

async function executeParallel(
  subtasks: TaskDecomposition["subtasks"],
  workers: AgentConfig[],
  tokenBudget: number,
  timeoutMs: number
): Promise<AgentResult[]> {
  // Allocate token budget equally across workers
  const budgetPerWorker = Math.floor(tokenBudget / subtasks.length);

  const promises = subtasks.map(async (subtask) => {
    const worker = workers.find((w) => w.id === subtask.assignTo);
    if (!worker) {
      return {
        agentId: subtask.assignTo,
        status: "error" as const,
        output: "",
        tokensUsed: { input: 0, output: 0 },
        durationMs: 0,
        error: `No worker found with id ${subtask.assignTo}`,
      };
    }

    // Run with timeout
    const workerWithBudget = { ...worker, tokenBudget: budgetPerWorker };
    return Promise.race([
      runAgent(workerWithBudget, subtask.description, subtask.context),
      createTimeout(subtask.assignTo, timeoutMs),
    ]);
  });

  return Promise.all(promises);
}

function createTimeout(agentId: string, ms: number): Promise<AgentResult> {
  return new Promise((resolve) =>
    setTimeout(
      () =>
        resolve({
          agentId,
          status: "timeout",
          output: "",
          tokensUsed: { input: 0, output: 0 },
          durationMs: ms,
          error: `Agent timed out after ${ms}ms`,
        }),
      ms
    )
  );
}
```

### 3.2 Parallel with Concurrency Control

In production, you cannot just fire 100 agents simultaneously. You will hit API rate limits. You need a concurrency pool:

```typescript
// concurrency.ts — Bounded parallel execution

async function executeWithConcurrency<T>(
  tasks: (() => Promise<T>)[],
  maxConcurrent: number
): Promise<T[]> {
  const results: T[] = new Array(tasks.length);
  let nextIndex = 0;

  async function runNext(): Promise<void> {
    while (nextIndex < tasks.length) {
      const currentIndex = nextIndex++;
      results[currentIndex] = await tasks[currentIndex]();
    }
  }

  // Create pool of workers
  const pool = Array.from(
    { length: Math.min(maxConcurrent, tasks.length) },
    () => runNext()
  );

  await Promise.all(pool);
  return results;
}

// Usage: process 100 documents with 10 concurrent agents
const tasks = documents.map(
  (doc) => () => runAgent(analyzerConfig, `Analyze this document:\n${doc}`)
);
const results = await executeWithConcurrency(tasks, 10);
```

### 3.3 Map-Reduce with Agents

A powerful pattern for batch processing: map over inputs with parallel agents, then reduce results with a synthesis agent.

```typescript
// map-reduce.ts — Fan-out processing, fan-in synthesis

interface MapReduceConfig {
  mapAgent: AgentConfig;      // Processes each item
  reduceAgent: AgentConfig;   // Synthesizes all map outputs
  maxConcurrency: number;
  mapTokenBudget: number;     // Per-item token budget
  reduceTokenBudget: number;  // Synthesis token budget
}

async function mapReduce(
  items: string[],
  instruction: string,
  config: MapReduceConfig
): Promise<{ synthesis: string; itemResults: AgentResult[] }> {
  // MAP phase: process each item in parallel
  console.log(`[map-reduce] Mapping ${items.length} items...`);
  const mapTasks = items.map(
    (item, i) => () =>
      runAgent(
        { ...config.mapAgent, id: `mapper-${i}`, tokenBudget: config.mapTokenBudget },
        `${instruction}\n\nItem to process:\n${item}`
      )
  );

  const mapResults = await executeWithConcurrency(mapTasks, config.maxConcurrency);

  // REDUCE phase: synthesize all results
  console.log(`[map-reduce] Reducing ${mapResults.length} results...`);
  const successfulResults = mapResults
    .filter((r) => r.status === "success")
    .map((r, i) => `## Result ${i + 1} (from ${r.agentId})\n${r.output}`)
    .join("\n\n---\n\n");

  const failedCount = mapResults.filter((r) => r.status !== "success").length;

  const reduceTask = `You received results from ${mapResults.length} workers ` +
    `(${failedCount} failed). Synthesize the successful results into a ` +
    `coherent summary.\n\nOriginal instruction: ${instruction}\n\n` +
    `Results:\n${successfulResults}`;

  const synthesis = await runAgent(
    { ...config.reduceAgent, tokenBudget: config.reduceTokenBudget },
    reduceTask
  );

  return {
    synthesis: synthesis.output,
    itemResults: mapResults,
  };
}
```

**Real-world example:** Ramp's Glass tool uses this pattern for codebase analysis. When an engineer asks "find all API endpoints that don't have rate limiting," Glass maps over each service directory in parallel, then reduces the findings into a prioritized report.

---

## 4. Sequential Pipelines

### 4.1 The Chain Pattern

When each stage needs the previous stage's output, you run agents sequentially. The output of Agent A becomes the input of Agent B.

```typescript
// pipeline.ts — Sequential agent chain

interface PipelineStage {
  agent: AgentConfig;
  transformInput?: (previousOutput: string, originalInput: string) => string;
}

async function executePipeline(
  input: string,
  stages: PipelineStage[],
  tokenBudget: number
): Promise<{
  finalOutput: string;
  stageResults: AgentResult[];
}> {
  const stageResults: AgentResult[] = [];
  let currentInput = input;
  let remainingBudget = tokenBudget;

  for (const stage of stages) {
    // Transform input if a transformer is provided
    const stageInput = stage.transformInput
      ? stage.transformInput(currentInput, input)
      : currentInput;

    // Allocate remaining budget
    const stageBudget = Math.floor(remainingBudget / (stages.length - stageResults.length));

    const result = await runAgent(
      { ...stage.agent, tokenBudget: stageBudget },
      stageInput
    );

    stageResults.push(result);
    remainingBudget -= result.tokensUsed.input + result.tokensUsed.output;

    if (result.status !== "success") {
      console.error(`[pipeline] Stage ${stage.agent.id} failed: ${result.error}`);
      // Decision: fail the pipeline or continue with degraded output
      break;
    }

    currentInput = result.output;
  }

  return {
    finalOutput: currentInput,
    stageResults,
  };
}

// Example: Code review pipeline
const codeReviewPipeline: PipelineStage[] = [
  {
    agent: {
      id: "security-reviewer",
      model: "claude-sonnet-4-20250514",
      systemPrompt: "You are a security expert. Review the code for security vulnerabilities. Output a structured report with severity, location, and remediation for each finding.",
      maxTokens: 4096,
    },
  },
  {
    agent: {
      id: "performance-reviewer",
      model: "claude-sonnet-4-20250514",
      systemPrompt: "You are a performance expert. Review the code for performance issues. You also have the security review from a previous stage — reference it if relevant. Output a structured report.",
      maxTokens: 4096,
    },
    transformInput: (securityOutput, originalCode) =>
      `## Original code\n${originalCode}\n\n## Security review\n${securityOutput}\n\nNow review for performance issues.`,
  },
  {
    agent: {
      id: "synthesis-reviewer",
      model: "claude-sonnet-4-20250514",
      systemPrompt: "You are a senior engineer. You receive security and performance reviews. Synthesize them into a single actionable review with prioritized recommendations.",
      maxTokens: 4096,
    },
    transformInput: (performanceOutput, originalCode) =>
      `## Original code\n${originalCode}\n\n## Combined review\n${performanceOutput}\n\nSynthesize into a final actionable review.`,
  },
];
```

### 4.2 Pipeline with Validation Gates

In production, you often want a gate between stages: a check that the previous stage's output meets quality standards before proceeding.

```typescript
// gated-pipeline.ts — Pipeline with quality gates

interface GatedPipelineStage extends PipelineStage {
  validate?: (output: string) => Promise<{
    passed: boolean;
    reason?: string;
    retry?: boolean;
  }>;
  maxRetries?: number;
}

async function executeGatedPipeline(
  input: string,
  stages: GatedPipelineStage[],
  tokenBudget: number
): Promise<{
  finalOutput: string;
  stageResults: AgentResult[];
  gateResults: { stageId: string; passed: boolean; attempts: number }[];
}> {
  const stageResults: AgentResult[] = [];
  const gateResults: { stageId: string; passed: boolean; attempts: number }[] = [];
  let currentInput = input;
  let remainingBudget = tokenBudget;

  for (const stage of stages) {
    const maxRetries = stage.maxRetries ?? 2;
    let attempts = 0;
    let passed = false;

    while (attempts < maxRetries && !passed) {
      attempts++;

      const stageInput = stage.transformInput
        ? stage.transformInput(currentInput, input)
        : currentInput;

      const stageBudget = Math.floor(
        remainingBudget / (stages.length - stageResults.length)
      );

      const result = await runAgent(
        { ...stage.agent, tokenBudget: stageBudget },
        stageInput
      );

      remainingBudget -= result.tokensUsed.input + result.tokensUsed.output;

      if (result.status !== "success") {
        stageResults.push(result);
        break;
      }

      // Run validation gate
      if (stage.validate) {
        const validation = await stage.validate(result.output);
        if (validation.passed) {
          passed = true;
          stageResults.push(result);
          currentInput = result.output;
        } else if (!validation.retry || attempts >= maxRetries) {
          console.warn(
            `[gated-pipeline] Gate failed for ${stage.agent.id}: ${validation.reason}`
          );
          stageResults.push(result);
          currentInput = result.output; // Proceed with best effort
          break;
        }
        // else: retry
      } else {
        passed = true;
        stageResults.push(result);
        currentInput = result.output;
      }
    }

    gateResults.push({ stageId: stage.agent.id, passed, attempts });
  }

  return { finalOutput: currentInput, stageResults, gateResults };
}
```

---

## 5. The Council Pattern

### 5.1 What the Council Pattern Is

Inspired by Andrej Karpathy's suggestion for high-stakes AI decisions: dispatch the same question to multiple models with different perspectives, have them cross-review each other's answers, then synthesize a verdict.

**What it is:** A multi-model deliberation system where each model acts as an "expert" with a specific stance or perspective. After each expert gives their initial opinion, they review each other's opinions. A final synthesis agent (or voting mechanism) produces the decision.

**When to use it:** High-stakes decisions where the cost of a wrong answer far exceeds the cost of running multiple models. Code architecture decisions, security assessments, financial analyses, legal document review.

**Trade-offs:** 3-5x the token cost of a single model call, but dramatically higher quality and reliability for complex judgments.

```
THE COUNCIL PATTERN

              ┌──────────────┐
              │   Question    │
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Expert A │ │Expert B │ │Expert C │
    │(Claude) │ │(GPT-4o) │ │(Gemini) │
    │Advocate │ │Skeptic  │ │Pragmat. │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         ▼           ▼           ▼
    Round 1: Initial opinions
         │           │           │
    ┌────┴───────────┴───────────┴────┐
    │      Cross-review phase         │
    │  Each expert reads all others'  │
    │  opinions and responds          │
    └────┬───────────┬───────────┬────┘
         │           │           │
         ▼           ▼           ▼
    Round 2: Revised opinions
         │           │           │
         └───────────┼───────────┘
                     ▼
              ┌──────────────┐
              │  Synthesizer  │
              │  (verdict)    │
              └──────────────┘
```

### 5.2 Building the Council

```typescript
// council.ts — Multi-model deliberation

interface CouncilMember {
  id: string;
  model: string;
  perspective: string; // Advocacy role: "advocate", "skeptic", "pragmatist"
  systemPrompt: string;
  provider: "anthropic" | "openai" | "google"; // Multi-provider for diversity
}

interface CouncilConfig {
  members: CouncilMember[];
  synthesizer: AgentConfig;
  crossReviewRounds: number; // Usually 1-2
  votingThreshold?: number; // For binary decisions
}

interface CouncilResult {
  verdict: string;
  opinions: {
    memberId: string;
    initialOpinion: string;
    revisedOpinion: string;
    confidence: number;
  }[];
  consensus: "strong" | "moderate" | "weak" | "split";
  totalTokens: { input: number; output: number };
}

async function runCouncil(
  question: string,
  config: CouncilConfig
): Promise<CouncilResult> {
  const totalTokens = { input: 0, output: 0 };

  // ROUND 1: Gather initial opinions
  console.log("[council] Round 1: gathering initial opinions...");
  const initialOpinions = await Promise.all(
    config.members.map(async (member) => {
      const result = await callModel(member, question);
      totalTokens.input += result.tokens.input;
      totalTokens.output += result.tokens.output;
      return { memberId: member.id, opinion: result.text };
    })
  );

  // ROUND 2: Cross-review
  console.log("[council] Round 2: cross-review...");
  const revisedOpinions = await Promise.all(
    config.members.map(async (member) => {
      const othersOpinions = initialOpinions
        .filter((o) => o.memberId !== member.id)
        .map(
          (o) =>
            `### ${o.memberId}'s opinion:\n${o.opinion}`
        )
        .join("\n\n");

      const crossReviewPrompt =
        `You previously answered this question:\n\n"${question}"\n\n` +
        `Your initial opinion:\n${initialOpinions.find((o) => o.memberId === member.id)?.opinion}\n\n` +
        `Other council members' opinions:\n${othersOpinions}\n\n` +
        `Now, considering their perspectives:\n` +
        `1. What do you agree with?\n` +
        `2. What do you disagree with and why?\n` +
        `3. Has your opinion changed? If so, how?\n` +
        `4. What is your revised opinion?\n` +
        `5. Confidence level (0-100)?`;

      const result = await callModel(member, crossReviewPrompt);
      totalTokens.input += result.tokens.input;
      totalTokens.output += result.tokens.output;

      // Extract confidence (simple heuristic)
      const confidenceMatch = result.text.match(/confidence.*?(\d+)/i);
      const confidence = confidenceMatch
        ? parseInt(confidenceMatch[1]) / 100
        : 0.5;

      return {
        memberId: member.id,
        initialOpinion: initialOpinions.find((o) => o.memberId === member.id)!
          .opinion,
        revisedOpinion: result.text,
        confidence,
      };
    })
  );

  // SYNTHESIS: Produce the verdict
  console.log("[council] Synthesizing verdict...");
  const allOpinions = revisedOpinions
    .map(
      (o) =>
        `### ${o.memberId} (confidence: ${Math.round(o.confidence * 100)}%)\n` +
        `Initial: ${o.initialOpinion.slice(0, 500)}...\n` +
        `Revised: ${o.revisedOpinion}`
    )
    .join("\n\n---\n\n");

  const synthesisPrompt =
    `A council of experts deliberated on this question:\n\n"${question}"\n\n` +
    `Their opinions after cross-review:\n\n${allOpinions}\n\n` +
    `Synthesize a final verdict that:\n` +
    `1. Identifies areas of consensus\n` +
    `2. Acknowledges remaining disagreements\n` +
    `3. Provides a clear recommendation\n` +
    `4. Rates the consensus strength: strong/moderate/weak/split`;

  const synthesis = await runAgent(config.synthesizer, synthesisPrompt);
  totalTokens.input += synthesis.tokensUsed.input;
  totalTokens.output += synthesis.tokensUsed.output;

  // Determine consensus level
  const consensusMatch = synthesis.output.match(
    /consensus.*?(strong|moderate|weak|split)/i
  );
  const consensus =
    (consensusMatch?.[1]?.toLowerCase() as CouncilResult["consensus"]) ||
    "moderate";

  return {
    verdict: synthesis.output,
    opinions: revisedOpinions,
    consensus,
    totalTokens,
  };
}

// Model-agnostic call — supports Anthropic, OpenAI, Google
async function callModel(
  member: CouncilMember,
  prompt: string
): Promise<{ text: string; tokens: { input: number; output: number } }> {
  if (member.provider === "anthropic") {
    const client = new Anthropic();
    const response = await client.messages.create({
      model: member.model,
      max_tokens: 4096,
      system: member.systemPrompt,
      messages: [{ role: "user", content: prompt }],
    });
    const text = response.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");
    return {
      text,
      tokens: {
        input: response.usage.input_tokens,
        output: response.usage.output_tokens,
      },
    };
  }

  // OpenAI and Google follow similar patterns
  // Implementation depends on your SDK setup
  throw new Error(`Provider ${member.provider} not implemented in this example`);
}
```

### 5.3 When to Use the Council

| Scenario | Use Council? | Why |
|----------|-------------|-----|
| Code review for security-critical system | Yes | Wrong assessment = breach |
| Generating a marketing email | No | Low stakes, single model is fine |
| Architecture decision for new service | Yes | Wrong architecture = months of rework |
| Classifying customer support tickets | No | High volume, low per-item stakes |
| Evaluating whether to approve a large financial transaction | Yes | Wrong decision = significant loss |
| Summarizing a meeting | No | Low stakes, single model is fine |

**Cost rule of thumb:** If the cost of a wrong answer is more than 100x the cost of running the council, use the council.

---

## 6. Error Handling in Multi-Agent Systems

### 6.1 Failure Modes

Multi-agent systems have failure modes that single agents do not:

**Individual agent failure.** One agent in a fleet errors out. Do you fail the whole operation? Continue without that result? Retry?

**Cascade failure.** Agent A fails, Agent B depends on Agent A's output, Agent C depends on Agent B. One failure brings down the chain.

**Partial result degradation.** Three of five agents succeed. Is the aggregate result trustworthy? How do you communicate partial failure to the user?

**Budget exhaustion.** Early agents consume too many tokens, leaving insufficient budget for later (often more important) agents.

**Timeout imbalance.** One agent takes 10x longer than others, holding up the entire operation.

### 6.2 Error Handling Strategies

```typescript
// error-handling.ts — Strategies for multi-agent failure

type ErrorStrategy = "fail_fast" | "best_effort" | "retry_with_fallback" | "quorum";

interface ErrorConfig {
  strategy: ErrorStrategy;
  maxRetries: number;
  retryDelayMs: number;
  quorumSize?: number; // For quorum strategy: minimum successful results
  fallbackModel?: string; // For retry_with_fallback: cheaper/faster model
}

async function executeWithErrorHandling(
  agents: AgentConfig[],
  tasks: string[],
  errorConfig: ErrorConfig
): Promise<AgentResult[]> {
  switch (errorConfig.strategy) {
    case "fail_fast":
      return executeFailFast(agents, tasks);

    case "best_effort":
      return executeBestEffort(agents, tasks);

    case "retry_with_fallback":
      return executeWithRetry(agents, tasks, errorConfig);

    case "quorum":
      return executeWithQuorum(
        agents,
        tasks,
        errorConfig.quorumSize ?? Math.ceil(agents.length / 2)
      );
  }
}

// Fail fast: any failure cancels everything
async function executeFailFast(
  agents: AgentConfig[],
  tasks: string[]
): Promise<AgentResult[]> {
  const controller = new AbortController();
  const results: AgentResult[] = [];

  try {
    const promises = agents.map(async (agent, i) => {
      const result = await runAgent(agent, tasks[i]);
      if (result.status !== "success") {
        controller.abort();
        throw new Error(`Agent ${agent.id} failed: ${result.error}`);
      }
      results[i] = result;
      return result;
    });

    await Promise.all(promises);
    return results;
  } catch (error) {
    throw new Error(
      `Multi-agent execution failed (fail_fast): ${error}`
    );
  }
}

// Best effort: collect whatever succeeds
async function executeBestEffort(
  agents: AgentConfig[],
  tasks: string[]
): Promise<AgentResult[]> {
  const promises = agents.map((agent, i) =>
    runAgent(agent, tasks[i]).catch(
      (error): AgentResult => ({
        agentId: agent.id,
        status: "error",
        output: "",
        tokensUsed: { input: 0, output: 0 },
        durationMs: 0,
        error: String(error),
      })
    )
  );

  return Promise.all(promises);
}

// Retry with fallback: retry failed agents with a cheaper model
async function executeWithRetry(
  agents: AgentConfig[],
  tasks: string[],
  config: ErrorConfig
): Promise<AgentResult[]> {
  const results = await executeBestEffort(agents, tasks);

  // Retry failed agents
  for (let i = 0; i < results.length; i++) {
    if (results[i].status !== "success" && config.maxRetries > 0) {
      for (let retry = 0; retry < config.maxRetries; retry++) {
        await sleep(config.retryDelayMs * (retry + 1)); // Exponential-ish backoff

        // On last retry, fall back to cheaper model
        const retryAgent =
          retry === config.maxRetries - 1 && config.fallbackModel
            ? { ...agents[i], model: config.fallbackModel }
            : agents[i];

        const retryResult = await runAgent(retryAgent, tasks[i]);
        if (retryResult.status === "success") {
          results[i] = retryResult;
          break;
        }
      }
    }
  }

  return results;
}

// Quorum: succeed if enough agents succeed
async function executeWithQuorum(
  agents: AgentConfig[],
  tasks: string[],
  quorumSize: number
): Promise<AgentResult[]> {
  const results = await executeBestEffort(agents, tasks);

  const successCount = results.filter((r) => r.status === "success").length;

  if (successCount < quorumSize) {
    throw new Error(
      `Quorum not met: ${successCount}/${agents.length} succeeded, ` +
      `needed ${quorumSize}`
    );
  }

  return results;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

### 6.3 Choosing an Error Strategy

| Strategy | Use When | Example |
|----------|---------|---------|
| `fail_fast` | All results are required; partial is worse than none | Financial report generation |
| `best_effort` | Partial results are still useful | Batch customer ticket analysis |
| `retry_with_fallback` | Individual failures are transient, and a cheaper model can fill in | Multi-service health check |
| `quorum` | You need agreement from a majority, not all | Council voting on a decision |

---

## 7. Communication Patterns

### 7.1 Shared Memory

Agents communicate through a shared data structure that any agent can read or write. Simple, fast, but requires careful coordination to avoid conflicts.

```typescript
// shared-memory.ts — Shared state for agent communication

interface SharedMemory {
  data: Map<string, unknown>;
  log: { timestamp: number; agentId: string; key: string; action: "read" | "write" }[];
  locks: Map<string, string>; // key -> agentId holding the lock
}

function createSharedMemory(): SharedMemory {
  return {
    data: new Map(),
    log: [],
    locks: new Map(),
  };
}

async function writeToMemory(
  memory: SharedMemory,
  agentId: string,
  key: string,
  value: unknown
): Promise<boolean> {
  // Simple lock check (in production, use proper distributed locks)
  const lockHolder = memory.locks.get(key);
  if (lockHolder && lockHolder !== agentId) {
    return false; // Key is locked by another agent
  }

  memory.data.set(key, value);
  memory.log.push({
    timestamp: Date.now(),
    agentId,
    key,
    action: "write",
  });
  return true;
}

function readFromMemory(
  memory: SharedMemory,
  agentId: string,
  key: string
): unknown {
  memory.log.push({
    timestamp: Date.now(),
    agentId,
    key,
    action: "read",
  });
  return memory.data.get(key);
}

// Make shared memory available as a tool
function createMemoryTools(memory: SharedMemory, agentId: string): Anthropic.Tool[] {
  return [
    {
      name: "read_shared",
      description: "Read a value from shared memory by key",
      input_schema: {
        type: "object" as const,
        properties: {
          key: { type: "string", description: "The key to read" },
        },
        required: ["key"],
      },
    },
    {
      name: "write_shared",
      description: "Write a value to shared memory",
      input_schema: {
        type: "object" as const,
        properties: {
          key: { type: "string", description: "The key to write" },
          value: { type: "string", description: "The value to store" },
        },
        required: ["key", "value"],
      },
    },
    {
      name: "list_keys",
      description: "List all keys currently in shared memory",
      input_schema: {
        type: "object" as const,
        properties: {},
      },
    },
  ];
}
```

### 7.2 Message Passing

Agents communicate through an explicit message queue. More structured than shared memory, better for agent-to-agent communication.

```typescript
// message-queue.ts — Agent-to-agent messaging

interface AgentMessage {
  id: string;
  from: string;
  to: string;
  type: "request" | "response" | "notification" | "broadcast";
  content: string;
  timestamp: number;
  replyTo?: string; // For response messages
}

class MessageBus {
  private queues: Map<string, AgentMessage[]> = new Map();
  private subscribers: Map<string, ((msg: AgentMessage) => void)[]> = new Map();

  send(message: AgentMessage): void {
    // Add to recipient's queue
    if (!this.queues.has(message.to)) {
      this.queues.set(message.to, []);
    }
    this.queues.get(message.to)!.push(message);

    // Notify subscribers
    const subs = this.subscribers.get(message.to) || [];
    for (const callback of subs) {
      callback(message);
    }
  }

  broadcast(from: string, content: string, exclude?: string[]): void {
    for (const [agentId] of this.queues) {
      if (agentId !== from && !exclude?.includes(agentId)) {
        this.send({
          id: crypto.randomUUID(),
          from,
          to: agentId,
          type: "broadcast",
          content,
          timestamp: Date.now(),
        });
      }
    }
  }

  receive(agentId: string): AgentMessage[] {
    const messages = this.queues.get(agentId) || [];
    this.queues.set(agentId, []); // Clear after reading
    return messages;
  }

  subscribe(agentId: string, callback: (msg: AgentMessage) => void): void {
    if (!this.subscribers.has(agentId)) {
      this.subscribers.set(agentId, []);
    }
    this.subscribers.get(agentId)!.push(callback);
  }

  register(agentId: string): void {
    if (!this.queues.has(agentId)) {
      this.queues.set(agentId, []);
    }
  }
}
```

### 7.3 Event-Driven Coordination

For complex multi-agent systems, an event bus decouples agents completely. Agents publish events, and other agents subscribe to events they care about.

```typescript
// event-bus.ts — Decoupled agent coordination

interface AgentEvent {
  type: string;
  source: string;
  data: unknown;
  timestamp: number;
}

type EventHandler = (event: AgentEvent) => Promise<void>;

class EventBus {
  private handlers: Map<string, EventHandler[]> = new Map();
  private eventLog: AgentEvent[] = [];

  on(eventType: string, handler: EventHandler): void {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType)!.push(handler);
  }

  async emit(event: AgentEvent): Promise<void> {
    this.eventLog.push(event);
    const handlers = this.handlers.get(event.type) || [];
    await Promise.all(handlers.map((h) => h(event)));
  }

  getLog(filter?: { source?: string; type?: string }): AgentEvent[] {
    return this.eventLog.filter(
      (e) =>
        (!filter?.source || e.source === filter.source) &&
        (!filter?.type || e.type === filter.type)
    );
  }
}

// Usage: agents that react to each other
const bus = new EventBus();

// Security agent publishes findings
bus.on("security_finding", async (event) => {
  console.log(`[remediation-agent] Received security finding from ${event.source}`);
  const finding = event.data as { severity: string; location: string };
  if (finding.severity === "critical") {
    // Auto-generate a fix
    const fix = await runAgent(remediationAgent, `Fix this critical security issue: ${JSON.stringify(finding)}`);
    await bus.emit({
      type: "fix_proposed",
      source: "remediation-agent",
      data: { finding, fix: fix.output },
      timestamp: Date.now(),
    });
  }
});
```

---

## 8. Resource Management

### 8.1 Token Budgets Across a Fleet

The most important resource in a multi-agent system is tokens. Without budget management, agents will consume tokens without limit.

```typescript
// budget-manager.ts — Token budget allocation and tracking

interface BudgetAllocation {
  agentId: string;
  allocated: number;
  consumed: number;
  remaining: number;
}

class TokenBudgetManager {
  private totalBudget: number;
  private allocations: Map<string, BudgetAllocation> = new Map();
  private reserved: number = 0; // Tokens reserved for synthesis/overhead

  constructor(totalBudget: number, reservedForOverhead: number = 0) {
    this.totalBudget = totalBudget;
    this.reserved = reservedForOverhead;
  }

  allocate(agentId: string, tokens: number): boolean {
    const currentTotal = this.getTotalAllocated();
    if (currentTotal + tokens > this.totalBudget - this.reserved) {
      return false;
    }

    this.allocations.set(agentId, {
      agentId,
      allocated: tokens,
      consumed: 0,
      remaining: tokens,
    });
    return true;
  }

  allocateEvenly(agentIds: string[]): void {
    const available = this.totalBudget - this.reserved;
    const perAgent = Math.floor(available / agentIds.length);

    for (const id of agentIds) {
      this.allocations.set(id, {
        agentId: id,
        allocated: perAgent,
        consumed: 0,
        remaining: perAgent,
      });
    }
  }

  // Weighted allocation: critical agents get more budget
  allocateWeighted(
    agents: { id: string; weight: number }[]
  ): void {
    const available = this.totalBudget - this.reserved;
    const totalWeight = agents.reduce((sum, a) => sum + a.weight, 0);

    for (const agent of agents) {
      const allocation = Math.floor(
        (agent.weight / totalWeight) * available
      );
      this.allocations.set(agent.id, {
        agentId: agent.id,
        allocated: allocation,
        consumed: 0,
        remaining: allocation,
      });
    }
  }

  recordUsage(agentId: string, tokens: number): void {
    const allocation = this.allocations.get(agentId);
    if (allocation) {
      allocation.consumed += tokens;
      allocation.remaining = allocation.allocated - allocation.consumed;
    }
  }

  canSpend(agentId: string, tokens: number): boolean {
    const allocation = this.allocations.get(agentId);
    return allocation ? allocation.remaining >= tokens : false;
  }

  // Redistribute unused budget from completed agents to remaining ones
  redistribute(completedAgentIds: string[], activeAgentIds: string[]): void {
    let reclaimable = 0;
    for (const id of completedAgentIds) {
      const alloc = this.allocations.get(id);
      if (alloc) {
        reclaimable += alloc.remaining;
        alloc.remaining = 0;
        alloc.allocated = alloc.consumed;
      }
    }

    if (reclaimable > 0 && activeAgentIds.length > 0) {
      const bonus = Math.floor(reclaimable / activeAgentIds.length);
      for (const id of activeAgentIds) {
        const alloc = this.allocations.get(id);
        if (alloc) {
          alloc.allocated += bonus;
          alloc.remaining += bonus;
        }
      }
    }
  }

  getSummary(): {
    total: number;
    allocated: number;
    consumed: number;
    remaining: number;
    agents: BudgetAllocation[];
  } {
    const agents = Array.from(this.allocations.values());
    const consumed = agents.reduce((sum, a) => sum + a.consumed, 0);
    const allocated = agents.reduce((sum, a) => sum + a.allocated, 0);

    return {
      total: this.totalBudget,
      allocated,
      consumed,
      remaining: this.totalBudget - consumed,
      agents,
    };
  }

  private getTotalAllocated(): number {
    return Array.from(this.allocations.values()).reduce(
      (sum, a) => sum + a.allocated,
      0
    );
  }
}
```

### 8.2 Cost Tracking Per Agent

```typescript
// cost-tracker.ts — Per-agent cost tracking for multi-agent systems

interface ModelPricing {
  model: string;
  inputPer1M: number;  // USD per 1M input tokens
  outputPer1M: number; // USD per 1M output tokens
}

const PRICING: ModelPricing[] = [
  { model: "claude-sonnet-4-20250514", inputPer1M: 3, outputPer1M: 15 },
  { model: "claude-opus-4-20250514", inputPer1M: 15, outputPer1M: 75 },
  { model: "claude-haiku-3-5-20241022", inputPer1M: 0.8, outputPer1M: 4 },
  { model: "gpt-4o", inputPer1M: 2.5, outputPer1M: 10 },
  { model: "gpt-4o-mini", inputPer1M: 0.15, outputPer1M: 0.6 },
];

function calculateCost(
  model: string,
  tokens: { input: number; output: number }
): number {
  const pricing = PRICING.find((p) => p.model === model);
  if (!pricing) return 0;

  return (
    (tokens.input / 1_000_000) * pricing.inputPer1M +
    (tokens.output / 1_000_000) * pricing.outputPer1M
  );
}

function generateCostReport(
  results: (AgentResult & { model: string })[]
): string {
  let totalCost = 0;
  const lines = results.map((r) => {
    const cost = calculateCost(r.model, r.tokensUsed);
    totalCost += cost;
    return (
      `  ${r.agentId.padEnd(25)} ` +
      `${r.model.padEnd(30)} ` +
      `in:${r.tokensUsed.input.toString().padStart(8)} ` +
      `out:${r.tokensUsed.output.toString().padStart(8)} ` +
      `$${cost.toFixed(4)}`
    );
  });

  return [
    `Multi-Agent Cost Report`,
    `${"=".repeat(80)}`,
    ...lines,
    `${"=".repeat(80)}`,
    `  TOTAL: $${totalCost.toFixed(4)}`,
  ].join("\n");
}
```

---

## 9. Putting It All Together: A Complete Multi-Agent System

### 9.1 The Codebase Analyzer

Let's build a real system: a multi-agent codebase analyzer that takes a question about a codebase and answers it using specialized agents working in parallel.

```typescript
// codebase-analyzer.ts — Complete multi-agent system

interface AnalysisRequest {
  question: string;
  codebasePath: string;
  maxBudgetUSD: number;
}

async function analyzeCodebase(request: AnalysisRequest): Promise<string> {
  // Token budget from dollar budget (rough estimate using Sonnet pricing)
  const estimatedTokenBudget = Math.floor(
    (request.maxBudgetUSD / 3) * 1_000_000 // $3/M input as baseline
  );

  const budget = new TokenBudgetManager(estimatedTokenBudget, 50000); // Reserve 50K for synthesis

  // Define specialist workers
  const workers: AgentConfig[] = [
    {
      id: "architecture-analyst",
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You are a software architecture expert. Analyze code structure,
        dependency graphs, module boundaries, and architectural patterns.
        Be specific — cite file paths and function names.`,
      maxTokens: 4096,
    },
    {
      id: "security-analyst",
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You are a security expert. Look for vulnerabilities: injection,
        auth bypass, data exposure, dependency risks. Rate each finding by severity.
        Be specific — cite exact lines and files.`,
      maxTokens: 4096,
    },
    {
      id: "performance-analyst",
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You are a performance engineering expert. Identify bottlenecks:
        N+1 queries, unbounded loops, missing caching, memory leaks,
        synchronous operations that should be async. Estimate impact.`,
      maxTokens: 4096,
    },
    {
      id: "quality-analyst",
      model: "claude-sonnet-4-20250514",
      systemPrompt: `You are a code quality expert. Assess test coverage,
        error handling, documentation, naming conventions, complexity metrics.
        Identify areas of technical debt. Prioritize by business impact.`,
      maxTokens: 4096,
    },
  ];

  // Allocate budget weighted by typical importance
  budget.allocateWeighted([
    { id: "architecture-analyst", weight: 3 },
    { id: "security-analyst", weight: 3 },
    { id: "performance-analyst", weight: 2 },
    { id: "quality-analyst", weight: 2 },
  ]);

  // Add file reading tools to each worker
  const fileTools = createFileReadingTools(request.codebasePath);
  const workersWithTools = workers.map((w) => ({
    ...w,
    tools: fileTools,
    tokenBudget: budget.allocations?.get(w.id)?.allocated,
  }));

  // Run all analysts in parallel
  console.log("[analyzer] Running specialist analyses in parallel...");
  const results = await executeParallel(
    workersWithTools.map((w) => ({
      id: w.id,
      description: `${request.question}\n\nAnalyze the codebase at ${request.codebasePath} from your specialist perspective.`,
      assignTo: w.id,
      dependencies: [],
    })),
    workersWithTools,
    estimatedTokenBudget - 50000,
    120000 // 2 minute timeout
  );

  // Track costs
  for (const result of results) {
    budget.recordUsage(
      result.agentId,
      result.tokensUsed.input + result.tokensUsed.output
    );
  }

  // Synthesize
  console.log("[analyzer] Synthesizing findings...");
  const synthesizer: AgentConfig = {
    id: "synthesizer",
    model: "claude-sonnet-4-20250514",
    systemPrompt: `You are a principal engineer synthesizing reports from specialist
      analysts. Produce a single coherent analysis with:
      1. Executive summary (3-5 bullets)
      2. Critical findings (prioritized)
      3. Recommended actions (prioritized)
      4. Areas where analysts disagreed
      Cross-reference findings across analysts when relevant.`,
    maxTokens: 8192,
  };

  const successfulResults = results.filter((r) => r.status === "success");
  const synthesisInput = successfulResults
    .map((r) => `## ${r.agentId}\n${r.output}`)
    .join("\n\n---\n\n");

  const synthesis = await runAgent(
    synthesizer,
    `Question: ${request.question}\n\nSpecialist Reports:\n${synthesisInput}`
  );

  // Print cost report
  const costReport = generateCostReport(
    results.map((r) => ({
      ...r,
      model: "claude-sonnet-4-20250514",
    }))
  );
  console.log(costReport);

  return synthesis.output;
}
```

### 9.2 File Reading Tools for Agents

```typescript
// file-tools.ts — Give agents access to the filesystem (sandboxed)

import { readFile, readdir, stat } from "fs/promises";
import { join, resolve, relative } from "path";

function createFileReadingTools(basePath: string): Anthropic.Tool[] {
  return [
    {
      name: "read_file",
      description: "Read the contents of a file. Returns the full text content.",
      input_schema: {
        type: "object" as const,
        properties: {
          path: {
            type: "string",
            description: "Relative path from the project root",
          },
        },
        required: ["path"],
      },
    },
    {
      name: "list_directory",
      description: "List files and directories at a path. Returns names with type indicators.",
      input_schema: {
        type: "object" as const,
        properties: {
          path: {
            type: "string",
            description: "Relative path from the project root (use '.' for root)",
          },
        },
        required: ["path"],
      },
    },
    {
      name: "search_files",
      description: "Search for files matching a glob pattern. Returns matching file paths.",
      input_schema: {
        type: "object" as const,
        properties: {
          pattern: {
            type: "string",
            description: "Glob pattern like '**/*.ts' or 'src/**/*.py'",
          },
        },
        required: ["pattern"],
      },
    },
  ];
}
```

---

## 10. Multi-Agent Orchestration in Python

### 10.1 The Same Pattern, Different Language

For ML-heavy workloads, here is the core orchestration in Python.

```python
# orchestrator.py — Multi-agent orchestration in Python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Optional
import anthropic

client = anthropic.Anthropic()


@dataclass
class AgentConfig:
    id: str
    model: str
    system_prompt: str
    max_tokens: int = 4096
    token_budget: Optional[int] = None


@dataclass
class AgentResult:
    agent_id: str
    status: str  # "success", "error", "timeout", "budget_exceeded"
    output: str
    tokens_used: dict = field(default_factory=lambda: {"input": 0, "output": 0})
    duration_ms: float = 0
    error: Optional[str] = None


async def run_agent(config: AgentConfig, task: str, context: str = "") -> AgentResult:
    """Run a single agent loop to completion."""
    start = time.time()
    tokens = {"input": 0, "output": 0}

    user_message = task
    if context:
        user_message = f"## Context\n{context}\n\n## Your task\n{task}"

    messages = [{"role": "user", "content": user_message}]

    for _ in range(20):  # Max iterations
        if config.token_budget and sum(tokens.values()) > config.token_budget:
            return AgentResult(
                agent_id=config.id,
                status="budget_exceeded",
                output=f"Budget exceeded ({config.token_budget} tokens)",
                tokens_used=tokens,
                duration_ms=(time.time() - start) * 1000,
            )

        try:
            response = client.messages.create(
                model=config.model,
                max_tokens=config.max_tokens,
                system=config.system_prompt,
                messages=messages,
            )

            tokens["input"] += response.usage.input_tokens
            tokens["output"] += response.usage.output_tokens

            # Check if agent is done
            if response.stop_reason == "end_turn":
                text = "".join(
                    b.text for b in response.content if b.type == "text"
                )
                return AgentResult(
                    agent_id=config.id,
                    status="success",
                    output=text,
                    tokens_used=tokens,
                    duration_ms=(time.time() - start) * 1000,
                )

            # Handle tool use (simplified — add your tool registry)
            messages.append({"role": "assistant", "content": response.content})
            # ... execute tools and append results ...

        except Exception as e:
            return AgentResult(
                agent_id=config.id,
                status="error",
                output="",
                tokens_used=tokens,
                duration_ms=(time.time() - start) * 1000,
                error=str(e),
            )

    return AgentResult(
        agent_id=config.id,
        status="error",
        output="Max iterations exceeded",
        tokens_used=tokens,
        duration_ms=(time.time() - start) * 1000,
    )


async def run_parallel(
    agents: list[AgentConfig],
    tasks: list[str],
    timeout_seconds: float = 120,
) -> list[AgentResult]:
    """Run multiple agents in parallel with timeout."""

    async def run_with_timeout(agent: AgentConfig, task: str) -> AgentResult:
        try:
            return await asyncio.wait_for(
                run_agent(agent, task),
                timeout=timeout_seconds,
            )
        except asyncio.TimeoutError:
            return AgentResult(
                agent_id=agent.id,
                status="timeout",
                output="",
                error=f"Timed out after {timeout_seconds}s",
            )

    return await asyncio.gather(
        *[run_with_timeout(a, t) for a, t in zip(agents, tasks)]
    )


async def run_council(
    question: str,
    members: list[AgentConfig],
    synthesizer: AgentConfig,
) -> str:
    """Run the council pattern: independent opinions, cross-review, synthesis."""

    # Round 1: Independent opinions
    opinions = await run_parallel(
        members,
        [question] * len(members),
    )

    # Round 2: Cross-review
    review_tasks = []
    for i, member in enumerate(members):
        others = [
            f"**{o.agent_id}**: {o.output[:1000]}"
            for j, o in enumerate(opinions)
            if j != i and o.status == "success"
        ]
        review_task = (
            f"Your initial opinion on '{question}':\n{opinions[i].output}\n\n"
            f"Other experts' opinions:\n{''.join(others)}\n\n"
            f"Revise your opinion. What changed? Confidence (0-100)?"
        )
        review_tasks.append(review_task)

    revised = await run_parallel(members, review_tasks)

    # Synthesis
    all_views = "\n---\n".join(
        f"**{r.agent_id}**: {r.output}" for r in revised if r.status == "success"
    )
    synthesis = await run_agent(
        synthesizer,
        f"Synthesize these expert opinions on '{question}':\n\n{all_views}",
    )

    return synthesis.output
```

---

## 11. Production Considerations

### 11.1 Observability for Multi-Agent Systems

You cannot debug a multi-agent system without tracing. Every agent needs a trace ID that connects to the parent operation.

```typescript
// observability.ts — Tracing for multi-agent systems

interface AgentSpan {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  agentId: string;
  operation: string;
  startTime: number;
  endTime?: number;
  status?: string;
  attributes: Record<string, string | number>;
}

class MultiAgentTracer {
  private spans: AgentSpan[] = [];
  private traceId: string;

  constructor() {
    this.traceId = crypto.randomUUID();
  }

  startSpan(
    agentId: string,
    operation: string,
    parentSpanId?: string
  ): AgentSpan {
    const span: AgentSpan = {
      traceId: this.traceId,
      spanId: crypto.randomUUID(),
      parentSpanId,
      agentId,
      operation,
      startTime: Date.now(),
      attributes: {},
    };
    this.spans.push(span);
    return span;
  }

  endSpan(span: AgentSpan, status: string): void {
    span.endTime = Date.now();
    span.status = status;
  }

  // Export as OpenTelemetry-compatible format
  export(): object[] {
    return this.spans.map((s) => ({
      traceId: s.traceId,
      spanId: s.spanId,
      parentSpanId: s.parentSpanId,
      name: `${s.agentId}/${s.operation}`,
      startTimeUnixNano: s.startTime * 1_000_000,
      endTimeUnixNano: (s.endTime || Date.now()) * 1_000_000,
      status: { code: s.status === "success" ? 1 : 2 },
      attributes: Object.entries(s.attributes).map(([k, v]) => ({
        key: k,
        value: { stringValue: String(v) },
      })),
    }));
  }

  // Pretty print for debugging
  printTree(): string {
    const roots = this.spans.filter((s) => !s.parentSpanId);
    const lines: string[] = [];

    function printSpan(span: AgentSpan, depth: number, spans: AgentSpan[]): void {
      const indent = "  ".repeat(depth);
      const duration = span.endTime
        ? `${span.endTime - span.startTime}ms`
        : "running";
      lines.push(
        `${indent}[${span.status || "?"}] ${span.agentId}/${span.operation} (${duration})`
      );
      const children = spans.filter((s) => s.parentSpanId === span.spanId);
      for (const child of children) {
        printSpan(child, depth + 1, spans);
      }
    }

    for (const root of roots) {
      printSpan(root, 0, this.spans);
    }

    return lines.join("\n");
  }
}
```

### 11.2 When to Use Multi-Agent vs. Single Agent

```
USE SINGLE AGENT WHEN:
├── Task fits in one context window
├── Task requires one type of expertise
├── Latency matters more than thoroughness
├── Budget is tight (multi-agent = multi-cost)
└── Task is conversational (needs memory continuity)

USE MULTI-AGENT WHEN:
├── Task exceeds one context window
├── Task benefits from multiple perspectives (council)
├── Throughput matters (batch processing)
├── Task has clear decomposition into subtasks
├── Different subtasks need different tools
└── You need fault isolation (one failure should not kill everything)
```

### 11.3 Anti-Patterns

**The chatty agents anti-pattern.** Agents that exchange 20 messages to coordinate something that a single agent could do in one pass. Communication has a cost.

**The god coordinator anti-pattern.** A coordinator that micromanages every step instead of delegating meaningfully. If the coordinator's instructions are as detailed as the work itself, you do not need workers.

**The infinite loop anti-pattern.** Two agents that keep requesting clarification from each other forever. Always set maximum iterations and timeouts.

**The homogeneous council anti-pattern.** Running the council pattern with three instances of the same model and same system prompt. You get three similar opinions, not genuine diversity. Use different models, different temperatures, or different perspective prompts.

---

## 12. Chapter Summary

Multi-agent orchestration is distributed systems engineering applied to AI. The fundamental patterns are parent-child (hierarchical coordination), peer-to-peer (direct communication), and pipeline (sequential processing). The council pattern adds adversarial deliberation for high-stakes decisions. Every multi-agent system needs explicit error handling, token budget management, and observability.

The key insight from this chapter: multi-agent systems are not about making agents smarter. They are about making *work* decomposable. If you can break a task into independent subtasks, you can parallelize it. If you can identify distinct expertise areas, you can specialize agents. If you can define quality gates between stages, you can pipeline them.

In Chapter 59, we take this orchestration capability and package it inside an internal tool that anyone in your organization can use.

---

*Previous: [Ch 57 — Scaling & Cost at the Infra Level](../part-11-deployment-infrastructure/57-scaling-cost-infra.md)* | *Next: [Ch 59 — Building Internal AI Tools](./59-internal-ai-tools.md)*

<!--
  CHAPTER: 32
  TITLE: Multi-Agent Coordination
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 9 (Agent Loop), Ch 26 (AI-Augmented Development), Ch 27 (Harness Architecture)
  KEY_TOPICS: coordinator mode, tick loop, SleepTool, background agents, SendUserMessage, worktrees, parallel agents, Stripe Minions, delegation patterns
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 32: Multi-Agent Coordination

> **Part 6 — Anatomy of AI Developer Tools** | Phase 2: Become an Expert | Prerequisites: Ch 9, Ch 26, Ch 27 | Difficulty: Advanced | Language: TypeScript + Architecture

A single agent working on a single task is straightforward. But production work often requires multiple things happening at once: running tests while editing code, searching the web while reading files, processing different features in parallel. The Claude Code leak revealed sophisticated coordination mechanisms: a tick loop that uses `setTimeout(0)` to give the agent opportunities to act during idle time, a SleepTool that lets agents pace themselves, a SendUserMessage system for background agents to communicate, and worktree-based isolation that enables parallel agents on the same codebase without merge conflicts.

This chapter dissects how one agent becomes many, how they communicate, and how they stay out of each other's way.

### In This Chapter

1. Why multi-agent coordination matters
2. Coordinator mode: spawning parallel workers
3. The tick loop: idle-time agency
4. SleepTool: pacing decisions
5. The 15-second blocking budget
6. SendUserMessage: background agent communication
7. Worktrees: parallel agents on the same repo
8. Stripe's Minions: one-shot agents at scale
9. The delegation pattern: Claude to Devin via tickets
10. Advanced delegation architecture: dependency-aware planning and session resume
11. Designing multi-agent systems for your own tools

### Related Chapters

- **Ch 9 (The Agent Loop)** -- introduced the single loop; this chapter shows many loops coordinating
- **Ch 26 (AI-Augmented Development)** -- introduced coding agents; this chapter shows their coordination layer
- **Ch 27 (Harness Architecture)** -- showed the single-agent harness; this chapter extends it
- **Ch 58 (Multi-Agent Orchestration)** -- will scale these patterns to platform-level orchestration

---

## 1. Why Multi-Agent Coordination Matters

### 1.1 The Limitations of a Single Agent

A single agent loop has inherent limitations:

```
┌────────────────────────────────────────────────────┐
│          SINGLE AGENT LIMITATIONS                   │
│                                                      │
│  SERIAL EXECUTION                                    │
│  The agent does one thing at a time:                 │
│  read file -> edit file -> run tests -> fix bug      │
│  -> run tests -> fix next bug -> ...                 │
│  Total time: sum of all operations                   │
│                                                      │
│  SINGLE CONTEXT                                      │
│  Everything shares one context window:               │
│  All file reads, all tool results, all search        │
│  results compete for the same limited space          │
│                                                      │
│  SINGLE FOCUS                                        │
│  Cannot work on multiple independent tasks:          │
│  Cannot fix bug A while investigating bug B          │
│  Cannot run tests while writing new code             │
│  Cannot search docs while editing a file             │
│                                                      │
│  BLOCKING                                            │
│  Long-running operations block everything:           │
│  While `npm install` runs (60s), agent does nothing  │
│  While tests run (30s), agent waits                  │
│  While a build runs (120s), everything stalls        │
└────────────────────────────────────────────────────┘
```

### 1.2 The Multi-Agent Solution

Multiple agents solve these problems through parallelism and isolation:

```
┌────────────────────────────────────────────────────────┐
│          MULTI-AGENT ADVANTAGES                         │
│                                                         │
│  PARALLEL EXECUTION                                     │
│  Agent 1: fix bug A                                     │
│  Agent 2: fix bug B                                     │
│  Agent 3: write tests                                   │
│  Total time: max of all operations (not sum)            │
│                                                         │
│  ISOLATED CONTEXTS                                      │
│  Each agent has its own context window:                  │
│  Agent 1 reads auth files (its context)                 │
│  Agent 2 reads database files (its context)             │
│  No context pollution between agents                    │
│                                                         │
│  SPECIALIZED FOCUS                                      │
│  Each agent works on exactly one thing:                  │
│  Agent 1: expert on the auth system                     │
│  Agent 2: expert on the database layer                  │
│  Agent 3: expert on writing tests                       │
│                                                         │
│  NON-BLOCKING                                           │
│  Long operations run in background agents:               │
│  Parent agent continues working while                    │
│  background agent runs tests                            │
└────────────────────────────────────────────────────────┘
```

---

## 2. Coordinator Mode: Spawning Parallel Workers

### 2.1 The Architecture

Claude Code's coordinator mode (also called the Agent tool) spawns sub-agents that operate independently:

```
┌─────────────────────────────────────────────────────┐
│                COORDINATOR MODE                      │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │           PARENT AGENT (Coordinator)          │    │
│  │  - Receives the user's task                   │    │
│  │  - Decomposes into sub-tasks                  │    │
│  │  - Spawns worker agents                       │    │
│  │  - Collects results                           │    │
│  │  - Synthesizes final response                 │    │
│  └──────┬──────────┬──────────┬─────────────────┘    │
│         │          │          │                       │
│         ▼          ▼          ▼                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Worker 1 │ │ Worker 2 │ │ Worker 3 │             │
│  │          │ │          │ │          │             │
│  │ Own      │ │ Own      │ │ Own      │             │
│  │ context  │ │ context  │ │ context  │             │
│  │ Own      │ │ Own      │ │ Own      │             │
│  │ tools    │ │ tools    │ │ tools    │             │
│  │ Own      │ │ Own      │ │ Own      │             │
│  │ cache    │ │ cache    │ │ cache    │             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                      │
│  Workers run in parallel, each with:                 │
│  - Independent context window                        │
│  - Independent prompt cache                          │
│  - Subset of tools (limited by coordinator)          │
│  - Scoped file access                                │
│  - Their own compaction lifecycle                    │
└─────────────────────────────────────────────────────┘
```

### 2.2 How the Agent Tool Works

```typescript
// The Agent tool spawns a sub-agent
const AGENT_TOOL = {
  name: 'Agent',
  description: `Spawn a sub-agent to handle a specific task.
The sub-agent has its own context window and tool access.
Use when: the task is independent, benefits from focused 
context, or you need parallel execution.
Do NOT use when: the task requires the current conversation
context or user interaction.`,
  
  input_schema: {
    type: 'object',
    properties: {
      prompt: {
        type: 'string',
        description: 'The task description for the sub-agent'
      },
      tools: {
        type: 'array',
        items: { type: 'string' },
        description: 'Tools the sub-agent can use (default: Read, Glob, Grep)'
      }
    },
    required: ['prompt']
  }
};

// Internal implementation
async function executeAgentTool(
  input: { prompt: string; tools?: string[] },
  parentSession: Session
): Promise<string> {
  // Create a new session for the sub-agent
  const subSession = createSubSession({
    parentId: parentSession.id,
    model: parentSession.config.model,  // Same model (preserves cache patterns)
    
    // Sub-agent gets a subset of tools
    tools: input.tools || ['Read', 'Glob', 'Grep'],
    
    // Sub-agent inherits project memory
    claudeMd: parentSession.config.claudeMd,
    
    // Sub-agent gets its own context window
    messages: [],
    
    // Sub-agent cannot spawn further sub-agents (depth limit)
    maxDepth: parentSession.depth + 1,
    depthLimit: 3,
  });
  
  // Run the sub-agent's loop
  const result = await agentLoop(subSession, input.prompt);
  
  // Return the sub-agent's final response to the parent
  return result;
}
```

### 2.3 Worker Tool Restrictions

Sub-agents typically get a restricted set of tools:

```typescript
// Default tool sets for sub-agents
const SUB_AGENT_TOOL_PRESETS = {
  // Research agent: read-only
  research: ['Read', 'Glob', 'Grep', 'WebSearch', 'WebFetch'],
  
  // Editor agent: can modify files
  editor: ['Read', 'Write', 'Edit', 'Glob', 'Grep'],
  
  // Test agent: can run commands
  test: ['Read', 'Glob', 'Grep', 'Bash'],
  
  // Full agent: everything except spawning more agents
  full: ['Read', 'Write', 'Edit', 'Glob', 'Grep', 'Bash',
         'WebSearch', 'WebFetch'],
  
  // Default (most restrictive)
  default: ['Read', 'Glob', 'Grep'],
};
```

The restriction is deliberate: sub-agents are less trusted than the parent because they have less context. Limiting their tools reduces the blast radius of mistakes.

### 2.4 When to Spawn Sub-Agents

```
┌──────────────────────────────────────────────────┐
│          WHEN TO USE SUB-AGENTS                   │
│                                                    │
│  GOOD USE CASES:                                   │
│  ├── Search multiple files in parallel             │
│  │   "Find all usages of deprecated API"           │
│  ├── Independent investigations                    │
│  │   "Check if module A and module B both work"    │
│  ├── Focused deep-dives                            │
│  │   "Analyze the entire test suite"               │
│  ├── Different model for different tasks           │
│  │   Parent: Opus for complex reasoning            │
│  │   Sub-agent: Haiku for simple searches          │
│  └── Avoiding context pollution                    │
│      "Read that 2000-line file without             │
│       filling up my context"                       │
│                                                    │
│  BAD USE CASES:                                    │
│  ├── Tasks that need current conversation context  │
│  ├── Tasks that need user interaction              │
│  ├── Trivially small tasks (overhead > benefit)    │
│  └── Tasks that depend on each other's output      │
└──────────────────────────────────────────────────┘
```

---

## 3. The Tick Loop: Idle-Time Agency

### 3.1 What the Tick Loop Is

The leak revealed an unusual pattern: Claude Code uses `setTimeout(0)` to inject "tick" events into the event loop during idle periods. This gives the agent opportunities to act even when the user has not sent a message.

```typescript
// The tick loop (reconstructed from the leak)
class TickLoop {
  private running: boolean = false;
  private session: Session;
  
  start(session: Session): void {
    this.session = session;
    this.running = true;
    this.tick();
  }
  
  stop(): void {
    this.running = false;
  }
  
  private tick(): void {
    if (!this.running) return;
    
    // Schedule the next tick using setTimeout(0)
    // This runs after all pending I/O is processed
    setTimeout(() => {
      this.processTick();
      if (this.running) {
        this.tick();  // Schedule the next tick
      }
    }, 0);
  }
  
  private async processTick(): Promise<void> {
    // Inject a <tick> marker into the agent's context
    // The agent decides whether to act or sleep
    
    const tickMessage = {
      role: 'system' as const,
      content: '<tick>'
    };
    
    // The agent sees the tick and decides:
    // 1. Is there background work to do?
    // 2. Are there pending operations to check?
    // 3. Should I just sleep?
    
    const decision = await this.session.agent.decideTick({
      pendingOperations: this.session.getPendingOps(),
      backgroundTasks: this.session.getBackgroundTasks(),
      timeSinceLastActivity: this.session.getIdleTime(),
    });
    
    switch (decision.type) {
      case 'act':
        // Agent wants to do something
        await this.session.executeAction(decision.action);
        break;
      case 'sleep':
        // Agent has nothing to do -- use SleepTool
        break;
      case 'check':
        // Agent wants to check on a background operation
        await this.session.checkOperation(decision.operationId);
        break;
    }
  }
}
```

### 3.2 Why setTimeout(0)?

`setTimeout(0)` does not actually wait 0 milliseconds. It schedules the callback to run after the current event loop cycle completes and all pending I/O is processed. This means:

```
Event Loop:
┌────────────────────────────────────────────────┐
│  1. Process user input (if any)                │
│  2. Process network responses (if any)         │
│  3. Process file I/O completions (if any)      │
│  4. Process timers (including setTimeout(0))   │
│     └── Tick fires HERE                        │
│  5. Back to 1                                  │
└────────────────────────────────────────────────┘
```

The tick fires AFTER all real work is processed, making it a "do this when idle" mechanism. This prevents the tick from blocking real operations.

### 3.3 What the Agent Does During Ticks

During tick events, the agent might:

```
┌──────────────────────────────────────────────────┐
│         TICK LOOP ACTIVITIES                      │
│                                                    │
│  CHECK BACKGROUND OPERATIONS                       │
│  ├── Has the test run finished?                    │
│  ├── Has the build completed?                      │
│  ├── Has the long-running command returned?         │
│  └── Has the search finished?                      │
│                                                    │
│  PROACTIVE ACTIONS                                 │
│  ├── Pre-read files the agent expects to need      │
│  ├── Pre-index project structure                   │
│  ├── Check for file changes since last activity    │
│  └── Update memory with recent observations        │
│                                                    │
│  BACKGROUND AGENT COMMUNICATION                    │
│  ├── Check if a sub-agent has sent a message       │
│  ├── Process sub-agent results                     │
│  └── Send status updates to the user               │
│                                                    │
│  NOTHING (most common)                             │
│  └── SleepTool: agent decides to do nothing        │
└──────────────────────────────────────────────────┘
```

---

## 4. SleepTool: Pacing Decisions

### 4.1 What SleepTool Does

SleepTool lets the agent explicitly decide to not act during a tick. This might seem trivial, but it solves a real problem: the cost-vs-cache balance.

```typescript
// SleepTool (reconstructed)
const SLEEP_TOOL = {
  name: 'SleepTool',
  description: `Pause for a specified duration. Use when:
- Waiting for a background operation to complete
- No immediate action is needed
- You want to check back after a delay

The cache expires after ~5 minutes of inactivity.
Consider sleeping for less than 5 minutes to preserve
the prompt cache.`,
  
  input_schema: {
    type: 'object',
    properties: {
      duration_ms: {
        type: 'number',
        description: 'How long to sleep in milliseconds'
      },
      reason: {
        type: 'string',
        description: 'Why sleeping (for logging)'
      }
    },
    required: ['duration_ms']
  }
};
```

### 4.2 The Cache-vs-Cost Balance

The SleepTool creates an interesting optimization problem:

```
┌────────────────────────────────────────────────────────┐
│           THE CACHE-COST BALANCE                        │
│                                                         │
│  If the agent acts during every tick:                   │
│  + Prompt cache stays warm (fast, cheap API calls)      │
│  - Uses API calls even when there's nothing to do       │
│  - Costs money for no-op operations                     │
│                                                         │
│  If the agent sleeps for too long:                      │
│  + No wasted API calls during sleep                     │
│  - Prompt cache expires (~5 min TTL)                    │
│  - Next real call pays full price (cache miss)          │
│                                                         │
│  OPTIMAL STRATEGY:                                      │
│  Sleep for less than the cache TTL                      │
│  Wake up, do a minimal operation to refresh cache       │
│  Sleep again                                            │
│                                                         │
│  Cost of cache-keeping operation: ~$0.001               │
│  Cost of cache miss: ~$0.02                             │
│  Break-even: keep cache alive if you expect to need     │
│  it within the next 20 cache-keeping operations         │
└────────────────────────────────────────────────────────┘
```

### 4.3 Implementation Pattern

```typescript
// Smart sleep with cache management
async function smartSleep(
  session: Session,
  waitForCondition: () => Promise<boolean>,
  options: {
    checkInterval: number;      // How often to check
    maxWait: number;            // Maximum wait time
    keepCacheAlive: boolean;    // Whether to ping to keep cache
    cacheTTL: number;           // Cache expiration time
  }
): Promise<boolean> {
  const startTime = Date.now();
  
  while (Date.now() - startTime < options.maxWait) {
    // Check if the condition is met
    if (await waitForCondition()) {
      return true;  // Condition met, stop sleeping
    }
    
    if (options.keepCacheAlive) {
      // Calculate time until cache expires
      const timeSinceLastCall =
        Date.now() - session.lastApiCallTime;
      
      if (timeSinceLastCall > options.cacheTTL * 0.8) {
        // Cache is about to expire -- do a minimal API call
        await session.sendKeepAlive();
      }
    }
    
    // Sleep for the check interval
    await new Promise(r =>
      setTimeout(r, options.checkInterval)
    );
  }
  
  return false;  // Timed out
}
```

---

## 5. The 15-Second Blocking Budget

### 5.1 What It Is

Claude Code auto-backgrounds any operation that takes longer than approximately 15 seconds:

```
┌────────────────────────────────────────────────────────┐
│           THE 15-SECOND BLOCKING BUDGET                 │
│                                                         │
│  Operation starts                                       │
│  │                                                      │
│  │  < 15 seconds: runs in foreground                    │
│  │  User sees output in real-time                       │
│  │  Agent waits for completion                          │
│  │                                                      │
│  │  > 15 seconds: auto-backgrounded                     │
│  │  Agent continues working                             │
│  │  Background task runs independently                  │
│  │  Agent checks on it via tick loop                    │
│  │  User notified when it completes                     │
│  │                                                      │
│  ▼                                                      │
└────────────────────────────────────────────────────────┘
```

### 5.2 Implementation

```typescript
// Auto-backgrounding long operations
async function executeWithBackgrounding(
  operation: () => Promise<string>,
  timeout: number = 15000
): Promise<string | BackgroundHandle> {
  // Start the operation
  const startTime = Date.now();
  
  // Race: operation vs timeout
  const result = await Promise.race([
    operation().then(r => ({ type: 'complete' as const, result: r })),
    sleep(timeout).then(() => ({ type: 'timeout' as const })),
  ]);
  
  if (result.type === 'complete') {
    // Finished within budget -- return directly
    return result.result;
  }
  
  // Operation exceeded budget -- background it
  const handle = new BackgroundHandle();
  
  // The operation continues running
  operation().then(
    (result) => handle.resolve(result),
    (error) => handle.reject(error)
  );
  
  return handle;
}

class BackgroundHandle {
  private _resolve: (value: string) => void;
  private _reject: (error: Error) => void;
  private _promise: Promise<string>;
  public completed: boolean = false;
  public result: string | null = null;
  
  constructor() {
    this._promise = new Promise((resolve, reject) => {
      this._resolve = resolve;
      this._reject = reject;
    });
    this._promise.then(r => {
      this.completed = true;
      this.result = r;
    });
  }
  
  resolve(value: string): void { this._resolve(value); }
  reject(error: Error): void { this._reject(error); }
  
  async wait(): Promise<string> { return this._promise; }
}
```

### 5.3 Why 15 Seconds

The 15-second threshold balances two concerns:

```
Too short (e.g., 2 seconds):
  - Most npm install commands would be backgrounded
  - Simple test runs would be backgrounded
  - Agent would lose visibility into common operations
  - More complexity, less transparency

Too long (e.g., 60 seconds):
  - Agent blocks for a full minute on slow builds
  - User stares at a spinner
  - Wasted time when the agent could be doing other things

15 seconds (sweet spot):
  - Most quick operations complete in time
  - Only genuinely long operations are backgrounded
  - User does not wait too long
  - Agent stays productive
```

---

## 6. SendUserMessage: Background Agent Communication

### 6.1 The Problem

Background agents need to communicate with the user, but they cannot interrupt the foreground agent's conversation. SendUserMessage is the messaging layer that solves this:

```
┌───────────────────────────────────────────────────────┐
│          BACKGROUND AGENT COMMUNICATION                │
│                                                        │
│  WITHOUT SendUserMessage:                              │
│  ┌──────────┐     ┌──────────┐                        │
│  │ Parent   │     │Background│                        │
│  │ Agent    │     │ Agent    │                        │
│  └──────────┘     └──────────┘                        │
│  No connection. Background agent's results             │
│  are lost or delayed until it completes.               │
│                                                        │
│  WITH SendUserMessage:                                 │
│  ┌──────────┐ <─── ┌──────────┐                       │
│  │ Parent   │ msgs │Background│                       │
│  │ Agent    │ ───> │ Agent    │                       │
│  └──────────┘      └──────────┘                       │
│  Background agent can send status updates,             │
│  ask questions, and report results.                    │
└───────────────────────────────────────────────────────┘
```

### 6.2 The Three-Tier UI Filtering

Not all background messages are shown to the user. The leak revealed a three-tier filtering system:

```typescript
// Message filtering (reconstructed)
interface BackgroundMessage {
  type: 'status' | 'question' | 'result' | 'error';
  priority: 'low' | 'medium' | 'high';
  content: string;
  agentId: string;
  timestamp: Date;
}

function shouldShowToUser(message: BackgroundMessage): boolean {
  // Tier 1: Always shown (high priority)
  if (message.priority === 'high') return true;
  if (message.type === 'error') return true;
  if (message.type === 'question') return true;
  
  // Tier 2: Shown if user has opted in (medium priority)
  if (message.priority === 'medium') {
    return userPreferences.showBackgroundUpdates;
  }
  
  // Tier 3: Logged but not shown (low priority)
  // Progress updates, intermediate results, etc.
  return false;
}
```

The three tiers:

```
┌──────────────────────────────────────────────────┐
│      BACKGROUND MESSAGE TIERS                     │
│                                                    │
│  TIER 1: ALWAYS SHOWN                              │
│  ├── Errors ("Test suite failed")                  │
│  ├── Questions ("Which database should I use?")    │
│  ├── Completion ("All tasks done, PR created")     │
│  └── Critical status ("Build broken")              │
│                                                    │
│  TIER 2: USER-CONFIGURABLE                         │
│  ├── Progress updates ("3 of 5 tasks complete")    │
│  ├── Intermediate results ("Found 12 matches")     │
│  └── Non-critical status ("Installing deps...")    │
│                                                    │
│  TIER 3: LOGGED ONLY                               │
│  ├── Detailed progress ("Reading file X...")       │
│  ├── Internal decisions ("Trying approach B...")   │
│  └── Routine operations ("Running linter...")      │
└──────────────────────────────────────────────────┘
```

### 6.3 Implementation Pattern

```typescript
// Background agent messaging system
class BackgroundAgentMessenger {
  private messageQueue: BackgroundMessage[] = [];
  private listeners: Map<string, MessageListener[]> = new Map();
  
  // Background agent sends a message
  send(
    agentId: string,
    message: Omit<BackgroundMessage, 'agentId' | 'timestamp'>
  ): void {
    const fullMessage: BackgroundMessage = {
      ...message,
      agentId,
      timestamp: new Date(),
    };
    
    this.messageQueue.push(fullMessage);
    
    // Notify listeners
    const agentListeners = this.listeners.get(agentId) || [];
    for (const listener of agentListeners) {
      listener(fullMessage);
    }
  }
  
  // Parent agent listens for messages
  subscribe(
    agentId: string,
    listener: MessageListener
  ): () => void {
    const listeners = this.listeners.get(agentId) || [];
    listeners.push(listener);
    this.listeners.set(agentId, listeners);
    
    return () => {
      const idx = listeners.indexOf(listener);
      if (idx >= 0) listeners.splice(idx, 1);
    };
  }
  
  // Get all messages from an agent
  getMessages(agentId: string): BackgroundMessage[] {
    return this.messageQueue.filter(m => m.agentId === agentId);
  }
}
```

---

## 7. Worktrees: Parallel Agents on the Same Repo

### 7.1 The Problem

When multiple agents work on the same codebase simultaneously, they create conflicts:

```
WITHOUT WORKTREES:
  Agent 1 edits src/auth.ts (line 50)
  Agent 2 edits src/auth.ts (line 75)
  Agent 1 runs tests -- sees Agent 2's changes
  Agent 2 runs tests -- sees Agent 1's changes
  CHAOS: agents interfere with each other
```

### 7.2 Git Worktrees as Isolation

Git worktrees create independent working directories from the same repository:

```
┌──────────────────────────────────────────────────────┐
│              GIT WORKTREE ISOLATION                    │
│                                                        │
│  Main worktree (user's working directory):             │
│  /project/                                             │
│  └── .git/                                             │
│                                                        │
│  Agent 1 worktree:                                     │
│  /project/.worktrees/agent-1/                          │
│  ├── (independent copy of all files)                   │
│  ├── Own branch: agent-1/fix-auth-bug                  │
│  └── Own changes don't affect main or Agent 2          │
│                                                        │
│  Agent 2 worktree:                                     │
│  /project/.worktrees/agent-2/                          │
│  ├── (independent copy of all files)                   │
│  ├── Own branch: agent-2/add-tests                     │
│  └── Own changes don't affect main or Agent 1          │
│                                                        │
│  ISOLATION:                                            │
│  ├── Agents cannot see each other's changes            │
│  ├── Each agent has a clean working directory          │
│  ├── Tests run against only one agent's changes        │
│  └── Merge happens after agent work is complete        │
└──────────────────────────────────────────────────────┘
```

### 7.3 Worktree Management

```typescript
// Worktree manager for parallel agents
class WorktreeManager {
  private projectRoot: string;
  private worktrees: Map<string, WorktreeInfo> = new Map();
  
  async createWorktree(
    agentId: string,
    baseBranch: string = 'main'
  ): Promise<WorktreeInfo> {
    const worktreePath = path.join(
      this.projectRoot,
      '.worktrees',
      agentId
    );
    
    const branchName = `agent/${agentId}/${Date.now()}`;
    
    // Create the worktree with a new branch
    await execAsync(
      `git worktree add -b ${branchName} ${worktreePath} ${baseBranch}`,
      { cwd: this.projectRoot }
    );
    
    const info: WorktreeInfo = {
      agentId,
      path: worktreePath,
      branch: branchName,
      baseBranch,
      createdAt: new Date(),
    };
    
    this.worktrees.set(agentId, info);
    return info;
  }
  
  async cleanupWorktree(agentId: string): Promise<void> {
    const info = this.worktrees.get(agentId);
    if (!info) return;
    
    // Remove the worktree
    await execAsync(
      `git worktree remove ${info.path} --force`,
      { cwd: this.projectRoot }
    );
    
    // Optionally delete the branch
    await execAsync(
      `git branch -D ${info.branch}`,
      { cwd: this.projectRoot }
    );
    
    this.worktrees.delete(agentId);
  }
  
  async mergeWorktree(
    agentId: string,
    targetBranch: string = 'main'
  ): Promise<MergeResult> {
    const info = this.worktrees.get(agentId);
    if (!info) throw new Error('Worktree not found');
    
    // Switch to target branch and merge
    await execAsync(
      `git checkout ${targetBranch}`,
      { cwd: this.projectRoot }
    );
    
    try {
      await execAsync(
        `git merge ${info.branch}`,
        { cwd: this.projectRoot }
      );
      return { success: true };
    } catch {
      return {
        success: false,
        conflicts: await this.getConflicts(),
      };
    }
  }
}
```

### 7.4 The Parallel Agent Pattern

Combining worktrees with sub-agents:

```typescript
// Parallel agent execution with worktree isolation
async function parallelAgentExecution(
  tasks: AgentTask[],
  session: Session
): Promise<AgentResult[]> {
  const worktreeManager = new WorktreeManager(session.projectRoot);
  const messenger = new BackgroundAgentMessenger();
  
  // Create a worktree for each task
  const agents = await Promise.all(
    tasks.map(async (task) => {
      const worktree = await worktreeManager.createWorktree(
        task.id
      );
      
      return {
        task,
        worktree,
        session: createSubSession({
          ...session.config,
          projectRoot: worktree.path,  // Agent works in worktree
          tools: task.tools || ['Read', 'Write', 'Edit', 'Glob', 'Grep', 'Bash'],
        }),
      };
    })
  );
  
  // Run all agents in parallel
  const results = await Promise.all(
    agents.map(async ({ task, worktree, session: subSession }) => {
      try {
        const result = await agentLoop(subSession, task.prompt);
        
        messenger.send(task.id, {
          type: 'result',
          priority: 'high',
          content: `Task "${task.name}" completed: ${result}`,
        });
        
        return { taskId: task.id, result, success: true };
      } catch (error) {
        messenger.send(task.id, {
          type: 'error',
          priority: 'high',
          content: `Task "${task.name}" failed: ${error.message}`,
        });
        
        return {
          taskId: task.id,
          result: error.message,
          success: false
        };
      }
    })
  );
  
  // Merge successful worktrees
  for (const result of results) {
    if (result.success) {
      await worktreeManager.mergeWorktree(result.taskId);
    }
    await worktreeManager.cleanupWorktree(result.taskId);
  }
  
  return results;
}
```

---

## 8. Stripe's Minions: One-Shot Agents at Scale

### 8.1 The Architecture

Stripe's Minions system (described in public talks and blog posts) uses one-shot agents triggered by Linear tickets:

```
┌───────────────────────────────────────────────────────────┐
│                 STRIPE MINIONS ARCHITECTURE                │
│                                                            │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │ Engineer  │    │   Linear     │    │   Minions    │     │
│  │ creates   │ -> │   Ticket     │ -> │ Orchestrator │     │
│  │ ticket    │    │  (task spec) │    │              │     │
│  └──────────┘    └──────────────┘    └──────┬───────┘     │
│                                             │              │
│                                    ┌────────▼────────┐    │
│                                    │  Agent spawned   │    │
│                                    │  in container    │    │
│                                    │  with repo clone │    │
│                                    └────────┬────────┘    │
│                                             │              │
│                              ┌──────────────▼──────────┐  │
│                              │  Agent loop:             │  │
│                              │  1. Read ticket          │  │
│                              │  2. Explore codebase     │  │
│                              │  3. Write code           │  │
│                              │  4. Run tests            │  │
│                              │  5. If fail: fix + retry │  │
│                              │  6. Push branch          │  │
│                              │  7. Create PR            │  │
│                              └──────────────┬──────────┘  │
│                                             │              │
│                              ┌──────────────▼──────────┐  │
│                              │  PR created             │  │
│                              │  CI runs automatically  │  │
│                              │  Engineer reviews        │  │
│                              └─────────────────────────┘  │
│                                                            │
│  KEY: CI IS THE FEEDBACK LOOP, NOT HUMAN INTERACTION      │
└───────────────────────────────────────────────────────────┘
```

### 8.2 The CI Feedback Loop

Instead of asking a human for guidance, Minions use CI as the feedback mechanism:

```typescript
// CI-driven feedback loop (conceptual)
async function minionLoop(
  ticket: LinearTicket,
  repo: Repository
): Promise<PullRequest> {
  const MAX_ATTEMPTS = 5;
  
  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
    // Generate code changes
    const changes = await agent.generateChanges(
      ticket.description,
      attempt > 0 ? lastCIResult : null  // Include CI feedback
    );
    
    // Apply changes
    await repo.applyChanges(changes);
    
    // Run CI (tests, linting, type checking)
    const ciResult = await repo.runCI();
    
    if (ciResult.success) {
      // All tests pass -- create PR
      return await repo.createPR({
        title: ticket.title,
        body: generatePRDescription(ticket, changes),
        branch: `minion/${ticket.id}`,
      });
    }
    
    // CI failed -- the failure output becomes context for retry
    lastCIResult = ciResult;
    
    // Revert changes before retrying
    await repo.resetChanges();
  }
  
  // Max attempts reached -- create PR anyway with failure notes
  return await repo.createPR({
    title: `[WIP] ${ticket.title}`,
    body: `Minion attempted ${MAX_ATTEMPTS} times. ` +
      `CI still failing. Human review needed.`,
    branch: `minion/${ticket.id}`,
    draft: true,
  });
}
```

### 8.3 Why Minions Work at Scale

```
┌──────────────────────────────────────────────────┐
│        WHY MINIONS SCALE                          │
│                                                    │
│  1. NO HUMAN IN THE LOOP (during execution)        │
│     Agent works autonomously                       │
│     Human reviews the output (PR), not the process │
│                                                    │
│  2. CI PROVIDES OBJECTIVE FEEDBACK                 │
│     Did tests pass? Binary answer.                 │
│     No ambiguity, no interpretation needed.         │
│                                                    │
│  3. CONTAINER ISOLATION                            │
│     Each minion runs in its own container          │
│     No interference between minions                │
│     Clean state on every attempt                   │
│                                                    │
│  4. TICKET = COMPLETE SPECIFICATION                │
│     The Linear ticket contains everything           │
│     the agent needs to know                        │
│     Well-specified tasks = better agent output      │
│                                                    │
│  5. PR = REVIEW CHECKPOINT                         │
│     The output is a standard PR                    │
│     Fits into existing code review workflows       │
│     Engineers review AI work like human work        │
└──────────────────────────────────────────────────┘
```

---

## 9. The Delegation Pattern

### 9.1 Claude to Devin via Tickets

A fascinating pattern that emerged: using one AI tool to create tasks for another:

```
┌──────────────────────────────────────────────────────┐
│           THE DELEGATION PATTERN                      │
│                                                        │
│  Step 1: User works with Claude Code on architecture   │
│  "Design the new auth system"                          │
│  Claude Code produces: design doc, API spec, tasks     │
│                                                        │
│  Step 2: Claude Code creates Linear tickets            │
│  - Ticket 1: "Implement JWT middleware"                │
│  - Ticket 2: "Add session storage to Redis"            │
│  - Ticket 3: "Create login/logout endpoints"           │
│  - Ticket 4: "Write auth integration tests"            │
│                                                        │
│  Step 3: Devin (or Minions) picks up tickets           │
│  Each ticket is independently implementable            │
│  Each ticket has clear acceptance criteria              │
│  Each ticket can be verified by CI                     │
│                                                        │
│  Step 4: PRs come back for review                      │
│  Claude Code (or human) reviews the PRs                │
│  Provides feedback, requests changes                   │
│                                                        │
│  Result: Claude Code as architect, Devin as builder    │
└──────────────────────────────────────────────────────┘
```

### 9.2 Why Delegation Works

Different AI tools have different strengths:

```
┌─────────────────────┬─────────────────────────────────┐
│ Tool                │ Strength                        │
├─────────────────────┼─────────────────────────────────┤
│ Claude Code         │ Architecture, design, complex   │
│                     │ reasoning, interactive work     │
│                     │                                 │
│ Devin               │ Long-running implementation,    │
│                     │ autonomous execution, CI loop   │
│                     │                                 │
│ Codex               │ One-shot tasks, PR generation,  │
│                     │ container-isolated execution    │
│                     │                                 │
│ Cursor              │ IDE-integrated editing, fast    │
│                     │ completions, visual context     │
└─────────────────────┴─────────────────────────────────┘
```

Delegation lets you use each tool for what it does best, connected by standard interfaces (tickets, PRs, APIs).

---

## 10. Advanced Delegation Architecture: Lessons from claw-code-agent

The open-source claw-code-agent reimplementation (75,000+ stars, pure Python) reverse-engineered and extended the delegation patterns described above. Its parity checklist reveals architectural details about how production multi-agent delegation actually works at the implementation level.

### 10.1 Dependency-Aware Subtasks with Topological Batch Planning

Section 2 showed simple parallel agents: spawn workers, collect results. But real tasks have dependencies. Task B needs the output of Task A. Task C and D are independent. Task E needs both C and D.

claw-code-agent implements this through `plan_runtime.py` -- a topological sort over a task dependency graph:

```
┌──────────────────────────────────────────────────────────┐
│       TOPOLOGICAL BATCH PLANNING                          │
│                                                           │
│  Task graph:                                              │
│    A (no deps)  ──┐                                       │
│    B (no deps)  ──┤──> D (needs A, B)  ──┐               │
│    C (no deps)  ──┘                      ├──> F (needs D,E)│
│    E (needs C)  ─────────────────────────┘               │
│                                                           │
│  Execution batches (topological sort):                    │
│                                                           │
│  Batch 1: [A, B, C]     ← all independent, run parallel  │
│  Batch 2: [D, E]        ← D waits for A+B, E waits for C │
│  Batch 3: [F]           ← waits for D+E                  │
│                                                           │
│  Total time: max(Batch1) + max(Batch2) + max(Batch3)     │
│  Not: A + B + C + D + E + F (sequential)                  │
└──────────────────────────────────────────────────────────┘
```

This is the same scheduling algorithm used in build systems (Make, Bazel) and CI pipelines. The insight is that multi-agent delegation is a scheduling problem, and the solutions are well-known.

### 10.2 Agent-Manager Lineage Tracking

When agents spawn sub-agents, which spawn further sub-agents, you need to track the lineage. claw-code-agent's `agent_manager.py` and `team_runtime.py` maintain:

- **Parent-child relationships** across spawned agents. Every agent knows its parent and can access the parent's context summary.
- **Managed agent-group membership** with child indices. A coordinator can enumerate all its descendants, not just direct children.
- **Sequential multi-subtask delegation with parent-context carryover.** When a parent delegates tasks sequentially, each subsequent task receives a summary of what the previous tasks accomplished. This prevents later tasks from repeating work or contradicting earlier results.

```
┌───────────────────────────────────────────────────────┐
│          AGENT LINEAGE TRACKING                        │
│                                                        │
│  Coordinator (depth 0)                                 │
│  ├── Agent A (depth 1, index 0)                        │
│  │   ├── Agent A1 (depth 2, index 0)                   │
│  │   └── Agent A2 (depth 2, index 1)                   │
│  ├── Agent B (depth 1, index 1)                        │
│  └── Agent C (depth 1, index 2)                        │
│                                                        │
│  Each agent tracks:                                    │
│  ├── parentId: who spawned me                          │
│  ├── depth: how deep in the tree                       │
│  ├── childIndex: my position among siblings            │
│  ├── groupId: which agent-group I belong to            │
│  └── lineage: [coordinator, parent, ..., me]           │
│                                                        │
│  WHY THIS MATTERS:                                     │
│  ├── Debugging: trace which agent made which change    │
│  ├── Budgeting: aggregate cost across a lineage        │
│  ├── Depth limits: prevent infinite agent spawning     │
│  └── Context: parent summaries flow to children        │
└───────────────────────────────────────────────────────┘
```

### 10.3 Child-Session Resume by Saved Session ID

One of the most practical features in claw-code-agent's delegation system is **session persistence for child agents**. When a delegated sub-agent is interrupted (timeout, user cancellation, system restart), its session state is saved to `session_store.py` with a unique ID. The parent agent can later resume that exact child session, picking up where it left off.

This is the difference between delegation that works for 30-second tasks and delegation that works for 30-minute tasks. Without resume, any interruption means restarting the entire subtask from scratch. With resume:

1. Child agent works on Task B for 10 minutes
2. System restarts (deployment, crash, user closes terminal)
3. Parent agent resumes, finds saved session ID for Task B
4. Child agent resumes from its last transcript state -- all tool results, all context, all progress preserved
5. Task B continues where it left off

The session state includes: the full message transcript, compaction metadata, file history journal (which files were read/written/edited), and the current tool-call state. This is transcript-aware persistence -- not just saving a prompt, but saving the entire reasoning trace so the agent can genuinely continue rather than start over.

**Practical lesson:** If you are building a multi-agent system where tasks take more than a few minutes, session persistence for child agents is not optional. Without it, any interruption cascades into wasted work and cost.

---

## 11. Designing Multi-Agent Systems

### 10.1 The Decision Framework

```
┌──────────────────────────────────────────────────────┐
│       MULTI-AGENT DESIGN DECISIONS                    │
│                                                        │
│  Q1: Do tasks need to share state?                     │
│  ├── Yes: single agent with sub-tasks                  │
│  └── No:  parallel agents with worktrees               │
│                                                        │
│  Q2: Do tasks need to communicate?                     │
│  ├── No:  fire-and-forget (Minions pattern)            │
│  ├── One-way: parent polls children (SendUserMessage)  │
│  └── Two-way: message bus (complex, often unnecessary) │
│                                                        │
│  Q3: How long do tasks run?                            │
│  ├── Seconds: foreground sub-agents                    │
│  ├── Minutes: backgrounded with tick-loop checking     │
│  └── Hours: one-shot with CI feedback (Minions)        │
│                                                        │
│  Q4: How much isolation is needed?                     │
│  ├── None: sub-agent in same context                   │
│  ├── Context: sub-agent with own context               │
│  ├── Filesystem: worktree per agent                    │
│  └── Full: container per agent                         │
│                                                        │
│  Q5: What is the feedback loop?                        │
│  ├── Human: interactive approval                       │
│  ├── CI: automated test results                        │
│  ├── Agent: parent reviews child output                │
│  └── None: fire-and-forget                             │
└──────────────────────────────────────────────────────┘
```

### 10.2 Reference Architecture

```typescript
// A multi-agent coordinator
class MultiAgentCoordinator {
  private worktreeManager: WorktreeManager;
  private messenger: BackgroundAgentMessenger;
  private tickLoop: TickLoop;
  
  async executeParallel(
    tasks: AgentTask[]
  ): Promise<CoordinationResult> {
    // Partition tasks by isolation requirements
    const { isolated, shared } = this.partitionTasks(tasks);
    
    // Run shared-context tasks sequentially
    const sharedResults = [];
    for (const task of shared) {
      const result = await this.executeSubAgent(task);
      sharedResults.push(result);
    }
    
    // Run isolated tasks in parallel worktrees
    const isolatedResults = await Promise.all(
      isolated.map(task => this.executeIsolatedAgent(task))
    );
    
    // Merge results
    return {
      results: [...sharedResults, ...isolatedResults],
      summary: await this.summarizeResults(
        [...sharedResults, ...isolatedResults]
      ),
    };
  }
  
  private async executeIsolatedAgent(
    task: AgentTask
  ): Promise<AgentResult> {
    // Create worktree for isolation
    const worktree = await this.worktreeManager.createWorktree(
      task.id
    );
    
    try {
      // Run the agent in the worktree
      const result = await agentLoop(
        createSubSession({
          projectRoot: worktree.path,
          tools: task.tools,
        }),
        task.prompt
      );
      
      // Merge successful changes
      if (task.mergeOnSuccess) {
        await this.worktreeManager.mergeWorktree(task.id);
      }
      
      return { taskId: task.id, result, success: true };
    } finally {
      await this.worktreeManager.cleanupWorktree(task.id);
    }
  }
}
```

---

## Build It: An Agent Coordinator in 75 Lines

Multi-agent coordination sounds like a distributed systems problem, but the core pattern is surprisingly simple with asyncio. The implementation below shows the complete flow: a coordinator takes a task, uses an LLM to break it into subtasks with dependency annotations, executes independent subtasks in parallel, waits for dependencies before starting dependent ones, collects all results, and synthesizes a final answer. It also tracks parent-child lineage so you can trace which worker produced which result.

```bash
pip install openai
```

```python
# Agent coordinator — parallel workers with dependency-aware execution
# Usage: OPENAI_API_KEY=sk-... python coordinator.py

import asyncio, json
from dataclasses import dataclass, field
from openai import AsyncOpenAI

@dataclass
class Subtask:
    id: str
    description: str
    depends_on: list[str] = field(default_factory=list)  # IDs of prerequisite subtasks
    result: str = ""
    parent_id: str = "coordinator"

# --- Task Decomposition -------------------------------------------------------

async def decompose_task(client: AsyncOpenAI, task: str, model: str) -> list[Subtask]:
    """Use the LLM to break a task into subtasks with dependencies."""
    response = await client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": (
                "Break the given task into 2-5 independent or dependent subtasks. "
                "Return JSON: [{\"id\": \"t1\", \"description\": \"...\", \"depends_on\": []}]. "
                "Use depends_on to list IDs of subtasks that must complete first. "
                "Maximize parallelism: only add dependencies when truly required."
            )},
            {"role": "user", "content": task},
        ],
        response_format={"type": "json_object"},
    )
    raw = json.loads(response.choices[0].message.content)
    subtasks_data = raw.get("subtasks", raw.get("tasks", list(raw.values())[0]))
    return [Subtask(id=s["id"], description=s["description"],
                    depends_on=s.get("depends_on", [])) for s in subtasks_data]

# --- Worker Agent -------------------------------------------------------------

async def run_worker(client: AsyncOpenAI, subtask: Subtask, context: dict[str, str],
                     model: str) -> str:
    """A worker agent that executes a single subtask."""
    # Include results from dependencies as context
    dep_context = ""
    for dep_id in subtask.depends_on:
        if dep_id in context:
            dep_context += f"\n[Result from {dep_id}]: {context[dep_id]}\n"

    response = await client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "Complete the given subtask concisely. Use any provided context from prior subtasks."},
            {"role": "user", "content": f"Subtask: {subtask.description}{dep_context}"},
        ],
    )
    return response.choices[0].message.content

# --- Dependency-Aware Executor ------------------------------------------------

async def execute_with_dependencies(client: AsyncOpenAI, subtasks: list[Subtask],
                                     model: str) -> dict[str, str]:
    """Execute subtasks respecting dependencies. Parallelize where possible."""
    results: dict[str, str] = {}
    completed: set[str] = set()
    pending = {s.id: s for s in subtasks}

    while pending:
        # Find subtasks whose dependencies are all completed
        ready = [s for s in pending.values() if all(d in completed for d in s.depends_on)]
        if not ready:
            raise RuntimeError(f"Circular dependency detected. Pending: {list(pending.keys())}")

        # Run all ready subtasks in parallel
        print(f"  [PARALLEL] Launching {len(ready)} workers: {[s.id for s in ready]}")
        tasks = [run_worker(client, s, results, model) for s in ready]
        worker_results = await asyncio.gather(*tasks)

        for subtask, result in zip(ready, worker_results):
            results[subtask.id] = result
            subtask.result = result
            completed.add(subtask.id)
            del pending[subtask.id]
            print(f"  [DONE] {subtask.id}: {result[:80]}...")

    return results

# --- Result Synthesis ---------------------------------------------------------

async def synthesize(client: AsyncOpenAI, task: str, subtasks: list[Subtask],
                     results: dict[str, str], model: str) -> str:
    """Combine all worker results into a final response."""
    parts = "\n\n".join(f"## {s.id}: {s.description}\n{results[s.id]}" for s in subtasks)
    response = await client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "Synthesize the worker results into a clear, unified response to the original task."},
            {"role": "user", "content": f"Original task: {task}\n\nWorker results:\n{parts}"},
        ],
    )
    return response.choices[0].message.content

# --- Coordinator --------------------------------------------------------------

async def coordinate(task: str, model: str = "gpt-4o"):
    """The full coordination flow: decompose -> execute -> synthesize."""
    client = AsyncOpenAI()

    print(f"[COORDINATOR] Task: {task}\n")

    # Step 1: Decompose
    subtasks = await decompose_task(client, task, model)
    print(f"[PLAN] {len(subtasks)} subtasks:")
    for s in subtasks:
        deps = f" (after {s.depends_on})" if s.depends_on else " (independent)"
        print(f"  {s.id}: {s.description}{deps}")
    print()

    # Step 2: Execute with dependency awareness
    results = await execute_with_dependencies(client, subtasks, model)

    # Step 3: Synthesize
    print("\n[SYNTHESIZING]...")
    final = await synthesize(client, task, subtasks, results, model)

    # Lineage tracking
    print("\n=== Lineage ===")
    for s in subtasks:
        print(f"  coordinator -> {s.id}: {s.description[:60]}")

    print(f"\n=== Final Response ===\n{final}")
    return final


if __name__ == "__main__":
    asyncio.run(coordinate(
        "Create a Python web API for a todo app: design the data model, "
        "write the API endpoints, and write tests for the endpoints."
    ))
```

The key insight is the dependency-aware executor: on each iteration, it finds all subtasks whose dependencies are satisfied and runs them concurrently. This naturally maximizes parallelism -- independent subtasks run at the same time, while dependent ones wait only for their specific prerequisites. This is the same pattern that Claude Code's coordinator mode and Stripe's Minions use, just without the git worktree isolation.

---

## 12. Key Takeaways

1. **Multi-agent coordination solves parallelism, isolation, and specialization.** Single agents are serial, share context, and cannot multitask.

2. **The Agent tool spawns sub-agents** with independent context windows, caches, and tool sets. Workers are more restricted than the coordinator.

3. **The tick loop uses setTimeout(0)** to give agents idle-time agency. During ticks, agents can check background operations, pre-fetch context, or sleep.

4. **SleepTool enables pacing decisions.** The cache-vs-cost balance means sleeping too long wastes the cache, but acting too often wastes API calls.

5. **The 15-second blocking budget** auto-backgrounds long operations. This keeps the agent productive during builds, tests, and installs.

6. **SendUserMessage** is the messaging layer for background agents. Three-tier filtering (always/configurable/logged) prevents notification overload.

7. **Git worktrees** provide filesystem isolation for parallel agents. Each agent gets its own branch and working directory. Changes merge after completion.

8. **Stripe's Minions** demonstrate one-shot agents at scale: ticket in, PR out, CI as the feedback loop. No human in the loop during execution.

9. **The delegation pattern** uses different tools for their strengths: Claude Code for architecture, Devin/Minions for implementation, connected by tickets and PRs.

10. **Dependency-aware delegation is a scheduling problem.** Topological batch planning (same algorithm as build systems) maximizes parallelism while respecting task dependencies. Agent lineage tracking and child-session resume make long-running delegated work reliable.

11. **Design decisions:** shared state vs. isolation, communication pattern, task duration, isolation level, and feedback loop. Answer these questions before building.

---

*Previous: [Chapter 31 -- Web Search & Knowledge Pipelines](./31-web-search-pipelines.md)* · *Next: [Chapter 33 -- Skills, Plugins & Distribution](./33-skills-plugins-internals.md)* explores how the harness is extended, customized, and distributed across organizations.

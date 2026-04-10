<!--
  CHAPTER: 29
  TITLE: Context Window Internals
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 12 (Context Window Management), Ch 27 (Harness Architecture), Ch 28 (Memory Systems)
  KEY_TOPICS: compaction service, token budgets, cache-break vectors, model switching costs, session limits, Statsig feature flags, db8 token drain, cost implications
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 29: Context Window Internals

> **Part 6 — Anatomy of AI Developer Tools** | Phase 2: Become an Expert | Prerequisites: Ch 12, Ch 27, Ch 28 | Difficulty: Advanced | Language: TypeScript + Architecture

The context window is the most valuable real estate in AI engineering. Every token in the context costs money on every model call, and the window has a hard maximum size. When it fills up, the session degrades or dies. Managing this finite resource is what separates a toy agent from a production tool.

In Ch 12 you learned the basic concepts: token counting, summarization, sliding windows. This chapter goes deep into how production systems actually implement context management. The Claude Code leak revealed a compaction service, 14 distinct cache-break vectors, Statsig-powered feature flags for A/B testing session limits, and a mysterious "db8" function that community researchers identified as a significant source of token drain. Every one of these details has cost implications.

### In This Chapter

1. The context window as a budget
2. How compaction actually works
3. The token threshold system
4. The 14 cache-break vectors
5. Why switching models mid-session kills your prompt cache
6. The compaction service architecture
7. Statsig and feature flags for session limits
8. The db8 function and token drain
9. Cost implications of every design choice
10. Building your own context manager

### Related Chapters

- **Ch 12 (Context Window Management)** -- introduced basic concepts; this chapter shows production implementation
- **Ch 27 (Harness Architecture)** -- showed where context management fits in the harness
- **Ch 28 (Memory Systems)** -- CLAUDE.md injection is a major consumer of context budget
- **Ch 49 (Cost Engineering)** -- will build on these insights for production cost optimization
- **Ch 50 (Advanced Context Strategies)** -- will explore more sophisticated strategies

---

## 1. The Context Window as a Budget

### 1.1 The Budget Metaphor

Think of the context window as a fixed budget. On every model call, you must allocate tokens across competing priorities:

```
┌──────────────────────────────────────────────────┐
│        CONTEXT WINDOW BUDGET (200K tokens)        │
│                                                    │
│  ┌──────────────────────────────────┐  FIXED      │
│  │  System Prompt      (~3,000)    │  OVERHEAD    │
│  │  CLAUDE.md           (~800)     │  (paid       │
│  │  Active Skills       (~2,000)   │   every      │
│  │  Tool Definitions    (~1,500)   │   turn)      │
│  │  Permission Context    (~500)   │              │
│  └──────────────────────────────────┘  ~7,800     │
│                                                    │
│  ┌──────────────────────────────────┐  DYNAMIC    │
│  │  Conversation History           │  (grows      │
│  │  (messages + tool results)      │   over       │
│  │  Budget: everything that's left │   time)      │
│  └──────────────────────────────────┘             │
│                                                    │
│  ┌──────────────────────────────────┐  RESERVED   │
│  │  Output Token Reserve (~8,000)  │  (for the    │
│  │  (model's response space)       │   model's    │
│  └──────────────────────────────────┘   reply)    │
│                                                    │
│  Available for history:                            │
│  200,000 - 7,800 - 8,000 = 184,200 tokens        │
└──────────────────────────────────────────────────┘
```

### 1.2 How the Budget Gets Consumed

Here is a realistic session showing how the budget gets consumed:

```
Turn  1: User prompt (50) + Response (200)
         Total: 250 tokens in history
         Budget remaining: 183,950

Turn  5: After 4 tool calls (file reads)
         Total: 12,000 tokens in history
         Budget remaining: 172,200

Turn 10: After code exploration
         Total: 45,000 tokens in history
         Budget remaining: 139,200

Turn 20: After significant editing
         Total: 95,000 tokens in history
         Budget remaining: 89,200

Turn 30: Heavy session with many file reads
         Total: 150,000 tokens in history
         Budget remaining: 34,200

Turn 35: COMPACTION TRIGGERED
         Old messages summarized
         Total: 30,000 tokens (post-compaction)
         Budget remaining: 154,200

Turn 45: History grows again
         Total: 90,000 tokens
         Budget remaining: 94,200

Turn 55: SECOND COMPACTION
         Total: 25,000 tokens (post-compaction)
         Budget remaining: 159,200
```

**Key insight:** Compaction buys you runway. Without it, the session dies around turn 30-35. With it, sessions can run for hundreds of turns -- but at the cost of losing detail from earlier in the conversation.

### 1.3 What Consumes the Most Tokens

Across typical sessions, here is where tokens go:

```
┌────────────────────────────────────────────────┐
│          TOKEN CONSUMPTION BREAKDOWN            │
│                                                  │
│  File reads (Read tool):          ~40% of total │
│  ┌────────────────────────────────┐             │
│  │████████████████████████████████│             │
│  └────────────────────────────────┘             │
│  A single 500-line file = ~5000 tokens          │
│                                                  │
│  Command outputs (Bash tool):     ~20% of total │
│  ┌────────────────────────┐                     │
│  │████████████████████████│                     │
│  └────────────────────────┘                     │
│  Test output, build logs, directory listings    │
│                                                  │
│  Model responses:                 ~15% of total │
│  ┌──────────────────┐                           │
│  │██████████████████│                           │
│  └──────────────────┘                           │
│  Code generation, explanations                  │
│                                                  │
│  System prompt (fixed):           ~10% of total │
│  ┌──────────────┐                               │
│  │██████████████│                               │
│  └──────────────┘                               │
│  But paid on EVERY turn (cumulative cost is high)│
│                                                  │
│  User messages:                    ~5% of total │
│  ┌────────┐                                     │
│  │████████│                                     │
│  └────────┘                                     │
│                                                  │
│  Other (search results, errors):  ~10% of total │
│  ┌──────────────┐                               │
│  │██████████████│                               │
│  └──────────────┘                               │
└────────────────────────────────────────────────┘
```

File reads dominate. This is why Claude Code's Read tool has `offset` and `limit` parameters -- reading 50 lines instead of 500 saves 90% of the token cost.

---

## 2. How Compaction Actually Works

### 2.1 The Compaction Algorithm

Compaction is the process of replacing old conversation messages with a shorter summary. Here is how Claude Code's compaction service works:

```typescript
// The compaction service (reconstructed from the leak)
class CompactionService {
  private tokenCounter: TokenCounter;
  private summaryModel: string;
  
  constructor(config: CompactionConfig) {
    this.tokenCounter = new TokenCounter(config.model);
    this.summaryModel = config.summaryModel || config.model;
  }
  
  async shouldCompact(
    systemPrompt: string,
    messages: Message[]
  ): Promise<boolean> {
    const totalTokens = this.tokenCounter.count(
      systemPrompt,
      messages
    );
    return totalTokens > this.config.tokenThreshold;
  }
  
  async compact(
    messages: Message[],
    options: CompactOptions
  ): Promise<Message[]> {
    // Step 1: Identify the split point
    // Keep the most recent messages untouched
    const keepCount = options.keepRecentMessages || 10;
    const recentMessages = messages.slice(-keepCount);
    const oldMessages = messages.slice(0, -keepCount);
    
    if (oldMessages.length === 0) {
      // Nothing to compact
      return messages;
    }
    
    // Step 2: Generate a summary of old messages
    const summary = await this.summarize(oldMessages);
    
    // Step 3: Create the compacted message array
    const compacted: Message[] = [
      {
        role: 'user',
        content: this.formatSummary(summary)
      },
      {
        role: 'assistant',
        content: 'Understood. I have the context from our '
          + 'previous conversation. How can I help?'
      },
      ...recentMessages
    ];
    
    return compacted;
  }
  
  private async summarize(messages: Message[]): Promise<string> {
    // Use a model call to generate the summary
    const summaryPrompt = `
Summarize the following conversation between a user and an AI
coding assistant. Focus on:
1. What task was being worked on
2. What files were read or modified
3. What decisions were made
4. What problems were encountered and how they were resolved
5. What the current state of the work is

Be concise but preserve all information needed to continue
the work.

<conversation>
${this.formatMessagesForSummary(messages)}
</conversation>
`;
    
    const response = await this.callModel(
      this.summaryModel,
      summaryPrompt
    );
    
    return response;
  }
  
  private formatSummary(summary: string): string {
    return [
      '[Previous conversation summary]',
      '',
      summary,
      '',
      '[End of summary. The conversation continues below.]'
    ].join('\n');
  }
}
```

### 2.2 What Gets Lost in Compaction

Compaction is inherently lossy. Here is what typically survives and what is lost:

```
┌──────────────────────────────────────────────┐
│          COMPACTION: WHAT SURVIVES            │
│                                                │
│  SURVIVES:                                     │
│  ✓ Task description and goals                  │
│  ✓ Files that were modified                    │
│  ✓ Key decisions and their rationale           │
│  ✓ Current state of the work                   │
│  ✓ Errors encountered and solutions            │
│                                                │
│  LOST:                                         │
│  ✗ Exact file contents that were read          │
│  ✗ Full command outputs                        │
│  ✗ Intermediate exploration steps              │
│  ✗ Rejected approaches (and why)               │
│  ✗ Nuanced context from user messages          │
│  ✗ The exact wording of error messages         │
└──────────────────────────────────────────────┘
```

**Real-world consequence:** After compaction, the agent may re-read files it already read, re-try approaches it already rejected, or forget constraints the user mentioned earlier. This is why keeping recent messages intact is critical -- it provides a "working memory" buffer that is never compacted.

### 2.3 Compaction Strategies

Different situations call for different compaction approaches:

```typescript
// Strategy 1: Summarize-and-replace (Claude Code's default)
// Best for: general purpose, good balance of retention and savings
async function summarizeAndReplace(
  oldMessages: Message[]
): Promise<Message[]> {
  const summary = await summarize(oldMessages);
  return [{ role: 'user', content: summary }];
}

// Strategy 2: Sliding window (simplest)
// Best for: when exact history doesn't matter much
function slidingWindow(
  messages: Message[],
  maxTokens: number
): Message[] {
  let total = 0;
  const kept: Message[] = [];
  
  // Walk backward from most recent
  for (let i = messages.length - 1; i >= 0; i--) {
    const msgTokens = countTokens(messages[i]);
    if (total + msgTokens > maxTokens) break;
    total += msgTokens;
    kept.unshift(messages[i]);
  }
  
  return kept;
}

// Strategy 3: Importance-weighted compaction
// Best for: complex sessions with mixed importance
async function importanceWeighted(
  messages: Message[]
): Promise<Message[]> {
  // Score each message's importance
  const scored = await scoreImportance(messages);
  
  // Keep high-importance messages, summarize the rest
  const highImportance = scored.filter(
    m => m.importance > 0.7
  );
  const lowImportance = scored.filter(
    m => m.importance <= 0.7
  );
  
  const summary = await summarize(
    lowImportance.map(m => m.message)
  );
  
  return [
    { role: 'user', content: summary },
    ...highImportance.map(m => m.message)
  ];
}
```

---

## 3. The Token Threshold System

### 3.1 How Thresholds Work

Claude Code uses a multi-threshold system for context management:

```typescript
// Token threshold configuration (reconstructed)
interface TokenThresholds {
  // Primary threshold: triggers compaction
  compactionThreshold: number;   // e.g., 150,000 tokens
  
  // Warning threshold: alerts the user
  warningThreshold: number;      // e.g., 170,000 tokens
  
  // Hard limit: refuses new tool calls
  hardLimit: number;             // e.g., 190,000 tokens
  
  // Target after compaction: how small to compact TO
  compactionTarget: number;      // e.g., 50,000 tokens
  
  // Minimum recent messages to keep
  keepRecentMessages: number;    // e.g., 10
}
```

### 3.2 The Threshold Lifecycle

```
Token count over time:
                                    Hard limit (190K)
- - - - - - - - - - - - - - - - - - - - - - - - - - -
                                Warning (170K)
- - - - - - - - - - - - - - - - - - - - - - - - - - -
                          Compact trigger (150K)
- - - - - - - - - - - - - ╱- - - - - - - - - ╱- - - -
                         ╱                   ╱
                        ╱                   ╱
                       ╱   compaction      ╱
                      ╱   drops to        ╱
                     ╱    target         ╱
                    ╱        │          ╱
                   ╱         │         ╱
                  ╱          ▼        ╱
                 ╱     ┌─────────┐  ╱
                ╱      │  50K    │ ╱
               ╱       │ target  │╱
              ╱        └─────────┘
             ╱              ╱
            ╱              ╱
           ╱              ╱
    ──────╱──────────────╱────────────────── Time
    Turn 1         Turn 35          Turn 65
```

### 3.3 Dynamic Threshold Adjustment

The leak revealed that thresholds are not static -- they can be adjusted by feature flags:

```typescript
// Feature flag-driven thresholds
async function getThresholds(
  userId: string,
  plan: string
): Promise<TokenThresholds> {
  // Statsig feature flag check
  const config = await statsig.getConfig(
    userId,
    'context_management_config'
  );
  
  return {
    compactionThreshold: config.get(
      'compaction_threshold',
      150000  // default
    ),
    warningThreshold: config.get(
      'warning_threshold',
      170000
    ),
    hardLimit: config.get(
      'hard_limit',
      190000
    ),
    compactionTarget: config.get(
      'compaction_target',
      50000
    ),
    keepRecentMessages: config.get(
      'keep_recent',
      10
    ),
  };
}
```

This means Anthropic can A/B test different threshold configurations across user segments. Pro plan users might get higher thresholds. Enterprise users might get different compaction strategies.

---

## 4. The 14 Cache-Break Vectors

### 4.1 Why Prompt Caching Matters

Anthropic's API supports prompt caching: if the beginning of your prompt is identical to a previous call, the cached portion is served at a 90% discount. This is a massive cost savings because the system prompt and CLAUDE.md are the same on every turn.

```
Without caching:
  Turn 1: 7,800 system tokens at full price = $0.023
  Turn 2: 7,800 system tokens at full price = $0.023
  Turn 3: 7,800 system tokens at full price = $0.023
  ...
  Turn 100: Total system token cost = $2.34

With caching:
  Turn 1: 7,800 system tokens at full price = $0.023
  Turn 2: 7,800 system tokens at CACHED price = $0.0023
  Turn 3: 7,800 system tokens at CACHED price = $0.0023
  ...
  Turn 100: Total system token cost = $0.25

Savings: ~89%
```

But the cache is fragile. Many things can break it.

### 4.2 The 14 Vectors

The community identified 14 distinct events that break Claude Code's prompt cache:

```
┌──────────────────────────────────────────────────────────┐
│              THE 14 CACHE-BREAK VECTORS                   │
│                                                           │
│  MODEL CHANGES                                            │
│  1. Switching models mid-session (e.g., Sonnet -> Opus)   │
│  2. Model version rollover (Anthropic updates the model)  │
│  3. Max tokens change                                     │
│                                                           │
│  SYSTEM PROMPT CHANGES                                    │
│  4. CLAUDE.md is modified (file changed on disk)          │
│  5. Skill is loaded or unloaded                           │
│  6. Permission rules change                               │
│  7. System prompt template update (Anthropic pushes new)  │
│                                                           │
│  TOOL CHANGES                                             │
│  8. MCP server connects or disconnects                    │
│  9. Tool definition changes (new tools, modified schemas) │
│  10. Tool availability changes (feature flag toggle)      │
│                                                           │
│  SESSION CHANGES                                          │
│  11. Session timeout (cache TTL expires, e.g., 5 min)     │
│  12. Rate limit hit (forces new request path)             │
│  13. API error and retry (different request)              │
│                                                           │
│  INFRASTRUCTURE                                           │
│  14. Server-side cache eviction (Anthropic's infra)       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### 4.3 The Most Expensive Vector: Model Switching

The single most expensive cache-break is switching models mid-session:

```
Session with consistent model:
  Turn 1:  Cache miss (full price)      $0.023
  Turn 2:  Cache HIT (90% discount)     $0.0023
  Turn 3:  Cache HIT                    $0.0023
  ...
  Turn 50: Cache HIT                    $0.0023
  Total: $0.023 + (49 * $0.0023) = $0.136

Session with model switch at Turn 25:
  Turn 1:  Cache miss                   $0.023
  Turn 2-24: Cache HIT (23 turns)       $0.053
  Turn 25: MODEL SWITCH - Cache miss    $0.023
  Turn 26-50: Cache HIT (24 turns)      $0.055
  Total: $0.023 + $0.053 + $0.023 + $0.055 = $0.154

Extra cost from one model switch: ~$0.02 (13% increase)
```

Seems small? Multiply by thousands of sessions per day across an organization and it adds up fast.

**Why it breaks the cache:** Prompt caching is keyed on the exact bytes sent to the API. Different models have different tokenizers, different system prompt templates, and different tool schemas. When you switch models, the entire prompt changes.

### 4.4 Protecting Your Cache

Strategies to minimize cache breaks:

```typescript
// Cache-friendly session management
class CacheAwareSession {
  private model: string;
  private systemPrompt: string;
  private tools: ToolDefinition[];
  
  // Freeze the model for the session
  constructor(config: SessionConfig) {
    this.model = config.model;  // Never change mid-session
    this.systemPrompt = this.buildSystemPrompt(config);
    this.tools = this.buildTools(config);
  }
  
  // Watch for CLAUDE.md changes
  private watchClaudeMd(): void {
    // Don't reload CLAUDE.md on every turn
    // Instead, reload only when the file actually changes
    fs.watch(this.claudeMdPath, () => {
      this.invalidateCache();
      this.systemPrompt = this.buildSystemPrompt(this.config);
    });
  }
  
  // Batch MCP server connections at session start
  private async connectMcpServers(): Promise<void> {
    // Connect ALL servers at startup, not one-by-one
    // Each connection changes tool definitions = cache break
    await Promise.all(
      this.config.mcpServers.map(s => this.connectServer(s))
    );
    // Now build tools ONCE
    this.tools = this.buildTools(this.config);
  }
}
```

### 4.5 The SYSTEM_PROMPT_DYNAMIC_BOUNDARY Pattern

The 14 cache-break vectors above tell you what breaks the cache. But Claude Code's architecture contains a deeper insight about how to protect it: the system prompt is deliberately split into two halves at a stable boundary.

```
┌──────────────────────────────────────────────────────────┐
│           SYSTEM PROMPT CACHE BOUNDARY                    │
│                                                           │
│  ┌────────────────────────────────────────────────┐       │
│  │  STABLE FRONT HALF (cached across turns)       │       │
│  │  ├── Core identity and behavior instructions   │       │
│  │  ├── Tool definitions and schemas              │       │
│  │  ├── Permission rules and safety constraints   │       │
│  │  ├── Coding conventions                        │       │
│  │  └── CLAUDE.md content (changes rarely)        │       │
│  │                                                │       │
│  │  This portion is byte-identical across turns.  │       │
│  │  Anthropic's API caches it at a 90% discount.  │       │
│  ├────────────────────────────────────────────────┤       │
│  │  === DYNAMIC BOUNDARY ===                      │       │
│  ├────────────────────────────────────────────────┤       │
│  │  DYNAMIC BACK HALF (changes per turn)          │       │
│  │  ├── Current file context                      │       │
│  │  ├── Recent changes and diffs                  │       │
│  │  ├── Active skill injections                   │       │
│  │  ├── System reminders (turn-specific)          │       │
│  │  └── Feature flag overrides                    │       │
│  └────────────────────────────────────────────────┘       │
│                                                           │
│  Only the FRONT HALF benefits from prompt caching.        │
│  Everything after the boundary is paid at full price.     │
└──────────────────────────────────────────────────────────┘
```

The source code reveals a tag called `DANGEROUS_uncachedSystemPromptSection`. Any content placed in a section marked with this tag explicitly breaks the prompt cache boundary. The naming is deliberate -- it forces engineers to acknowledge the cost before modifying it. Adding ten lines to the cached portion costs nothing. Adding ten lines that cross the boundary costs real money on every turn of every session.

**Why subagent forking is cheap.** This boundary design explains a surprising cost property: spawning five subagents costs barely more than spawning one. When Claude Code forks a subagent, the child inherits the parent's system prompt as a byte-identical copy. Because the stable front half is identical across parent and children, all of them share the same cache entry. The API serves the cached portion at 90% discount for every subagent. The only per-agent cost is the dynamic back half (which is small) and the conversation-specific content.

```
Parent agent:
  System prompt (stable): CACHED     $0.0023
  System prompt (dynamic): full       $0.002
  Conversation: full                  $0.015
  Total:                              $0.019

5 subagents (forked from parent):
  System prompt (stable): CACHED x5  $0.0115  (5 * $0.0023)
  System prompt (dynamic): full x5   $0.010
  Conversations: full x5             $0.075
  Total:                             $0.097

Without cache sharing, 5 subagents would cost:
  System prompt (full): x5           $0.115
  Conversations: full x5             $0.075
  Total:                             $0.190

Cache boundary savings for 5 subagents: ~49%
```

**Practical lesson for your own agents.** If you are building any system that makes repeated API calls with a shared system prompt -- which is every agent -- split your system prompt at a stable boundary. Put everything that does not change turn-to-turn in the front. Put everything dynamic after the boundary. This is the single highest-ROI optimization for multi-turn agents:

```typescript
// Cache-boundary-aware prompt construction
function buildSystemPrompt(config: AgentConfig): SystemPrompt {
  // STABLE SECTION: never changes within a session
  // Order matters — this must come first for cache hits
  const stableSection = [
    config.coreInstructions,      // Identity, behavior rules
    config.toolDefinitions,        // Tool schemas (frozen at session start)
    config.permissionRules,        // Allow/deny lists
    config.projectContext,         // CLAUDE.md equivalent
  ].join('\n');

  // DYNAMIC SECTION: may change every turn
  const dynamicSection = [
    config.currentFileContext,     // What file is open now
    config.recentChanges,          // Git diff, recent edits
    config.activeSkills,           // Currently loaded skills
    config.turnSpecificReminders,  // System reminders for this turn
  ].join('\n');

  return {
    // The API will cache everything up to stableSection
    // dynamicSection is paid at full price every turn
    content: stableSection + '\n' + dynamicSection,
    cacheBreakpoint: stableSection.length,
  };
}
```

---

## 5. Why Switching Models Mid-Session Kills Your Prompt Cache

### 5.1 The Full Picture

Model switching is such a common and expensive mistake that it deserves its own section.

```
┌────────────────────────────────────────────────────────┐
│         WHY MODEL SWITCHING IS EXPENSIVE                │
│                                                         │
│  Cache key includes:                                    │
│  ┌─────────────────────────────────────────────┐       │
│  │  model_id + system_prompt + tools + messages │       │
│  └─────────────────────────────────────────────┘       │
│                                                         │
│  When you switch from claude-sonnet to claude-opus:     │
│                                                         │
│  1. model_id changes           → cache key changes     │
│  2. system prompt may change   → cache key changes     │
│  3. tool schemas may differ    → cache key changes     │
│  4. tokenization differs       → cache key changes     │
│                                                         │
│  Result: COMPLETE cache miss on the next turn           │
│  The entire system prompt + history is re-processed     │
│  at full price                                          │
│                                                         │
│  AND: the OLD model's cache is now useless             │
│  You paid to build it and now it's abandoned            │
└────────────────────────────────────────────────────────┘
```

### 5.2 When People Switch Models

Common reasons users switch models mid-session (and why they should not):

| Reason | Better Alternative |
|--------|-------------------|
| "Sonnet can't handle this complex task" | Start a new session with Opus |
| "I want Haiku for this simple part" | Use a sub-agent with Haiku (separate cache) |
| "Opus is too slow for this" | Use the `Agent` tool which can use a different model |
| "I want to try a different model" | Start a new session |

The key insight: sub-agents (Ch 32) have their own cache. You can use a different model in a sub-agent without breaking the parent agent's cache.

### 5.3 The Session-Model Contract

```typescript
// Best practice: lock the model at session creation
interface Session {
  readonly model: string;  // Immutable after creation
  
  // If you need a different model, spawn a sub-agent
  async spawnSubAgent(config: {
    model: string;  // Can be different from parent
    prompt: string;
    tools?: string[];
  }): Promise<string>;
}
```

---

## 6. The Compaction Service Architecture

### 6.1 The Full Service

The compaction service is not just a function -- it is a stateful service with monitoring, triggering, and recovery:

```typescript
// Compaction service architecture (reconstructed)
class CompactionService {
  private monitor: TokenMonitor;
  private summarizer: Summarizer;
  private config: CompactionConfig;
  
  // Monitor token usage after each turn
  async afterTurn(
    systemPrompt: string,
    messages: Message[]
  ): Promise<CompactionResult | null> {
    const tokenCount = await this.monitor.count(
      systemPrompt,
      messages
    );
    
    // Check thresholds
    if (tokenCount < this.config.compactionThreshold) {
      return null;  // No compaction needed
    }
    
    // Determine compaction strategy
    const strategy = this.selectStrategy(tokenCount, messages);
    
    // Execute compaction
    const result = await this.execute(strategy, messages);
    
    // Verify compaction worked
    const newCount = await this.monitor.count(
      systemPrompt,
      result.messages
    );
    
    if (newCount > this.config.compactionTarget * 1.5) {
      // Compaction was not aggressive enough
      // Try again with more aggressive settings
      return this.execute(
        { ...strategy, aggressive: true },
        result.messages
      );
    }
    
    return result;
  }
  
  private selectStrategy(
    tokenCount: number,
    messages: Message[]
  ): CompactionStrategy {
    // How far over the threshold are we?
    const overageRatio = tokenCount / this.config.compactionThreshold;
    
    if (overageRatio < 1.2) {
      // Slightly over: gentle compaction
      return {
        type: 'summarize',
        keepRecent: 15,
        summaryDetail: 'detailed'
      };
    }
    
    if (overageRatio < 1.5) {
      // Moderately over: standard compaction
      return {
        type: 'summarize',
        keepRecent: 10,
        summaryDetail: 'standard'
      };
    }
    
    // Way over: aggressive compaction
    return {
      type: 'summarize',
      keepRecent: 5,
      summaryDetail: 'brief',
      aggressive: true
    };
  }
  
  private async execute(
    strategy: CompactionStrategy,
    messages: Message[]
  ): Promise<CompactionResult> {
    const recent = messages.slice(-strategy.keepRecent);
    const old = messages.slice(0, -strategy.keepRecent);
    
    // Summarize old messages
    const summary = await this.summarizer.summarize(
      old,
      strategy.summaryDetail
    );
    
    const compacted: Message[] = [
      {
        role: 'user',
        content: `[Context from earlier in this session]\n${summary}`
      },
      {
        role: 'assistant',
        content: 'I have the context. Let me continue.'
      },
      ...recent
    ];
    
    return {
      messages: compacted,
      tokensSaved: this.monitor.quickCount(old)
        - this.monitor.quickCount(compacted.slice(0, 2)),
      messagesRemoved: old.length,
      messagesKept: recent.length,
    };
  }
}
```

### 6.2 The Cost of Compaction

Compaction itself costs tokens:

```
Compaction cost breakdown:
  1. Read old messages to build summary prompt:  FREE (already in memory)
  2. Send summary prompt to model:               ~1000-5000 input tokens
  3. Model generates summary:                    ~500-2000 output tokens
  4. Total compaction cost:                      ~$0.005-0.02 per compaction

But compaction SAVES:
  Before: 150,000 token context on every subsequent turn
  After:  50,000 token context on every subsequent turn
  Savings: 100,000 tokens per turn * remaining turns
  
  If 20 more turns remain: 2,000,000 tokens saved
  At $3/M input: $6.00 saved
  
  ROI: spend $0.02, save $6.00 = 300x return
```

Compaction is almost always worth it economically. The exception is very short sessions where the overhead of the compaction call is not amortized over many subsequent turns.

---

## 7. Statsig and Feature Flags for Session Limits

### 7.1 What the Leak Revealed

The Claude Code source contained Statsig integration for A/B testing session parameters:

```typescript
// Feature flag integration (reconstructed from the leak)
import { Statsig } from 'statsig-node';

async function getSessionConfig(
  userId: string,
  userTier: 'free' | 'pro' | 'enterprise'
): Promise<SessionConfig> {
  // Dynamic configuration from Statsig
  const dynamicConfig = await Statsig.getExperiment(
    userId,
    'session_limits_v3'
  );
  
  return {
    maxTurns: dynamicConfig.get('max_turns', 200),
    maxTokensPerTurn: dynamicConfig.get('max_tokens_per_turn', 16384),
    compactionThreshold: dynamicConfig.get(
      'compaction_threshold', 150000
    ),
    compactionModel: dynamicConfig.get(
      'compaction_model', 'claude-haiku-4-20250514'
    ),
    sessionTimeout: dynamicConfig.get(
      'session_timeout_minutes', 120
    ),
    maxConcurrentSessions: dynamicConfig.get(
      'max_concurrent', 5
    ),
  };
}
```

### 7.2 What This Means

Feature flags mean that different users may have different experiences:

```
┌────────────────────────────────────────────────────┐
│          FEATURE FLAG IMPLICATIONS                  │
│                                                      │
│  Different users might have:                         │
│  ├── Different max turns per session                 │
│  ├── Different compaction thresholds                 │
│  ├── Different models for compaction (Haiku vs       │
│  │   Sonnet for summaries)                           │
│  ├── Different session timeouts                      │
│  ├── Different max concurrent sessions               │
│  └── Different available tools                       │
│                                                      │
│  This enables:                                       │
│  ├── A/B testing: "does a 120K threshold produce     │
│  │   better outcomes than 150K?"                     │
│  ├── Gradual rollout: "try new compaction with 5%    │
│  │   of users first"                                 │
│  ├── Tier differentiation: "Pro gets longer sessions" │
│  └── Cost control: "reduce thresholds if costs spike" │
└────────────────────────────────────────────────────┘
```

### 7.3 The Implication for Your Own Tools

If you build AI tools, consider feature flags for:

```typescript
// Feature flags worth controlling
const AI_FEATURE_FLAGS = {
  // Cost controls
  'max_tokens_per_request': 8192,
  'max_requests_per_session': 200,
  'compaction_threshold': 150000,
  
  // Quality controls
  'compaction_model': 'claude-haiku-4-20250514',
  'default_model': 'claude-sonnet-4-20250514',
  
  // Feature access
  'web_search_enabled': true,
  'multi_agent_enabled': false,
  'auto_mode_enabled': true,
  
  // Rate limits
  'requests_per_minute': 30,
  'tokens_per_hour': 1_000_000,
};
```

---

## 8. The db8 Function and Token Drain

### 8.1 Community Discovery

After the leak, community researchers identified a function they nicknamed "db8" (the actual obfuscated name) that appeared to consume significant tokens without obvious benefit:

```typescript
// The db8 pattern (simplified reconstruction)
// This function was called on certain turns, adding tokens
// to the context that were not visible to the user

function db8(messages: Message[], config: Config): Message[] {
  // Injected diagnostic/telemetry content into the message stream
  // The exact purpose was debated in the community
  
  // Hypothesis 1: Anti-abuse monitoring
  // Adds hidden markers that help detect misuse
  
  // Hypothesis 2: Context quality signals
  // Adds signals that help the model understand
  // the quality/reliability of context
  
  // Hypothesis 3: Debug instrumentation
  // Left-over debugging code that was not fully removed
  
  // Whatever the purpose, it added ~200-500 tokens
  // on certain turns, contributing to faster context fill
  
  return augmentedMessages;
}
```

### 8.2 The Token Drain Pattern

The broader lesson is that token drain can come from unexpected sources:

```
┌────────────────────────────────────────────────────┐
│          SOURCES OF TOKEN DRAIN                     │
│                                                      │
│  OBVIOUS:                                            │
│  ├── Large file reads                                │
│  ├── Long command outputs                            │
│  ├── Verbose model responses                         │
│  └── CLAUDE.md injection                             │
│                                                      │
│  SUBTLE:                                             │
│  ├── Tool definitions (re-sent every turn)           │
│  ├── Permission context (re-sent every turn)         │
│  ├── Error messages from failed tools                │
│  ├── Diagnostic/telemetry injections                 │
│  ├── System reminder injections                      │
│  ├── Skill loading/unloading messages                │
│  └── Invisible system messages between turns         │
│                                                      │
│  The subtle sources add up:                          │
│  Tool definitions: ~1,500 tokens/turn                │
│  Permission context: ~500 tokens/turn                │
│  System reminders: ~200 tokens/turn (variable)       │
│  ─────────────────────────────────────               │
│  Total subtle overhead: ~2,200+ tokens/turn          │
│                                                      │
│  Over 100 turns: 220,000 tokens of overhead          │
│  (More than the context window!)                     │
│                                                      │
│  This is why prompt caching is essential.             │
│  Without it, the overhead cost is enormous.           │
└────────────────────────────────────────────────────┘
```

---

## 9. Cost Implications of Every Design Choice

### 9.1 The Cost Model

Every architectural decision in context management has a cost:

```typescript
// Cost model for context management decisions
function estimateSessionCost(session: SessionParams): CostEstimate {
  const inputCostPerToken = session.model === 'opus' ? 15 / 1_000_000
    : session.model === 'sonnet' ? 3 / 1_000_000
    : 0.25 / 1_000_000;  // haiku
  
  const outputCostPerToken = inputCostPerToken * 5;  // 5x for output
  
  // Fixed cost per turn (system prompt + tools)
  const fixedCostPerTurn =
    session.systemPromptTokens * inputCostPerToken;
  
  // Cache discount (if cache holds)
  const cachedCostPerTurn = session.cacheHitRate > 0
    ? fixedCostPerTurn * (1 - session.cacheHitRate * 0.9)
    : fixedCostPerTurn;
  
  // Variable cost per turn (history)
  const avgHistoryTokens = session.avgHistoryTokens;
  const historyCostPerTurn =
    avgHistoryTokens * inputCostPerToken;
  
  // Compaction cost (when it happens)
  const compactionCost =
    session.compactionInputTokens * inputCostPerToken
    + session.compactionOutputTokens * outputCostPerToken;
  
  // Total
  const turnsBeforeCompaction = Math.floor(
    (session.contextWindowSize - session.systemPromptTokens)
    / session.avgTokensPerTurn
  );
  
  return {
    costPerTurn: cachedCostPerTurn + historyCostPerTurn,
    compactionCost,
    turnsBeforeCompaction,
    estimatedSessionCost:
      session.expectedTurns * (cachedCostPerTurn + historyCostPerTurn)
      + Math.ceil(session.expectedTurns / turnsBeforeCompaction)
        * compactionCost,
  };
}
```

### 9.2 Decision Cost Matrix

```
┌────────────────────────────┬───────────┬──────────────┐
│ Decision                   │ Per-turn  │ Per-session   │
│                            │ cost      │ cost (100t)   │
├────────────────────────────┼───────────┼──────────────┤
│ +100 tokens in CLAUDE.md   │ +$0.0003  │ +$0.03       │
│ +500 tokens in CLAUDE.md   │ +$0.0015  │ +$0.15       │
│ Add a skill (+2000 tokens) │ +$0.006   │ +$0.60       │
│ Model switch (cache break) │ +$0.02    │ +$0.02 (1x)  │
│ Read 500-line file         │ +$0.015   │ +$0.015 (1x) │
│ Compaction event           │ +$0.01    │ -$2.00+      │
│ Tool result not truncated  │ +$0.03    │ +$0.03 (1x)  │
│ Cache hit rate 90% -> 50%  │ +$0.005   │ +$0.50       │
└────────────────────────────┴───────────┴──────────────┘
  (Estimated for Sonnet at $3/M input tokens)
```

### 9.3 The Optimization Hierarchy

In order of impact, optimize:

```
1. PROMPT CACHING (biggest impact)
   Keep the cache alive. Avoid the 14 break vectors.
   Savings: up to 90% of fixed overhead.

2. TOOL RESULT TRUNCATION (second biggest)
   Don't put 10,000-line files in context.
   Use offset/limit. Truncate command outputs.
   Savings: prevents premature compaction.

3. CONCISE CLAUDE.md (third biggest)
   Every token costs on every turn.
   Target: under 500 tokens.
   Savings: proportional to session length.

4. SMART COMPACTION (fourth)
   Compact early enough to avoid the hard limit.
   Use a cheap model for summaries.
   Savings: extends session lifetime.

5. SKILL MANAGEMENT (fifth)
   Don't load skills you don't need.
   Unload skills when done.
   Savings: proportional to skill size * turns active.
```

---

## 10. Building Your Own Context Manager

### 10.1 A Production-Ready Context Manager

```typescript
// context-manager.ts
// A reusable context management system

interface ContextManagerConfig {
  maxContextTokens: number;      // Model's context window
  reservedOutputTokens: number;  // Space for model response
  compactionThreshold: number;   // When to trigger compaction
  compactionTarget: number;      // Target size after compaction
  keepRecentMessages: number;    // Messages to protect
  maxToolResultTokens: number;   // Per-result truncation limit
  compactionModel: string;       // Model for summaries
}

class ContextManager {
  private config: ContextManagerConfig;
  private tokenCounter: TokenCounter;
  
  constructor(config: ContextManagerConfig) {
    this.config = config;
    this.tokenCounter = new TokenCounter();
  }
  
  // Check if compaction is needed before a model call
  needsCompaction(
    systemPrompt: string,
    messages: Message[]
  ): boolean {
    const total = this.tokenCounter.countAll(
      systemPrompt, messages
    );
    return total > this.config.compactionThreshold;
  }
  
  // Get available token budget for the model's response
  availableBudget(
    systemPrompt: string,
    messages: Message[]
  ): number {
    const used = this.tokenCounter.countAll(
      systemPrompt, messages
    );
    return this.config.maxContextTokens
      - used
      - this.config.reservedOutputTokens;
  }
  
  // Truncate a tool result to fit within budget
  truncateToolResult(result: string): string {
    const tokens = this.tokenCounter.count(result);
    if (tokens <= this.config.maxToolResultTokens) {
      return result;
    }
    
    // Binary search for the right truncation point
    let lo = 0;
    let hi = result.length;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      const truncated = result.slice(0, mid) + '\n[...truncated]';
      if (
        this.tokenCounter.count(truncated)
          <= this.config.maxToolResultTokens
      ) {
        lo = mid + 1;
      } else {
        hi = mid;
      }
    }
    
    return result.slice(0, lo - 1) + '\n[...truncated]';
  }
  
  // Perform compaction
  async compact(
    messages: Message[],
    client: Anthropic
  ): Promise<CompactionResult> {
    const keepCount = this.config.keepRecentMessages;
    const recent = messages.slice(-keepCount);
    const old = messages.slice(0, -keepCount);
    
    if (old.length === 0) {
      return { messages, didCompact: false };
    }
    
    // Generate summary using a (potentially cheaper) model
    const summaryResponse = await client.messages.create({
      model: this.config.compactionModel,
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: this.buildSummaryPrompt(old)
      }]
    });
    
    const summary = summaryResponse.content
      .filter(b => b.type === 'text')
      .map(b => b.text)
      .join('');
    
    const compacted: Message[] = [
      {
        role: 'user',
        content: `[Summary of previous ${old.length} messages]\n\n${summary}`
      },
      {
        role: 'assistant',
        content: 'Understood, I have the context. Continuing.'
      },
      ...recent
    ];
    
    return {
      messages: compacted,
      didCompact: true,
      tokensSaved: this.tokenCounter.countMessages(old)
        - this.tokenCounter.countMessages(compacted.slice(0, 2)),
    };
  }
  
  private buildSummaryPrompt(messages: Message[]): string {
    const formatted = messages.map(m => {
      if (typeof m.content === 'string') {
        return `${m.role}: ${m.content}`;
      }
      // Handle tool use/result blocks
      return `${m.role}: [complex content with ${
        Array.isArray(m.content) ? m.content.length : 1
      } blocks]`;
    }).join('\n\n');
    
    return `Summarize this conversation excerpt concisely.
Focus on: decisions made, files modified, current task state,
problems encountered, and important context.

${formatted}`;
  }
}

// Token counter (simplified -- production would use tiktoken)
class TokenCounter {
  // Rough approximation: 4 chars per token for English
  count(text: string): number {
    return Math.ceil(text.length / 4);
  }
  
  countMessages(messages: Message[]): number {
    return messages.reduce((total, msg) => {
      if (typeof msg.content === 'string') {
        return total + this.count(msg.content) + 4; // overhead
      }
      if (Array.isArray(msg.content)) {
        return total + msg.content.reduce((sum, block) => {
          if (block.type === 'text') return sum + this.count(block.text);
          if (block.type === 'tool_result') {
            return sum + this.count(
              typeof block.content === 'string'
                ? block.content
                : JSON.stringify(block.content)
            );
          }
          return sum + 50; // estimate for other block types
        }, 0) + 4;
      }
      return total + 50;
    }, 0);
  }
  
  countAll(systemPrompt: string, messages: Message[]): number {
    return this.count(systemPrompt) + this.countMessages(messages);
  }
}
```

### 10.2 Integration with the Agent Loop

```typescript
// Using the context manager in an agent loop
async function managedAgentLoop(
  config: AgentConfig
): Promise<void> {
  const contextManager = new ContextManager({
    maxContextTokens: 200_000,
    reservedOutputTokens: 8_192,
    compactionThreshold: 150_000,
    compactionTarget: 50_000,
    keepRecentMessages: 10,
    maxToolResultTokens: 5_000,
    compactionModel: 'claude-haiku-4-20250514',
  });
  
  const messages: Message[] = [];
  const systemPrompt = buildSystemPrompt(config);
  
  while (true) {
    // Check if compaction is needed
    if (contextManager.needsCompaction(systemPrompt, messages)) {
      const result = await contextManager.compact(messages, client);
      if (result.didCompact) {
        messages.length = 0;
        messages.push(...result.messages);
        console.log(`Compacted. Saved ~${result.tokensSaved} tokens.`);
      }
    }
    
    // Make the model call
    const response = await client.messages.create({
      model: config.model,
      system: systemPrompt,
      messages,
      tools: config.tools,
      max_tokens: config.maxTokens,
    });
    
    // Process tool results with truncation
    for (const block of response.content) {
      if (block.type === 'tool_use') {
        const rawResult = await executeTool(block);
        const truncated = contextManager.truncateToolResult(rawResult);
        // Use truncated result in messages
      }
    }
  }
}
```

---

## Build It: A Context Compaction Service in 60 Lines

The compaction service is the heart of any long-running agent. Without it, your context window fills up and the session dies. The implementation below shows the complete pattern: token counting, budget checking, summarizing old messages into a single summary, preserving recent messages, and splitting the system prompt into a stable (cacheable) part and a dynamic part that gets rebuilt each turn. This is exactly what Claude Code's compaction service does, stripped to the essential algorithm.

```bash
pip install openai tiktoken
```

```python
# Context compaction service — the pattern that keeps agent sessions alive
# Usage: python compaction.py

import tiktoken
from openai import OpenAI
from typing import Optional

# --- Token Counting -----------------------------------------------------------

_encoder = tiktoken.get_encoding("cl100k_base")

def count_tokens(text: str) -> int:
    """Count tokens using tiktoken (same tokenizer as GPT-4 / Claude-ish)."""
    return len(_encoder.encode(text))

def count_message_tokens(messages: list[dict]) -> int:
    """Count total tokens across all messages."""
    total = 0
    for msg in messages:
        total += 4  # message overhead (role, separators)
        total += count_tokens(msg.get("content", "") or "")
    return total

# --- Cache Boundary: Stable vs Dynamic System Prompt -------------------------

def build_system_prompt(stable_instructions: str, dynamic_context: str) -> list[dict]:
    """Split system prompt into stable (cached) and dynamic (rebuilt each turn).
    The stable part stays identical across calls -> prompt cache hit.
    The dynamic part (CLAUDE.md, active skills) changes per turn."""
    return [
        # Stable part: identical every call, eligible for prompt caching
        {"role": "system", "content": stable_instructions},
        # Dynamic part: rebuilt each turn with current context
        {"role": "system", "content": f"[Current context]\n{dynamic_context}"},
    ]

# --- Compaction: Summarize Old Messages, Keep Recent --------------------------

def compact_messages(
    messages: list[dict],
    max_tokens: int = 100_000,
    preserved_tail: int = 10,
    model: str = "gpt-4o-mini",
) -> list[dict]:
    """If messages exceed the token budget, summarize old messages into one.
    
    Strategy:
    1. Always keep the system messages (index 0, 1) 
    2. Always keep the last `preserved_tail` messages (recent context)
    3. Summarize everything in between into a single summary message
    """
    total_tokens = count_message_tokens(messages)
    if total_tokens <= max_tokens:
        return messages  # under budget, no compaction needed

    print(f"[COMPACTION] {total_tokens} tokens exceeds {max_tokens} budget. Compacting...")

    # Separate system messages, middle (compactable), and tail (preserved)
    system_msgs = [m for m in messages[:2] if m["role"] == "system"]
    non_system = [m for m in messages if m["role"] != "system"]

    if len(non_system) <= preserved_tail:
        return messages  # not enough messages to compact

    to_summarize = non_system[:-preserved_tail]
    to_keep = non_system[-preserved_tail:]

    # Build a text representation of messages to summarize
    summary_input = ""
    for msg in to_summarize:
        role = msg["role"]
        content = (msg.get("content", "") or "")[:500]  # truncate long messages
        summary_input += f"[{role}]: {content}\n"

    # Call a cheap model to summarize
    client = OpenAI()
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": (
                "Summarize the following conversation history concisely. "
                "Preserve: key decisions, file paths mentioned, errors encountered, "
                "and the user's goals. Drop: verbose tool outputs, repeated attempts, "
                "routine acknowledgments. Be factual and brief."
            )},
            {"role": "user", "content": summary_input},
        ],
    )
    summary = response.choices[0].message.content

    # Build compacted message array
    summary_msg = {
        "role": "user",
        "content": f"[Previous conversation summary]\n{summary}"
    }

    compacted = system_msgs + [summary_msg] + to_keep
    new_tokens = count_message_tokens(compacted)
    saved = total_tokens - new_tokens
    print(f"[COMPACTION] {total_tokens} -> {new_tokens} tokens (saved {saved})")
    return compacted

# --- Full Context Manager -----------------------------------------------------

class ContextManager:
    def __init__(self, stable_prompt: str, max_tokens: int = 100_000, preserved_tail: int = 10):
        self.stable_prompt = stable_prompt
        self.dynamic_context = ""
        self.max_tokens = max_tokens
        self.preserved_tail = preserved_tail
        self.messages: list[dict] = []

    def initialize(self, dynamic_context: str = ""):
        """Set up initial messages with cache-boundary system prompt."""
        self.dynamic_context = dynamic_context
        self.messages = build_system_prompt(self.stable_prompt, dynamic_context)

    def add_message(self, role: str, content: str):
        """Add a message and compact if over budget."""
        self.messages.append({"role": role, "content": content})
        self.messages = compact_messages(
            self.messages, self.max_tokens, self.preserved_tail
        )

    def get_messages(self) -> list[dict]:
        return self.messages

    def token_usage(self) -> dict:
        total = count_message_tokens(self.messages)
        return {"total": total, "budget": self.max_tokens, "remaining": self.max_tokens - total}


if __name__ == "__main__":
    ctx = ContextManager(
        stable_prompt="You are a helpful coding assistant.",
        max_tokens=500,  # low threshold for demo purposes
        preserved_tail=3,
    )
    ctx.initialize(dynamic_context="Project: demo-app, Language: Python")

    # Simulate a conversation that exceeds the budget
    exchanges = [
        ("user", "Read the file src/main.py"),
        ("assistant", "Here is the content of src/main.py:\n" + "x = 1\n" * 50),
        ("user", "Now read tests/test_main.py"),
        ("assistant", "Here is test_main.py:\n" + "def test(): pass\n" * 50),
        ("user", "Fix the failing test"),
        ("assistant", "I updated the test to check the return value."),
        ("user", "Run the tests"),
        ("assistant", "All 12 tests passed."),
        ("user", "Great, now add a docstring to main.py"),
    ]

    for role, content in exchanges:
        ctx.add_message(role, content)
        usage = ctx.token_usage()
        print(f"  [{role}] tokens: {usage['total']}/{usage['budget']}")

    print(f"\nFinal message count: {len(ctx.get_messages())}")
    print(f"Final token usage: {ctx.token_usage()}")
```

The critical insight is the preserved tail: recent messages are never summarized because they contain the context the model needs to continue working. Old messages get compressed into a factual summary. This is the same trade-off Claude Code makes -- and why long sessions still work after hundreds of turns.

---

## 11. Key Takeaways

1. **The context window is a budget.** Fixed overhead (system prompt, CLAUDE.md, tools) is paid on every turn. Dynamic content (history, tool results) grows over time. Reserve space for the model's response.

2. **Compaction is essential for long sessions.** Without it, sessions die around turn 30-35. With it, sessions can run for hundreds of turns. The ROI of compaction is typically 300x or better.

3. **The 14 cache-break vectors are real costs.** Model switching is the most expensive. Protect your cache by freezing the model, batching MCP connections, and monitoring CLAUDE.md changes.

4. **File reads dominate token consumption.** Use offset/limit parameters. Truncate tool results. Never read a whole file when you need 50 lines.

5. **Statsig/feature flags control session parameters.** Different users may have different limits. This enables A/B testing, gradual rollout, and tier differentiation.

6. **The db8 pattern shows that token drain comes from unexpected sources.** Audit your context for invisible overhead: tool definitions, permission context, system reminders, diagnostic injections.

7. **Concise CLAUDE.md saves money.** Every token costs on every turn. Target under 500 tokens for CLAUDE.md.

8. **The optimization hierarchy:** caching > truncation > concise memory > smart compaction > skill management.

---

*Previous: [Chapter 28 -- Memory Systems: KAIROS & Beyond](./28-memory-systems.md)* · *Next: [Chapter 30 -- Tool Execution & Permissions](./30-tool-execution.md)* dives into how the harness converts model intentions into real-world actions -- and how it decides what the model is allowed to do.

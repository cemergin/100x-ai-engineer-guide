<!--
  CHAPTER: 28
  TITLE: Memory Systems: KAIROS & Beyond
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 10 (Agent Memory & State), Ch 23 (Claude Code Mastery), Ch 27 (Harness Architecture)
  KEY_TOPICS: KAIROS, three-layer memory, CLAUDE.md injection, AutoDream, append-only logs, semantic memory merging, codebase indexing, memory architecture design
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 28: Memory Systems: KAIROS & Beyond

> **Part 6 — Anatomy of AI Developer Tools** | Phase 2: Become an Expert | Prerequisites: Ch 10, Ch 23, Ch 27 | Difficulty: Advanced | Language: TypeScript + Architecture

An LLM has no memory. Every time you call the API, the model starts fresh -- it does not know who you are, what project you are working on, or what you discussed five minutes ago. Everything the model "remembers" is something the harness explicitly puts into the context window. The quality of an AI developer tool is therefore directly proportional to the quality of its memory system: what it remembers, how it organizes memories, and when it decides to forget.

The March 2026 Claude Code leak revealed KAIROS, a three-layer memory system far more sophisticated than anyone expected. It also revealed AutoDream, a background consolidation process that reviews, prunes, and merges memories while you sleep. This chapter dissects both systems and teaches you how to design memory for your own agents.

### In This Chapter

1. The memory problem: why LLMs need external memory
2. KAIROS: the three-layer architecture
3. Layer 1: Session memory (conversation history)
4. Layer 2: Project memory (CLAUDE.md and injection mechanics)
5. Layer 3: Persistent memory (long-term files and logs)
6. AutoDream: background memory consolidation
7. Cursor's approach: codebase indexing as memory
8. Append-only logs vs. rewriting strategies
9. Semantic memory merging and deduplication
10. Designing memory for your own agents

### Related Chapters

- **Ch 10 (Agent Memory & State)** -- introduced memory concepts; this chapter shows production architecture
- **Ch 23 (Claude Code Mastery)** -- taught CLAUDE.md usage; this chapter explains injection mechanics
- **Ch 27 (Harness Architecture)** -- showed where memory fits in the harness
- **Ch 50 (Advanced Context Strategies)** -- will build on these patterns for production systems
- **Ch 59 (Building Internal AI Tools)** -- will use these memory patterns in platform tools

---

## 1. The Memory Problem

### 1.1 What the Model Cannot Do

A language model is a stateless function:

```
f(tokens_in) -> tokens_out
```

It has no built-in ability to:
- Remember previous conversations
- Learn your project's conventions
- Recall decisions made last week
- Track what files it has already read
- Build up knowledge over time

Everything the model "knows" about your specific situation must be placed into the context window on every call. This is expensive (tokens cost money), limited (context windows have a maximum size), and lossy (old context gets evicted or summarized).

### 1.2 The Three Horizons of Memory

AI developer tools need memory at three different time scales:

```
┌──────────────────────────────────────────────────────────┐
│                  MEMORY TIME HORIZONS                     │
│                                                           │
│  SESSION (minutes to hours)                               │
│  ├── "I just read this file"                              │
│  ├── "The user asked me to use async/await"               │
│  └── "The tests failed because of a missing import"       │
│                                                           │
│  PROJECT (days to months)                                 │
│  ├── "This project uses Next.js with the App Router"      │
│  ├── "Tests go in __tests__ directories"                  │
│  └── "Never import from the legacy/ directory"            │
│                                                           │
│  PERSISTENT (months to forever)                           │
│  ├── "This user prefers functional over class components"  │
│  ├── "The database migration pattern we settled on"       │
│  └── "That one weird webpack config issue and its fix"    │
└──────────────────────────────────────────────────────────┘
```

Each horizon has different requirements for storage, retrieval, and eviction. KAIROS addresses all three.

---

## 2. KAIROS: The Three-Layer Architecture

KAIROS (the name found in the leaked source) organizes memory into three layers, each with different characteristics:

```
┌─────────────────────────────────────────────────────────┐
│                       KAIROS                             │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  LAYER 3: PERSISTENT MEMORY                        │  │
│  │  Storage: ~/.claude/memory/ files                  │  │
│  │  Lifetime: Across all sessions, all projects       │  │
│  │  Managed by: AutoDream (background consolidation)  │  │
│  │  Examples: User preferences, learned patterns,     │  │
│  │            cross-project knowledge                 │  │
│  └────────────────────────────────────────────────────┘  │
│                          ↑                               │
│                    promotion │                           │
│                          │                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │  LAYER 2: PROJECT MEMORY                           │  │
│  │  Storage: CLAUDE.md files (project root + nested)  │  │
│  │  Lifetime: Duration of the project                 │  │
│  │  Injected: On every turn (system prompt)           │  │
│  │  Examples: Coding conventions, architecture notes, │  │
│  │            known issues, team preferences          │  │
│  └────────────────────────────────────────────────────┘  │
│                          ↑                               │
│                   extraction │                           │
│                          │                               │
│  ┌────────────────────────────────────────────────────┐  │
│  │  LAYER 1: SESSION MEMORY                           │  │
│  │  Storage: Conversation history (message array)     │  │
│  │  Lifetime: Current session only                    │  │
│  │  Managed by: Compaction service (Ch 29)            │  │
│  │  Examples: Recent tool results, user instructions, │  │
│  │            current task context                    │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

The layers interact through two processes:
1. **Extraction** -- useful patterns discovered during sessions are extracted into project memory (CLAUDE.md)
2. **Promotion** -- patterns that appear across multiple projects are promoted to persistent memory

---

## 3. Layer 1: Session Memory (Conversation History)

### 3.1 What It Contains

Session memory is the conversation history: every message the user sent, every response the model generated, every tool call and its result. It is the most immediate and detailed form of memory.

```typescript
// Session memory is literally the messages array
interface SessionMemory {
  messages: Message[];  // The full conversation
  
  // Each message can be:
  // - user text ("Fix the login bug")
  // - assistant text ("I'll look at the auth module...")
  // - assistant tool_use (Read file, Edit file, etc.)
  // - user tool_result (file contents, command output, etc.)
}
```

### 3.2 The Problem With Session Memory

Session memory is perfect for short tasks. But for long sessions:

```
Turn  1: 200 tokens   (user prompt + response)
Turn  5: 3,000 tokens (accumulated tool results)
Turn 15: 25,000 tokens (several file reads)
Turn 30: 80,000 tokens (extensive exploration)
Turn 50: 150,000 tokens (approaching context limit)
Turn 60: CONTEXT WINDOW FULL -- session degrades or dies
```

Every tool result, every file read, every command output accumulates in session memory. A single `cat` of a large file can add thousands of tokens. Without management, session memory fills the context window within a few dozen turns.

### 3.3 How Session Memory Is Managed

The compaction service (detailed in Ch 29) manages session memory by:

1. **Monitoring** -- tracking total token count after each turn
2. **Triggering** -- when tokens exceed a threshold, trigger compaction
3. **Summarizing** -- using a model to summarize old messages
4. **Preserving** -- keeping the most recent N messages intact
5. **Replacing** -- swapping old messages with their summary

```typescript
// Simplified compaction logic
async function compactSessionMemory(
  messages: Message[],
  config: CompactionConfig
): Promise<Message[]> {
  const recent = messages.slice(-config.keepRecentCount);
  const old = messages.slice(0, -config.keepRecentCount);
  
  // Summarize old messages with a separate model call
  const summary = await summarize(old, config.summaryModel);
  
  // Return: summary + recent messages
  return [
    { role: 'user', content: `[Previous context summary]\n${summary}` },
    ...recent
  ];
}
```

**The cost of compaction:** Each compaction requires a model call to generate the summary. This costs tokens and takes time. But the alternative -- running out of context window -- is worse.

---

## 4. Layer 2: Project Memory (CLAUDE.md)

### 4.1 What CLAUDE.md Is

CLAUDE.md is a markdown file at the root of your project (and optionally in subdirectories) that contains project-specific knowledge for the AI agent. Think of it as documentation that the agent reads on every turn.

```markdown
# CLAUDE.md

## Project Overview
This is a Next.js 15 app using the App Router with TypeScript.

## Architecture
- /src/app/ - Route handlers and pages
- /src/components/ - React components (named exports only)
- /src/lib/ - Shared utilities
- /src/db/ - Drizzle schema and queries

## Conventions
- Use `async/await` not `.then()` chains
- All components must have explicit TypeScript types (no `any`)
- Tests go in `__tests__/` directories next to source files
- Use `vi.mock()` for test mocks, never `jest.mock()`

## Known Issues
- The auth middleware has a race condition on cold starts (see #142)
- Don't import from `src/legacy/` - it's being deprecated

## Build & Test
- `pnpm dev` for development
- `pnpm test` for tests
- `pnpm build` for production build
```

### 4.2 The Injection Mechanics

The critical discovery from the leak: CLAUDE.md is not loaded once at session start. It is re-injected into the system prompt on every single turn.

```typescript
// On EVERY model call, not just the first one
function buildSystemPrompt(config: Config): string {
  let prompt = BASE_SYSTEM_PROMPT;
  
  // This runs every turn, not once
  const claudeMd = loadClaudeMd(config.projectRoot);
  if (claudeMd) {
    prompt += `\n\n<user-info>\n${claudeMd}\n</user-info>`;
  }
  
  return prompt;
}
```

**Why re-inject?** Because the model is stateless. It does not remember reading CLAUDE.md on turn 1. If you want the model to follow project conventions on turn 50, CLAUDE.md must be in the context on turn 50.

**The cost implication:**

```
CLAUDE.md size: 500 tokens
Turns per session: 100
Total CLAUDE.md tokens: 50,000 input tokens just for memory injection

At $3/M input tokens (Sonnet):
Cost per session just for CLAUDE.md: $0.15

With prompt caching (if cache holds):
Cost per session: ~$0.015 (90% savings)

But if cache breaks (Ch 29 has 14 reasons it can):
Cost per session: back to $0.15
```

This is why CLAUDE.md should be concise. Every extra line costs tokens on every turn.

### 4.3 Hierarchical CLAUDE.md

Claude Code supports nested CLAUDE.md files:

```
project-root/
├── CLAUDE.md            <-- Project-wide conventions
├── src/
│   ├── CLAUDE.md        <-- Source-specific notes
│   ├── api/
│   │   └── CLAUDE.md    <-- API-specific notes
│   └── components/
│       └── CLAUDE.md    <-- Component-specific notes
└── tests/
    └── CLAUDE.md        <-- Test-specific notes
```

The loading behavior:
1. Always load the root CLAUDE.md
2. When working in a subdirectory, also load that directory's CLAUDE.md
3. Walk up the directory tree, loading each CLAUDE.md found

```typescript
// Hierarchical loading (reconstructed)
function loadClaudeMdHierarchy(
  projectRoot: string,
  activeDirectory: string
): string {
  const memories: string[] = [];
  
  // Always load root
  const rootMd = loadFile(path.join(projectRoot, 'CLAUDE.md'));
  if (rootMd) memories.push(rootMd);
  
  // Walk from active directory up to project root
  let current = activeDirectory;
  while (current !== projectRoot && current.startsWith(projectRoot)) {
    const dirMd = loadFile(path.join(current, 'CLAUDE.md'));
    if (dirMd) memories.push(dirMd);
    current = path.dirname(current);
  }
  
  return memories.join('\n\n---\n\n');
}
```

**When to use nested CLAUDE.md:** When different parts of your codebase have different conventions. An API directory might have different naming conventions than a components directory. Nested files keep context focused.

### 4.4 The CLAUDE.md Lifecycle

CLAUDE.md evolves over time through a cycle:

```
┌─────────────────┐
│  Agent works on  │
│  the project     │
│  (session)       │
├─────────────────┤
│  Discovers       │     ┌─────────────────┐
│  patterns and    │ --> │  Updates         │
│  conventions     │     │  CLAUDE.md       │
└─────────────────┘     │  (manual or auto)│
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │  Next session    │
                         │  loads updated   │
                         │  CLAUDE.md       │
                         └─────────────────┘
```

Claude Code can update CLAUDE.md during a session (adding notes about discovered patterns). Some teams also update it manually as part of their development workflow.

---

## 5. Layer 3: Persistent Memory (Long-Term Files)

### 5.1 What Persistent Memory Contains

Persistent memory survives across sessions and potentially across projects. In the leaked architecture:

```
~/.claude/
├── memory/
│   ├── user-preferences.md      # How the user likes to work
│   ├── learned-patterns.md      # Patterns discovered across projects
│   └── logs/
│       └── 2026/
│           └── 03/
│               ├── 2026-03-15.md   # Session log for March 15
│               ├── 2026-03-16.md   # Session log for March 16
│               └── ...
├── projects/
│   ├── my-app/
│   │   └── context.md           # Per-project persistent context
│   └── other-project/
│       └── context.md
└── settings.json
```

### 5.2 Append-Only Daily Logs

The leak revealed an append-only logging pattern for persistent memory:

```typescript
// Daily log append pattern
function appendToSessionLog(entry: LogEntry): void {
  const today = new Date();
  const logDir = path.join(
    MEMORY_DIR,
    'logs',
    today.getFullYear().toString(),
    (today.getMonth() + 1).toString().padStart(2, '0')
  );
  const logFile = path.join(
    logDir,
    `${today.toISOString().split('T')[0]}.md`
  );
  
  fs.mkdirSync(logDir, { recursive: true });
  
  const logLine = [
    `## ${today.toISOString()}`,
    `**Project:** ${entry.project}`,
    `**Task:** ${entry.taskSummary}`,
    `**Key decisions:** ${entry.decisions.join(', ')}`,
    `**Files modified:** ${entry.filesModified.join(', ')}`,
    ''
  ].join('\n');
  
  fs.appendFileSync(logFile, logLine);
}
```

**Why append-only?** Append-only logs are:
- Safe from data loss (you never overwrite previous entries)
- Easy to implement (just append to a file)
- Chronological (natural time ordering)
- Good for auditing (you can see what the agent did and when)

**The downside:** Append-only logs grow without bound. After weeks of heavy use, the logs become too large to inject into context. This is where AutoDream comes in.

### 5.3 When Persistent Memory Is Loaded

Not all persistent memory is loaded on every turn. The loading strategy:

```
On session start:
  1. Always load: user-preferences.md (if it exists)
  2. Always load: current project's CLAUDE.md
  3. Conditionally load: recent session logs (last 1-3 days)
  4. Never auto-load: old session logs (too large)

On explicit request:
  5. Search: user asks "what did we decide about X?"
  6. Agent searches persistent memory files
  7. Relevant entries injected into context
```

---

## 6. AutoDream: Background Memory Consolidation

### 6.1 What AutoDream Does

AutoDream is a background process that runs when the user is not actively using Claude Code (hence the name -- it "dreams" while you are away). Its job is to consolidate memory:

```
┌─────────────────────────────────────────────────────────┐
│                     AutoDream Pipeline                    │
│                                                          │
│  1. REVIEW                                               │
│     Read all session logs from the past N days           │
│     Read current CLAUDE.md                               │
│     Read persistent memory files                         │
│                                                          │
│  2. EXTRACT                                              │
│     Identify new patterns, conventions, decisions        │
│     Identify stale or contradicted information           │
│     Identify duplicate entries                           │
│                                                          │
│  3. PRUNE                                                │
│     Remove stale entries (superseded by newer ones)      │
│     Remove one-off context (not recurring patterns)      │
│     Remove duplicates                                    │
│                                                          │
│  4. MERGE                                                │
│     Combine related entries into single coherent notes   │
│     Update CLAUDE.md with new stable patterns            │
│     Promote recurring cross-project patterns             │
│                                                          │
│  5. RE-INDEX                                             │
│     Update semantic embeddings for memory search         │
│     Rebuild the memory index for fast retrieval          │
└─────────────────────────────────────────────────────────┘
```

### 6.2 The Consolidation Model

AutoDream uses a model call to perform the consolidation:

```typescript
// AutoDream consolidation (conceptual reconstruction)
async function autoDeamConsolidate(
  config: AutoDreamConfig
): Promise<void> {
  // 1. Gather recent session logs
  const recentLogs = await loadRecentLogs(config.lookbackDays);
  
  // 2. Load current memory state
  const currentMemory = await loadPersistentMemory();
  const claudeMd = await loadClaudeMd(config.projectRoot);
  
  // 3. Ask the model to consolidate
  const consolidationPrompt = `
You are a memory consolidation agent. Review the following:

<current-memory>
${currentMemory}
</current-memory>

<claude-md>
${claudeMd}
</claude-md>

<recent-sessions>
${recentLogs}
</recent-sessions>

Your tasks:
1. Identify new patterns or conventions that should be 
   added to CLAUDE.md
2. Identify stale entries in CLAUDE.md that are no longer 
   accurate
3. Identify entries in persistent memory that can be 
   pruned (one-off, superseded)
4. Identify duplicate entries that should be merged
5. Output the updated CLAUDE.md and persistent memory

Respond with structured JSON containing the updates.
`;
  
  const response = await anthropic.messages.create({
    model: config.consolidationModel,  // Can use a cheaper model
    messages: [{ role: 'user', content: consolidationPrompt }],
    max_tokens: 4096,
  });
  
  // 4. Apply the updates
  const updates = parseConsolidationResponse(response);
  
  if (updates.claudeMdChanges) {
    await applyClaudeMdUpdates(updates.claudeMdChanges);
  }
  
  if (updates.memoryPruning) {
    await pruneMemoryEntries(updates.memoryPruning);
  }
  
  if (updates.memoryMerges) {
    await mergeMemoryEntries(updates.memoryMerges);
  }
}
```

### 6.3 When AutoDream Runs

AutoDream is designed to run during idle periods:

```
┌─────────────────────────────────────────┐
│         AutoDream Scheduling            │
│                                         │
│  Trigger 1: Session end                 │
│    After a session ends, schedule       │
│    consolidation for the next idle      │
│    period                               │
│                                         │
│  Trigger 2: Daily schedule              │
│    Run once per day (e.g., 3 AM)        │
│    if there are new session logs        │
│                                         │
│  Trigger 3: Memory threshold            │
│    When persistent memory files         │
│    exceed a size threshold, trigger     │
│    aggressive pruning                   │
│                                         │
│  NOT triggered: During active sessions  │
│    AutoDream never runs while the       │
│    user is actively working             │
└─────────────────────────────────────────┘
```

### 6.4 The Dream Analogy

The name "AutoDream" is not accidental. It parallels how human memory consolidation works during sleep:

| Human Sleep | AutoDream |
|-------------|-----------|
| Replays experiences | Reviews session logs |
| Strengthens important memories | Promotes recurring patterns |
| Prunes unneeded details | Removes one-off context |
| Consolidates related memories | Merges duplicate entries |
| Runs during inactivity | Runs during idle periods |

This is a genuinely elegant design. Memory consolidation during inactivity is biologically inspired and practically sound -- it avoids consuming resources during active work.

---

## 7. Cursor's Approach: Codebase Indexing as Memory

Cursor takes a fundamentally different approach to memory. Instead of maintaining explicit memory files, it treats the codebase itself as the memory and builds an index over it.

### 7.1 The Indexing Pipeline

```
┌────────────────────────────────────────────────────┐
│          CURSOR CODEBASE INDEXING                    │
│                                                      │
│  1. FILE DISCOVERY                                   │
│     Scan the project for all source files            │
│     Respect .gitignore and .cursorignore             │
│                                                      │
│  2. PARSING                                          │
│     Parse each file into an AST                      │
│     Extract symbols: functions, classes, types       │
│     Extract imports and dependencies                 │
│                                                      │
│  3. CHUNKING                                         │
│     Split files into semantic chunks                 │
│     (function bodies, class definitions, etc.)       │
│                                                      │
│  4. EMBEDDING                                        │
│     Generate embeddings for each chunk               │
│     Store in a local vector index                    │
│                                                      │
│  5. GRAPH BUILDING                                   │
│     Build import/dependency graph                    │
│     Build call graph (who calls what)                │
│     Build type graph (who uses what types)           │
│                                                      │
│  6. RETRIEVAL                                        │
│     When the model needs context:                    │
│     - Semantic search over embeddings                │
│     - Graph traversal for related files              │
│     - Symbol resolution for definitions              │
└────────────────────────────────────────────────────┘
```

### 7.2 Index as Memory

```typescript
// Cursor's context retrieval (conceptual)
interface CursorMemory {
  // "What file defines this function?"
  resolveSymbol(name: string): FileLocation[];
  
  // "What files are related to this query?"
  semanticSearch(query: string, limit: number): ChunkResult[];
  
  // "What files depend on this file?"
  getDependents(file: string): string[];
  
  // "What has changed recently?"
  getRecentlyModified(since: Date): FileChange[];
  
  // "What errors exist in the project?"
  getDiagnostics(): Diagnostic[];
}
```

### 7.3 Trade-offs: Explicit Memory vs. Index Memory

```
┌─────────────────┬─────────────────────┬──────────────────────┐
│                 │ Claude Code (KAIROS) │ Cursor (Index)       │
├─────────────────┼─────────────────────┼──────────────────────┤
│ Memory type     │ Explicit (written)  │ Implicit (derived)   │
│ Content         │ Conventions, notes  │ Code structure       │
│ Update          │ Manual + AutoDream  │ Auto-reindex         │
│ Persistence     │ Files on disk       │ In-memory + cache    │
│ Cross-session   │ Yes (files persist) │ Partial (re-indexes) │
│ Cross-project   │ Yes (global memory) │ No (per-project)     │
│ User control    │ Direct (edit files) │ Indirect (edit code) │
│ Cost            │ Tokens per turn     │ Compute at startup   │
│ Flexibility     │ Arbitrary knowledge │ Code structure only  │
│ Accuracy        │ Can become stale    │ Always current       │
└─────────────────┴─────────────────────┴──────────────────────┘
```

The key insight: these are complementary approaches. Claude Code's explicit memory captures things that are not in the code -- conventions, decisions, known issues. Cursor's index captures the actual state of the code. The ideal system would do both.

---

## 8. Append-Only Logs vs. Rewriting Strategies

### 8.1 Two Philosophies

There are two schools of thought on how to manage persistent memory:

**Append-Only (Logging)**
```
Day 1: "User prefers named exports"
Day 3: "User prefers named exports for components"
Day 7: "User prefers named exports for components, default for pages"
Day 12: "Actually, user switched to all named exports"
```

All four entries exist. The history is preserved. But the memory file grows, and entries can contradict each other.

**Rewrite (Living Document)**
```
Day 1: "User prefers named exports"
Day 3: "User prefers named exports for components"   <-- replaced
Day 7: "User prefers named exports for components, default for pages"  <-- replaced
Day 12: "User prefers all named exports"              <-- replaced
```

Only the latest entry exists. The file stays small. But history is lost, and a bad rewrite can destroy good information.

### 8.2 The Hybrid Approach (What KAIROS Does)

KAIROS uses a hybrid:

```
┌──────────────────────────────────────────────────┐
│            HYBRID MEMORY STRATEGY                 │
│                                                    │
│  SESSION LOGS: Append-only                         │
│  ├── Never modify past session logs                │
│  ├── Each day gets its own file                    │
│  └── AutoDream processes these into summaries      │
│                                                    │
│  CLAUDE.md: Rewrite (living document)              │
│  ├── Updated by the agent during sessions          │
│  ├── Updated by AutoDream during consolidation     │
│  └── Always represents "current truth"             │
│                                                    │
│  PERSISTENT MEMORY: Append + periodic pruning      │
│  ├── New learnings are appended                    │
│  ├── AutoDream prunes stale entries                │
│  └── Entries that survive multiple pruning cycles  │
│      are considered stable knowledge               │
└──────────────────────────────────────────────────┘
```

This hybrid captures the benefits of both approaches: session logs preserve history, CLAUDE.md stays concise, and persistent memory gradually stabilizes.

### 8.3 Implementation Pattern

```typescript
// Hybrid memory manager
class HybridMemoryManager {
  // Append-only: session logs
  async logSession(entry: SessionLogEntry): Promise<void> {
    const logPath = this.getLogPath(new Date());
    await fs.promises.appendFile(
      logPath,
      this.formatLogEntry(entry)
    );
  }
  
  // Rewrite: CLAUDE.md
  async updateProjectMemory(
    projectRoot: string,
    updates: MemoryUpdate[]
  ): Promise<void> {
    const claudeMdPath = path.join(projectRoot, 'CLAUDE.md');
    let content = await this.readOrCreate(claudeMdPath);
    
    for (const update of updates) {
      switch (update.type) {
        case 'add':
          content = this.addSection(content, update);
          break;
        case 'replace':
          content = this.replaceSection(content, update);
          break;
        case 'remove':
          content = this.removeSection(content, update);
          break;
      }
    }
    
    await fs.promises.writeFile(claudeMdPath, content);
  }
  
  // Append + prune: persistent memory
  async addPersistentMemory(entry: PersistentEntry): Promise<void> {
    const memoryPath = this.getPersistentMemoryPath();
    await fs.promises.appendFile(
      memoryPath,
      this.formatPersistentEntry(entry)
    );
  }
  
  async prunePersistentMemory(
    staleEntries: string[]
  ): Promise<void> {
    const memoryPath = this.getPersistentMemoryPath();
    let content = await fs.promises.readFile(memoryPath, 'utf-8');
    
    for (const stale of staleEntries) {
      content = content.replace(stale, '');
    }
    
    await fs.promises.writeFile(memoryPath, content.trim());
  }
}
```

---

## 9. Semantic Memory Merging and Deduplication

### 9.1 The Duplication Problem

Over time, memory systems accumulate duplicate or near-duplicate entries:

```
Entry 1 (March 1): "Use Vitest for testing"
Entry 2 (March 5): "This project uses Vitest, not Jest"
Entry 3 (March 12): "Tests should use Vitest. Import from vitest, not jest."
Entry 4 (March 20): "Testing framework: Vitest. Use vi.mock() for mocks."
```

These are all saying variations of the same thing. Without deduplication, they waste token budget when loaded into context.

### 9.2 Semantic Similarity for Merging

AutoDream uses semantic similarity to identify candidates for merging:

```typescript
// Semantic deduplication pipeline
async function deduplicateMemories(
  entries: MemoryEntry[]
): Promise<MemoryEntry[]> {
  // 1. Generate embeddings for each entry
  const embeddings = await generateEmbeddings(
    entries.map(e => e.content)
  );
  
  // 2. Find clusters of similar entries
  const clusters = clusterBySimilarity(
    embeddings,
    { threshold: 0.85 }  // cosine similarity threshold
  );
  
  // 3. For each cluster, merge entries
  const merged: MemoryEntry[] = [];
  
  for (const cluster of clusters) {
    if (cluster.length === 1) {
      merged.push(cluster[0]);
      continue;
    }
    
    // Use LLM to merge cluster into single entry
    const mergedEntry = await mergeCluster(cluster);
    merged.push(mergedEntry);
  }
  
  return merged;
}

async function mergeCluster(
  entries: MemoryEntry[]
): Promise<MemoryEntry> {
  // Sort by date (newest first) to prefer recent information
  entries.sort((a, b) =>
    b.timestamp.getTime() - a.timestamp.getTime()
  );
  
  const prompt = `
Merge these memory entries into a single, concise entry.
Prefer the most recent information if entries conflict.
Keep all unique details.

Entries (newest first):
${entries.map(e =>
  `[${e.timestamp.toISOString()}] ${e.content}`
).join('\n')}

Output a single merged entry:
`;
  
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-20250514',  // Cheap model for merging
    messages: [{ role: 'user', content: prompt }],
    max_tokens: 500,
  });
  
  return {
    content: extractText(response),
    timestamp: entries[0].timestamp,  // Use newest timestamp
    mergedFrom: entries.map(e => e.id),
  };
}
```

### 9.3 The Merge Quality Problem

Merging is lossy. Every merge risks losing nuance:

```
Before merge:
  "Use Vitest for testing"
  "Tests should use Vitest. Import from vitest, not jest."
  "Testing framework: Vitest. Use vi.mock() for mocks."

After merge:
  "Testing framework: Vitest. Import from vitest, not jest.
   Use vi.mock() for mocks."
```

This is a good merge -- it preserved all useful details. But merges can go wrong:

```
Before merge:
  "Use Redis for caching in production"
  "Use in-memory cache for development (not Redis)"

Bad merge:
  "Use Redis for caching"  <-- lost the dev/prod distinction

Good merge:
  "Caching: Redis in production, in-memory in development"
```

The quality of the merge depends on the model and the prompt. This is why AutoDream uses careful prompting with explicit instructions to preserve nuance and prefer recent information.

---

## 10. Designing Memory for Your Own Agents

### 10.1 The Decision Framework

When designing memory for your agent, answer these questions:

```
┌──────────────────────────────────────────────────┐
│          MEMORY DESIGN DECISIONS                  │
│                                                    │
│  Q1: What time scales matter?                      │
│  ├── Session only? (chatbot)                       │
│  ├── Session + project? (dev tool)                 │
│  └── Session + project + global? (personal agent)  │
│                                                    │
│  Q2: Who controls the memory?                      │
│  ├── Only the system? (implicit)                   │
│  ├── System + user? (editable memory files)        │
│  └── Only the user? (configuration files)          │
│                                                    │
│  Q3: How is memory retrieved?                      │
│  ├── Always injected? (CLAUDE.md pattern)          │
│  ├── Retrieved on demand? (search pattern)         │
│  └── Both? (small core + searchable archive)       │
│                                                    │
│  Q4: How is memory updated?                        │
│  ├── Explicitly by user? (manual)                  │
│  ├── Automatically by agent? (learning)            │
│  └── Background consolidation? (AutoDream)         │
│                                                    │
│  Q5: What is the cost budget?                      │
│  ├── Cheap: minimal injection, search only         │
│  ├── Medium: small core + occasional search        │
│  └── High: full KAIROS with AutoDream              │
└──────────────────────────────────────────────────┘
```

### 10.2 Three Reference Architectures

**Architecture 1: Minimal (for simple agents)**

```typescript
// Just CLAUDE.md equivalent -- no persistence, no consolidation
interface MinimalMemory {
  projectContext: string;  // Loaded from a file, injected every turn
  sessionHistory: Message[];  // Conversation so far
}
```

Best for: single-purpose agents, chatbots, simple tools. Cost: lowest.

**Architecture 2: Standard (for developer tools)**

```typescript
// CLAUDE.md + session logs + basic compaction
interface StandardMemory {
  projectContext: string;     // CLAUDE.md equivalent
  sessionHistory: Message[];  // With compaction
  sessionLogs: string[];      // Append-only logs
  
  compact(): Promise<void>;        // Summarize old messages
  logSession(): Promise<void>;     // Save session summary
  loadRecentLogs(): Promise<string>; // Load last N days
}
```

Best for: developer tools, project assistants, code review bots. Cost: moderate.

**Architecture 3: Full KAIROS (for personal/platform agents)**

```typescript
// Three-layer with consolidation
interface FullMemory {
  // Layer 1: Session
  sessionHistory: Message[];
  compactionService: CompactionService;
  
  // Layer 2: Project
  projectMemory: ProjectMemory;  // CLAUDE.md management
  
  // Layer 3: Persistent
  persistentMemory: PersistentMemory;
  sessionLogs: SessionLogStore;
  
  // Background
  autoDream: AutoDreamService;  // Consolidation
  semanticIndex: SemanticIndex; // Embedding-based search
  
  // Operations
  inject(): string;             // Build context for model call
  remember(entry: string): void; // Add to appropriate layer
  search(query: string): MemoryResult[]; // Find relevant memories
  consolidate(): Promise<void>; // Run AutoDream
}
```

Best for: personal AI assistants, platform-level tools, agents that work across many projects. Cost: highest.

### 10.3 The Golden Rule of Memory Design

> **Memory should be invisible to the user until it fails.**

A good memory system works silently: the agent remembers conventions, recalls decisions, and builds on past work without the user having to explicitly remind it. The user only notices memory when it breaks -- when the agent forgets something important or remembers something stale.

Design for the invisible success case, and build good error recovery for the visible failure case (stale memory, missing context, conflicting entries).

---

## Build It: A KAIROS-Style Memory System in 70 Lines

The three-layer memory architecture sounds complex, but the core idea is file-based and buildable in a single script. The implementation below gives you an index layer (always loaded, one-line summaries), topic files (loaded on demand when relevant), and a transcript layer (grep-only, never fully loaded). It also includes the AutoDream consolidation pattern -- a function that reads all topic files, deduplicates entries, merges related ones, and writes them back. No database, no embeddings, no external services. Just files.

```bash
pip install openai
```

```python
# KAIROS-style 3-layer memory system — file-based, no database needed
# Usage: python memory.py

import os, json, re
from pathlib import Path
from typing import Optional
from openai import OpenAI

MEMORY_DIR = Path("./agent_memory")
INDEX_FILE = MEMORY_DIR / "index.jsonl"       # Layer 1: always loaded (~150 chars/entry)
TOPICS_DIR = MEMORY_DIR / "topics"             # Layer 2: loaded on demand
TRANSCRIPTS_DIR = MEMORY_DIR / "transcripts"   # Layer 3: grep-only, never fully loaded

def init_memory():
    """Create memory directory structure."""
    for d in [MEMORY_DIR, TOPICS_DIR, TRANSCRIPTS_DIR]:
        d.mkdir(parents=True, exist_ok=True)
    if not INDEX_FILE.exists():
        INDEX_FILE.touch()

# --- Layer 1: Index (always loaded) ------------------------------------------

def load_index() -> list[dict]:
    """Load the full index into memory. Each entry is {topic, summary}."""
    if not INDEX_FILE.exists():
        return []
    entries = []
    for line in INDEX_FILE.read_text().strip().split("\n"):
        if line.strip():
            entries.append(json.loads(line))
    return entries

def add_to_index(topic: str, summary: str):
    """Append a one-line summary to the index. Keep summaries under 150 chars."""
    summary = summary[:150]
    with open(INDEX_FILE, "a") as f:
        f.write(json.dumps({"topic": topic, "summary": summary}) + "\n")

# --- Layer 2: Topic Files (loaded on demand) ----------------------------------

def write_topic(topic: str, content: str):
    """Write to a topic file. Call this FIRST, then update the index."""
    safe_name = re.sub(r"[^a-z0-9_-]", "_", topic.lower())
    topic_file = TOPICS_DIR / f"{safe_name}.md"
    # Append to existing topic file
    with open(topic_file, "a") as f:
        f.write(f"\n---\n{content}\n")
    return topic_file

def load_topic(topic: str) -> Optional[str]:
    """Load a topic file's full content."""
    safe_name = re.sub(r"[^a-z0-9_-]", "_", topic.lower())
    topic_file = TOPICS_DIR / f"{safe_name}.md"
    if topic_file.exists():
        return topic_file.read_text()
    return None

# --- Layer 3: Transcripts (grep-only) ----------------------------------------

def save_transcript(session_id: str, messages: list[dict]):
    """Save a full session transcript. Never loaded whole -- grep only."""
    transcript_file = TRANSCRIPTS_DIR / f"{session_id}.jsonl"
    with open(transcript_file, "w") as f:
        for msg in messages:
            f.write(json.dumps({"role": msg["role"], "content": msg.get("content", "")[:500]}) + "\n")

def grep_transcripts(query: str) -> list[str]:
    """Search transcripts for a keyword. Returns matching lines."""
    matches = []
    for tf in TRANSCRIPTS_DIR.glob("*.jsonl"):
        for line in tf.read_text().split("\n"):
            if query.lower() in line.lower():
                matches.append(f"[{tf.stem}] {line[:200]}")
                if len(matches) >= 20:
                    return matches
    return matches

# --- Memory Write Discipline: topic first, index second -----------------------

def remember(topic: str, detail: str, summary: str):
    """The correct write order: topic file first, then index."""
    write_topic(topic, detail)       # Step 1: durable detail
    add_to_index(topic, summary)     # Step 2: index pointer

# --- Memory Recall: search index, load matching topics ------------------------

def recall(query: str) -> str:
    """Search the index, load matching topic files, return combined context."""
    index = load_index()
    query_lower = query.lower()
    # Find matching index entries
    matches = [e for e in index if query_lower in e["summary"].lower() or query_lower in e["topic"].lower()]
    if not matches:
        return f"No memories found for '{query}'."
    # Load the topic files for matches (deduplicate topics)
    seen_topics = set()
    context_parts = []
    for entry in matches:
        if entry["topic"] not in seen_topics:
            seen_topics.add(entry["topic"])
            content = load_topic(entry["topic"])
            if content:
                context_parts.append(f"## {entry['topic']}\n{content}")
    return "\n\n".join(context_parts) if context_parts else "Index matched but topic files missing."

# --- AutoDream: Background Consolidation --------------------------------------

def autodream():
    """The AutoDream pattern: consolidate, deduplicate, and merge topic files.
    In production, this runs during idle periods (like sleep)."""
    client = OpenAI()
    topic_files = list(TOPICS_DIR.glob("*.md"))
    if not topic_files:
        print("Nothing to consolidate.")
        return

    for topic_file in topic_files:
        content = topic_file.read_text()
        sections = [s.strip() for s in content.split("---") if s.strip()]
        if len(sections) <= 1:
            continue  # nothing to consolidate

        print(f"Consolidating {topic_file.name} ({len(sections)} entries)...")
        response = client.chat.completions.create(
            model="gpt-4o-mini",  # cheap model for consolidation
            messages=[
                {"role": "system", "content": (
                    "You are a memory consolidation agent. Given multiple memory entries "
                    "about a topic, merge duplicates, remove stale info, and produce a "
                    "clean consolidated version. Keep all unique facts. Be concise."
                )},
                {"role": "user", "content": f"Consolidate these entries:\n\n{content}"},
            ],
        )
        consolidated = response.choices[0].message.content
        topic_file.write_text(consolidated)
        print(f"  -> Consolidated to {len(consolidated)} chars")

    # Rebuild index from consolidated topic files
    INDEX_FILE.write_text("")  # clear
    for topic_file in topic_files:
        topic_name = topic_file.stem.replace("_", " ")
        content = topic_file.read_text()
        # Use first line as summary
        first_line = content.split("\n")[0][:150]
        add_to_index(topic_name, first_line)
    print("Index rebuilt.")


if __name__ == "__main__":
    init_memory()

    # --- Demo: store some memories ---
    remember("project-setup", 
             "This project uses Next.js 15 with App Router. Tests are in __tests__/ dirs. "
             "We use pnpm, not npm. CI runs on GitHub Actions.",
             "Next.js 15 + App Router, pnpm, GitHub Actions CI")

    remember("api-conventions",
             "All API routes return {data, error} shape. Auth uses JWT in httpOnly cookies. "
             "Rate limiting is 100 req/min per user.",
             "API returns {data,error}, JWT auth, 100 req/min limit")

    remember("project-setup",
             "Database is Postgres via Prisma. Migrations run on deploy. "
             "Seed data is in prisma/seed.ts.",
             "Postgres + Prisma, migrations on deploy")

    # --- Demo: recall memories ---
    print("=== Recall 'project' ===")
    print(recall("project"))
    print()
    print("=== Recall 'api' ===")
    print(recall("api"))
    print()

    # --- Demo: consolidation ---
    print("=== AutoDream Consolidation ===")
    # autodream()  # uncomment to run (requires OPENAI_API_KEY)
    print("(uncomment autodream() and set OPENAI_API_KEY to run consolidation)")
```

Notice the write discipline: always write the topic file first, then update the index. If the process crashes between the two operations, you lose an index entry but never lose data. This is the same pattern KAIROS uses. The recall function mirrors the two-step lookup: scan the cheap index, then load the expensive topic files only for matches.

---

## 11. Key Takeaways

1. **KAIROS is three layers:** session memory (conversation), project memory (CLAUDE.md), persistent memory (long-term files). Each layer has different storage, lifetime, and management characteristics.

2. **CLAUDE.md is re-injected every turn.** This is the most important cost insight. Keep CLAUDE.md concise -- every token costs on every turn.

3. **AutoDream consolidates memory in the background.** It reviews session logs, prunes stale entries, merges duplicates, and updates CLAUDE.md. It runs during idle periods, paralleling human sleep-based memory consolidation.

4. **Append-only logs preserve history; rewriting keeps files concise.** The hybrid approach uses append-only for session logs and rewriting for CLAUDE.md.

5. **Semantic merging uses embeddings to find duplicates** and LLM calls to merge them. Merge quality depends on careful prompting.

6. **Cursor takes a different approach:** the codebase index IS the memory. This is always current but only captures code structure, not conventions or decisions.

7. **Choose your memory architecture based on your use case:** minimal for simple agents, standard for developer tools, full KAIROS for platform-level systems.

8. **The golden rule:** Memory should be invisible until it fails. Design for silent success.

---

*Previous: [Chapter 27 -- The Agent Harness Architecture](./27-harness-architecture.md)* · *Next: [Chapter 29 -- Context Window Internals](./29-context-internals.md)* explores how the finite context window is managed when memory, tools, and conversation all compete for space.

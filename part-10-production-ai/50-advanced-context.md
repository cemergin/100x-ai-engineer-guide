<!--
  CHAPTER: 50
  TITLE: Advanced Context Strategies
  PART: 10 — Production AI Systems
  PHASE: 2 — Become an Expert
  PREREQS: Ch 12 (context window management), Ch 28 (KAIROS memory systems), Ch 29 (context window internals)
  KEY_TOPICS: recursive compaction, sub-agent delegation, RAG-augmented context, tiered memory, context-aware routing, conversation branching, KAIROS in production
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 50: Advanced Context Strategies

> **Part 10 — Production AI Systems** | Phase 2: Become an Expert | Prerequisites: Ch 12, Ch 28, Ch 29 | Difficulty: Advanced | Language: TypeScript + Python

You've hit 120K tokens in a conversation. The model is starting to lose track of details mentioned 50 turns ago. Compacting once helped, but now the compacted summary itself is 20K tokens and growing. Your RAG system retrieves 8 documents per query, but only 2 are actually relevant, and the other 6 are eating context space and confusing the model.

This chapter is about the strategies that production AI systems use when basic context management (Ch 12) isn't enough. These are the patterns that let Claude Code handle 8-hour coding sessions, that let customer support bots manage 200-turn conversations, and that let agent systems coordinate across multiple LLM instances without blowing their context budgets.

### In This Chapter
- Recursive compaction: compacting summaries of summaries
- Sub-agent delegation: spawning focused child agents to offload context
- RAG-augmented context: just-in-time knowledge injection
- Tiered memory: hot (in-context) vs warm (RAG) vs cold (archival)
- Context-aware routing: different prompts/models based on context length
- Conversation branching: splitting conversations when they diverge
- The KAIROS approach applied: session + project + persistent layers in production

### Related Chapters
- **Ch 12 (Context Window Management)** — spirals back: basic compaction, sliding windows, token counting
- **Ch 28 (Memory Systems: KAIROS & Beyond)** — spirals back: the three-layer memory architecture
- **Ch 29 (Context Window Internals)** — spirals back: how Claude Code manages its context internally
- **Ch 59 (Building Internal AI Tools)** — spirals forward: context management in platform tools

---

## 1. Why Basic Context Management Fails

### 1.1 The Problem with Single-Pass Compaction

In Ch 12, you learned the basic compaction pattern: when the conversation gets long, summarize the older messages and replace them with the summary. This works for short-to-medium conversations (10-30 turns). It breaks down in three scenarios:

**Scenario 1: The summary itself gets too long.**
A 100-turn conversation compacts to a 5,000-token summary. By turn 200, the summary is 10,000 tokens. By turn 500, it's 25,000 tokens. You're back to the same problem.

**Scenario 2: Important details get lost.**
When you summarize 50 turns into 500 tokens, you necessarily lose detail. If turn 12 mentioned a specific configuration flag that becomes relevant again at turn 80, the summary might not have preserved it.

**Scenario 3: Multi-topic conversations.**
A long conversation might cover 5 different topics. A linear summary mixes them all together. When the user returns to topic 2 after a detour through topic 4, the model has to reconstruct the topic 2 context from a blended summary.

### 1.2 The Context Budget Mental Model

Think of your context window as a budget with competing line items:

```
┌─────────────── Context Window (200K tokens) ──────────────┐
│                                                            │
│  System Prompt          2,000 tokens  (1%)                 │
│  Tool Definitions       3,000 tokens  (1.5%)               │
│  Memory (KAIROS)        5,000 tokens  (2.5%)               │
│  Conversation History  80,000 tokens  (40%)                │
│  RAG Documents         20,000 tokens  (10%)                │
│  Current Turn           2,000 tokens  (1%)                 │
│  ─────────────────────────────────                         │
│  Used:                112,000 tokens  (56%)                │
│  Available for output: 88,000 tokens  (44%)                │
│                                                            │
│  But effective attention degrades after ~60K tokens         │
│  Real useful window: ~60,000 tokens                        │
└────────────────────────────────────────────────────────────┘
```

The "lost in the middle" problem means that even if you have 200K tokens available, the model's attention degrades for content in the middle of the context. Your 80K of conversation history isn't all equally "seen" by the model. Content at the beginning and end gets more attention than content in the middle.

This means the *effective* context window is smaller than the *technical* context window. Advanced strategies account for this.

---

## 2. Recursive Compaction

### 2.1 The Idea

Instead of a single summary that grows forever, maintain a *hierarchy* of summaries. Recent turns are kept in full. Older turns are summarized into a "level 1" summary. When the level 1 summary gets too long, it's summarized into a "level 2" summary. And so on.

```
┌─────────────────────────────────────────────────────────┐
│ Level 2 Summary: "In this conversation, the user set up │
│ a Next.js project with Prisma, resolved a CORS issue,   │
│ and is now working on authentication." (200 tokens)      │
├─────────────────────────────────────────────────────────┤
│ Level 1 Summary: "The user added NextAuth with GitHub    │
│ and Google providers. They configured the session        │
│ strategy as JWT. They ran into a callback URL mismatch   │
│ error and fixed it by updating .env." (500 tokens)       │
├─────────────────────────────────────────────────────────┤
│ Recent turns (full detail):                              │
│ Turn 45: User: "Now I need to protect the /dashboard     │
│   route. Only authenticated users should access it."     │
│ Turn 46: Assistant: "You can use NextAuth middleware..."  │
│ Turn 47: User: "That works! But the redirect goes to     │
│   /api/auth/signin instead of my custom login page."     │
│ Turn 48: (current)                                       │
│ (2,000 tokens of full context)                           │
└─────────────────────────────────────────────────────────┘

Total context for history: ~2,700 tokens (instead of ~20,000+)
```

### 2.2 Implementation

```typescript
// TypeScript: Recursive compaction system

import Anthropic from "@anthropic-ai/sdk";

interface CompactionLevel {
  level: number;
  summary: string;
  tokenCount: number;
  turnsCompacted: number;
  lastUpdated: Date;
}

interface ConversationState {
  levels: CompactionLevel[];
  recentTurns: Array<{ role: string; content: string; turnNumber: number }>;
  maxRecentTurns: number;
  maxTokensPerLevel: number;
}

class RecursiveCompactor {
  private client: Anthropic;

  constructor() {
    this.client = new Anthropic();
  }

  async compact(state: ConversationState): Promise<ConversationState> {
    // Check if recent turns need compaction
    if (state.recentTurns.length <= state.maxRecentTurns) {
      return state;  // No compaction needed
    }

    // Take the oldest half of recent turns for compaction
    const turnsToCompact = state.recentTurns.slice(
      0,
      Math.floor(state.recentTurns.length / 2)
    );
    const remainingTurns = state.recentTurns.slice(
      Math.floor(state.recentTurns.length / 2)
    );

    // Compact into level 0 (or merge with existing level 0)
    const newSummary = await this.summarizeTurns(
      turnsToCompact,
      state.levels[0]?.summary,
    );

    // Update level 0
    const level0: CompactionLevel = {
      level: 0,
      summary: newSummary,
      tokenCount: this.estimateTokens(newSummary),
      turnsCompacted: (state.levels[0]?.turnsCompacted || 0) + turnsToCompact.length,
      lastUpdated: new Date(),
    };

    // Check if level 0 needs to be rolled up into level 1
    const updatedLevels = [level0, ...state.levels.slice(1)];
    const finalLevels = await this.rollUpLevels(updatedLevels, state.maxTokensPerLevel);

    return {
      ...state,
      levels: finalLevels,
      recentTurns: remainingTurns,
    };
  }

  private async rollUpLevels(
    levels: CompactionLevel[],
    maxTokensPerLevel: number,
  ): Promise<CompactionLevel[]> {
    const result = [...levels];

    for (let i = 0; i < result.length; i++) {
      if (result[i].tokenCount > maxTokensPerLevel) {
        // This level is too long — summarize it into the next level
        const nextLevel = i + 1;
        const existingSummary = result[nextLevel]?.summary || "";

        const rolledUp = await this.summarizeSummary(
          result[i].summary,
          existingSummary,
          nextLevel,
        );

        // Replace current level with a shorter version
        result[i] = {
          ...result[i],
          summary: await this.shortenSummary(result[i].summary, maxTokensPerLevel / 2),
          tokenCount: maxTokensPerLevel / 2,
        };

        // Create or update next level
        if (nextLevel < result.length) {
          result[nextLevel] = {
            level: nextLevel,
            summary: rolledUp,
            tokenCount: this.estimateTokens(rolledUp),
            turnsCompacted: result[nextLevel].turnsCompacted + result[i].turnsCompacted,
            lastUpdated: new Date(),
          };
        } else {
          result.push({
            level: nextLevel,
            summary: rolledUp,
            tokenCount: this.estimateTokens(rolledUp),
            turnsCompacted: result[i].turnsCompacted,
            lastUpdated: new Date(),
          });
        }
      }
    }

    return result;
  }

  private async summarizeTurns(
    turns: Array<{ role: string; content: string }>,
    existingSummary?: string,
  ): Promise<string> {
    const turnsText = turns
      .map((t) => `${t.role}: ${t.content}`)
      .join("\n");

    const prompt = existingSummary
      ? `Here is a summary of the conversation so far:
${existingSummary}

Here are the new turns to incorporate:
${turnsText}

Create an updated summary that preserves key decisions, configurations, 
errors encountered, and current goals. Be specific about technical details 
(file names, function names, error messages, configuration values).`
      : `Summarize this conversation segment. Preserve key decisions, 
configurations, errors encountered, and current goals. Be specific 
about technical details.

${turnsText}`;

    const response = await this.client.messages.create({
      model: "claude-haiku-4-20250414",  // Use cheap model for compaction
      max_tokens: 1000,
      system: "You create concise, technically precise conversation summaries. Preserve specific details that might be needed later: file names, function signatures, error messages, configuration values, decisions made.",
      messages: [{ role: "user", content: prompt }],
    });

    return response.content[0].type === "text" ? response.content[0].text : "";
  }

  private async summarizeSummary(
    currentSummary: string,
    higherLevelSummary: string,
    targetLevel: number,
  ): Promise<string> {
    const response = await this.client.messages.create({
      model: "claude-haiku-4-20250414",
      max_tokens: 500,
      system: `Create a level-${targetLevel} summary. Higher levels are more abstract.
Level 0: Specific actions, commands, errors, fixes.
Level 1: Goals, approaches, key decisions, outcomes.
Level 2+: Overall themes, project trajectory, major milestones.`,
      messages: [
        {
          role: "user",
          content: `Existing higher-level summary:\n${higherLevelSummary || "(none)"}\n\nNew material to incorporate:\n${currentSummary}`,
        },
      ],
    });

    return response.content[0].type === "text" ? response.content[0].text : "";
  }

  private estimateTokens(text: string): number {
    return Math.ceil(text.length / 4);
  }

  private async shortenSummary(summary: string, targetTokens: number): Promise<string> {
    const response = await this.client.messages.create({
      model: "claude-haiku-4-20250414",
      max_tokens: targetTokens,
      system: "Shorten this summary while preserving the most important technical details.",
      messages: [{ role: "user", content: summary }],
    });

    return response.content[0].type === "text" ? response.content[0].text : "";
  }
}
```

```python
# Python: Recursive compaction system

import anthropic
from dataclasses import dataclass, field
from datetime import datetime

client = anthropic.Anthropic()

@dataclass
class CompactionLevel:
    level: int
    summary: str
    token_count: int
    turns_compacted: int
    last_updated: datetime = field(default_factory=datetime.now)

@dataclass
class ConversationState:
    levels: list[CompactionLevel] = field(default_factory=list)
    recent_turns: list[dict] = field(default_factory=list)
    max_recent_turns: int = 20
    max_tokens_per_level: int = 2000

class RecursiveCompactor:
    def __init__(self):
        self.client = anthropic.Anthropic()

    def compact(self, state: ConversationState) -> ConversationState:
        if len(state.recent_turns) <= state.max_recent_turns:
            return state

        # Take oldest half for compaction
        split = len(state.recent_turns) // 2
        to_compact = state.recent_turns[:split]
        remaining = state.recent_turns[split:]

        existing = state.levels[0].summary if state.levels else None
        new_summary = self._summarize_turns(to_compact, existing)

        level0 = CompactionLevel(
            level=0,
            summary=new_summary,
            token_count=len(new_summary) // 4,
            turns_compacted=(state.levels[0].turns_compacted if state.levels else 0) + len(to_compact),
        )

        levels = [level0] + state.levels[1:]
        levels = self._roll_up(levels, state.max_tokens_per_level)

        state.levels = levels
        state.recent_turns = remaining
        return state

    def _summarize_turns(self, turns: list[dict], existing: str | None) -> str:
        turns_text = "\n".join(f"{t['role']}: {t['content']}" for t in turns)

        if existing:
            prompt = f"Existing summary:\n{existing}\n\nNew turns:\n{turns_text}\n\nUpdate the summary."
        else:
            prompt = f"Summarize preserving technical details:\n{turns_text}"

        response = self.client.messages.create(
            model="claude-haiku-4-20250414",
            max_tokens=1000,
            system="Create concise, technically precise summaries.",
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text

    def _roll_up(self, levels: list[CompactionLevel], max_tokens: int) -> list[CompactionLevel]:
        for i in range(len(levels)):
            if levels[i].token_count > max_tokens:
                # Roll into next level
                next_existing = levels[i + 1].summary if i + 1 < len(levels) else ""
                rolled = self._summarize_at_level(levels[i].summary, next_existing, i + 1)

                if i + 1 < len(levels):
                    levels[i + 1].summary = rolled
                    levels[i + 1].token_count = len(rolled) // 4
                else:
                    levels.append(CompactionLevel(
                        level=i + 1, summary=rolled,
                        token_count=len(rolled) // 4,
                        turns_compacted=levels[i].turns_compacted,
                    ))
        return levels

    def _summarize_at_level(self, current: str, existing: str, level: int) -> str:
        response = self.client.messages.create(
            model="claude-haiku-4-20250414",
            max_tokens=500,
            system=f"Create a level-{level} summary. Higher = more abstract.",
            messages=[{"role": "user", "content": f"Existing: {existing}\nNew: {current}"}],
        )
        return response.content[0].text
```

### 2.3 Building the Context from Levels

When it's time to build the actual prompt, assemble from all levels:

```typescript
// TypeScript: Assembling context from compaction levels

function buildContextFromLevels(state: ConversationState): string {
  const parts: string[] = [];

  // Add summaries from highest level to lowest (most abstract first)
  const sortedLevels = [...state.levels].sort((a, b) => b.level - a.level);

  for (const level of sortedLevels) {
    if (level.summary) {
      parts.push(`<conversation_context level="${level.level}" turns_covered="${level.turnsCompacted}">
${level.summary}
</conversation_context>`);
    }
  }

  // Add recent turns in full
  parts.push("<recent_conversation>");
  for (const turn of state.recentTurns) {
    parts.push(`${turn.role}: ${turn.content}`);
  }
  parts.push("</recent_conversation>");

  return parts.join("\n\n");
}
```

---

## 3. Sub-Agent Delegation

### 3.1 The Pattern

When a conversation involves multiple distinct tasks, delegate each task to a focused child agent with its own clean context window. The parent agent manages the orchestration and collects results.

```
Parent Agent (coordinating context: 5K tokens)
├── Child Agent 1: "Analyze the error logs" (context: 15K tokens of logs)
├── Child Agent 2: "Review the database schema" (context: 10K tokens of DDL)
└── Child Agent 3: "Check the API documentation" (context: 8K tokens of docs)
```

Each child agent has a focused context — it only sees what it needs. The parent agent never needs to hold all the logs, schema, and documentation simultaneously.

### 3.2 Implementation

```typescript
// TypeScript: Sub-agent delegation for context management

import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface SubTask {
  id: string;
  description: string;
  context: string;  // The specific context this subtask needs
  systemPrompt: string;
}

interface SubTaskResult {
  id: string;
  result: string;
  tokensUsed: number;
}

async function delegateToSubAgent(task: SubTask): Promise<SubTaskResult> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: task.systemPrompt,
    messages: [
      {
        role: "user",
        content: `Context:\n${task.context}\n\nTask: ${task.description}`,
      },
    ],
  });

  return {
    id: task.id,
    result: response.content[0].type === "text" ? response.content[0].text : "",
    tokensUsed: response.usage.input_tokens + response.usage.output_tokens,
  };
}

async function coordinateSubAgents(
  userRequest: string,
  availableContext: Record<string, string>,  // Named context chunks
): Promise<string> {
  // Step 1: Parent agent plans the subtasks
  const planResponse = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 500,
    system: `You are a task coordinator. Given a user request and available context sources,
plan which subtasks to delegate. Each subtask should use only the context it needs.

Available context sources: ${Object.keys(availableContext).join(", ")}

Respond with JSON: {"subtasks": [{"id": "...", "description": "...", "context_sources": ["..."]}]}`,
    messages: [{ role: "user", content: userRequest }],
  });

  const planText = planResponse.content[0].type === "text" ? planResponse.content[0].text : "{}";
  const plan = JSON.parse(planText);

  // Step 2: Execute subtasks in parallel
  const subtasks: SubTask[] = plan.subtasks.map((st: any) => ({
    id: st.id,
    description: st.description,
    context: st.context_sources
      .map((source: string) => availableContext[source] || "")
      .join("\n\n"),
    systemPrompt: "You are a focused analyst. Answer the specific question based on the provided context. Be concise and precise.",
  }));

  const results = await Promise.all(subtasks.map(delegateToSubAgent));

  // Step 3: Parent agent synthesizes results
  const synthesisPrompt = results
    .map((r) => `## ${r.id}\n${r.result}`)
    .join("\n\n");

  const synthesis = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: "Synthesize the analysis results into a coherent response for the user.",
    messages: [
      {
        role: "user",
        content: `User's original request: ${userRequest}\n\nAnalysis results:\n${synthesisPrompt}`,
      },
    ],
  });

  return synthesis.content[0].type === "text" ? synthesis.content[0].text : "";
}
```

```python
# Python: Sub-agent delegation

import anthropic
import asyncio

client = anthropic.Anthropic()

async def delegate_to_sub_agent(
    task_id: str,
    description: str,
    context: str,
) -> dict:
    """Execute a focused subtask with its own clean context."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system="Focused analyst. Answer based on provided context. Be concise.",
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nTask: {description}",
        }],
    )
    return {
        "id": task_id,
        "result": response.content[0].text,
        "tokens": response.usage.input_tokens + response.usage.output_tokens,
    }

async def coordinate(user_request: str, contexts: dict[str, str]) -> str:
    """Parent agent coordinates sub-agents."""
    # Plan subtasks
    plan = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        system=f"Plan subtasks. Available contexts: {list(contexts.keys())}. "
               f"Respond JSON: {{\"subtasks\": [{{\"id\": ..., \"desc\": ..., \"sources\": [...]}}]}}",
        messages=[{"role": "user", "content": user_request}],
    )

    import json
    tasks = json.loads(plan.content[0].text)["subtasks"]

    # Execute in parallel
    results = await asyncio.gather(*[
        delegate_to_sub_agent(
            t["id"], t["desc"],
            "\n\n".join(contexts.get(s, "") for s in t["sources"])
        )
        for t in tasks
    ])

    # Synthesize
    synthesis_input = "\n\n".join(f"## {r['id']}\n{r['result']}" for r in results)
    final = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system="Synthesize these analysis results into a coherent response.",
        messages=[{"role": "user", "content": f"Request: {user_request}\n\nResults:\n{synthesis_input}"}],
    )
    return final.content[0].text
```

### 3.3 When to Delegate vs. Keep In-Context

| Situation | Strategy | Why |
|-----------|----------|-----|
| Single-topic deep conversation | Recursive compaction | Context is linearly related; delegation would lose coherence |
| Multi-topic investigation | Sub-agent delegation | Each topic has independent context |
| Large document analysis | Sub-agent per section | Each section can be analyzed independently |
| Code review (multi-file) | Sub-agent per file group | Files have local context |
| Long-running agent with tools | Hybrid: compact history + delegate tasks | Keep orchestration in parent, offload execution |

---

## 4. RAG-Augmented Context

### 4.1 Just-in-Time Knowledge Injection

Instead of stuffing all potentially-relevant context into the prompt upfront, use RAG to inject knowledge *only when needed* — and remove it when it's no longer needed.

```typescript
// TypeScript: Dynamic RAG context management

interface RAGContext {
  documents: Array<{
    id: string;
    content: string;
    tokenCount: number;
    relevanceScore: number;
    addedAtTurn: number;
  }>;
  maxTokenBudget: number;
  currentTokens: number;
}

class DynamicRAGContext {
  private context: RAGContext;

  constructor(maxTokenBudget: number = 15000) {
    this.context = {
      documents: [],
      maxTokenBudget: maxTokenBudget,
      currentTokens: 0,
    };
  }

  async updateForTurn(
    userMessage: string,
    currentTurn: number,
    retriever: (query: string) => Promise<Array<{ id: string; content: string; score: number }>>,
  ): Promise<void> {
    // 1. Retrieve relevant documents for this turn
    const retrieved = await retriever(userMessage);

    // 2. Add new high-relevance documents
    for (const doc of retrieved) {
      if (doc.score < 0.7) continue;  // Skip low-relevance results

      // Check if already in context
      if (this.context.documents.find((d) => d.id === doc.id)) continue;

      const tokenCount = Math.ceil(doc.content.length / 4);

      this.context.documents.push({
        id: doc.id,
        content: doc.content,
        tokenCount,
        relevanceScore: doc.score,
        addedAtTurn: currentTurn,
      });
      this.context.currentTokens += tokenCount;
    }

    // 3. Evict stale documents if over budget
    await this.evictIfNeeded(currentTurn);
  }

  private async evictIfNeeded(currentTurn: number): Promise<void> {
    while (this.context.currentTokens > this.context.maxTokenBudget) {
      // Evict the oldest, lowest-relevance document
      const sorted = [...this.context.documents].sort((a, b) => {
        // Score: lower relevance and older = evict first
        const ageA = currentTurn - a.addedAtTurn;
        const ageB = currentTurn - b.addedAtTurn;
        const scoreA = a.relevanceScore - ageA * 0.05;  // Decay by age
        const scoreB = b.relevanceScore - ageB * 0.05;
        return scoreA - scoreB;
      });

      const evicted = sorted[0];
      this.context.documents = this.context.documents.filter(
        (d) => d.id !== evicted.id
      );
      this.context.currentTokens -= evicted.tokenCount;
    }
  }

  getContextString(): string {
    return this.context.documents
      .sort((a, b) => b.relevanceScore - a.relevanceScore)
      .map((d) => `<reference id="${d.id}" relevance="${d.relevanceScore.toFixed(2)}">\n${d.content}\n</reference>`)
      .join("\n\n");
  }

  getTokenUsage(): number {
    return this.context.currentTokens;
  }
}
```

### 4.2 Relevance Decay

Documents that were relevant 30 turns ago may no longer be relevant. Implement a relevance decay mechanism:

```python
# Python: Relevance decay for RAG context

from dataclasses import dataclass
from math import exp

@dataclass
class RAGDocument:
    doc_id: str
    content: str
    token_count: int
    base_relevance: float  # Original retrieval score
    added_at_turn: int

def calculate_effective_relevance(
    doc: RAGDocument,
    current_turn: int,
    decay_rate: float = 0.05,
    re_retrieval_bonus: float = 0.3,
    re_retrieved_at: list[int] | None = None,
) -> float:
    """
    Calculate effective relevance with exponential decay.
    Documents that are re-retrieved in later turns get a relevance boost.
    """
    turns_since_added = current_turn - doc.added_at_turn
    decayed = doc.base_relevance * exp(-decay_rate * turns_since_added)

    # Boost if re-retrieved recently
    if re_retrieved_at:
        for turn in re_retrieved_at:
            turns_since_re = current_turn - turn
            decayed += re_retrieval_bonus * exp(-decay_rate * turns_since_re)

    return min(decayed, 1.0)  # Cap at 1.0


def evict_stale_documents(
    documents: list[RAGDocument],
    current_turn: int,
    max_tokens: int,
) -> list[RAGDocument]:
    """Remove least-relevant documents until under token budget."""
    # Calculate effective relevance for all documents
    scored = [
        (doc, calculate_effective_relevance(doc, current_turn))
        for doc in documents
    ]

    # Sort by effective relevance (highest first)
    scored.sort(key=lambda x: x[1], reverse=True)

    # Keep documents until budget is exhausted
    kept = []
    total_tokens = 0
    for doc, score in scored:
        if total_tokens + doc.token_count <= max_tokens:
            kept.append(doc)
            total_tokens += doc.token_count
        else:
            break  # Budget exhausted

    return kept
```

---

## 5. Tiered Memory Architecture

### 5.1 Hot, Warm, and Cold Memory

Production AI systems need three tiers of memory, each with different access speeds, capacities, and costs:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  HOT MEMORY (In-Context)                                     │
│  ────────────────────────                                    │
│  Capacity: 10-50K tokens                                     │
│  Access: Immediate (already in the prompt)                   │
│  Cost: High (every token costs money every turn)             │
│  Contents: Recent conversation, active documents, system     │
│            prompt, tool definitions                          │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  WARM MEMORY (RAG / Fast Retrieval)                          │
│  ─────────────────────────────────                           │
│  Capacity: 100K-10M tokens                                   │
│  Access: ~200ms (embedding search + retrieval)               │
│  Cost: Low (only retrieved chunks enter context)             │
│  Contents: Conversation summaries, project documents,        │
│            knowledge base, previous session notes            │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  COLD MEMORY (Archival / Database)                           │
│  ────────────────────────────────                            │
│  Capacity: Unlimited                                         │
│  Access: ~500ms-2s (database query, may need processing)     │
│  Cost: Minimal (storage only, no LLM cost)                   │
│  Contents: Full conversation logs, all historical documents, │
│            user profiles, system logs                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Implementing Tiered Memory

```typescript
// TypeScript: Tiered memory system

interface TieredMemory {
  hot: HotMemory;
  warm: WarmMemory;
  cold: ColdMemory;
}

interface HotMemory {
  systemPrompt: string;
  recentTurns: Array<{ role: string; content: string }>;
  activeDocuments: Array<{ id: string; content: string }>;
  compactedHistory: string;
  totalTokens: number;
}

interface WarmMemory {
  vectorStore: VectorStore;  // Embeddings-based retrieval
  sessionSummaries: Map<string, string>;  // Past session summaries
  projectNotes: string[];
}

interface ColdMemory {
  database: Database;  // Full conversation logs
  archiveStore: ObjectStore;  // Historical documents
}

class TieredMemoryManager {
  private hot: HotMemory;
  private warm: WarmMemory;
  private cold: ColdMemory;
  private hotBudget: number;

  constructor(hotBudget: number = 40000) {
    this.hotBudget = hotBudget;
    // Initialize tiers...
  }

  async processUserMessage(
    userId: string,
    sessionId: string,
    message: string,
  ): Promise<{ context: string; tokenCount: number }> {
    // 1. Check warm memory for relevant context
    const warmResults = await this.warm.vectorStore.search(message, { topK: 5 });
    const relevantWarm = warmResults.filter((r) => r.score > 0.75);

    // 2. Promote relevant warm memories to hot
    for (const result of relevantWarm) {
      if (this.hot.totalTokens + result.tokenCount <= this.hotBudget) {
        this.hot.activeDocuments.push({
          id: result.id,
          content: result.content,
        });
        this.hot.totalTokens += result.tokenCount;
      }
    }

    // 3. If still under budget, check cold memory for critical context
    if (this.hot.totalTokens < this.hotBudget * 0.7) {
      const coldResults = await this.cold.database.query(
        `SELECT content FROM memories WHERE user_id = ? AND relevance_score > 0.9`,
        [userId]
      );
      // Promote high-relevance cold items to warm for future access
      for (const item of coldResults) {
        await this.warm.vectorStore.upsert(item);
      }
    }

    // 4. Build the hot context
    const contextParts = [
      this.hot.systemPrompt,
      this.hot.compactedHistory,
      ...this.hot.activeDocuments.map((d) => d.content),
      ...this.hot.recentTurns.map((t) => `${t.role}: ${t.content}`),
    ];

    const context = contextParts.join("\n\n");
    return { context, tokenCount: this.hot.totalTokens };
  }

  async onTurnComplete(
    turn: { role: string; content: string },
    sessionId: string,
  ): Promise<void> {
    // Add to hot memory
    this.hot.recentTurns.push(turn);

    // Demote old hot items to warm if over budget
    while (this.hot.totalTokens > this.hotBudget) {
      const oldest = this.hot.activeDocuments.shift();
      if (oldest) {
        this.hot.totalTokens -= Math.ceil(oldest.content.length / 4);
        // Item stays in warm memory automatically
      }
    }

    // Archive to cold
    await this.cold.database.insert({
      session_id: sessionId,
      turn: turn,
      timestamp: new Date(),
    });
  }
}
```

---

## 6. Context-Aware Routing

### 6.1 Different Strategies for Different Context Lengths

As conversations grow, you may want to change your approach:

```typescript
// TypeScript: Context-aware strategy selection

interface ContextStrategy {
  model: string;
  maxOutputTokens: number;
  systemPromptVersion: "full" | "compact" | "minimal";
  ragEnabled: boolean;
  compactionAggression: "conservative" | "moderate" | "aggressive";
}

function selectStrategy(currentContextTokens: number): ContextStrategy {
  if (currentContextTokens < 10_000) {
    // Short conversation — use full context, full system prompt
    return {
      model: "claude-sonnet-4-20250514",
      maxOutputTokens: 2048,
      systemPromptVersion: "full",
      ragEnabled: true,
      compactionAggression: "conservative",
    };
  }

  if (currentContextTokens < 50_000) {
    // Medium conversation — start compacting, moderate system prompt
    return {
      model: "claude-sonnet-4-20250514",
      maxOutputTokens: 1024,
      systemPromptVersion: "compact",
      ragEnabled: true,
      compactionAggression: "moderate",
    };
  }

  if (currentContextTokens < 120_000) {
    // Long conversation — aggressive compaction, minimal system prompt
    return {
      model: "claude-sonnet-4-20250514",
      maxOutputTokens: 512,
      systemPromptVersion: "minimal",
      ragEnabled: false,  // RAG adds too many tokens
      compactionAggression: "aggressive",
    };
  }

  // Very long conversation — consider spawning a fresh agent
  return {
    model: "claude-sonnet-4-20250514",
    maxOutputTokens: 512,
    systemPromptVersion: "minimal",
    ragEnabled: false,
    compactionAggression: "aggressive",
  };
}

// System prompt versions at different sizes
const SYSTEM_PROMPTS = {
  full: `You are a customer support agent for Acme Corp.

## Capabilities
- Look up orders, products, and policies
- Issue refunds for orders within 30 days
- Escalate complex issues to human agents

## Policies
[... 1500 tokens of policy details ...]

## Formatting
- Use markdown for structured responses
- Include order numbers when referencing orders
- Always confirm actions before executing them`,

  compact: `Acme Corp support agent. Look up orders/products/policies. 
Issue refunds (30-day limit). Escalate complex issues. 
Confirm before acting. Use markdown.`,

  minimal: `Acme support. Orders, refunds, escalation. Confirm before acting.`,
};
```

### 6.2 Conversation Branching

When a conversation naturally splits into two distinct threads, branch it:

```typescript
// TypeScript: Conversation branching

interface ConversationBranch {
  branchId: string;
  parentBranch: string | null;
  topic: string;
  turns: Array<{ role: string; content: string }>;
  summary: string;  // Summary of the branch so far
  active: boolean;
}

class BranchingConversation {
  private branches: Map<string, ConversationBranch> = new Map();
  private activeBranch: string = "main";

  async detectBranchPoint(
    userMessage: string,
    currentBranch: ConversationBranch,
  ): Promise<{ shouldBranch: boolean; newTopic?: string }> {
    // Use a cheap model to detect topic shifts
    const response = await client.messages.create({
      model: "claude-haiku-4-20250414",
      max_tokens: 100,
      system: `Determine if this message continues the current topic or starts a new one.
Current topic: "${currentBranch.topic}"

Respond JSON: {"continues": true/false, "new_topic": "..." (if not continuing)}`,
      messages: [{ role: "user", content: userMessage }],
    });

    const result = JSON.parse(
      response.content[0].type === "text" ? response.content[0].text : "{}"
    );

    return {
      shouldBranch: !result.continues,
      newTopic: result.new_topic,
    };
  }

  async switchBranch(branchId: string): Promise<string> {
    const branch = this.branches.get(branchId);
    if (!branch) throw new Error(`Branch ${branchId} not found`);

    this.activeBranch = branchId;

    // Build context from this branch's history
    return `[Returning to topic: ${branch.topic}]\n\n${branch.summary}\n\n` +
      branch.turns.slice(-10).map((t) => `${t.role}: ${t.content}`).join("\n");
  }
}
```

---

## 7. The KAIROS Approach in Production

### 7.1 Recap: Three-Layer Memory

In Ch 28, you studied KAIROS — the three-layer memory system used by Claude Code:

1. **Session memory** — what happened in this conversation (ephemeral)
2. **Project memory** — what's true about this project (CLAUDE.md, persisted)
3. **Persistent memory** — what's true about the user across all projects

### 7.2 Applying KAIROS to Your AI Features

```typescript
// TypeScript: KAIROS-inspired memory for a production AI feature

interface KAIROSMemory {
  session: SessionMemory;
  project: ProjectMemory;
  persistent: PersistentMemory;
}

interface SessionMemory {
  conversationId: string;
  turns: Array<{ role: string; content: string }>;
  compactedHistory: string;
  currentGoal: string;
  decisionsThisSession: string[];
}

interface ProjectMemory {
  projectId: string;
  preferences: Record<string, string>;  // e.g., {"framework": "Next.js", "db": "Postgres"}
  conventions: string[];  // e.g., ["Use TypeScript strict mode", "Tabs not spaces"]
  knownIssues: string[];  // e.g., ["CORS issue on staging — use proxy"]
  lastUpdated: Date;
}

interface PersistentMemory {
  userId: string;
  communicationStyle: string;  // "technical" | "brief" | "detailed"
  expertise: Record<string, string>;  // {"react": "expert", "rust": "beginner"}
  frequentTasks: string[];
  lastUpdated: Date;
}

function buildKAIROSPrompt(memory: KAIROSMemory): string {
  const parts: string[] = [];

  // Persistent layer (injected first — always present)
  parts.push(`<user_profile>
Communication style: ${memory.persistent.communicationStyle}
Expertise: ${Object.entries(memory.persistent.expertise).map(([k, v]) => `${k}: ${v}`).join(", ")}
</user_profile>`);

  // Project layer (injected after persistent)
  if (Object.keys(memory.project.preferences).length > 0) {
    parts.push(`<project_context>
Tech stack: ${Object.entries(memory.project.preferences).map(([k, v]) => `${k}: ${v}`).join(", ")}
Conventions: ${memory.project.conventions.join("; ")}
Known issues: ${memory.project.knownIssues.join("; ")}
</project_context>`);
  }

  // Session layer (most recent, most detailed)
  if (memory.session.compactedHistory) {
    parts.push(`<session_history>
${memory.session.compactedHistory}
</session_history>`);
  }

  if (memory.session.currentGoal) {
    parts.push(`<current_goal>${memory.session.currentGoal}</current_goal>`);
  }

  return parts.join("\n\n");
}

// After each session, update the project and persistent layers
async function updateMemoryPostSession(
  memory: KAIROSMemory,
  sessionTranscript: string,
): Promise<void> {
  // Use LLM to extract updates for project memory
  const projectUpdates = await client.messages.create({
    model: "claude-haiku-4-20250414",
    max_tokens: 500,
    system: "Extract any new project preferences, conventions, or known issues from this session. Respond with JSON.",
    messages: [{ role: "user", content: sessionTranscript }],
  });

  // Use LLM to extract updates for persistent memory
  const persistentUpdates = await client.messages.create({
    model: "claude-haiku-4-20250414",
    max_tokens: 300,
    system: "Extract any updates about the user's expertise, preferences, or communication style. Respond with JSON.",
    messages: [{ role: "user", content: sessionTranscript }],
  });

  // Apply updates...
  // (Parse JSON and merge into existing memory)
}
```

---

## 8. Production Context Budget Template

### 8.1 The Budget Allocation

Here's a practical allocation for a production AI feature:

```python
# Python: Context budget allocation

from dataclasses import dataclass

@dataclass
class ContextBudget:
    """How to allocate your context window."""
    total_window: int
    system_prompt: int
    tool_definitions: int
    persistent_memory: int
    project_memory: int
    session_history: int
    rag_documents: int
    current_turn: int
    output_reserved: int

    @property
    def allocated(self) -> int:
        return (
            self.system_prompt + self.tool_definitions +
            self.persistent_memory + self.project_memory +
            self.session_history + self.rag_documents +
            self.current_turn + self.output_reserved
        )

    @property
    def utilization(self) -> float:
        return self.allocated / self.total_window

    def validate(self) -> list[str]:
        issues = []
        if self.allocated > self.total_window:
            issues.append(f"Over budget by {self.allocated - self.total_window} tokens")
        if self.output_reserved < 1000:
            issues.append("Output budget too small — risk of truncated responses")
        if self.system_prompt > self.total_window * 0.05:
            issues.append("System prompt uses >5% of context — consider compacting")
        if self.rag_documents > self.total_window * 0.15:
            issues.append("RAG budget >15% — risk of diluting conversation context")
        return issues

# Recommended budget for a 200K-token model
RECOMMENDED_BUDGET = ContextBudget(
    total_window=200_000,
    system_prompt=2_000,       # 1% — lean system prompt
    tool_definitions=3_000,     # 1.5% — tool schemas
    persistent_memory=1_000,    # 0.5% — user profile
    project_memory=3_000,       # 1.5% — project context
    session_history=40_000,     # 20% — conversation history
    rag_documents=15_000,       # 7.5% — retrieved documents
    current_turn=2_000,         # 1% — current user message
    output_reserved=4_000,      # 2% — response generation
)

print(f"Allocated: {RECOMMENDED_BUDGET.allocated:,} / {RECOMMENDED_BUDGET.total_window:,}")
print(f"Utilization: {RECOMMENDED_BUDGET.utilization:.1%}")
print(f"Available buffer: {RECOMMENDED_BUDGET.total_window - RECOMMENDED_BUDGET.allocated:,} tokens")

issues = RECOMMENDED_BUDGET.validate()
for issue in issues:
    print(f"  Warning: {issue}")
```

---

## 9. Key Takeaways

Advanced context management is about making every token count. The strategies in this chapter give you the tools to handle conversations of any length, tasks of any complexity, and knowledge bases of any size.

**The toolkit:**

| Strategy | When to Use | Token Impact |
|----------|-------------|-------------|
| Recursive compaction | Long single-topic conversations | 80-90% reduction in history tokens |
| Sub-agent delegation | Multi-topic or multi-document tasks | Splits context across agents |
| Dynamic RAG context | Knowledge-intensive conversations | Inject only what's relevant per turn |
| Tiered memory | All production features | Hot/warm/cold keeps costs proportional |
| Context-aware routing | Growing conversations | Adapts strategy as context grows |
| Conversation branching | Multi-topic conversations | Clean context per topic |
| KAIROS layers | Features with returning users | Persistent context without per-turn cost |

The key insight: context management is not just about fitting more text into a window. It's about getting the *right* information in front of the model at the *right* time. A 10K-token context with precisely the right information outperforms a 100K-token context packed with noise.

> **Next chapter:** Your context is managed, your costs are controlled, and your security is in place. But how do you know if the system is actually working well? Ch 51 covers production eval pipelines — continuous evaluation that catches quality regressions before your users do.

---

*Previous: [Cost Engineering](./49-cost-engineering.md)* | *Next: [Production Eval Pipelines](./51-production-evals.md)*

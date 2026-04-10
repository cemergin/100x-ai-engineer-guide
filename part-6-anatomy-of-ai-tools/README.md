<!--
  TYPE: part-index
  PART: 6
  TITLE: Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  CHAPTERS: 27, 28, 29, 30, 31, 32, 33
  UPDATED: 2026-04-10
-->

# Part 6 — Anatomy of AI Developer Tools

> **Phase 2: Become an Expert** | 7 chapters | Inter-Adv | Language: TypeScript + Architecture

How Claude Code, Cursor, and coding agents actually work inside. This Part reverse-engineers the systems you learned to *use* in Part 5 and teaches you to *understand* and *build* them.

In March 2026, Claude Code's source was leaked. The community discovered systems like KAIROS (three-layer memory), AutoDream (background memory consolidation), a tick loop for idle-time agency, 14 cache-break vectors, a compaction service, and a permission architecture far more nuanced than anyone expected. This Part takes those discoveries and turns them into engineering knowledge you can apply. We are not interested in gossip — we are interested in architecture.

The seven chapters follow a tight spiral:

1. **The harness** — the system AROUND the model that makes it useful (Ch 27)
2. **Memory** — how the harness remembers across sessions, projects, and time (Ch 28)
3. **Context** — how the harness manages its finite token window (Ch 29)
4. **Tools** — how the harness executes actions and controls permissions (Ch 30)
5. **Search** — how the harness acquires external knowledge (Ch 31)
6. **Coordination** — how multiple harnesses work together (Ch 32)
7. **Extension** — how the harness is customized and distributed (Ch 33)

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 27 | [The Agent Harness Architecture](./27-harness-architecture.md) | Inter-Adv | CLI->loop->tools->permissions->UI. Claude Code vs Cursor vs Codex internals |
| 28 | [Memory Systems: KAIROS & Beyond](./28-memory-systems.md) | Advanced | 3-layer memory, CLAUDE.md injection, AutoDream, append-only logs, semantic merging |
| 29 | [Context Window Internals](./29-context-internals.md) | Advanced | Compaction service, token budgets, cache-break vectors, session limits |
| 30 | [Tool Execution & Permissions](./30-tool-execution.md) | Advanced | Permission models, approval flows, risk tiers, sandboxing |
| 31 | [Web Search & Knowledge Pipelines](./31-web-search-pipelines.md) | Advanced | Search APIs, content extraction, Turndown, paraphrase limits, pre-approved domains |
| 32 | [Multi-Agent Coordination](./32-multi-agent-coordination.md) | Advanced | Coordinator mode, tick loops, background agents, worker delegation |
| 33 | [Skills, Plugins & Distribution](./33-skills-plugins-internals.md) | Advanced | Skill architecture, Skillify, hook system, anti-distillation, org distribution |

## How This Part Connects

```
Part 5 (Harness Engineering — using the tools)
  │
  └──> Part 6 (you are here — understanding the tools)
         Ch 27 spirals the agent loop from Ch 9, frameworks from Ch 13
         Ch 28 spirals memory concepts from Ch 10
         Ch 29 spirals context management from Ch 12
         Ch 30 spirals tool calling from Ch 8, approvals from Ch 11
         Ch 31 spirals RAG pipelines from Ch 15, advanced retrieval from Ch 17
         Ch 32 spirals the agent loop from Ch 9, coding agents from Ch 26
         Ch 33 spirals skills/plugins from Ch 23-25, prompt engineering from Ch 4
         │
         ├──> Part 10: Production AI
         │      Ch 48: security patterns from Ch 30 (permissions)
         │      Ch 49: cost engineering from Ch 29 (cache management)
         │      Ch 50: advanced context from Ch 28 (memory), Ch 29 (compaction)
         │
         ├──> Part 11: Deployment & Infrastructure
         │      Ch 55: sandboxing from Ch 30 (tool execution)
         │
         └──> Part 12: AI Platform Engineering
                Ch 58: multi-agent orchestration from Ch 32 (coordination)
                Ch 59: internal tools from Ch 27 (architecture), Ch 28 (memory)
                Ch 60: skills marketplaces from Ch 33 (distribution)
```

## The Within-Part Spiral

```
Ch 27: The Agent Harness Architecture
  └──> Ch 28: Memory Systems (the harness needs to remember)
        └──> Ch 29: Context Window Internals (memory lives in the context)
              └──> Ch 30: Tool Execution & Permissions (context contains tool calls)
                    └──> Ch 31: Web Search & Knowledge Pipelines (a special kind of tool)
                          └──> Ch 32: Multi-Agent Coordination (many harnesses working together)
                                └──> Ch 33: Skills, Plugins & Distribution (extending and sharing harnesses)
```

## Reading Time

Expect about 6-8 hours to read all seven chapters. This is the densest Part in the guide. There is no code to deploy — instead you will build deep mental models of production AI architectures that inform every system you build going forward. By the end you will understand how to design memory systems, context managers, permission layers, and multi-agent coordinators for your own tools.

---

*Previous: [Part 5 — Harness Engineering](../part-5-harness-engineering/)* | *Next: [Part 7 — The Hard Parts](../part-7-hard-parts/)*

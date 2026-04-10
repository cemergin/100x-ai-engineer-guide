<!--
  TYPE: part-index
  PART: 2
  TITLE: Agent Engineering
  PHASE: 1 — Get Dangerous
  CHAPTERS: 8, 9, 10, 11, 12, 13
  UPDATED: 2026-04-10
-->

# Part 2 — Agent Engineering

> **Phase 1: Get Dangerous** | 6 chapters | Intermediate | Language: TypeScript

From API calls to autonomous agents. This is the core of what makes an AI engineer. After Part 1 you can make an LLM do things. After Part 2 you can make an LLM *decide* what to do, *do it*, and *keep going* until the job is done.

The six chapters in this Part follow a tight spiral. Each chapter introduces the next problem:

1. **Tools** give the LLM hands (Ch 8)
2. **The loop** lets it use those hands repeatedly (Ch 9)
3. **Memory** lets the loop remember what happened (Ch 10)
4. **Approvals** let humans stay in control (Ch 11)
5. **Context management** keeps the loop from choking (Ch 12)
6. **Frameworks** package all five concepts into libraries (Ch 13)

By the end you'll have built an agent from scratch — and you'll understand exactly what every framework is doing under the hood.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 8 | [Tool Calling](./08-tool-calling.md) | Intermediate | Tool definitions, Zod schemas, execute functions, description quality |
| 9 | [The Agent Loop](./09-agent-loop.md) | Intermediate | Prompt -> LLM -> tool -> execute -> append -> repeat, stop conditions, streaming |
| 10 | [Agent Memory & State](./10-agent-memory.md) | Inter -> Adv | Short-term, working, long-term memory architectures |
| 11 | [Human-in-the-Loop](./11-human-in-the-loop.md) | Intermediate | Sync/async approvals, trust spectrum, approval architectures |
| 12 | [Context Window Management](./12-context-management.md) | Inter -> Adv | Token counting, compaction, summarization, sliding windows |
| 13 | [Agent Patterns & Frameworks](./13-agent-patterns.md) | Inter -> Adv | ReAct, Plan-and-Execute, LangChain, Mastra, Vercel AI SDK |

## How This Part Connects

```
Part 1 (Building with LLM APIs)
  │
  └──→ Part 2 (you are here)
         Ch 8 uses Zod schemas from Ch 5
         Ch 9 uses conversation history from Ch 6
         Ch 9 uses streaming from Ch 3
         │
         ├──→ Part 3: RAG & Knowledge
         │      Ch 15 uses tools + retrieval (Ch 8)
         │      Ch 17 uses agent loops for advanced retrieval
         │
         ├──→ Part 4: Evals & Quality
         │      Ch 19 evaluates tool selection (Ch 8)
         │      Ch 20 evaluates multi-turn agent conversations (Ch 9)
         │
         ├──→ Part 5: Harness Engineering
         │      Ch 23: Claude Code IS an agent loop (Ch 9) with memory (Ch 10)
         │      Ch 24: MCP servers ARE tools (Ch 8)
         │      Ch 25: Automation uses approvals (Ch 11)
         │
         └──→ Part 6: Anatomy of AI Tools (Phase 2)
                Ch 27-33 spiral ALL of Part 2 to expert depth
                Ch 27: production agent architecture (Ch 9)
                Ch 28: KAIROS memory (Ch 10)
                Ch 29: context internals (Ch 12)
                Ch 30: tool execution internals (Ch 8)
```

## The Within-Part Spiral

```
Ch 8: Tool Calling
  └──→ Ch 9: The Agent Loop (tools run inside the loop)
        └──→ Ch 10: Agent Memory (the loop needs memory)
              └──→ Ch 11: Human-in-the-Loop (some actions need approval)
                    └──→ Ch 12: Context Management (the loop fills up)
                          └──→ Ch 13: Frameworks (package all of this)
```

## Reading Time

Expect about 3-4 hours to read all six chapters and build the code examples. By the end you'll have a working CLI agent that can call tools, manage memory, ask for approval, and handle context window limits.

---

*Previous: [Part 1 -- Building with LLM APIs](../part-1-building-with-llms/)* | *Next: [Part 3 -- RAG & Knowledge Systems](../part-3-rag-knowledge/)*

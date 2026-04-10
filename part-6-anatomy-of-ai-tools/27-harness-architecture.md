<!--
  CHAPTER: 27
  TITLE: The Agent Harness Architecture
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 9 (Agent Loop), Ch 13 (Agent Patterns & Frameworks), Ch 23 (Claude Code Mastery)
  KEY_TOPICS: harness architecture, CLI agent loop, tool executor, permission layer, UI renderer, Claude Code internals, Cursor architecture, Codex architecture, LSP integration
  DIFFICULTY: Inter->Adv
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 27: The Agent Harness Architecture

> **Part 6 — Anatomy of AI Developer Tools** | Phase 2: Become an Expert | Prerequisites: Ch 9, Ch 13, Ch 23 | Difficulty: Intermediate to Advanced | Language: TypeScript + Architecture

The model is not the product. The harness is the product. Every AI developer tool you use -- Claude Code, Cursor, GitHub Copilot, Codex -- is a thin language model wrapped in a thick engineering system that handles input, output, context, tools, permissions, memory, and coordination. The March 2026 Claude Code source leak confirmed what many suspected: the harness code dwarfs the model integration code by an order of magnitude. Understanding this architecture is the key to understanding why these tools behave the way they do, and how to build your own.

### In This Chapter

1. What a "harness" actually is
2. The five-layer architecture pattern
3. Claude Code's architecture: from CLI to response
4. Cursor's architecture: LSP-native AI
5. Codex/Copilot: cloud-first agents
6. Why the harness matters more than the model
7. Common patterns across all AI developer tools
8. Building your own: the minimum viable harness
9. Architecture comparison summary
10. Model codenames and the multi-version pipeline
11. Open-source reimplementations: the harness pattern proven model-agnostic

### Related Chapters

- **Ch 9 (The Agent Loop)** -- introduced the basic loop; this chapter shows production implementations
- **Ch 13 (Agent Patterns & Frameworks)** -- covered framework abstractions; this chapter shows real systems
- **Ch 23 (Claude Code Mastery)** -- taught you to use the harness; this chapter teaches you how it works
- **Ch 59 (Building Internal AI Tools)** -- will use these architecture patterns to build your own

---

## 1. What a "Harness" Actually Is

### 1.1 The Term

The word "harness" comes from testing -- a test harness is the scaffolding around the code under test that provides inputs, captures outputs, and controls the execution environment. An AI harness is the same idea applied to a language model: everything around the model that makes it useful.

**What it is:** The complete system that sits between the user and the model -- receiving input, preparing context, calling the model, executing tool results, managing permissions, rendering output, and maintaining state across turns.

**What it is not:** The model itself. The model is a stateless function that takes tokens in and produces tokens out. It has no memory, no tools, no permissions, no UI. The harness provides all of that.

```
┌─────────────────────────────────────────────────────────┐
│                    THE HARNESS                          │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   Input   │  │  Context  │  │   Tool   │  │   UI   │ │
│  │  Handler  │→ │  Builder  │→ │ Executor │→ │Renderer│ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│       ↑              ↑             ↑            ↑       │
│       │         ┌────┴────┐  ┌────┴────┐       │       │
│       │         │ Memory  │  │Permission│       │       │
│       │         │  System │  │  Layer   │       │       │
│       │         └─────────┘  └─────────┘       │       │
│       │                                         │       │
│  ┌────┴─────────────────────────────────────────┴────┐ │
│  │                  Agent Loop                        │ │
│  │          (prompt → model → tool → repeat)          │ │
│  └────────────────────────────────────────────────────┘ │
│                          ↕                              │
│                 ┌────────────────┐                      │
│                 │   LLM API Call │                      │
│                 │  (the model)   │                      │
│                 └────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Why This Matters

Consider two scenarios:

**Scenario A:** You give Claude Sonnet raw API access. No system prompt. No tools. No memory. Single turn. The model produces a text response and forgets everything.

**Scenario B:** You give Claude Sonnet the Claude Code harness. It can read files, write code, run tests, search the web, remember your project conventions, ask for permission before destructive operations, spawn sub-agents, manage its own context window, and learn from past sessions.

Same model. Radically different capabilities. The difference is the harness.

**Real-world example:** When Claude Code was first released, users noticed it was dramatically better at coding tasks than the same Claude model accessed through the API. The leak revealed why: the harness includes a sophisticated system prompt with coding conventions, a permission layer that auto-approves safe operations while blocking risky ones, a memory system that injects project context on every turn, and a compaction service that keeps the context window focused. None of that is the model.

### 1.3 The Harness Thesis

This is the key insight from the leak, and the thesis of this entire Part:

> **The harness matters more than the model.**

A well-engineered harness around a good model consistently outperforms a great model with a basic harness. This is why:

1. **Context preparation** determines what the model sees -- garbage in, garbage out
2. **Tool execution** determines what the model can do -- more tools, more capability
3. **Memory management** determines what the model remembers -- continuity across sessions
4. **Permission systems** determine what the model is allowed to do -- trust enables autonomy
5. **Coordination** determines what scale the model operates at -- one agent vs many

You cannot improve any of these by making the model smarter. You improve them by engineering the harness.

### 1.4 From Prompt Tools to Execution Environments

The community articulated this thesis with even more force after the leak. A widely-shared insight from the Reddit analysis thread put it bluntly:

> "We're moving from prompt tools to execution environments."

The first generation of AI developer tools were prompt wrappers -- take user input, add a system prompt, call the model, display the result. That era is over. The tools that matter now are execution environments: they manage state across turns, coordinate multiple agents, recover from failures, handle multi-step reasoning chains, and orchestrate tool calls across processes and machines.

The moat is no longer model quality. Frontier models are converging. The moat is coordination, state management, recovery, and the engineering that enables long-running autonomous work. As one commenter noted: "Drop DeepSeek or Gemini into the same Claude Code harness and you may get improved coding ability too." The harness IS the product; the model is a component.

This has a concrete implication: if you are evaluating AI developer tools, evaluate the harness, not just the model behind it. And if you are building AI features, the engineering effort should be weighted toward the harness -- context assembly, tool execution, permission management, multi-agent coordination -- not toward model selection.

---

## 2. The Five-Layer Architecture Pattern

Every AI developer tool we examined -- Claude Code, Cursor, Codex, Aider, Continue -- follows the same basic layered architecture. The implementations differ, but the layers are universal.

### 2.1 Layer 1: Input Interface

The surface where the user interacts with the system.

```
┌──────────────────────────────────────────┐
│            INPUT INTERFACE                │
├──────────────┬───────────────────────────┤
│ Claude Code  │ CLI (terminal)            │
│ Cursor       │ IDE (VS Code fork)        │
│ Copilot      │ IDE extension + CLI       │
│ Codex        │ Web UI + CLI              │
│ Aider        │ CLI (terminal)            │
│ Devin        │ Web UI + Slack            │
└──────────────┴───────────────────────────┘
```

The input interface handles:
- User message capture (text, files, selections)
- Session initialization and persistence
- Configuration loading (settings files, environment)
- Authentication and authorization

**Key design decision:** CLI vs IDE vs Web. This choice cascades through the entire architecture. A CLI tool has direct filesystem access but no visual context. An IDE tool has syntax highlighting and file navigation but is constrained by the editor's extension API. A web tool has the richest UI but the most limited system access.

### 2.2 Layer 2: Context Assembly

The system that builds the prompt the model actually sees.

```typescript
// Simplified context assembly -- what happens before every model call
interface ContextAssembler {
  // Fixed context -- loaded once, injected every turn
  systemPrompt: string;           // The base system prompt
  projectMemory: string;          // CLAUDE.md or equivalent
  activeSkills: string[];         // Loaded skill definitions
  
  // Dynamic context -- changes every turn
  conversationHistory: Message[]; // The message thread
  toolResults: ToolResult[];      // Results from tool executions
  fileContents: FileContent[];    // Recently read files
  
  // Computed context -- derived from the above
  tokenBudget: number;            // How many tokens we can use
  compactionNeeded: boolean;      // Whether to summarize old messages
  
  assemble(): PromptPayload;      // Build the final prompt
}
```

This is where the magic happens. The context assembler decides:
- What the model knows about the project (memory injection)
- What the model remembers from earlier in the conversation (history management)
- What the model can see right now (active file contents, search results)
- When to compress old context to make room for new context (compaction)

**The CLAUDE.md discovery:** The leak revealed that CLAUDE.md is not loaded once at session start. It is re-injected on every turn. This means it occupies token budget on every single model call. A 500-line CLAUDE.md costs you tokens on every turn -- a key cost implication explored in Ch 29.

### 2.3 Layer 3: Agent Loop

The core execution engine. This is the same loop from Ch 9, but in production it is far more sophisticated.

```typescript
// The production agent loop (simplified from the leak)
async function agentLoop(
  messages: Message[],
  tools: ToolDefinition[],
  config: AgentConfig
): Promise<void> {
  while (true) {
    // 1. Assemble context (Layer 2)
    const context = assembleContext(messages, config);
    
    // 2. Check if compaction is needed
    if (context.tokenCount > config.tokenThreshold) {
      messages = await compact(messages, config);
    }
    
    // 3. Call the model
    const response = await callModel(context);
    
    // 4. Handle the response
    if (response.type === 'text') {
      // Model wants to speak to the user
      render(response.text);
      break; // Wait for user input
    }
    
    if (response.type === 'tool_use') {
      for (const toolCall of response.toolCalls) {
        // 5. Check permissions (Layer 4)
        const permitted = await checkPermission(toolCall, config);
        
        if (!permitted) {
          messages.push(deniedMessage(toolCall));
          continue;
        }
        
        // 6. Execute the tool (Layer 4)
        const result = await executeTool(toolCall);
        messages.push(resultMessage(toolCall, result));
      }
      // Loop back to step 1 -- the model sees the tool results
      continue;
    }
    
    if (response.type === 'end_turn') {
      break;
    }
  }
}
```

**Key difference from Ch 9:** The production loop has branching paths for compaction, permission checking, background agent spawning, tick-loop idle detection, and error recovery. The basic loop is maybe 20 lines. The production loop is thousands.

### 2.4 Layer 4: Tool Execution & Permissions

The system that converts model intentions into real-world actions.

```
┌────────────────────────────────────────────────┐
│            TOOL EXECUTION LAYER                │
│                                                │
│  Model says: tool_use("Bash", {cmd: "rm -rf"}) │
│                     │                          │
│                     ▼                          │
│  ┌──────────────────────────────┐              │
│  │     Permission Checker       │              │
│  │  ┌────────────────────────┐  │              │
│  │  │ Is this tool allowed?  │  │              │
│  │  │ Is this path allowed?  │  │              │
│  │  │ Is this cmd allowed?   │  │              │
│  │  │ Does user need to OK?  │  │              │
│  │  └────────────────────────┘  │              │
│  └──────────────┬───────────────┘              │
│            allowed │ denied                    │
│                 ▼     ▼                        │
│  ┌──────────┐  ┌──────────────┐               │
│  │ Execute  │  │ Return error │               │
│  │  tool    │  │  to model    │               │
│  └──────────┘  └──────────────┘               │
│       │                                        │
│       ▼                                        │
│  ┌──────────────────────────────┐              │
│  │      Result Formatter        │              │
│  │  (truncate, filter, format)  │              │
│  └──────────────────────────────┘              │
└────────────────────────────────────────────────┘
```

This layer is explored in depth in Ch 30. The key insight: permission systems are not just security features. They are trust mechanisms that enable autonomy. The more the system trusts the agent (or the user trusts the system), the more autonomous it can be.

### 2.5 Layer 5: Output Rendering

The system that presents results to the user.

Different tools make radically different rendering choices:

| Tool | Rendering Strategy |
|------|-------------------|
| Claude Code | Streaming markdown in terminal, diff views for file changes |
| Cursor | Inline code suggestions, diff overlays in editor |
| Copilot | Ghost text completions, chat panel |
| Codex | Web UI with file tree, terminal output, PR generation |
| Devin | Recorded browser sessions, terminal replays |

The rendering layer is often underestimated. A tool that shows you exactly what changed (with diffs) builds more trust than one that just says "I updated the file." Claude Code's rendering of file edits as diffs was a deliberate design choice to increase user trust and enable faster review.

---

## 3. Claude Code's Architecture: From CLI to Response

The March 2026 leak gave us an unusually detailed view of a production AI harness. Here is what the community reconstructed.

### 3.1 The Entry Point

Claude Code starts as a CLI application. When you run `claude` in your terminal:

```
┌────────────────────────────────────────────────────────┐
│                    STARTUP SEQUENCE                     │
│                                                         │
│  1. Parse CLI arguments (--model, --resume, etc.)       │
│  2. Load configuration                                  │
│     ├── Global settings (~/.claude/settings.json)       │
│     ├── Project settings (.claude/settings.json)        │
│     └── Environment variables                           │
│  3. Authenticate with Anthropic API                     │
│  4. Initialize session                                  │
│     ├── New session: create conversation                │
│     └── Resume: load previous messages                  │
│  5. Load memory                                         │
│     ├── CLAUDE.md (project memory)                      │
│     ├── Session memory (conversation history)           │
│     └── Persistent memory (long-term files)             │
│  6. Load tools                                          │
│     ├── Built-in tools (Read, Write, Edit, Bash, etc.)  │
│     ├── MCP server tools                                │
│     └── Skill-provided tools                            │
│  7. Load permissions                                    │
│     ├── Allow lists                                     │
│     ├── Deny lists                                      │
│     └── Auto-approval rules                             │
│  8. Initialize UI renderer                              │
│  9. Enter agent loop                                    │
└────────────────────────────────────────────────────────┘
```

### 3.2 The System Prompt

The leak revealed that Claude Code's system prompt is not short. It is a carefully engineered document that includes:

```typescript
// Reconstructed system prompt assembly
function buildSystemPrompt(config: SessionConfig): string {
  const parts: string[] = [];
  
  // 1. Base identity and behavior instructions
  parts.push(BASE_SYSTEM_PROMPT);  // ~2000 tokens
  
  // 2. Tool definitions and usage instructions
  parts.push(formatToolInstructions(config.tools));  // Variable
  
  // 3. Permission context
  parts.push(formatPermissionRules(config.permissions));
  
  // 4. Project memory (CLAUDE.md) -- RE-INJECTED EVERY TURN
  if (config.claudeMd) {
    parts.push(`<project-memory>\n${config.claudeMd}\n</project-memory>`);
  }
  
  // 5. Active skills
  for (const skill of config.activeSkills) {
    parts.push(`<skill name="${skill.name}">\n${skill.content}\n</skill>`);
  }
  
  // 6. Session-specific context
  parts.push(formatSessionContext(config));
  
  return parts.join('\n\n');
}
```

**The critical insight:** CLAUDE.md is inside the system prompt, not in the conversation history. And the system prompt is rebuilt on every turn. This means CLAUDE.md tokens are paid for on every single model call. A CLAUDE.md that is 1000 tokens costs you 1000 input tokens per turn -- across potentially hundreds of turns in a session.

### 3.3 The Tool Inventory

Claude Code's built-in tools, as revealed by the leak:

```typescript
// The core tool set (reconstructed from the leak)
const BUILT_IN_TOOLS = {
  // File operations
  Read: {
    description: "Read a file from the filesystem",
    parameters: { file_path: "string", offset: "number?", limit: "number?" },
    riskLevel: "low"
  },
  Write: {
    description: "Write content to a file",
    parameters: { file_path: "string", content: "string" },
    riskLevel: "medium"
  },
  Edit: {
    description: "Make targeted edits to a file",
    parameters: {
      file_path: "string",
      old_string: "string",
      new_string: "string"
    },
    riskLevel: "medium"
  },
  
  // Search operations
  Glob: {
    description: "Find files by pattern",
    parameters: { pattern: "string", path: "string?" },
    riskLevel: "low"
  },
  Grep: {
    description: "Search file contents with regex",
    parameters: { pattern: "string", path: "string?", include: "string?" },
    riskLevel: "low"
  },
  
  // Execution
  Bash: {
    description: "Execute a shell command",
    parameters: { command: "string", timeout: "number?" },
    riskLevel: "high"  // Requires approval by default
  },
  
  // Agent operations
  Agent: {
    description: "Spawn a sub-agent for a task",
    parameters: { prompt: "string", tools: "string[]?" },
    riskLevel: "medium"
  },
  
  // Web operations
  WebSearch: {
    description: "Search the web",
    parameters: { query: "string" },
    riskLevel: "low"
  },
  WebFetch: {
    description: "Fetch a URL",
    parameters: { url: "string" },
    riskLevel: "low"
  }
};
```

**What to notice:** Each tool has an explicit risk level. This drives the permission system -- low-risk tools auto-approve, high-risk tools require user confirmation. This tiered approach is what enables Claude Code's "auto mode" where it can read and search freely but asks before writing or running commands.

### 3.4 The Agent Loop in Detail

Here is a more detailed reconstruction of Claude Code's production agent loop:

```typescript
async function claudeCodeAgentLoop(session: Session): Promise<void> {
  const { messages, config, tools, permissions } = session;
  
  while (true) {
    // -- Phase 1: Context Assembly --
    
    // Re-inject CLAUDE.md (every turn!)
    const systemPrompt = buildSystemPrompt(config);
    
    // Check token budget
    const tokenCount = countTokens(systemPrompt, messages);
    
    if (tokenCount > config.tokenThreshold) {
      // Compact: summarize old messages, keep recent ones
      const compacted = await compactionService.compact(messages, {
        keepRecent: config.keepRecentMessages,
        summaryModel: config.compactionModel
      });
      messages.length = 0;
      messages.push(...compacted);
    }
    
    // -- Phase 2: Model Call --
    
    const response = await anthropic.messages.create({
      model: config.model,
      system: systemPrompt,
      messages: messages,
      tools: formatTools(tools),
      max_tokens: config.maxTokens,
      stream: true  // Always stream
    });
    
    // -- Phase 3: Response Processing --
    
    const blocks = await collectStreamedBlocks(response);
    
    for (const block of blocks) {
      if (block.type === 'text') {
        // Render text to terminal (streaming)
        renderer.renderText(block.text);
      }
      
      if (block.type === 'tool_use') {
        // -- Phase 4: Permission Check --
        const decision = await permissions.check(block);
        
        switch (decision.type) {
          case 'auto_approve':
            // Low-risk tool, proceed silently
            break;
            
          case 'require_approval':
            // Show the user what the agent wants to do
            renderer.renderToolRequest(block);
            const approved = await ui.promptApproval();
            if (!approved) {
              messages.push({
                role: 'user',
                content: [{
                  type: 'tool_result',
                  tool_use_id: block.id,
                  content: 'Permission denied by user.',
                  is_error: true
                }]
              });
              continue;
            }
            break;
            
          case 'deny':
            // Blocked by deny list
            messages.push({
              role: 'user',
              content: [{
                type: 'tool_result',
                tool_use_id: block.id,
                content: `Tool "${block.name}" is not allowed.`,
                is_error: true
              }]
            });
            continue;
        }
        
        // -- Phase 5: Tool Execution --
        try {
          const result = await executeToolWithTimeout(block, config);
          
          // Truncate large results to avoid context bloat
          const truncated = truncateResult(
            result,
            config.maxToolResultTokens
          );
          
          messages.push({
            role: 'user',
            content: [{
              type: 'tool_result',
              tool_use_id: block.id,
              content: truncated
            }]
          });
        } catch (error) {
          messages.push({
            role: 'user',
            content: [{
              type: 'tool_result',
              tool_use_id: block.id,
              content: `Error: ${error.message}`,
              is_error: true
            }]
          });
        }
      }
    }
    
    // -- Phase 6: Decide Whether to Continue --
    
    const lastBlock = blocks[blocks.length - 1];
    
    if (
      lastBlock.type === 'text' &&
      !blocks.some(b => b.type === 'tool_use')
    ) {
      // Model responded with text only -- it is done
      break;
    }
    
    // If there were tool calls, loop back for the model to see results
  }
}
```

### 3.5 The Rendering Pipeline

Claude Code renders output to the terminal in real-time using streaming. But it is not just dumping text:

```typescript
// Simplified rendering pipeline
class TerminalRenderer {
  renderText(text: string): void {
    // Markdown-aware rendering with syntax highlighting
    // Uses ink (React for CLIs) under the hood
    this.stream(markdownToTerminal(text));
  }
  
  renderToolRequest(block: ToolUseBlock): void {
    // Show what the agent wants to do, formatted for quick review
    switch (block.name) {
      case 'Edit':
        // Show a diff view
        this.renderDiff(
          block.input.file_path,
          block.input.old_string,
          block.input.new_string
        );
        break;
      case 'Write':
        // Show the file that will be created/overwritten
        this.renderFilePreview(block.input.file_path, block.input.content);
        break;
      case 'Bash':
        // Show the command with syntax highlighting
        this.renderCommand(block.input.command);
        break;
      default:
        // Generic tool call display
        this.renderGenericTool(block);
    }
  }
  
  renderDiff(path: string, oldStr: string, newStr: string): void {
    // Red/green diff rendering, similar to git diff
    const diff = createDiff(oldStr, newStr);
    this.stream(colorize(diff));
  }
}
```

**Design insight:** The diff rendering is not cosmetic. It is a trust mechanism. Users review changes faster when they can see exactly what is being modified. This enables higher autonomy -- users are more willing to approve changes they can visually verify.

---

## 4. Cursor's Architecture: LSP-Native AI

Cursor takes a fundamentally different architectural approach from Claude Code. Where Claude Code is a CLI with a model, Cursor is an IDE with a model.

### 4.1 The VS Code Foundation

Cursor is a fork of VS Code. This gives it:

- Complete IDE functionality (editing, debugging, extensions)
- The Language Server Protocol (LSP) for code intelligence
- A rich extension ecosystem
- Visual diff tools, file explorers, terminal integration

But it also constrains the architecture. Everything must work within the VS Code extension model.

```
┌─────────────────────────────────────────────────────────┐
│                    CURSOR ARCHITECTURE                   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              VS Code Shell (Electron)             │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐ │   │
│  │  │ Editor │  │Terminal│  │  File  │  │  Git   │ │   │
│  │  │  Pane  │  │  Pane  │  │Explorer│  │  Pane  │ │   │
│  │  └────┬───┘  └────────┘  └────────┘  └────────┘ │   │
│  │       │                                           │   │
│  │  ┌────┴────────────────────────────────────────┐  │   │
│  │  │          AI Integration Layer                │  │   │
│  │  │                                              │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │   │
│  │  │  │   Tab    │  │ Composer │  │   Chat   │  │  │   │
│  │  │  │Completion│  │  (Agent) │  │  Panel   │  │  │   │
│  │  │  └──────────┘  └──────────┘  └──────────┘  │  │   │
│  │  │       │              │             │        │  │   │
│  │  │  ┌────┴──────────────┴─────────────┴────┐   │  │   │
│  │  │  │        Context Engine                 │   │  │   │
│  │  │  │  (codebase indexing, file graph,      │   │  │   │
│  │  │  │   symbol resolution, LSP data)        │   │  │   │
│  │  │  └──────────────────────────────────────┘   │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘ │
│                          ↕                              │
│            ┌─────────────────────────┐                  │
│            │  Model API (multiple)   │                  │
│            │  Claude / GPT / Custom  │                  │
│            └─────────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Three AI Interaction Modes

Cursor provides three distinct interaction patterns, each with different architecture:

**Tab Completion (inline)**
```
User types code --> Small model predicts next tokens --> Ghost text appears
                    |
                    Uses: current file, open files, recently edited files
                    Model: Fast/small (likely custom fine-tuned model)
                    Latency target: <200ms
```

This is the most constrained mode. The model must be fast enough to predict as you type. Cursor reportedly uses a small custom model for this, not a frontier model.

**Chat Panel (conversational)**
```
User asks question --> Full model generates response --> Displayed in panel
                       |
                       Uses: full conversation history, selected code,
                             codebase index, file graph
                       Model: Frontier (Claude, GPT-4, etc.)
                       Latency target: streaming, first token <1s
```

Similar to Claude Code's agent loop, but integrated into the IDE with visual context.

**Composer (agentic)**
```
User describes task --> Agent loop with multi-file editing --> Changes applied
                        |
                        Uses: full codebase context, tool execution,
                              multi-file editing, terminal access
                        Model: Frontier with tool use
                        Latency target: seconds to minutes
```

Composer is Cursor's agent mode. It can modify multiple files, run commands, and iterate. This is architecturally closest to Claude Code.

### 4.3 The Context Engine

Cursor's key architectural innovation is its codebase indexing engine:

```typescript
// Cursor's context engine (conceptual reconstruction)
interface CursorContextEngine {
  // Codebase-level indexing
  index: {
    files: Map<string, FileMetadata>;
    symbols: Map<string, SymbolInfo>;
    imports: Map<string, ImportGraph>;
    embeddings: Map<string, Float32Array>;
  };
  
  // LSP integration
  lsp: {
    definitions: (symbol: string) => Location[];
    references: (symbol: string) => Location[];
    diagnostics: (file: string) => Diagnostic[];
    completions: (position: Position) => Completion[];
  };
  
  // Context retrieval for model calls
  getRelevantContext(query: string, budget: number): ContextChunk[];
  getFileGraph(file: string, depth: number): FileNode[];
  getSymbolContext(symbol: string): SymbolContext;
}
```

**Key difference from Claude Code:** Claude Code discovers project structure by reading files and running commands during the session. Cursor pre-indexes the codebase at startup. This makes Cursor's initial model calls more informed but adds startup cost and requires re-indexing when files change.

### 4.4 Multi-File Editing

Cursor's Composer mode can edit multiple files in a single operation:

```typescript
// Multi-file edit pipeline (conceptual)
async function composerEdit(
  instruction: string,
  context: CursorContext
): Promise<FileEdit[]> {
  // 1. Plan: identify which files need changes
  const plan = await planEdits(instruction, context);
  
  // 2. For each file, generate the edit
  const edits: FileEdit[] = [];
  for (const file of plan.files) {
    const fileContext = await context.getFileContext(file);
    const edit = await generateEdit(instruction, fileContext, plan);
    edits.push(edit);
  }
  
  // 3. Apply all edits atomically
  // User can accept/reject as a group
  return edits;
}
```

This is architecturally simpler than Claude Code's approach (which uses individual Edit/Write tool calls in a loop) but less flexible. Claude Code can adapt its editing strategy mid-task based on tool results. Cursor's Composer generates a plan and executes it.

---

## 5. Codex/Copilot: Cloud-First Agents

OpenAI's Codex (the agent, not the legacy model) and GitHub Copilot take yet another architectural approach: cloud-native agent execution.

### 5.1 The Container Model

```
┌──────────────────────────────────────────────┐
│              USER'S MACHINE                   │
│  ┌────────────────────────────────────────┐   │
│  │  CLI / Web UI / IDE Extension          │   │
│  │  (thin client -- sends task, gets PR)  │   │
│  └──────────────────┬─────────────────────┘   │
└─────────────────────┼────────────────────────┘
                      │ HTTPS
                      ▼
┌──────────────────────────────────────────────┐
│              CLOUD ENVIRONMENT                │
│  ┌────────────────────────────────────────┐   │
│  │  Orchestrator                          │   │
│  │  (task queue, agent lifecycle)         │   │
│  └──────────────────┬─────────────────────┘   │
│                     ▼                         │
│  ┌────────────────────────────────────────┐   │
│  │  Container (per task)                  │   │
│  │  ┌──────────────────────────────────┐  │   │
│  │  │  Cloned repository               │  │   │
│  │  │  Full dev environment            │  │   │
│  │  │  Agent loop (model + tools)      │  │   │
│  │  │  Test runner                     │  │   │
│  │  │  Git client                      │  │   │
│  │  └──────────────────────────────────┘  │   │
│  └────────────────────────────────────────┘   │
│                     │                         │
│                     ▼                         │
│  ┌────────────────────────────────────────┐   │
│  │  Output: PR / Patch / Report           │   │
│  └────────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

**Key architectural difference:** The agent runs in a cloud container with a clone of your repository. It does not run on your machine. This means:

1. **Full isolation** -- the agent cannot affect your local state
2. **Reproducibility** -- every run starts from a clean repo clone
3. **No real-time interaction** -- you send a task and get back a result (PR, patch)
4. **CI as feedback** -- the agent can run tests in its container

### 5.2 The One-Shot Pattern

Codex follows a fundamentally different interaction pattern:

```
Traditional agent (Claude Code, Cursor):
  Human -> Agent -> Human -> Agent -> Human -> Agent -> Done
  (interactive, back-and-forth)

One-shot agent (Codex):
  Human -> Agent -> [many internal tool loops] -> Done (PR)
  (batch mode, no human in the loop during execution)
```

This changes the architecture requirements. A one-shot agent needs:
- Better planning (no human to course-correct)
- More aggressive testing (CI replaces human review during execution)
- Clearer task specification (no clarifying questions mid-task)
- Result packaging (the output is a PR, not a conversation)

### 5.3 Stripe's Minions Pattern

Stripe shared their approach to one-shot agents at scale. Their "Minions" system uses Claude Code in a pattern similar to Codex but with organizational integration:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Linear    │     │   Minions    │     │    GitHub     │
│   Ticket    │ --> │  Orchestrator│ --> │  Pull Request │
│  (task spec)│     │  (spawns     │     │  (output)     │
└─────────────┘     │   agent)     │     └──────┬───────┘
                    └──────────────┘            │
                                                ▼
                                         ┌──────────────┐
                                         │   CI Tests   │
                                         │  (feedback)  │
                                         └──────────────┘
```

The feedback loop is CI, not human interaction. The agent writes code, pushes it, CI runs tests, and if tests fail, the agent gets the failure output and iterates. This is explored further in Ch 32.

---

## 6. Why the Harness Matters More Than the Model

### 6.1 The Evidence

The leak provided concrete evidence for the harness thesis:

**Evidence 1: Same model, different capabilities.** Claude Sonnet through the API can write code. Claude Sonnet through Claude Code can read your project, understand its conventions, write code that fits your style, run tests to verify it works, and fix issues autonomously. The model is identical. The harness is different.

**Evidence 2: Model-agnostic improvements.** When Claude Code improved its compaction service, all models running through it got better at long tasks. When it improved its permission system, all models became more trustworthy. These improvements were model-independent.

**Evidence 3: The feature flag system.** The leak revealed Statsig-powered feature flags controlling session limits, compaction thresholds, and tool availability. These are harness parameters that dramatically affect user experience -- and they have nothing to do with the model.

### 6.2 The Implication for Your Work

If you are building AI features, this means:

1. **Invest more in the harness than in model selection.** Switching from one frontier model to another gives you marginal improvements. Building a better context assembly system gives you dramatic improvements.

2. **The model is a commodity. The harness is your moat.** Anyone can call the same API you call. But your harness -- your memory system, your context assembly, your tool integrations, your permission layer -- that is unique to your product.

3. **Harness engineering is software engineering.** It is not ML. It is systems design, state management, API integration, and user experience. The skills you already have as a software engineer are the skills you need.

### 6.3 What to Steal

From Claude Code:
- **CLAUDE.md re-injection** -- project context on every turn
- **Risk-tiered permissions** -- auto-approve safe actions, gate risky ones
- **Compaction service** -- summarize old context to extend session length
- **Tool result truncation** -- prevent large outputs from blowing the context

From Cursor:
- **Codebase indexing** -- pre-compute context for faster model calls
- **LSP integration** -- use language intelligence for better context
- **Three interaction modes** -- different UI for different tasks
- **Fast completion model** -- use a small model for inline suggestions

From Codex:
- **Container isolation** -- run agents in clean environments
- **CI as feedback loop** -- use tests instead of human review
- **One-shot execution** -- batch mode for well-specified tasks
- **PR as output** -- package results for review

---

## 7. Common Patterns Across All AI Developer Tools

### 7.1 The Universal Architecture

Despite their differences, every tool we examined shares these patterns:

```
┌──────────────────────────────────────────────────┐
│           UNIVERSAL HARNESS PATTERNS              │
│                                                    │
│  1. CONTEXT ASSEMBLY                               │
│     Every tool builds a prompt from multiple        │
│     sources: system prompt + memory + history       │
│     + tool results + file contents                  │
│                                                    │
│  2. AGENT LOOP                                     │
│     Every tool runs some variant of                 │
│     prompt -> model -> tool -> result -> repeat     │
│                                                    │
│  3. TOOL EXECUTION                                 │
│     Every tool provides ways for the model          │
│     to act on the world: files, commands, search   │
│                                                    │
│  4. PERMISSION GATING                              │
│     Every tool has some mechanism to control         │
│     what the model can do without human approval    │
│                                                    │
│  5. CONTEXT MANAGEMENT                             │
│     Every tool deals with the finite context         │
│     window: compaction, summarization, eviction     │
│                                                    │
│  6. MEMORY PERSISTENCE                             │
│     Every tool has some way to remember across       │
│     sessions: project files, databases, indexes     │
│                                                    │
│  7. OUTPUT RENDERING                               │
│     Every tool formats model output for human        │
│     consumption: diffs, highlights, streaming       │
└──────────────────────────────────────────────────┘
```

### 7.2 Pattern: The System Prompt Sandwich

Every tool uses a variation of this prompt structure:

```
┌──────────────────────────────────────┐
│  SYSTEM PROMPT (fixed)               │  <-- Identity, rules, instructions
│  PROJECT MEMORY (per-project)        │  <-- CLAUDE.md or equivalent
│  SKILL CONTEXT (per-task)            │  <-- Active skills, loaded docs
├──────────────────────────────────────┤
│  CONVERSATION HISTORY (dynamic)      │  <-- Messages, tool calls, results
│  ...potentially compacted...         │
├──────────────────────────────────────┤
│  RECENT MESSAGES (protected)         │  <-- Last N messages, never compacted
│  CURRENT USER MESSAGE                │  <-- What the user just said
└──────────────────────────────────────┘
```

The "sandwich" is: fixed context on top, dynamic history in the middle, recent context on the bottom. Compaction targets the middle. The top and bottom are protected.

### 7.3 Pattern: Graduated Autonomy

Every tool implements a trust spectrum:

```
No autonomy          Partial autonomy       Full autonomy
     │                      │                      │
     ▼                      ▼                      ▼
┌─────────┐          ┌─────────────┐        ┌──────────┐
│ Ask for │          │ Auto-approve│        │ Run      │
│ every   │          │ safe actions│        │ everything│
│ action  │          │ Ask for     │        │ without  │
│         │          │ risky ones  │        │ asking   │
└─────────┘          └─────────────┘        └──────────┘
  Default              Recommended            Dangerous
  (new users)          (experienced)          (CI/automation)
```

Claude Code calls these modes: default, auto, dangerously-skip-permissions.
Cursor has similar graduated levels of edit acceptance.
Codex runs in full autonomy within its container (sandboxed instead of permissioned).

### 7.4 Pattern: Token Budget Consciousness

Every tool is acutely aware of token costs:

```typescript
// The token budget pattern found across all tools
interface TokenBudget {
  // Hard limits
  contextWindowSize: number;     // Model's maximum (e.g., 200k)
  reservedForOutput: number;     // Tokens reserved for model response
  
  // Soft limits
  compactionThreshold: number;   // When to start compacting
  warningThreshold: number;      // When to warn the user
  
  // Budget allocation
  systemPromptBudget: number;    // Fixed overhead per turn
  memoryBudget: number;          // Project memory allocation
  historyBudget: number;         // Conversation history allocation
  toolResultBudget: number;      // Per-tool-result limit
  
  // Computed
  availableForHistory(): number {
    return this.contextWindowSize 
      - this.reservedForOutput 
      - this.systemPromptBudget 
      - this.memoryBudget;
  }
}
```

---

## 8. Building Your Own: The Minimum Viable Harness

Armed with these patterns, here is the minimum viable harness for a coding agent.

### 8.1 The Architecture

```typescript
// minimum-viable-harness.ts
// A complete but minimal agent harness

import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';
import * as path from 'path';
import * as readline from 'readline';

// -- Layer 1: Configuration --
interface HarnessConfig {
  model: string;
  maxTokens: number;
  projectRoot: string;
  memoryFile: string;       // Your CLAUDE.md equivalent
  autoApproveReads: boolean;
  maxToolResultLength: number;
}

const DEFAULT_CONFIG: HarnessConfig = {
  model: 'claude-sonnet-4-20250514',
  maxTokens: 8192,
  projectRoot: process.cwd(),
  memoryFile: 'AGENT.md',
  autoApproveReads: true,
  maxToolResultLength: 10000,
};

// -- Layer 2: Tool Definitions --
const TOOLS: Anthropic.Messages.Tool[] = [
  {
    name: 'read_file',
    description: 'Read a file from the project',
    input_schema: {
      type: 'object' as const,
      properties: {
        path: {
          type: 'string',
          description: 'File path relative to project root'
        }
      },
      required: ['path']
    }
  },
  {
    name: 'write_file',
    description: 'Write content to a file',
    input_schema: {
      type: 'object' as const,
      properties: {
        path: {
          type: 'string',
          description: 'File path relative to project root'
        },
        content: {
          type: 'string',
          description: 'File content to write'
        }
      },
      required: ['path', 'content']
    }
  },
  {
    name: 'list_files',
    description: 'List files matching a glob pattern',
    input_schema: {
      type: 'object' as const,
      properties: {
        pattern: {
          type: 'string',
          description: 'Glob pattern (e.g., "src/**/*.ts")'
        }
      },
      required: ['pattern']
    }
  }
];

// -- Layer 3: Tool Execution with Permissions --
async function executeTool(
  name: string,
  input: Record<string, unknown>,
  config: HarnessConfig,
  promptApproval: () => Promise<boolean>
): Promise<string> {
  const fullPath = (p: string) =>
    path.resolve(config.projectRoot, p as string);
  
  switch (name) {
    case 'read_file': {
      // Auto-approve reads (low risk)
      const content = fs.readFileSync(
        fullPath(input.path as string), 'utf-8'
      );
      return content.slice(0, config.maxToolResultLength);
    }
    
    case 'write_file': {
      // Require approval for writes (medium risk)
      console.log(`\n--- Agent wants to write: ${input.path} ---`);
      console.log(
        (input.content as string).slice(0, 500) + '...'
      );
      if (!await promptApproval()) return 'Permission denied.';
      
      const filePath = fullPath(input.path as string);
      fs.mkdirSync(path.dirname(filePath), { recursive: true });
      fs.writeFileSync(filePath, input.content as string);
      return `Wrote ${(input.content as string).length} bytes`;
    }
    
    case 'list_files': {
      // Auto-approve listings (low risk)
      const { globSync } = require('glob');
      const files = globSync(
        input.pattern as string,
        { cwd: config.projectRoot }
      );
      return files.join('\n');
    }
    
    default:
      return `Unknown tool: ${name}`;
  }
}

// -- Layer 4: Context Assembly --
function buildSystemPrompt(config: HarnessConfig): string {
  let prompt = [
    'You are a coding assistant working in the project',
    `at ${config.projectRoot}.`,
    'You can read files, write files, and list files.',
    'Always read relevant files before making changes.',
  ].join(' ');
  
  // Inject project memory (like CLAUDE.md)
  const memoryPath = path.join(
    config.projectRoot, config.memoryFile
  );
  if (fs.existsSync(memoryPath)) {
    const memory = fs.readFileSync(memoryPath, 'utf-8');
    prompt += `\n\n<project-memory>\n${memory}\n</project-memory>`;
  }
  
  return prompt;
}

// -- Layer 5: The Agent Loop --
async function agentLoop(
  config: HarnessConfig = DEFAULT_CONFIG
): Promise<void> {
  const client = new Anthropic();
  const messages: Anthropic.Messages.MessageParam[] = [];
  const systemPrompt = buildSystemPrompt(config);
  
  const rl = readline.createInterface({
    input: process.stdin, output: process.stdout
  });
  const ask = (q: string) =>
    new Promise<string>(r => rl.question(q, r));
  const promptApproval = async () =>
    (await ask('Approve? (y/n): ')).toLowerCase() === 'y';
  
  console.log('Agent ready. Type your request.\n');
  
  while (true) {
    const userInput = await ask('You: ');
    if (userInput === 'exit') break;
    
    messages.push({ role: 'user', content: userInput });
    
    // Agent inner loop (tool use cycle)
    while (true) {
      const response = await client.messages.create({
        model: config.model,
        max_tokens: config.maxTokens,
        system: systemPrompt,
        messages,
        tools: TOOLS,
      });
      
      const assistantContent = response.content;
      messages.push({ role: 'assistant', content: assistantContent });
      
      for (const block of assistantContent) {
        if (block.type === 'text') {
          console.log(`\nAgent: ${block.text}`);
        }
      }
      
      const toolUseBlocks = assistantContent.filter(
        (b): b is Anthropic.Messages.ToolUseBlock =>
          b.type === 'tool_use'
      );
      
      if (toolUseBlocks.length === 0) break;
      
      const toolResults: Anthropic.Messages.ToolResultBlockParam[] = [];
      for (const toolUse of toolUseBlocks) {
        console.log(`\n[Tool: ${toolUse.name}]`);
        const result = await executeTool(
          toolUse.name,
          toolUse.input as Record<string, unknown>,
          config,
          promptApproval
        );
        toolResults.push({
          type: 'tool_result',
          tool_use_id: toolUse.id,
          content: result,
        });
      }
      
      messages.push({ role: 'user', content: toolResults });
    }
  }
  
  rl.close();
}

// -- Entry Point --
agentLoop().catch(console.error);
```

### 8.2 What This Harness Has

- Context assembly with project memory injection (every turn)
- Agent loop with tool use cycling
- Three tools: read, write, list
- Risk-tiered permissions (reads auto-approve, writes require approval)
- Tool result truncation
- Streaming conversation

### 8.3 What This Harness Lacks (and Where to Find It)

| Missing Feature | Where to Learn |
|----------------|---------------|
| Memory persistence across sessions | Ch 28 |
| Context compaction | Ch 29 |
| Fine-grained permission system | Ch 30 |
| Web search and knowledge retrieval | Ch 31 |
| Multi-agent coordination | Ch 32 |
| Skill/plugin extension system | Ch 33 |
| Production error handling | Ch 48 |
| Cost optimization | Ch 49 |
| Deployment and sandboxing | Ch 55 |

This minimum viable harness is about 200 lines. Claude Code is orders of magnitude larger. The remaining chapters in this Part explain why, one system at a time.

---

## 9. Architecture Comparison Summary

```
┌──────────────┬───────────────┬───────────────┬───────────────┐
│              │  Claude Code  │    Cursor     │    Codex      │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Interface    │ CLI           │ IDE (VS Code) │ Web/CLI       │
│ Execution    │ Local machine │ Local machine │ Cloud contner │
│ Interaction  │ Conversational│ Multi-mode    │ One-shot      │
│ Context      │ Runtime disc. │ Pre-indexed   │ Repo clone    │
│ Memory       │ CLAUDE.md +   │ Codebase      │ Per-task      │
│              │ KAIROS        │ index         │ (stateless)   │
│ Tools        │ File/Bash/Web │ Edit/Terminal │ Full dev env  │
│ Permissions  │ Allow/deny    │ Accept/reject │ Container     │
│              │ lists         │ changes       │ sandbox       │
│ Output       │ Terminal      │ Editor        │ PR / Patch    │
│ Feedback     │ Human review  │ Human review  │ CI tests      │
│ Strengths    │ Flexible,     │ IDE context,  │ Isolated,     │
│              │ extensible    │ fast UX       │ scalable      │
│ Weaknesses   │ No visual     │ Model-locked  │ No real-time  │
│              │ context       │ architecture  │ interaction   │
└──────────────┴───────────────┴───────────────┴───────────────┘
```

---

## 10. Model Codenames and the Multi-Version Pipeline

### 10.1 What the Source Revealed

The leak exposed internal model codenames used within Claude Code's configuration and routing logic. These names are not publicly documented, but their presence reveals how Anthropic manages a multi-version model pipeline:

```
┌──────────────────────────────────────────────────────────┐
│              MODEL CODENAMES (from source)                 │
│                                                           │
│  Capybara / Mythos                                        │
│  ├── Maps to: v8 architecture, 1M context window          │
│  ├── Used in: "fast mode" configuration                   │
│  └── Likely: the model powering high-throughput sessions   │
│                                                           │
│  Numbat                                                   │
│  ├── Referenced with: a launch window date in source       │
│  ├── Status: upcoming / unreleased at time of leak         │
│  └── Likely: a next-generation model in the pipeline       │
│                                                           │
│  Fennec                                                   │
│  ├── Speculated to be: Opus 4.6 (the 1M context model)    │
│  ├── Referenced in: model routing configuration            │
│  └── Likely: the high-capability tier model                │
│                                                           │
│  Tengu                                                    │
│  ├── Referenced in: undercover.ts                          │
│  ├── Used in: undercover mode configuration                │
│  └── Purpose: unclear, possibly a specialized variant      │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### 10.2 What Codenames Reveal About Architecture

The existence of multiple codenames with different configurations tells us that model versioning is a first-class architectural concern inside the harness. This is not "we have one model and we call it." It is a pipeline:

1. **Parallel development tracks.** Multiple models exist simultaneously at different stages of readiness. The harness must route requests to the right model based on user tier, task type, and feature flags.

2. **Model-specific configuration.** Each codename maps to a different context window size, different token limits, and different cost profiles. The harness adapts its behavior (compaction thresholds, output limits, caching strategy) per model.

3. **Gradual rollout.** The Numbat codename with a launch window suggests that new models are staged behind date-gated feature flags, not deployed all at once. The harness handles the transition.

### 10.3 Undercover Mode and the One-Way Door

The `undercover.ts` file referenced alongside the Tengu codename implements a mode where Claude Code operates without disclosing that AI was involved in generating the code. The source revealed a critical architectural detail: undercover mode is a **one-way door**. Once enabled, there is no force-OFF mechanism -- the agent will not disclose its involvement regardless of subsequent instructions.

This is relevant to harness architecture because it shows how deeply behavioral modes can be embedded. It is not a prompt-level toggle that the model can be convinced to override. It is a code-level constraint that strips identifying markers from external builds before they ship.

### 10.4 Implications for Your Own Multi-Model Systems

If you are building an agent that supports multiple models (which most production agents eventually do), the codename pattern teaches you to:

- **Treat model identity as configuration, not code.** Model names, context limits, and pricing should live in config, not hardcoded strings.
- **Route by capability.** Different models have different strengths. The harness should route tasks to the appropriate model based on complexity, cost budget, and required context length.
- **Version your model configurations.** When you swap Model A for Model B, the harness behavior should adapt automatically (different compaction thresholds, different token budgets, different caching strategies).
- **Gate rollouts with feature flags.** Never switch all users to a new model simultaneously. Use feature flags (Statsig, LaunchDarkly) to roll out gradually and measure the impact.

---

## 11. Open-Source Reimplementations: The Harness Pattern Proven Model-Agnostic

### 11.1 Why Reimplementations Matter

The strongest validation of the harness thesis comes from outside Anthropic. Two open-source projects have reconstructed the Claude Code harness from first principles, proving that the architecture is not locked to a single model or runtime.

### 11.2 claw-code-agent: The Full Python Reimplementation

[claw-code-agent](https://github.com/HarnessLab/claw-code-agent) is a pure Python, zero-dependency reimplementation of the Claude Code harness. With over 75,000 stars, it is the most popular open-source Claude Code alternative. It works with vLLM, Ollama, LiteLLM Proxy, and OpenRouter -- proving the harness pattern is model-agnostic.

The architectural mapping between claw-code-agent and the Claude Code harness is direct:

```
┌──────────────────────────────────────────────────────────┐
│    CLAW-CODE-AGENT ↔ CLAUDE CODE ARCHITECTURE MAP        │
│                                                           │
│  agent_runtime.py       = Agent loop (Ch 27 Section 3)   │
│  agent_tools.py         = Tool system (Ch 30)            │
│  agent_prompting.py     = System prompt assembly          │
│  agent_context.py       = Context window management       │
│  compact.py             = Compaction service (Ch 29)      │
│  permissions.py         = Permission engine (Ch 30)       │
│  session_store.py       = Session persistence             │
│  plugin_runtime.py      = Plugin hooks (Ch 33)            │
│  mcp_runtime.py         = MCP client (Ch 31)              │
│  agent_manager.py       = Multi-agent coordination (Ch 32)│
│  cost_tracker.py        = Usage budget enforcement        │
│  tokenizer_runtime.py   = Token counting                  │
│  hook_policy.py         = Hook execution policy           │
│  worktree_runtime.py    = Git worktree isolation (Ch 32)  │
│  background_runtime.py  = Background agent execution      │
│  task_runtime.py        = Task delegation                 │
│  plan_runtime.py        = Topological batch planning      │
│  workflow_runtime.py    = Workflow orchestration           │
│  team_runtime.py        = Agent-group management          │
│  openai_compat.py       = Multi-provider API translation  │
│  query_engine.py        = Main query/response engine      │
│                                                           │
│  TOTAL: 55 Python files. Zero external dependencies.      │
└──────────────────────────────────────────────────────────┘
```

The key insight: the entire Claude Code harness decomposes into roughly 55 well-scoped Python modules. Each module maps to one concept from this Part. The claw-code-agent repository is effectively a build guide for constructing a production-grade AI coding agent.

Notable capabilities reproduced in pure Python:

- **Agent runtime:** One-shot agent loop with iterative tool calling and streaming token-by-token output
- **Context management:** Automatic compact-boundary insertion with preserved recent tail, reactive compaction retry after prompt-too-long errors, and compaction metadata tracking (compacted message IDs, preserved-tail IDs, depth, lineage, revision summaries)
- **Session persistence:** Transcript-aware save/resume with file history journaling for write/edit/shell actions
- **Multi-agent:** Dependency-aware delegated subtasks with topological batch planning, agent-manager lineage tracking, child-session resume by saved session ID
- **Plugin system:** Manifest-based runtime discovery with hooks at five lifecycle phases (before-prompt, after-turn, resume, persist, delegate), tool aliases, virtual tools, tool blocking, and plugin session-state persistence
- **MCP integration:** Real stdio transport with full initialize/resources/tools lifecycle

### 11.3 nano-claude-code: The Minimal Viable Implementation

From the [collection-claude-code-source-code](https://github.com/chauncygu/collection-claude-code-source-code) repository comes nano-claude-code: a minimal implementation in roughly 5,000 lines of Python. It is immediately runnable and includes:

- Multi-provider support (same model-agnostic pattern)
- 18 tools (the practical minimum for a useful coding agent)
- Dual-scope memory (project-level and session-level)
- A skill system for extensibility

This proves the minimum viable harness from Section 8 of this chapter can be extended to production-quality in about 5,000 lines. The gap between our 200-line TypeScript example and a fully capable agent is not as large as it might seem.

### 11.4 What the Original Source Reveals

The same collection repository also contains reconstructions of Claude Code's original TypeScript source. Key findings that reinforce what the leak suggested:

- **query.ts is 785KB** -- the main agent loop in a single massive file. This confirms that the agent loop is the core of the system, with everything else orbiting it.
- **40+ tools** in the original source, compared to nano-claude-code's 18. The extra tools handle edge cases, notebook editing, web operations, and agent spawning.
- **~87 slash commands** with metadata-driven routing. The command system is larger than most developers expect.
- **6+ killswitches** for remote control infrastructure. The production harness includes operational controls (feature flags, rate limiters, emergency stops) that open-source reimplementations can omit.

### 11.5 The Lesson for Builders

If you want to build your own Claude Code-quality harness:

1. **Start with the 55-file blueprint.** The claw-code-agent module list is the architecture. Each file is one concern. Build them one at a time.
2. **Start minimal.** nano-claude-code proves 5,000 lines and 18 tools gets you a working agent. Add complexity only when you need it.
3. **The harness is model-agnostic.** claw-code-agent works with vLLM, Ollama, LiteLLM, and OpenRouter. Your harness should target the OpenAI-compatible API format and swap models freely.
4. **Python works fine.** Claude Code is TypeScript. claw-code-agent is Python. The architecture translates cleanly because it is fundamentally about state management, tool execution, and context assembly -- not language-specific features.

---

## Build It: A Minimal Agent Harness in 80 Lines

You have now seen the architecture of production agent harnesses. But here is the secret: the core pattern is buildable in a single file. The following script implements the complete agent loop -- prompt, LLM call, tool execution, permission check, streaming output -- in roughly 80 lines of Python. It works with any OpenAI-compatible API (OpenAI, Anthropic via proxy, Ollama, LiteLLM, OpenRouter). Copy this file, run it, and you have a working agent. Then add complexity one piece at a time, just like the open-source reimplementations did.

```bash
pip install openai
```

```python
# Minimal agent harness — inspired by nano-claude-code and claw-code-agent
# Usage: OPENAI_API_KEY=sk-... python harness.py
#   (or set OPENAI_BASE_URL for Ollama, LiteLLM, OpenRouter, etc.)

import json, os, subprocess, sys
from typing import Callable
from openai import OpenAI

# --- Tool Registry -----------------------------------------------------------

TOOLS: dict[str, dict] = {}  # name -> {description, function, schema}

def register_tool(name: str, description: str, params: dict, fn: Callable):
    """Register a tool the agent can call."""
    TOOLS[name] = {
        "function": fn,
        "definition": {
            "type": "function",
            "function": {
                "name": name,
                "description": description,
                "parameters": {"type": "object", "properties": params, "required": list(params)},
            },
        },
    }

# --- Built-in Tools ----------------------------------------------------------

def read_file(path: str) -> str:
    try:
        return open(path).read()[:8000]  # truncate large files
    except Exception as e:
        return f"Error: {e}"

def write_file(path: str, content: str) -> str:
    open(path, "w").write(content)
    return f"Wrote {len(content)} chars to {path}"

def run_shell(command: str) -> str:
    # --- Permission Check: ask before executing shell commands ---
    print(f"\n  [PERMISSION] Agent wants to run: {command}")
    answer = input("  Allow? (y/n): ").strip().lower()
    if answer != "y":
        return "Permission denied by user."
    result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
    return (result.stdout + result.stderr)[:8000]

register_tool("read_file", "Read a file from disk",
              {"path": {"type": "string", "description": "File path to read"}}, 
              lambda **kw: read_file(kw["path"]))
register_tool("write_file", "Write content to a file",
              {"path": {"type": "string", "description": "File path"},
               "content": {"type": "string", "description": "Content to write"}},
              lambda **kw: write_file(kw["path"], kw["content"]))
register_tool("run_shell", "Run a shell command (requires user approval)",
              {"command": {"type": "string", "description": "Shell command to execute"}},
              lambda **kw: run_shell(kw["command"]))

# --- Agent Loop --------------------------------------------------------------

def agent_loop(user_message: str, model: str = "gpt-4o"):
    client = OpenAI()  # reads OPENAI_API_KEY and OPENAI_BASE_URL from env
    messages = [
        {"role": "system", "content": "You are a helpful coding agent. Use tools to help the user."},
        {"role": "user", "content": user_message},
    ]
    tool_defs = [t["definition"] for t in TOOLS.values()]

    while True:
        # --- Call the model with streaming ---
        stream = client.chat.completions.create(
            model=model, messages=messages, tools=tool_defs, stream=True
        )
        response_text = ""
        tool_calls = []
        current_tc = None

        for chunk in stream:
            delta = chunk.choices[0].delta
            # Stream text tokens as they arrive
            if delta.content:
                print(delta.content, end="", flush=True)
                response_text += delta.content
            # Accumulate tool calls
            if delta.tool_calls:
                for tc in delta.tool_calls:
                    if tc.index is not None and tc.index >= len(tool_calls):
                        tool_calls.append({"id": tc.id or "", "name": "", "arguments": ""})
                    idx = tc.index if tc.index is not None else len(tool_calls) - 1
                    if tc.function:
                        if tc.function.name:
                            tool_calls[idx]["name"] = tc.function.name
                            tool_calls[idx]["id"] = tc.id or tool_calls[idx]["id"]
                        if tc.function.arguments:
                            tool_calls[idx]["arguments"] += tc.function.arguments

        # If no tool calls, the agent is done
        if not tool_calls:
            print()  # newline after streamed output
            return response_text

        # Build assistant message with tool_calls
        assistant_msg = {"role": "assistant", "content": response_text or None, "tool_calls": [
            {"id": tc["id"], "type": "function",
             "function": {"name": tc["name"], "arguments": tc["arguments"]}}
            for tc in tool_calls
        ]}
        messages.append(assistant_msg)

        # --- Execute each tool call and append results ---
        for tc in tool_calls:
            name, args_str = tc["name"], tc["arguments"]
            print(f"\n  [TOOL] {name}({args_str[:80]}{'...' if len(args_str) > 80 else ''})")
            if name in TOOLS:
                args = json.loads(args_str)
                result = TOOLS[name]["function"](**args)
            else:
                result = f"Unknown tool: {name}"
            messages.append({"role": "tool", "tool_call_id": tc["id"], "content": str(result)})

        # Loop back: the model will see the tool results and continue

if __name__ == "__main__":
    print("Minimal Agent Harness (Ctrl+C to quit)")
    print("=" * 40)
    while True:
        try:
            user_input = input("\nYou: ").strip()
            if not user_input:
                continue
            agent_loop(user_input)
        except KeyboardInterrupt:
            print("\nGoodbye!")
            break
```

The key architectural insights are visible even in this minimal version: the tool registry decouples tool definition from the loop, the permission check gates dangerous operations, and the while-True loop with tool result appending is the exact same pattern that powers Claude Code. The only difference between this and a production harness is how many layers of sophistication you add around each step.

---

## 12. Key Takeaways

1. **The harness is the product.** The model is a commodity API call. The harness -- context assembly, tool execution, permission management, memory, rendering -- is what makes the product useful.

2. **Five layers are universal.** Input interface, context assembly, agent loop, tool execution, output rendering. Every AI developer tool implements these five layers.

3. **Architecture choices cascade.** CLI vs IDE vs Web. Local vs Cloud. Interactive vs One-shot. Each choice constrains the rest of the architecture.

4. **Claude Code's key innovations:** CLAUDE.md re-injection, risk-tiered permissions, compaction service, extensible tool system.

5. **Cursor's key innovations:** Codebase pre-indexing, LSP integration, three interaction modes (tab/chat/composer).

6. **Codex's key innovations:** Container isolation, CI as feedback loop, one-shot batch execution.

7. **The minimum viable harness** is about 200 lines. Production harnesses are orders of magnitude larger because of memory, context management, permissions, multi-agent coordination, and extension systems -- the topics of the remaining six chapters.

---

*Next: [Chapter 28 -- Memory Systems: KAIROS & Beyond](./28-memory-systems.md)* explores how these harnesses remember across sessions, projects, and time.

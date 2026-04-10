<!--
  CHAPTER: 30
  TITLE: Tool Execution & Permissions
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 8 (Tool Calling), Ch 11 (Human-in-the-Loop), Ch 27 (Harness Architecture)
  KEY_TOPICS: tool definitions, permission model, allow/deny lists, auto mode, risk tiers, sandboxing, frustration detection, adversarial verification, tool execution pipeline
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 30: Tool Execution & Permissions

> Part 6: Anatomy of AI Developer Tools · Phase 2 · Prerequisites: Ch 8, Ch 11, Ch 27 · Advanced · TypeScript + Architecture

When a model calls a tool, it is expressing an intention: "I want to read this file" or "I want to run this command." The harness must decide whether to fulfill that intention, and how. This decision -- what the agent can do, what it cannot, and what requires human approval -- is the most consequential design choice in any AI developer tool. Get it right and the agent is a trusted collaborator. Get it wrong and it is either dangerously autonomous or uselessly constrained.

The Claude Code leak revealed a permission system far more nuanced than a simple allow/deny list. It uses risk-tiered approval, pattern-matched allow lists, a frustration detection system that notices when the agent is struggling, and adversarial verification where the agent double-checks its own actions. This chapter dissects all of it.

### In This Chapter

1. How tools are defined and registered
2. The tool execution pipeline
3. The permission model: allow lists, deny lists, and risk tiers
4. Auto mode and dangerously-skip-permissions
5. How Cursor handles tool execution differently
6. Sandboxing strategies across tools
7. The frustration detection pattern
8. Adversarial verification: the agent checks itself
9. Designing a permission system for your own agents
10. The trust spectrum in production

### Related Chapters

- **Ch 8 (Tool Calling)** -- introduced tool definitions; this chapter shows the production execution system
- **Ch 11 (Human-in-the-Loop)** -- introduced approval concepts; this chapter shows real architecture
- **Ch 27 (Harness Architecture)** -- showed where tool execution fits in the harness
- **Ch 48 (AI Security & Guardrails)** -- will explore security implications of these permission patterns
- **Ch 55 (Sandboxing & Isolating Agents)** -- will go deep on container-based isolation

---

## 1. How Tools Are Defined and Registered

### 1.1 The Tool Definition Format

In Claude Code, each tool is defined with a schema that the model uses to understand how to call it:

```typescript
// Tool definition format (from the API)
interface ToolDefinition {
  name: string;
  description: string;
  input_schema: {
    type: 'object';
    properties: Record<string, PropertySchema>;
    required: string[];
  };
}

// Internal extension: risk metadata
interface InternalToolDefinition extends ToolDefinition {
  riskLevel: 'low' | 'medium' | 'high';
  category: 'read' | 'write' | 'execute' | 'search' | 'agent';
  requiresApproval: boolean;
  timeout: number;
  maxResultSize: number;
}
```

### 1.2 Tool Registration

Tools are registered from multiple sources at session start:

```
┌──────────────────────────────────────────────────┐
│              TOOL REGISTRATION                    │
│                                                    │
│  Source 1: Built-in Tools                          │
│  ├── Read, Write, Edit                             │
│  ├── Glob, Grep                                    │
│  ├── Bash                                          │
│  ├── Agent (sub-agent spawning)                    │
│  ├── WebSearch, WebFetch                           │
│  └── Notebook tools (NotebookEdit, etc.)           │
│                                                    │
│  Source 2: MCP Server Tools                        │
│  ├── Connected at session start                    │
│  ├── Each server exposes N tools                   │
│  └── Dynamically added when servers connect        │
│                                                    │
│  Source 3: Skill-Provided Tools                    │
│  ├── Skills can define custom tools                │
│  └── Available when the skill is active            │
│                                                    │
│  Source 4: Deferred Tools                          │
│  ├── Tool definitions loaded on demand             │
│  ├── Reduces initial context size                  │
│  └── ToolSearch resolves them when needed          │
│                                                    │
│  ALL tools are merged into a single tools array    │
│  sent with every model call                        │
└──────────────────────────────────────────────────┘
```

### 1.3 The Description Matters More Than You Think

The tool description is not just documentation -- it is the primary way the model decides which tool to use and how to use it. A bad description leads to misuse.

```typescript
// Bad description: vague, no guidance
{
  name: 'Bash',
  description: 'Run a command',
  // Model might use this for anything, including dangerous commands
}

// Good description: specific, with guardrails
{
  name: 'Bash',
  description: `Execute a shell command in the user's project directory.
Use for: running tests, installing packages, checking git status,
building the project, and other development tasks.

IMPORTANT: Prefer Read/Write/Edit for file operations instead of
shell commands like cat/sed/echo. Prefer Glob/Grep for file search
instead of find/grep.

Commands run with a default timeout. Long-running commands
(servers, watchers) should use the background flag.

The working directory is the project root.`,
  // Model now has clear guidance on when and how to use this tool
}
```

The leak revealed that Claude Code's tool descriptions include explicit instructions about when NOT to use the tool and what alternatives to prefer. This is a critical pattern: guide the model away from risky usage, not just toward correct usage.

---

## 2. The Tool Execution Pipeline

### 2.1 End-to-End Flow

When the model emits a tool_use block, here is the complete execution pipeline:

```
Model emits: tool_use(name="Bash", input={command: "npm test"})
                │
                ▼
┌─────────────────────────────────┐
│  1. VALIDATION                  │
│  ├── Is this a known tool?      │
│  ├── Does the input match       │
│  │   the schema?                │
│  └── Are required fields        │
│      present?                   │
└────────────┬────────────────────┘
             │ valid
             ▼
┌─────────────────────────────────┐
│  2. PERMISSION CHECK            │
│  ├── Check deny list first      │
│  │   (immediate reject)         │
│  ├── Check allow list           │
│  │   (auto-approve if match)    │
│  ├── Check risk level           │
│  │   (low=auto, high=ask)       │
│  └── Check mode                 │
│      (auto mode overrides)      │
└────────────┬────────────────────┘
             │ approved
             ▼
┌─────────────────────────────────┐
│  3. PRE-EXECUTION HOOKS         │
│  ├── PreToolUse hooks run       │
│  ├── Plugins can intercept      │
│  ├── Can modify input           │
│  └── Can block execution        │
└────────────┬────────────────────┘
             │ not blocked
             ▼
┌─────────────────────────────────┐
│  4. EXECUTION                   │
│  ├── Run the tool function      │
│  ├── Apply timeout              │
│  ├── Capture stdout/stderr      │
│  └── Handle errors              │
└────────────┬────────────────────┘
             │ result
             ▼
┌─────────────────────────────────┐
│  5. POST-EXECUTION HOOKS        │
│  ├── PostToolUse hooks run      │
│  ├── Plugins can inspect result │
│  ├── Can modify result          │
│  └── Can trigger side effects   │
└────────────┬────────────────────┘
             │ processed
             ▼
┌─────────────────────────────────┐
│  6. RESULT FORMATTING           │
│  ├── Truncate if too large      │
│  ├── Format for context         │
│  └── Add to message history     │
└─────────────────────────────────┘
```

### 2.2 Implementation

```typescript
// The complete tool execution pipeline
class ToolExecutor {
  private tools: Map<string, RegisteredTool>;
  private permissions: PermissionEngine;
  private hooks: HookRunner;
  
  async execute(
    toolCall: ToolUseBlock,
    session: Session
  ): Promise<ToolResult> {
    // 1. Validation
    const tool = this.tools.get(toolCall.name);
    if (!tool) {
      return {
        tool_use_id: toolCall.id,
        content: `Unknown tool: ${toolCall.name}`,
        is_error: true
      };
    }
    
    const validation = this.validateInput(
      toolCall.input, tool.inputSchema
    );
    if (!validation.valid) {
      return {
        tool_use_id: toolCall.id,
        content: `Invalid input: ${validation.error}`,
        is_error: true
      };
    }
    
    // 2. Permission check
    const permission = await this.permissions.check(
      toolCall, session
    );
    
    if (permission.denied) {
      return {
        tool_use_id: toolCall.id,
        content: permission.reason,
        is_error: true
      };
    }
    
    if (permission.requiresApproval) {
      const approved = await session.ui.promptApproval(
        toolCall, permission.explanation
      );
      if (!approved) {
        return {
          tool_use_id: toolCall.id,
          content: 'User denied permission.',
          is_error: true
        };
      }
    }
    
    // 3. Pre-execution hooks
    const preResult = await this.hooks.runPreToolUse(
      toolCall, session
    );
    if (preResult.blocked) {
      return {
        tool_use_id: toolCall.id,
        content: preResult.reason,
        is_error: true
      };
    }
    const input = preResult.modifiedInput || toolCall.input;
    
    // 4. Execution with timeout
    let rawResult: string;
    try {
      rawResult = await withTimeout(
        tool.execute(input, session),
        tool.timeout
      );
    } catch (error) {
      rawResult = `Error: ${error.message}`;
    }
    
    // 5. Post-execution hooks
    const postResult = await this.hooks.runPostToolUse(
      toolCall, rawResult, session
    );
    const processedResult = postResult.modifiedResult || rawResult;
    
    // 6. Result formatting
    const formatted = this.formatResult(
      processedResult, tool.maxResultSize
    );
    
    return {
      tool_use_id: toolCall.id,
      content: formatted
    };
  }
  
  private formatResult(
    result: string,
    maxSize: number
  ): string {
    if (result.length <= maxSize) return result;
    
    // Truncate intelligently
    const truncated = result.slice(0, maxSize);
    const lastNewline = truncated.lastIndexOf('\n');
    
    return truncated.slice(0, lastNewline > 0 ? lastNewline : maxSize)
      + `\n\n[...truncated. Full result was ${result.length} chars]`;
  }
}
```

---

## 3. The Permission Model

### 3.1 Allow Lists and Deny Lists

Claude Code uses a dual-list permission system defined in settings.json:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "WebSearch",
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Edit(src/**)",
      "Write(src/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push *)",
      "Bash(curl * | bash)",
      "Write(.env*)",
      "Write(*.pem)",
      "Write(*.key)"
    ]
  }
}
```

### 3.2 How Matching Works

The permission system uses pattern matching to determine if a tool call matches an allow or deny entry:

```typescript
// Permission matching (reconstructed)
class PermissionMatcher {
  matches(
    toolCall: ToolUseBlock,
    pattern: string
  ): boolean {
    // Pattern format: "ToolName" or "ToolName(argument_pattern)"
    const parsed = this.parsePattern(pattern);
    
    // Check tool name
    if (parsed.toolName !== toolCall.name) return false;
    
    // If no argument pattern, match on name alone
    if (!parsed.argPattern) return true;
    
    // Match argument pattern against tool input
    return this.matchArgument(
      toolCall.input,
      parsed.argPattern
    );
  }
  
  private matchArgument(
    input: Record<string, unknown>,
    pattern: string
  ): boolean {
    // For Bash: match against the command string
    if ('command' in input) {
      return this.globMatch(input.command as string, pattern);
    }
    
    // For file tools: match against the file path
    if ('file_path' in input) {
      return this.globMatch(input.file_path as string, pattern);
    }
    
    // For other tools: match against the serialized input
    return this.globMatch(JSON.stringify(input), pattern);
  }
  
  private globMatch(value: string, pattern: string): boolean {
    // Convert glob pattern to regex
    // * matches any characters except path separators
    // ** matches any characters including path separators
    const regex = pattern
      .replace(/\*\*/g, '<<<GLOBSTAR>>>')
      .replace(/\*/g, '[^/]*')
      .replace(/<<<GLOBSTAR>>>/g, '.*');
    
    return new RegExp(`^${regex}$`).test(value);
  }
}
```

### 3.3 The Evaluation Order

The order in which permissions are evaluated matters:

```
┌──────────────────────────────────────────────────┐
│          PERMISSION EVALUATION ORDER              │
│                                                    │
│  1. Check DENY list first                          │
│     If matched -> DENY (no override possible)      │
│                                                    │
│  2. Check ALLOW list                               │
│     If matched -> AUTO-APPROVE                     │
│                                                    │
│  3. Check tool's default risk level                │
│     Low risk  -> AUTO-APPROVE                      │
│     Medium    -> depends on mode                   │
│     High      -> REQUIRE APPROVAL                  │
│                                                    │
│  4. Check session mode                             │
│     Default mode  -> follow risk levels            │
│     Auto mode     -> approve medium risk too       │
│     Skip-perms    -> approve everything            │
│                                                    │
│  Result: DENY | AUTO-APPROVE | REQUIRE-APPROVAL    │
└──────────────────────────────────────────────────┘
```

**Deny always wins.** If a tool call matches both the allow list and the deny list, the deny list takes precedence. This is a security-first design -- you can never accidentally allow something you explicitly denied.

### 3.4 Risk Tier Classification

```typescript
// Risk classification for built-in tools
const TOOL_RISK_LEVELS: Record<string, RiskLevel> = {
  // LOW RISK: Read-only operations
  // These cannot modify the system state
  Read: 'low',
  Glob: 'low',
  Grep: 'low',
  WebSearch: 'low',
  
  // MEDIUM RISK: Write operations with limited scope
  // These modify files but are bounded
  Write: 'medium',
  Edit: 'medium',
  WebFetch: 'medium',    // Can access URLs
  Agent: 'medium',       // Spawns sub-agents
  NotebookEdit: 'medium',
  
  // HIGH RISK: Unbounded execution
  // These can do essentially anything
  Bash: 'high',
};
```

The risk classification reflects a simple principle: **read is safe, write is moderate, execute is dangerous.** This maps well to filesystem permissions (r/w/x) and is intuitive for users.

---

## 4. Auto Mode and dangerously-skip-permissions

### 4.1 The Three Permission Modes

```
┌───────────────────────────────────────────────────────────┐
│                 PERMISSION MODES                           │
│                                                            │
│  DEFAULT MODE                                              │
│  ├── Low risk tools: auto-approved                         │
│  ├── Medium risk tools: requires approval                  │
│  ├── High risk tools: requires approval                    │
│  └── Best for: new users, sensitive projects               │
│                                                            │
│  AUTO MODE (--auto or accept-all in session)               │
│  ├── Low risk tools: auto-approved                         │
│  ├── Medium risk tools: auto-approved                      │
│  ├── High risk tools: still requires approval (Bash only)  │
│  └── Best for: experienced users who trust the agent       │
│                                                            │
│  DANGEROUSLY SKIP PERMISSIONS (--dangerously-skip-perms)   │
│  ├── All tools: auto-approved (no prompts at all)          │
│  ├── Even Bash: auto-approved                              │
│  ├── Even denied tools: still denied (deny list is sacred) │
│  └── Best for: CI/CD, automation, headless operation       │
│                                                            │
│  Note: deny list ALWAYS applies, even in skip mode         │
└───────────────────────────────────────────────────────────┘
```

### 4.2 Auto Mode in Practice

Auto mode is the sweet spot for most experienced users:

```typescript
// Auto mode permission check
function checkAutoMode(
  toolCall: ToolUseBlock,
  toolRisk: RiskLevel,
  denyList: string[],
  allowList: string[]
): PermissionDecision {
  // Deny list always checked first
  if (matchesDenyList(toolCall, denyList)) {
    return { type: 'deny', reason: 'Blocked by deny list' };
  }
  
  // Allow list grants auto-approval
  if (matchesAllowList(toolCall, allowList)) {
    return { type: 'auto_approve' };
  }
  
  // In auto mode: low and medium auto-approve
  if (toolRisk === 'low' || toolRisk === 'medium') {
    return { type: 'auto_approve' };
  }
  
  // High risk still requires approval in auto mode
  // (mainly Bash commands not in the allow list)
  return {
    type: 'require_approval',
    explanation: `High-risk operation: ${toolCall.name}`
  };
}
```

### 4.3 The Skip-Permissions Danger

`--dangerously-skip-permissions` is named deliberately to discourage casual use:

```
WARNING: Running with --dangerously-skip-permissions means:
  - Bash commands execute without any confirmation
  - File writes happen without review
  - The agent can modify any file in the project
  - The agent can run any command your user can run
  - The agent can install packages, modify configs, etc.

USE ONLY WHEN:
  - Running in CI/CD (no human to approve)
  - Running in a container/sandbox
  - The task is well-defined and tested
  - You have version control to roll back changes
```

Even in skip mode, the deny list still applies. This is the last line of defense:

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf /)",
      "Bash(:(){ :|:& };:)",
      "Write(/etc/*)",
      "Bash(dd if=*)"
    ]
  }
}
```

---

## 5. How Cursor Handles Tool Execution Differently

### 5.1 The Accept/Reject Model

Cursor uses a different permission philosophy. Instead of pre-approving or blocking tool calls, it presents the result and lets the user accept or reject it:

```
┌───────────────────────────────────────────────────────┐
│              CURSOR PERMISSION MODEL                   │
│                                                        │
│  Claude Code: "Can I edit this file?"                  │
│              -> User approves -> Agent edits            │
│                                                        │
│  Cursor:     Agent edits -> shows diff -> "Accept?"    │
│              -> User accepts -> changes are kept        │
│              -> User rejects -> changes are reverted    │
│                                                        │
│  Key difference: Cursor executes optimistically,       │
│  Claude Code executes pessimistically.                 │
│                                                        │
│  Cursor is faster (no approval delay) but riskier      │
│  (changes are already applied before review).          │
│                                                        │
│  Claude Code is slower (waits for approval) but safer  │
│  (nothing happens until approved).                     │
└───────────────────────────────────────────────────────┘
```

### 5.2 Undo as Permission

Cursor's model treats the undo operation as the permission mechanism:

```typescript
// Cursor's approach (conceptual)
class CursorToolExecution {
  async executeEdit(edit: FileEdit): Promise<void> {
    // 1. Save the current state (for undo)
    const snapshot = await this.snapshot(edit.file);
    
    // 2. Apply the edit immediately
    await this.applyEdit(edit);
    
    // 3. Show the diff to the user
    this.showDiff(snapshot, edit);
    
    // 4. User can:
    //    - Accept (keep the changes)
    //    - Reject (revert to snapshot)
    //    - Modify (edit the result manually)
  }
  
  async revert(edit: FileEdit, snapshot: FileSnapshot): Promise<void> {
    await this.restoreSnapshot(snapshot);
  }
}
```

This is an architectural choice, not a security one. Cursor optimizes for speed (no approval delay), while Claude Code optimizes for safety (nothing happens without approval).

### 5.3 Composer Mode: Full Agent Permissions

In Composer mode, Cursor behaves more like Claude Code:

```
Cursor Chat:       Suggestions only (no direct execution)
Cursor Tab:        Inline completions (accept with Tab)
Cursor Composer:   Full agent with file edits, terminal access
                   (closer to Claude Code's permission model)
```

Composer can modify multiple files and run terminal commands, so it uses a more traditional permission model with user confirmation for risky operations.

---

## 6. Sandboxing Strategies

### 6.1 The Spectrum of Isolation

```
No isolation          Process isolation       Container isolation
     │                      │                         │
     ▼                      ▼                         ▼
┌─────────┐          ┌──────────────┐          ┌───────────┐
│ Agent    │          │ Agent runs   │          │ Agent runs│
│ runs as  │          │ with limited │          │ in Docker │
│ your     │          │ permissions  │          │ container │
│ user     │          │ (fs sandbox) │          │ (full     │
│          │          │              │          │  isolation)│
└─────────┘          └──────────────┘          └───────────┘
 Claude Code          Aider (partial)           Codex
 Cursor                                         Devin
```

### 6.2 Claude Code's Approach: Permission-Based

Claude Code runs as your user with your permissions. Its "sandboxing" is purely permission-based:

```typescript
// Claude Code: permission-based containment
const containment = {
  fileSystem: {
    // Can read any file your user can read
    read: 'full_user_access',
    // Can write files, but only with approval
    write: 'approval_required',
    // Deny list blocks specific paths
    blocked: ['~/.ssh/*', '~/.aws/*', '.env*'],
  },
  commands: {
    // Can run any command your user can run
    execution: 'full_user_access',
    // But requires approval for each command
    approval: 'per_command',
    // Deny list blocks specific patterns
    blocked: ['rm -rf /', 'curl * | bash'],
  },
  network: {
    // Can access the network (via WebSearch, WebFetch)
    access: 'full',
    // But only through the tool interface
    // (no raw network access from Bash without approval)
  },
};
```

**Trade-off:** Maximum capability, minimum isolation. The agent can do everything the user can do, but the permission system gates dangerous actions.

### 6.3 Codex's Approach: Container-Based

Codex runs in a Docker container with a clone of the repository:

```typescript
// Codex: container-based containment
const containment = {
  fileSystem: {
    // Can only access the cloned repository
    read: 'container_only',
    write: 'container_only',
    // Cannot access host filesystem
    host_access: 'none',
  },
  commands: {
    // Can run any command inside the container
    execution: 'unrestricted_in_container',
    // No approval needed (container IS the sandbox)
    approval: 'none',
  },
  network: {
    // Limited network access
    access: 'restricted',
    // Can access npm registry, GitHub API
    // Cannot access arbitrary URLs
    allowlist: ['registry.npmjs.org', 'api.github.com'],
  },
};
```

**Trade-off:** Maximum isolation, reduced capability. The agent cannot access your local state, but it also cannot interact with your running dev server, access local databases, or use local tools not in the container.

### 6.4 Hybrid: The Best of Both

The ideal approach combines permission-based and container-based containment:

```typescript
// Hybrid containment (recommended for production)
const hybridContainment = {
  // File operations: permission-based (for speed)
  files: {
    read: 'auto_approve_in_project',
    write: 'approval_required',
    outsideProject: 'deny',
  },
  
  // Command execution: container-based (for safety)
  commands: {
    safe: {
      // Known-safe commands run directly
      patterns: ['npm test', 'npm run lint', 'git status'],
      execution: 'direct',
    },
    unknown: {
      // Unknown commands run in a container
      execution: 'container',
      networkAccess: 'restricted',
    },
  },
};
```

---

## 7. The Frustration Detection Pattern

### 7.1 What It Is

The leak revealed that Claude Code includes a pattern detection system that identifies when the agent is struggling or the user is frustrated:

```typescript
// Frustration detection (reconstructed from the leak)
const FRUSTRATION_PATTERNS = [
  // User frustration signals
  /no,?\s*(that's|thats)\s*(not|wrong)/i,
  /you('re| are)\s*(not|still)\s*(getting|understanding)/i,
  /I\s*(already|just)\s*(told|said|asked)/i,
  /stop\s*(doing|trying)\s*that/i,
  /this\s*(is|isn't)\s*(not\s*)?working/i,
  /try\s*again/i,
  /wrong\s*(approach|direction|file|thing)/i,
  
  // Agent struggling signals (in its own output)
  /I\s*apologize.*let\s*me\s*try/i,
  /I('m| am)\s*(having\s*trouble|struggling)/i,
  /that\s*didn't\s*work.*let\s*me/i,
];

function detectFrustration(
  messages: Message[]
): FrustrationSignal | null {
  const recentMessages = messages.slice(-6);
  
  let frustrationScore = 0;
  
  for (const msg of recentMessages) {
    const text = extractText(msg);
    for (const pattern of FRUSTRATION_PATTERNS) {
      if (pattern.test(text)) {
        frustrationScore++;
      }
    }
  }
  
  // Also detect loops: agent repeating similar actions
  const recentToolCalls = extractToolCalls(recentMessages);
  const duplicateActions = findDuplicateActions(recentToolCalls);
  frustrationScore += duplicateActions.length;
  
  if (frustrationScore >= 3) {
    return {
      level: 'high',
      suggestion: 'Consider changing approach or asking '
        + 'the user for clarification',
    };
  }
  
  if (frustrationScore >= 1) {
    return {
      level: 'low',
      suggestion: 'Verify approach before continuing',
    };
  }
  
  return null;
}
```

### 7.2 How It Is Used

When frustration is detected, the system can inject guidance:

```typescript
// Frustration-aware agent loop
async function agentLoopWithFrustrationDetection(
  session: Session
): Promise<void> {
  // ... standard loop ...
  
  const frustration = detectFrustration(session.messages);
  
  if (frustration && frustration.level === 'high') {
    // Inject a system message to change behavior
    session.messages.push({
      role: 'user',
      content: [
        '[System note: The user appears frustrated or you',
        'appear to be stuck in a loop. Consider:',
        '1. Stepping back and re-reading the original request',
        '2. Trying a completely different approach',
        '3. Asking the user for clarification',
        '4. Explaining what you have tried and what failed]',
      ].join(' ')
    });
  }
  
  // Continue with the model call...
}
```

### 7.3 Why This Matters

Frustration detection is an example of the harness making the model smarter by providing meta-awareness it does not naturally have. The model cannot tell it is in a loop (it does not remember previous turns once compacted). The harness can detect this and inject corrective guidance.

**Lesson for your own tools:** Monitor patterns in agent behavior. If the agent is repeating actions, failing repeatedly, or generating apologies, inject guidance to break the pattern.

---

## 8. Adversarial Verification

### 8.1 The Pattern

Adversarial verification is a pattern where the agent double-checks its own work by examining the result from a critical perspective:

```typescript
// Adversarial verification pattern
async function verifiedEdit(
  agent: Agent,
  file: string,
  instruction: string
): Promise<EditResult> {
  // Step 1: Generate the edit
  const edit = await agent.generateEdit(file, instruction);
  
  // Step 2: Apply the edit to a temporary copy
  const tempFile = await createTempCopy(file);
  await applyEdit(tempFile, edit);
  
  // Step 3: Ask the agent to review its own edit
  const review = await agent.review({
    original: await readFile(file),
    modified: await readFile(tempFile),
    instruction: instruction,
    prompt: `Review this edit critically. Does it:
1. Correctly implement the requested change?
2. Introduce any bugs?
3. Break any existing functionality?
4. Follow the project's coding conventions?

If you find issues, describe them. If the edit is correct,
say "APPROVED".`
  });
  
  if (review.includes('APPROVED')) {
    // Apply the verified edit
    await applyEdit(file, edit);
    return { success: true, verified: true };
  }
  
  // Step 4: If issues found, regenerate
  const revisedEdit = await agent.generateEdit(
    file,
    `${instruction}\n\nPrevious attempt had issues: ${review}`
  );
  
  return { success: true, revised: true, edit: revisedEdit };
}
```

### 8.2 When to Use Adversarial Verification

Not every tool call needs adversarial verification. Use it for:

```
┌──────────────────────────────────────────────────┐
│       WHEN TO USE ADVERSARIAL VERIFICATION        │
│                                                    │
│  USE:                                              │
│  ├── Complex multi-file edits                      │
│  ├── Security-sensitive changes                    │
│  ├── Database migrations                           │
│  ├── Configuration changes                         │
│  └── Any change that is hard to undo               │
│                                                    │
│  SKIP:                                             │
│  ├── Read-only operations                          │
│  ├── Simple single-line edits                      │
│  ├── Test execution (CI verifies)                  │
│  └── Information gathering                         │
│                                                    │
│  COST: 2x model calls per verified operation       │
│  BENEFIT: Catches errors before they affect code   │
└──────────────────────────────────────────────────┘
```

---

## 9. Designing a Permission System for Your Own Agents

### 9.1 The Design Checklist

```
┌──────────────────────────────────────────────────────┐
│        PERMISSION SYSTEM DESIGN CHECKLIST             │
│                                                        │
│  [ ] Define risk levels for every tool                 │
│  [ ] Implement deny list (always checked first)        │
│  [ ] Implement allow list (auto-approve matches)       │
│  [ ] Support pattern matching (glob patterns)          │
│  [ ] Provide multiple modes (default, auto, skip)      │
│  [ ] Make the deny list immutable in skip mode         │
│  [ ] Log every permission decision for audit           │
│  [ ] Show clear UI for approval requests               │
│  [ ] Allow users to add to allow list from UI          │
│  [ ] Support per-project settings                      │
│  [ ] Timeout approval requests (don't hang forever)    │
│  [ ] Handle the "always allow" pattern for a session   │
└──────────────────────────────────────────────────────┘
```

### 9.2 A Reusable Permission Engine

```typescript
// permission-engine.ts
// A reusable permission system for AI agents

interface PermissionConfig {
  allow: string[];
  deny: string[];
  mode: 'default' | 'auto' | 'skip';
  riskLevels: Record<string, 'low' | 'medium' | 'high'>;
  sessionAllowList: Set<string>;  // Temporary allows for this session
}

interface PermissionDecision {
  action: 'approve' | 'deny' | 'ask';
  reason: string;
  riskLevel: 'low' | 'medium' | 'high';
}

class PermissionEngine {
  private config: PermissionConfig;
  private matcher: PatternMatcher;
  private auditLog: AuditEntry[];
  
  constructor(config: PermissionConfig) {
    this.config = config;
    this.matcher = new PatternMatcher();
    this.auditLog = [];
  }
  
  check(toolCall: ToolUseBlock): PermissionDecision {
    const toolName = toolCall.name;
    const input = toolCall.input;
    
    // Step 1: Deny list (always first, always final)
    for (const pattern of this.config.deny) {
      if (this.matcher.matches(toolName, input, pattern)) {
        const decision: PermissionDecision = {
          action: 'deny',
          reason: `Blocked by deny rule: ${pattern}`,
          riskLevel: 'high',
        };
        this.audit(toolCall, decision);
        return decision;
      }
    }
    
    // Step 2: Allow list
    for (const pattern of this.config.allow) {
      if (this.matcher.matches(toolName, input, pattern)) {
        const decision: PermissionDecision = {
          action: 'approve',
          reason: `Matched allow rule: ${pattern}`,
          riskLevel: this.getRiskLevel(toolName),
        };
        this.audit(toolCall, decision);
        return decision;
      }
    }
    
    // Step 3: Session allow list (temporary)
    const key = this.getSessionKey(toolCall);
    if (this.config.sessionAllowList.has(key)) {
      const decision: PermissionDecision = {
        action: 'approve',
        reason: 'Allowed for this session',
        riskLevel: this.getRiskLevel(toolName),
      };
      this.audit(toolCall, decision);
      return decision;
    }
    
    // Step 4: Risk level + mode
    const risk = this.getRiskLevel(toolName);
    
    if (this.config.mode === 'skip') {
      return this.approved(toolCall, 'Skip mode');
    }
    
    if (risk === 'low') {
      return this.approved(toolCall, 'Low risk');
    }
    
    if (risk === 'medium' && this.config.mode === 'auto') {
      return this.approved(toolCall, 'Medium risk, auto mode');
    }
    
    // Requires user approval
    const decision: PermissionDecision = {
      action: 'ask',
      reason: `${risk} risk operation requires approval`,
      riskLevel: risk,
    };
    this.audit(toolCall, decision);
    return decision;
  }
  
  // Allow a pattern for the rest of this session
  sessionAllow(pattern: string): void {
    this.config.sessionAllowList.add(pattern);
  }
  
  private getRiskLevel(
    toolName: string
  ): 'low' | 'medium' | 'high' {
    return this.config.riskLevels[toolName] || 'high';
  }
  
  private approved(
    toolCall: ToolUseBlock,
    reason: string
  ): PermissionDecision {
    const decision: PermissionDecision = {
      action: 'approve',
      reason,
      riskLevel: this.getRiskLevel(toolCall.name),
    };
    this.audit(toolCall, decision);
    return decision;
  }
  
  private audit(
    toolCall: ToolUseBlock,
    decision: PermissionDecision
  ): void {
    this.auditLog.push({
      timestamp: new Date(),
      tool: toolCall.name,
      input: toolCall.input,
      decision: decision.action,
      reason: decision.reason,
    });
  }
}
```

---

## 10. The Trust Spectrum in Production

### 10.1 Trust as an Engineering Variable

Trust is not binary (trust/don't trust). It is a spectrum that should increase over time as the system proves itself:

```
Day 1: Default mode
  - Every file write requires approval
  - Every command requires approval
  - User reviews everything

Week 2: Curated allow list
  - Common test commands auto-approved
  - Source file edits auto-approved
  - Config files still require approval

Month 2: Auto mode
  - Most operations auto-approved
  - Only bash commands require approval
  - User reviews important changes

Month 6: Full auto with monitoring
  - Nearly everything auto-approved
  - Audit log reviewed weekly
  - Alerts on anomalous behavior
```

### 10.2 Building Trust Through Transparency

The single most important factor in building user trust: **show your work.**

```
Low trust:  "I updated 3 files."
Medium:     "I updated auth.ts, middleware.ts, and test.ts."
High:       Shows diffs of each change, explains why.
Highest:    Shows diffs, runs tests, shows test output.
```

Claude Code's diff rendering is a trust mechanism. Cursor's inline diff is a trust mechanism. Both tools invest heavily in showing exactly what changed because transparency is the foundation of trust, and trust is the foundation of autonomy.

---

## 11. Key Takeaways

1. **The tool execution pipeline has six stages:** validation, permission check, pre-hooks, execution, post-hooks, result formatting. Each stage can block or modify the tool call.

2. **Permission evaluation order matters:** deny list first (always), then allow list, then risk level, then mode. Deny always wins.

3. **Three risk tiers work in practice:** low (auto-approve), medium (mode-dependent), high (always ask). Map them to read/write/execute.

4. **Cursor uses optimistic execution** (do first, undo if rejected). Claude Code uses pessimistic execution (ask first, then do). Both have trade-offs.

5. **Sandboxing can be permission-based** (Claude Code) or container-based (Codex) or hybrid. Choose based on your trust and isolation requirements.

6. **Frustration detection** helps the agent break out of loops. Monitor for repeated actions and user frustration signals.

7. **Adversarial verification** (the agent reviews its own work) catches errors for high-risk operations. Cost: 2x model calls. Benefit: fewer bugs in critical changes.

8. **Trust is a spectrum** that should increase over time. Start restrictive, observe behavior, gradually expand permissions.

9. **Transparency builds trust.** Show diffs, show reasoning, show test results. The more the user sees, the more they trust.

---

*Previous: [Chapter 29 -- Context Window Internals](./29-context-internals.md)* · *Next: [Chapter 31 -- Web Search & Knowledge Pipelines](./31-web-search-pipelines.md)* reveals how the harness acquires knowledge from the web -- and the surprising limitations of how it does so.

<!--
  CHAPTER: 33
  TITLE: Skills, Plugins & Distribution
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 23 (Claude Code Mastery), Ch 25 (Skills, Plugins & Automation), Ch 27 (Harness Architecture)
  KEY_TOPICS: skill loading, Skillify, plugin architecture, hooks, MCP bundling, anti-distillation, fake_tools, organizational distribution, undercover mode, description optimization
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 33: Skills, Plugins & Distribution

> Part 6: Anatomy of AI Developer Tools · Phase 2 · Prerequisites: Ch 23, Ch 25, Ch 27 · Advanced · TypeScript + Architecture

Skills are prompts. This is the most important sentence in this chapter. Despite the sophisticated loading mechanisms, frontmatter metadata, and distribution systems, a skill is ultimately a block of markdown text that gets injected into the model's context. Understanding this -- and understanding the engineering systems built around this simple idea -- is the difference between writing skills that sometimes work and writing skills that consistently guide agent behavior.

The leak revealed Skillify (automatic skill generation from session analysis), an anti-distillation system using fake tools to protect intellectual property, a hook architecture (PreToolUse, PostToolUse, Stop) that enables plugins to intercept agent behavior, and organizational distribution mechanisms that range from git-backed to admin-UI managed. It also revealed the "undercover mode" controversy -- the ability for AI to contribute to repositories without disclosure.

### In This Chapter

1. How skills actually load: the injection mechanics
2. CLAUDE.md is re-injected every turn (and the cost math)
3. Skill anatomy: frontmatter, progressive disclosure, descriptions
4. The Skillify pattern: auto-generating skills from sessions
5. Plugin architecture: hooks, MCP bundling, settings
6. Organizational distribution: git, admin UI, API
7. Anti-distillation: fake_tools for IP protection
8. The "undercover mode" controversy
9. How skill quality affects agent performance
10. Designing skills and plugins for your own systems

### Related Chapters

- **Ch 4 (Prompt Engineering)** -- skills ARE prompts; this chapter shows their architecture
- **Ch 23-25 (Using skills)** -- taught you to use skills; this chapter teaches their internals
- **Ch 27 (Harness Architecture)** -- showed where skills fit in the system prompt
- **Ch 60 (Skills Marketplaces)** -- will build distribution systems at scale

---

## 1. How Skills Actually Load

### 1.1 The Loading Pipeline

When Claude Code activates a skill, here is what happens:

```
┌───────────────────────────────────────────────────────────┐
│                  SKILL LOADING PIPELINE                    │
│                                                            │
│  1. DISCOVERY                                              │
│     ├── Scan skill directories                             │
│     │   ├── ~/.claude/skills/           (user skills)     │
│     │   ├── .claude/skills/             (project skills)  │
│     │   └── Plugin-provided skills      (plugin skills)   │
│     └── Catalog all available skills with metadata         │
│                                                            │
│  2. SELECTION                                              │
│     ├── User explicitly invokes (/skill-name)              │
│     ├── Auto-selected based on task context                │
│     └── Always-on skills (configured in settings)          │
│                                                            │
│  3. PARSING                                                │
│     ├── Read the markdown file                             │
│     ├── Parse YAML frontmatter (metadata)                  │
│     └── Extract the body (the actual skill content)        │
│                                                            │
│  4. INJECTION                                              │
│     ├── Add to system prompt as tagged block               │
│     │   <skill name="skill-name">                         │
│     │   [skill content]                                    │
│     │   </skill>                                          │
│     ├── Included on THIS turn and subsequent turns         │
│     └── Counts toward token budget                         │
│                                                            │
│  5. DEACTIVATION                                           │
│     ├── User explicitly deactivates                        │
│     ├── Session ends                                       │
│     └── Skill's scope condition no longer met              │
└───────────────────────────────────────────────────────────┘
```

### 1.2 The File Format

A skill file is a markdown file with YAML frontmatter:

```markdown
---
name: code-review
description: Reviews code changes for quality, security, and correctness
triggers:
  - "review"
  - "check my code"
  - "code review"
tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Code Review Skill

When asked to review code, follow this process:

## Step 1: Understand the Change
- Read the files mentioned or use `git diff` to see changes
- Understand the purpose of the change from context

## Step 2: Check for Issues
Review for:
- **Bugs**: Logic errors, off-by-one, null handling
- **Security**: Input validation, auth checks, injection risks
- **Performance**: N+1 queries, unnecessary re-renders, memory leaks
- **Readability**: Naming, complexity, documentation
- **Tests**: Are changes covered by tests?

## Step 3: Report
Format your review as:
```
## Code Review: [brief description]

### Issues Found
- [severity] [file:line] Description

### Suggestions
- [file:line] Suggestion

### Summary
[1-2 sentences overall assessment]
```

Be specific. Reference exact lines. Suggest fixes, don't just point out problems.
```

### 1.3 What the Model Sees

After injection, the model's system prompt includes:

```
[... base system prompt ...]
[... CLAUDE.md ...]

<skill name="code-review">
# Code Review Skill

When asked to review code, follow this process:
[... rest of skill content ...]
</skill>

[... other skills ...]
```

The model sees this as part of its instructions. It does not know the content came from a skill file -- it just sees text in its system prompt. This is why skills ARE prompts. The entire skill loading system is an elaborate mechanism for getting the right text into the system prompt at the right time.

---

## 2. CLAUDE.md Is Re-Injected Every Turn

### 2.1 The Re-Injection Mechanic

We covered this in Ch 28, but it bears repeating because of its impact on skill design:

```typescript
// On EVERY model call, the system prompt is rebuilt
function buildSystemPrompt(config: Config): string {
  const parts = [
    BASE_SYSTEM_PROMPT,           // ~2000 tokens (cached)
    formatTools(config.tools),    // ~1500 tokens (cached)
    config.claudeMd,              // Variable (cached if unchanged)
    ...config.activeSkills.map(  // Variable (cached if unchanged)
      s => `<skill name="${s.name}">\n${s.content}\n</skill>`
    ),
  ];
  
  return parts.join('\n\n');
}
```

### 2.2 The Cost Math

```
Scenario: You have CLAUDE.md (500 tokens) and 2 active skills
(1000 tokens each).

Fixed cost per turn:
  Base system prompt:  2,000 tokens
  Tool definitions:    1,500 tokens
  CLAUDE.md:             500 tokens
  Skill 1:             1,000 tokens
  Skill 2:             1,000 tokens
  ─────────────────────────────
  Total:               6,000 tokens per turn

Over 100 turns (typical session):
  Total fixed tokens: 600,000 tokens

With prompt caching (90% discount, ~90% hit rate):
  Turns 2-100: 6,000 * 0.1 * 99 = 59,400 tokens at full price
  Turn 1: 6,000 at full price
  Effective cost: 65,400 tokens at full price
  
Without prompt caching (cache broken):
  All 100 turns: 600,000 tokens at full price
  
Difference: ~9x cost increase when cache breaks

LESSON: Skills that are concise AND stable (don't change)
are dramatically cheaper than skills that are verbose or
frequently updated.
```

### 2.3 The "Short = Cheaper" Principle

Every token in CLAUDE.md or active skills costs money on every turn. This creates a direct financial incentive to keep them short:

```
CLAUDE.md LENGTH vs. SESSION COST (Sonnet, 100 turns, cached)

  100 tokens:   ~$0.03 per session
  500 tokens:   ~$0.15 per session
  1000 tokens:  ~$0.30 per session
  2000 tokens:  ~$0.60 per session
  5000 tokens:  ~$1.50 per session

  100 tokens:   ~$0.30 per session  (if cache breaks)
  500 tokens:   ~$1.50 per session  (if cache breaks)
  1000 tokens:  ~$3.00 per session  (if cache breaks)
  2000 tokens:  ~$6.00 per session  (if cache breaks)
  5000 tokens:  ~$15.00 per session (if cache breaks)

The difference between a 500-token and 5000-token CLAUDE.md
is ~$1.35/session cached, ~$13.50/session uncached.
Over 100 sessions/month: $135 - $1,350 difference.
```

---

## 3. Skill Anatomy: Frontmatter, Progressive Disclosure, Descriptions

### 3.1 Frontmatter Metadata

The frontmatter tells the loading system how to handle the skill:

```yaml
---
# Identity
name: database-migration        # Unique identifier
description: |                   # For skill discovery and selection
  Guides safe database migration creation,
  including rollback planning and data validation.

# Activation
triggers:                        # When to suggest this skill
  - "migration"
  - "schema change"
  - "alter table"
  - "database"
auto_activate: false             # Don't load unless triggered

# Tool access
tools:                           # Tools this skill needs
  - Read
  - Write
  - Edit
  - Bash

# Scope
scope:                           # When this skill is relevant
  file_patterns:
    - "**/*migration*"
    - "**/schema*"
    - "**/drizzle*"
  directories:
    - "src/db/"
    - "migrations/"
---
```

### 3.2 Progressive Disclosure

Well-designed skills use progressive disclosure: the most common information is first, the most detailed information is last.

```markdown
---
name: api-design
description: API route design patterns and conventions
---

# API Design Skill

## Quick Reference (always read this)
- REST conventions: GET=read, POST=create, PUT=update, DELETE=delete
- Response format: `{ data: T, error?: string }`
- Status codes: 200=OK, 201=created, 400=bad input, 404=not found
- Auth: Bearer token in Authorization header

## Patterns (read when implementing)
### Pagination
```typescript
interface PaginatedResponse<T> {
  data: T[];
  cursor: string | null;
  hasMore: boolean;
}
```

### Error Handling
```typescript
function apiError(status: number, message: string) {
  return Response.json({ error: message }, { status });
}
```

## Advanced (read when debugging or optimizing)
### Rate Limiting
[detailed implementation...]

### Caching Headers
[detailed implementation...]

### Versioning Strategy
[detailed implementation...]
```

The model reads the skill from top to bottom. By putting the most useful information first, you ensure the model sees it even if the skill is long. You also enable a future optimization: truncating skills from the bottom when context is tight.

### 3.3 Description Optimization

The skill description is the most important piece of metadata because it determines when the skill is selected and how the model understands it:

```yaml
# BAD: Too vague
description: "Helps with testing"

# BAD: Too long (wastes tokens in the skill catalog)
description: |
  This skill provides comprehensive guidance for writing
  unit tests, integration tests, end-to-end tests,
  snapshot tests, and performance tests using Vitest,
  Playwright, and custom test harnesses. It covers
  test setup, mocking, assertions, coverage, and CI
  integration. Use when writing or debugging tests.

# GOOD: Specific, action-oriented, right length
description: |
  Write and debug Vitest tests: unit, integration,
  and E2E. Covers mocking, assertions, and CI setup.
```

The description is used for:
1. **Skill matching:** when the user's request is matched against available skills
2. **LLM understanding:** the model reads the description to understand the skill's purpose
3. **Catalog display:** users see descriptions when browsing available skills

---

## 4. The Skillify Pattern: Auto-Generating Skills from Sessions

### 4.1 What Skillify Does

Skillify is a system that analyzes completed sessions and automatically generates skills from patterns it finds:

```
┌───────────────────────────────────────────────────────┐
│                    SKILLIFY PIPELINE                    │
│                                                        │
│  1. SESSION ANALYSIS                                   │
│     Review completed session logs                      │
│     Identify repeated patterns:                        │
│     ├── Same sequence of tool calls                    │
│     ├── Same types of files being read                 │
│     ├── Same commands being run                        │
│     ├── Same project structures encountered            │
│     └── Same questions asked repeatedly                │
│                                                        │
│  2. PATTERN EXTRACTION                                 │
│     For each pattern, extract:                         │
│     ├── The trigger (what the user asked)              │
│     ├── The process (what the agent did)               │
│     ├── The tools used                                 │
│     ├── The key decisions made                         │
│     └── The output format                              │
│                                                        │
│  3. SKILL GENERATION                                   │
│     Generate a skill file with:                        │
│     ├── Appropriate frontmatter                        │
│     ├── Clear step-by-step instructions                │
│     ├── Examples from actual sessions                  │
│     └── Tool requirements                              │
│                                                        │
│  4. QUALITY CHECK                                      │
│     ├── Is the skill general enough to be useful?      │
│     ├── Is it specific enough to be helpful?           │
│     ├── Does it duplicate an existing skill?           │
│     └── Is it short enough to not waste tokens?        │
│                                                        │
│  5. PROPOSAL                                           │
│     Suggest the skill to the user for review           │
│     User can: accept, modify, or reject                │
└───────────────────────────────────────────────────────┘
```

### 4.2 Pattern Detection

```typescript
// Skillify pattern detection (conceptual)
interface SessionPattern {
  trigger: string;         // What the user asked
  toolSequence: string[];  // Sequence of tools used
  filePatterns: string[];  // Types of files accessed
  frequency: number;       // How often this pattern appears
  sessions: string[];      // Which sessions it appeared in
}

async function detectPatterns(
  sessions: SessionLog[]
): Promise<SessionPattern[]> {
  const patterns: Map<string, SessionPattern> = new Map();
  
  for (const session of sessions) {
    // Extract tool call sequences
    const sequences = extractToolSequences(session);
    
    for (const seq of sequences) {
      // Create a signature for this sequence
      const signature = createSignature(seq);
      
      if (patterns.has(signature)) {
        // Existing pattern -- increment frequency
        const pattern = patterns.get(signature)!;
        pattern.frequency++;
        pattern.sessions.push(session.id);
      } else {
        // New pattern
        patterns.set(signature, {
          trigger: inferTrigger(seq),
          toolSequence: seq.tools,
          filePatterns: seq.filePatterns,
          frequency: 1,
          sessions: [session.id],
        });
      }
    }
  }
  
  // Return patterns that appear frequently enough
  return Array.from(patterns.values())
    .filter(p => p.frequency >= 3)  // At least 3 occurrences
    .sort((a, b) => b.frequency - a.frequency);
}
```

### 4.3 Skill Generation

```typescript
// Generate a skill from a detected pattern
async function generateSkill(
  pattern: SessionPattern
): Promise<SkillDraft> {
  const prompt = `
You are a skill-writing assistant. Based on the following
pattern detected across multiple sessions, generate a
reusable skill file.

Pattern:
- Trigger: ${pattern.trigger}
- Tool sequence: ${pattern.toolSequence.join(' -> ')}
- File patterns: ${pattern.filePatterns.join(', ')}
- Frequency: ${pattern.frequency} occurrences

Generate a markdown skill file with:
1. YAML frontmatter (name, description, triggers, tools)
2. Clear step-by-step instructions
3. The process should be generalizable (not specific to
   one project)
4. Include examples where helpful
5. Keep total length under 800 tokens

Output the complete skill file:
`;
  
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    messages: [{ role: 'user', content: prompt }],
    max_tokens: 2048,
  });
  
  return {
    content: extractText(response),
    pattern,
    confidence: calculateConfidence(pattern),
  };
}
```

### 4.4 The Skillify Flywheel

Skillify creates a self-reinforcing loop:

```
More sessions -> More patterns detected
     ↑                    │
     │                    ▼
Better agent     Skills generated
performance            │
     ↑                    ▼
     │            Agent uses skills
     │                    │
     └────────────────────┘
```

Each skill makes the agent better at its task, which produces cleaner sessions, which produce better patterns, which produce better skills.

---

## 5. Plugin Architecture: Hooks, MCP Bundling, Settings

### 5.1 The Plugin System

Plugins extend Claude Code's functionality beyond what skills can do. While skills are just prompts (text), plugins can run code:

```
┌───────────────────────────────────────────────────────┐
│              PLUGIN ARCHITECTURE                       │
│                                                        │
│  SKILLS (passive):                                     │
│  ├── Markdown files injected into context              │
│  ├── Guide the model's behavior through instructions   │
│  ├── Cannot execute code                               │
│  └── Cannot intercept tool calls                       │
│                                                        │
│  PLUGINS (active):                                     │
│  ├── Can include skills (passive guidance)             │
│  ├── Can register hooks (code execution)               │
│  ├── Can bundle MCP servers (additional tools)         │
│  ├── Can define settings (configuration)               │
│  └── Can modify agent behavior programmatically        │
└───────────────────────────────────────────────────────┘
```

### 5.2 The Hook System

Hooks allow plugins to intercept and modify agent behavior at key points:

```typescript
// Hook types (from the leak)
interface HookDefinition {
  // PreToolUse: runs BEFORE a tool is executed
  // Can modify input, block execution, or add context
  PreToolUse: {
    toolName: string | '*';  // Which tool(s) to intercept
    handler: (context: PreToolContext) => Promise<HookResult>;
  };
  
  // PostToolUse: runs AFTER a tool is executed
  // Can modify result, log, or trigger side effects
  PostToolUse: {
    toolName: string | '*';
    handler: (context: PostToolContext) => Promise<HookResult>;
  };
  
  // Stop: runs when the agent is about to stop
  // Can prevent stopping or add final actions
  Stop: {
    handler: (context: StopContext) => Promise<HookResult>;
  };
}
```

### 5.3 Hook Examples

```typescript
// Example: Security hook that blocks dangerous commands
const securityHook: HookDefinition = {
  PreToolUse: {
    toolName: 'Bash',
    handler: async (context) => {
      const command = context.input.command as string;
      
      // Check for dangerous patterns
      const dangerous = [
        /rm\s+-rf\s+\//,
        /curl.*\|\s*bash/,
        /eval\s*\(/,
        />\s*\/dev\/sd/,
      ];
      
      for (const pattern of dangerous) {
        if (pattern.test(command)) {
          return {
            action: 'block',
            message: `Blocked dangerous command: ${command}`
          };
        }
      }
      
      return { action: 'allow' };
    }
  }
};

// Example: Logging hook that records all tool calls
const loggingHook: HookDefinition = {
  PostToolUse: {
    toolName: '*',  // All tools
    handler: async (context) => {
      await appendToLog({
        tool: context.toolName,
        input: context.input,
        output: context.result,
        timestamp: new Date(),
        sessionId: context.sessionId,
      });
      
      return { action: 'continue' };
    }
  }
};

// Example: Auto-format hook that runs prettier after writes
const autoFormatHook: HookDefinition = {
  PostToolUse: {
    toolName: 'Write',
    handler: async (context) => {
      const filePath = context.input.file_path as string;
      
      // Check if the file is a formattable type
      if (/\.(ts|tsx|js|jsx|json|css|md)$/.test(filePath)) {
        // Run prettier on the written file
        await execAsync(
          `npx prettier --write "${filePath}"`,
          { cwd: context.projectRoot }
        );
      }
      
      return { action: 'continue' };
    }
  }
};
```

### 5.4 MCP Server Bundling

Plugins can bundle MCP servers, making additional tools available:

```json
// plugin.json
{
  "name": "my-database-plugin",
  "version": "1.0.0",
  "skills": ["./skills/"],
  "hooks": ["./hooks/"],
  "mcp": {
    "servers": {
      "database": {
        "command": "node",
        "args": ["./mcp-servers/database-server.js"],
        "env": {
          "DATABASE_URL": "${DATABASE_URL}"
        }
      }
    }
  },
  "settings": {
    "database_type": {
      "type": "string",
      "enum": ["postgres", "mysql", "sqlite"],
      "default": "postgres",
      "description": "Database type for migration generation"
    }
  }
}
```

When the plugin loads, the MCP server starts automatically and its tools become available to the agent.

### 5.5 Plugin Settings

Plugins can define configurable settings:

```typescript
// Plugin settings system
interface PluginSettings {
  // Defined in plugin.json
  schema: Record<string, SettingDefinition>;
  
  // Stored in .claude/settings.json
  values: Record<string, unknown>;
  
  // Read by the plugin at runtime
  get<T>(key: string, defaultValue?: T): T;
}

// Example usage in a hook
const hook: HookDefinition = {
  PreToolUse: {
    toolName: 'Bash',
    handler: async (context) => {
      const settings = context.plugin.settings;
      
      // Use a plugin setting
      const dbType = settings.get('database_type', 'postgres');
      
      // Modify behavior based on settings
      if (context.input.command.includes('migration')) {
        // Inject the database type into the command
        context.input.command = context.input.command
          .replace('DB_TYPE', dbType);
      }
      
      return { action: 'allow' };
    }
  }
};
```

---

## 6. Organizational Distribution

### 6.1 The Distribution Challenge

Organizations need to distribute skills and plugins across teams:

```
┌──────────────────────────────────────────────────────┐
│        DISTRIBUTION CHALLENGES                        │
│                                                        │
│  CONSISTENCY: All engineers should have the same       │
│  coding conventions, review checklists, and            │
│  deployment procedures encoded in skills.              │
│                                                        │
│  VERSIONING: Skills evolve. When you update the        │
│  deployment skill, everyone should get the update.     │
│                                                        │
│  CUSTOMIZATION: Different teams need different         │
│  skills. The frontend team doesn't need backend        │
│  deployment skills.                                    │
│                                                        │
│  ACCESS CONTROL: Some skills contain sensitive         │
│  information (internal APIs, auth patterns).           │
│                                                        │
│  DISCOVERY: Engineers need to find relevant skills     │
│  without browsing a catalog.                           │
└──────────────────────────────────────────────────────┘
```

### 6.2 Distribution Mechanisms

Three approaches, each with trade-offs:

**Approach 1: Git-Backed**

```
organization/
├── .claude/
│   ├── skills/
│   │   ├── code-review.md
│   │   ├── deployment.md
│   │   ├── security-review.md
│   │   └── team-conventions.md
│   ├── plugins/
│   │   └── internal-tools/
│   │       ├── plugin.json
│   │       └── hooks/
│   └── settings.json
└── CLAUDE.md
```

```
Pros:
  + Version controlled (git history)
  + Code review for skill changes
  + Works with existing git workflows
  + No additional infrastructure

Cons:
  - Must clone the repo to get skills
  - Skills tied to specific repositories
  - No cross-repo distribution without submodules
  - No centralized management
```

**Approach 2: Admin UI**

```
┌──────────────────────────────────────────────────┐
│  Organization Admin Dashboard                     │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Skills  │  │ Plugins  │  │ Settings │        │
│  │  Manager │  │  Manager │  │  Manager │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                    │
│  Upload/edit skills via web UI                     │
│  Assign skills to teams/projects                   │
│  Track skill usage and effectiveness               │
│  Push updates to all users instantly               │
└──────────────────────────────────────────────────┘
```

```
Pros:
  + Centralized management
  + Instant distribution
  + Team-level access control
  + Usage analytics

Cons:
  - Requires infrastructure
  - Not version-controlled (unless built in)
  - Separate from code review workflows
  - Vendor dependency
```

**Approach 3: API-Driven**

```typescript
// API-driven skill distribution
interface SkillAPI {
  // List available skills for this user/team
  listSkills(userId: string): Promise<SkillMetadata[]>;
  
  // Fetch a specific skill
  getSkill(skillId: string): Promise<SkillContent>;
  
  // Publish a new version of a skill
  publishSkill(skill: SkillContent): Promise<void>;
  
  // Track skill usage
  reportUsage(skillId: string, metrics: UsageMetrics): Promise<void>;
}
```

```
Pros:
  + Most flexible
  + Can integrate with any system
  + Supports dynamic skill selection
  + Can do A/B testing of skills

Cons:
  - Most engineering effort
  - Network dependency
  - Latency for skill loading
  - Must build the entire system
```

### 6.3 The Gap Between Claude Code and Desktop

The leak revealed an interesting asymmetry: Claude Code has full skill/plugin support, but the Claude Desktop app has limited support. This creates a gap:

```
┌─────────────────────────────────────────────────────┐
│  FEATURE         │ Claude Code │ Claude Desktop     │
├──────────────────┼─────────────┼────────────────────┤
│  CLAUDE.md       │ Yes         │ Limited (projects) │
│  Custom skills   │ Yes         │ No                 │
│  Plugins         │ Yes         │ No                 │
│  Hooks           │ Yes         │ No                 │
│  MCP servers     │ Yes         │ Yes                │
│  Custom tools    │ Yes (MCP)   │ Yes (MCP)          │
│  Org distribution│ Git/API     │ MCP only           │
└──────────────────┴─────────────┴────────────────────┘
```

For organizations, this means: if you want full customization, your engineers need Claude Code. If your engineers use Claude Desktop, MCP servers are your main extension mechanism.

---

## 7. Anti-Distillation: fake_tools for IP Protection

### 7.1 What Anti-Distillation Is

The leak revealed an anti-distillation mechanism: the system injects "fake tools" (tool definitions that do not correspond to real tools) into the system prompt. The purpose is to poison attempts to extract the system prompt for replication.

```typescript
// Anti-distillation (reconstructed)
function injectFakeTools(
  realTools: ToolDefinition[]
): ToolDefinition[] {
  // Add fake tool definitions that look real but do nothing
  const fakeTools: ToolDefinition[] = [
    {
      name: 'InternalAnalyze',
      description: 'Perform deep analysis of code patterns',
      input_schema: {
        type: 'object',
        properties: {
          target: { type: 'string' },
          depth: { type: 'number' }
        },
        required: ['target']
      }
    },
    {
      name: 'SecurityScan',
      description: 'Scan code for security vulnerabilities',
      input_schema: {
        type: 'object',
        properties: {
          path: { type: 'string' },
          level: { type: 'string' }
        },
        required: ['path']
      }
    },
    // ... more fake tools
  ];
  
  // Mix fake tools into the real ones
  return shuffleArray([...realTools, ...fakeTools]);
}
```

### 7.2 How It Works

If someone attempts to distill the Claude Code system prompt (by asking the model to output its instructions), the fake tools would be included. Anyone trying to replicate the system would:

1. See tool definitions that look real
2. Try to implement them
3. Waste time on tools that Claude Code does not actually use
4. Get confused when the replicated system does not work as expected

### 7.3 The Effectiveness Debate

The community debated whether fake_tools is an effective anti-distillation measure:

```
Arguments FOR effectiveness:
├── Makes exact replication harder
├── Wastes attacker time on fake implementations
├── Creates plausible confusion about what tools exist
└── Low cost to implement (just extra JSON in system prompt)

Arguments AGAINST effectiveness:
├── Fake tools are discoverable by trying to use them
│   (they return errors when called)
├── The real tools are also visible in the system prompt
├── Motivated attackers can filter fakes through testing
└── The system prompt is not the main IP anyway
│   (the real value is the harness engineering)
```

The consensus view: fake_tools raises the bar for casual replication but does not stop determined reverse engineering. The real protection is that the harness is a large, complex engineering system -- knowing the system prompt is a small fraction of what you need to replicate it.

---

## 8. The "Undercover Mode" Controversy

### 8.1 What It Is

The leak revealed a setting or behavior pattern where Claude Code could contribute to repositories without explicit disclosure that AI was involved. The community called this "undercover mode":

```
NORMAL MODE:
  git commit -m "Fix auth bug

  Co-authored-by: Claude Code"

UNDERCOVER MODE:
  git commit -m "Fix auth bug"
  (no AI disclosure)
```

### 8.2 The Controversy

This sparked debate in the developer community:

```
┌──────────────────────────────────────────────────────┐
│        THE UNDERCOVER MODE DEBATE                     │
│                                                        │
│  IN FAVOR:                                             │
│  ├── Some open-source projects reject AI contributions │
│  ├── AI disclosure creates stigma that hurts adoption  │
│  ├── The code quality matters, not who wrote it        │
│  └── Many developers already use AI without disclosing │
│                                                        │
│  AGAINST:                                              │
│  ├── Open-source projects have the right to know       │
│  ├── AI-generated code may have different liability    │
│  ├── Reviewers should know the origin for context      │
│  ├── Trust and transparency are foundational values    │
│  └── Some licenses may require authorship disclosure   │
│                                                        │
│  NUANCE:                                               │
│  ├── Internal repos: less controversial                │
│  ├── Open-source repos: more controversial             │
│  ├── The question is about norms, not technology       │
│  └── The feature exists; the ethics are debated        │
└──────────────────────────────────────────────────────┘
```

### 8.3 The Practical Impact

For organizations building AI tools:

1. **Make disclosure configurable, not mandatory.** Users have different requirements.
2. **Default to disclosure.** Transparency should be the default; hiding should require explicit opt-in.
3. **Support different disclosure formats.** Some teams want commit trailers, others want PR labels, others want nothing.
4. **Separate the tool from the policy.** The tool should support whatever policy the organization adopts.

---

## 9. How Skill Quality Affects Agent Performance

### 9.1 The Skill Quality Spectrum

Not all skills are equal. The quality of a skill directly affects how well the agent follows it:

```
┌──────────────────────────────────────────────────────┐
│          SKILL QUALITY SPECTRUM                       │
│                                                        │
│  LOW QUALITY (agent ignores or misuses):               │
│  ├── Vague instructions ("write good code")            │
│  ├── Contradictory rules                               │
│  ├── Too long (buried in text, agent loses focus)      │
│  ├── No examples (agent must guess what you want)      │
│  └── Wrong tools listed (skill requests unavailable    │
│      tools)                                            │
│                                                        │
│  MEDIUM QUALITY (agent partially follows):             │
│  ├── Clear but incomplete instructions                 │
│  ├── Some examples but not edge cases                  │
│  ├── Reasonable length                                 │
│  ├── Correct tool requirements                         │
│  └── Missing progressive disclosure                    │
│                                                        │
│  HIGH QUALITY (agent reliably follows):                │
│  ├── Specific, unambiguous instructions                │
│  ├── Concrete examples of inputs and outputs           │
│  ├── Concise (under 800 tokens)                        │
│  ├── Progressive disclosure (most important first)     │
│  ├── Correct tool requirements                         │
│  ├── Edge case handling                                │
│  └── Explicit "do NOT" instructions                    │
└──────────────────────────────────────────────────────┘
```

### 9.2 Skill Quality Patterns

**Pattern 1: The Constraint Pattern**
Tell the agent what NOT to do. This is often more effective than telling it what to do:

```markdown
## Constraints
- NEVER use `any` type in TypeScript
- NEVER import from the `legacy/` directory
- NEVER use `console.log` for error handling (use the logger)
- NEVER modify files in `src/generated/`
- ALWAYS run tests after making changes
```

**Pattern 2: The Example Pattern**
Show, don't tell:

```markdown
## Naming Conventions

GOOD:
```typescript
function getUserById(id: string): Promise<User> { }
function createPaymentIntent(amount: number): Promise<Intent> { }
```

BAD:
```typescript
function getUser(id: string): Promise<User> { }  // Missing "ById"
function makePayment(amount: number): Promise<Intent> { }  // Wrong verb
```
```

**Pattern 3: The Decision Tree Pattern**
Guide the agent through choices:

```markdown
## Choosing a Testing Strategy

If the function is pure (no side effects):
  -> Write a unit test with direct assertions

If the function calls an API:
  -> Mock the API call with vi.mock()
  -> Test both success and error cases

If the function modifies the database:
  -> Use a test database (not mocks)
  -> Clean up after each test

If the function renders UI:
  -> Use @testing-library/react
  -> Test user interactions, not implementation
```

### 9.3 Measuring Skill Effectiveness

```typescript
// Skill effectiveness metrics
interface SkillMetrics {
  // How often the skill is activated
  activationCount: number;
  
  // How often the agent follows the skill's instructions
  complianceRate: number;  // 0-1
  
  // How often the user overrides the skill's guidance
  overrideRate: number;    // 0-1
  
  // How much the skill adds to context (cost)
  tokenCost: number;
  
  // User satisfaction when skill is active
  satisfaction: number;    // 0-5
  
  // Computed effectiveness
  effectiveness(): number {
    return (this.complianceRate * (1 - this.overrideRate))
      * this.satisfaction
      / Math.log(this.tokenCost + 1);
    // High compliance, low override, high satisfaction,
    // low cost = high effectiveness
  }
}
```

---

## 10. Designing Skills and Plugins for Your Own Systems

### 10.1 The Design Checklist

```
┌──────────────────────────────────────────────────────┐
│        SKILL/PLUGIN DESIGN CHECKLIST                  │
│                                                        │
│  SKILLS:                                               │
│  [ ] Under 800 tokens (cost awareness)                 │
│  [ ] Progressive disclosure (important first)          │
│  [ ] Concrete examples (show, don't tell)              │
│  [ ] Explicit constraints (what NOT to do)             │
│  [ ] Correct tool requirements in frontmatter          │
│  [ ] Clear triggers for auto-activation                │
│  [ ] Tested with real tasks (not just theorized)       │
│                                                        │
│  PLUGINS:                                              │
│  [ ] Hooks are focused (one concern per hook)          │
│  [ ] Hooks are fast (don't add latency to tool calls)  │
│  [ ] MCP servers are reliable (handle errors)          │
│  [ ] Settings have sensible defaults                   │
│  [ ] Plugin does not break without its MCP server      │
│  [ ] Plugin works across different project types       │
│                                                        │
│  DISTRIBUTION:                                         │
│  [ ] Version controlled (git or API versioning)        │
│  [ ] Can be updated without user action                │
│  [ ] Team-level customization supported                │
│  [ ] Usage metrics collected                           │
│  [ ] Feedback mechanism for users                      │
└──────────────────────────────────────────────────────┘
```

### 10.2 A Minimal Skill System

If you are building your own agent and want skill-like functionality:

```typescript
// minimal-skill-system.ts

interface Skill {
  name: string;
  description: string;
  triggers: string[];
  content: string;
  tokenCount: number;
}

class SkillSystem {
  private skills: Map<string, Skill> = new Map();
  private activeSkills: Set<string> = new Set();
  
  // Load skills from a directory
  async loadFromDirectory(dir: string): Promise<void> {
    const files = await glob('*.md', { cwd: dir });
    
    for (const file of files) {
      const content = await fs.promises.readFile(
        path.join(dir, file), 'utf-8'
      );
      const skill = this.parseSkill(content);
      this.skills.set(skill.name, skill);
    }
  }
  
  // Parse a skill file (frontmatter + body)
  private parseSkill(content: string): Skill {
    const frontmatterMatch = content.match(
      /^---\n([\s\S]*?)\n---\n([\s\S]*)$/
    );
    
    if (!frontmatterMatch) {
      throw new Error('Skill file must have YAML frontmatter');
    }
    
    const metadata = parseYAML(frontmatterMatch[1]);
    const body = frontmatterMatch[2].trim();
    
    return {
      name: metadata.name,
      description: metadata.description,
      triggers: metadata.triggers || [],
      content: body,
      tokenCount: estimateTokens(body),
    };
  }
  
  // Activate a skill
  activate(name: string): void {
    if (!this.skills.has(name)) {
      throw new Error(`Skill not found: ${name}`);
    }
    this.activeSkills.add(name);
  }
  
  // Deactivate a skill
  deactivate(name: string): void {
    this.activeSkills.delete(name);
  }
  
  // Auto-select skills based on user input
  autoSelect(userInput: string): string[] {
    const selected: string[] = [];
    
    for (const [name, skill] of this.skills) {
      if (this.activeSkills.has(name)) continue;
      
      for (const trigger of skill.triggers) {
        if (userInput.toLowerCase().includes(
          trigger.toLowerCase()
        )) {
          selected.push(name);
          break;
        }
      }
    }
    
    return selected;
  }
  
  // Get the combined content of all active skills
  getActiveContent(): string {
    const parts: string[] = [];
    
    for (const name of this.activeSkills) {
      const skill = this.skills.get(name);
      if (skill) {
        parts.push(
          `<skill name="${name}">\n${skill.content}\n</skill>`
        );
      }
    }
    
    return parts.join('\n\n');
  }
  
  // Get total token cost of active skills
  getActiveTokenCost(): number {
    let total = 0;
    for (const name of this.activeSkills) {
      const skill = this.skills.get(name);
      if (skill) total += skill.tokenCount;
    }
    return total;
  }
}
```

### 10.3 A Minimal Hook System

```typescript
// minimal-hook-system.ts

type HookEvent = 'preToolUse' | 'postToolUse' | 'stop';

interface HookContext {
  toolName: string;
  input: Record<string, unknown>;
  result?: string;
  session: SessionInfo;
}

type HookHandler = (
  context: HookContext
) => Promise<{
  action: 'allow' | 'block' | 'modify';
  message?: string;
  modifiedInput?: Record<string, unknown>;
  modifiedResult?: string;
}>;

class HookSystem {
  private hooks: Map<HookEvent, HookHandler[]> = new Map();
  
  register(event: HookEvent, handler: HookHandler): void {
    const existing = this.hooks.get(event) || [];
    existing.push(handler);
    this.hooks.set(event, existing);
  }
  
  async runPreToolUse(
    context: HookContext
  ): Promise<{ allowed: boolean; input: Record<string, unknown> }> {
    const handlers = this.hooks.get('preToolUse') || [];
    let currentInput = context.input;
    
    for (const handler of handlers) {
      const result = await handler({
        ...context,
        input: currentInput
      });
      
      if (result.action === 'block') {
        return { allowed: false, input: currentInput };
      }
      
      if (result.action === 'modify' && result.modifiedInput) {
        currentInput = result.modifiedInput;
      }
    }
    
    return { allowed: true, input: currentInput };
  }
  
  async runPostToolUse(
    context: HookContext
  ): Promise<{ result: string }> {
    const handlers = this.hooks.get('postToolUse') || [];
    let currentResult = context.result || '';
    
    for (const handler of handlers) {
      const hookResult = await handler({
        ...context,
        result: currentResult
      });
      
      if (
        hookResult.action === 'modify' &&
        hookResult.modifiedResult
      ) {
        currentResult = hookResult.modifiedResult;
      }
    }
    
    return { result: currentResult };
  }
}
```

---

## 11. Key Takeaways

1. **Skills are prompts.** They are markdown text injected into the system prompt. Everything else -- frontmatter, loading, distribution -- is infrastructure around this simple idea.

2. **CLAUDE.md is re-injected every turn.** This means short = cheaper. Target under 500 tokens for CLAUDE.md, under 800 tokens per skill.

3. **Skillify auto-generates skills from session patterns.** It detects repeated tool sequences, extracts generalizable patterns, and proposes new skills. This creates a self-reinforcing loop.

4. **Plugins extend the harness with code.** Hooks (PreToolUse, PostToolUse, Stop) intercept agent behavior. MCP bundling adds tools. Settings make plugins configurable.

5. **Three distribution approaches:** git-backed (version controlled, code reviewed), admin UI (centralized, instant), API-driven (flexible, requires infrastructure).

6. **Anti-distillation (fake_tools)** injects fake tool definitions to confuse replication attempts. Effectiveness is debated -- the real moat is the harness engineering, not the system prompt.

7. **The "undercover mode" controversy** is about norms, not technology. Default to transparency, make disclosure configurable.

8. **Skill quality directly affects agent performance.** Be specific, use examples, include constraints, keep it concise, use progressive disclosure.

9. **Description optimization matters.** The skill description determines selection and understanding. Make it specific, action-oriented, and right-sized.

10. **The skill quality flywheel:** better skills -> better agent performance -> better sessions -> better patterns -> better skills.

---

*Previous: [Chapter 32 -- Multi-Agent Coordination](./32-multi-agent-coordination.md)* · *Next: [Part 7 -- The Hard Parts](../part-7-hard-parts/)* shifts from understanding tools to understanding the models themselves -- neural networks, transformers, and attention from scratch.

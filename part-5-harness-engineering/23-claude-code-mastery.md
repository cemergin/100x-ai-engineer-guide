<!--
  CHAPTER: 23
  TITLE: Claude Code Mastery
  PART: 5 — Harness Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 9 (agent loop), Ch 10 (agent memory), Ch 13 (agent patterns)
  KEY_TOPICS: claude-code, claude-md, skills, plugins, hooks, plan-mode, worktrees, permissions, settings, 1code-workflow
  DIFFICULTY: Beg→Adv
  LANGUAGE: Markdown, JSON, Bash, TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 23: Claude Code Mastery

> Part 5: Harness Engineering · Phase 1: Get Dangerous · Prereqs: Ch 9, 10, 13 · Difficulty: Beg→Adv · Language: Markdown/JSON/TS

You built an agent loop in Chapter 9. You gave it memory in Chapter 10. You explored frameworks in Chapter 13. Now here is the punchline: **Claude Code is the best agent loop most engineers will ever use**, and mastering it is the single highest-leverage skill you can develop as an AI engineer.

This is not a chapter about "tips and tricks." This is a chapter about understanding a system — what it is, how it thinks, and how to reshape its behavior to match your exact workflow. By the end, you will treat Claude Code not as a chatbot with file access, but as a programmable agent runtime that happens to speak English.

### In This Chapter

1. What Claude Code actually is (and why it matters)
2. CLAUDE.md: the most important file in your project
3. Skills: teaching the agent specific workflows
4. Plugins: extending Claude Code with custom tools and agents
5. Hooks: intercepting agent behavior at every stage
6. Plan mode: thinking before acting
7. Worktrees: parallel agents on isolated copies of your codebase
8. Permissions: controlling what the agent can do
9. Settings configuration
10. The 1Code workflow for managing multiple instances

### Related Chapters

| Direction | Chapter | Connection |
|-----------|---------|------------|
| ↩ spirals back | Ch 9: The Agent Loop | Claude Code IS this loop — prompt→LLM→tool→execute→append→repeat |
| ↩ spirals back | Ch 10: Agent Memory | CLAUDE.md is the memory layer you designed in theory |
| ↩ spirals back | Ch 13: Agent Patterns | Claude Code is a framework that implements these patterns |
| ↪ spirals forward | Ch 24: MCP Servers | Extend Claude Code with external tools |
| ↪ spirals forward | Ch 27: Harness Architecture | See the full architecture under the hood |
| ↪ spirals forward | Ch 33: Skills Internals | How skills are parsed, matched, and injected |

---

## 1. What Claude Code Actually Is

### 1.1 An Agent Loop with Your Codebase as Context

Strip away the marketing and Claude Code is exactly what you built in Chapter 9: an agent loop. It reads a prompt, sends it to Claude along with context, receives a response that may include tool calls, executes those tools, appends the results, and loops until the task is done.

```
┌─────────────────────────────────────────────────┐
│                  CLAUDE CODE                      │
│                                                   │
│  ┌───────┐    ┌───────┐    ┌──────────┐          │
│  │ Input │───→│ Claude │───→│ Tool Use │──┐       │
│  │ (you) │    │  API   │    │ Decision │  │       │
│  └───────┘    └───────┘    └──────────┘  │       │
│       ↑                                   │       │
│       │    ┌──────────┐    ┌──────────┐  │       │
│       └────│ Append   │←───│ Execute  │←─┘       │
│            │ Results  │    │  Tools   │           │
│            └──────────┘    └──────────┘           │
│                                                   │
│  Tools: Read, Edit, Write, Bash, Glob, Grep,     │
│         WebSearch, MCP servers, ...               │
│                                                   │
│  Memory: CLAUDE.md, conversation history,         │
│          session context, project files           │
└─────────────────────────────────────────────────┘
```

The difference between your Chapter 9 agent and Claude Code is **scale**. Claude Code ships with:

- **Dozens of built-in tools** — file I/O, search, bash execution, web search, notebook editing
- **A permission system** — controlling which tools run automatically vs. need approval
- **Memory layers** — CLAUDE.md files at project, user, and organization levels
- **A plugin architecture** — for extending with custom tools, agents, and hooks
- **Session management** — conversation history, context compaction, token budgets
- **Multi-agent support** — spawning sub-agents for parallel work

**Why it matters:** You could build all of this yourself. But you should not. The harness is the most well-tested, most battle-hardened agent loop available. Your job is to configure it, extend it, and make it work for your specific context — not to reinvent it.

### 1.2 The Mental Model

Think of Claude Code as three layers:

```
┌─────────────────────────────────┐
│  Layer 3: YOUR CONFIGURATION    │  ← CLAUDE.md, skills, plugins, hooks
│  (what makes it yours)          │
├─────────────────────────────────┤
│  Layer 2: THE HARNESS           │  ← Agent loop, tools, permissions, memory
│  (the runtime)                  │
├─────────────────────────────────┤
│  Layer 1: THE MODEL             │  ← Claude (Sonnet, Opus, Haiku)
│  (the intelligence)             │
└─────────────────────────────────┘
```

Most people only interact with Layer 1 — they type prompts and hope for the best. A 10x engineer configures Layer 3 so aggressively that Layer 1 almost cannot fail. That is what this chapter teaches.

### 1.3 Installing and Starting

```bash
# Install Claude Code globally
npm install -g @anthropic-ai/claude-code

# Start in your project directory
cd your-project
claude

# Or start with a specific task
claude "explain the authentication flow in this codebase"

# Start in plan mode (think first, act later)
claude --plan

# Start with a specific model
claude --model opus

# Resume a previous conversation
claude --resume
```

Claude Code looks at your current directory and treats it as the project root. It will read files, run commands, and make changes relative to this directory. Choose wisely.

---

## 2. CLAUDE.md: The Most Important File in Your Project

### 2.1 What It Is

CLAUDE.md is a markdown file that gets injected into Claude Code's system prompt every time it starts a conversation in your project. It is the single most powerful lever you have for controlling agent behavior.

Think of it this way: every time Claude Code starts, it reads CLAUDE.md and treats it as instructions. If your CLAUDE.md says "always use Prettier before committing," Claude Code will run Prettier before committing. If it says "never modify files in the /migrations directory," Claude Code will avoid those files. If it says "when writing tests, use Vitest with the following pattern," Claude Code will follow that pattern.

**CLAUDE.md is not documentation. It is programming.**

### 2.2 The Hierarchy

CLAUDE.md files stack. Claude Code reads them in order of specificity:

```
~/.claude/CLAUDE.md              ← User-level (your personal preferences)
.claude/CLAUDE.md                ← Organization-level (shared via repo)
CLAUDE.md                        ← Project root (the main one)
src/CLAUDE.md                    ← Directory-level (loaded when working in src/)
src/components/CLAUDE.md         ← Deeper directory-level
```

**What goes where:**

| Level | What to put here | Example |
|-------|-----------------|---------|
| User (~/.claude/CLAUDE.md) | Personal style, editor preferences, identity | "I prefer functional style. My name is Cem." |
| Organization (.claude/CLAUDE.md) | Org-wide conventions, shared tooling | "We use pnpm. Our CI runs on GitHub Actions." |
| Project root (CLAUDE.md) | Project architecture, build commands, patterns | "This is a Next.js app with Drizzle ORM..." |
| Directory (src/CLAUDE.md) | Module-specific context | "Components in this dir use Radix UI primitives." |

### 2.3 Anatomy of a Great CLAUDE.md

Here is a real-world CLAUDE.md structure that works:

```markdown
# CLAUDE.md

## Project Overview
This is Nelo — a B2B fintech platform built with Next.js 15, TypeScript,
Drizzle ORM, and PostgreSQL. We serve ~2000 SMB customers.

## Architecture
- **Frontend:** Next.js App Router, React Server Components, Tailwind CSS
- **Backend:** Next.js API routes + tRPC for internal calls
- **Database:** PostgreSQL via Drizzle ORM (schema in src/db/schema/)
- **Auth:** Clerk (middleware in src/middleware.ts)
- **Payments:** Stripe (webhooks in src/app/api/webhooks/stripe/)

## Commands
- `pnpm dev` — start dev server
- `pnpm build` — production build
- `pnpm test` — run Vitest
- `pnpm lint` — ESLint + Prettier
- `pnpm db:push` — push schema changes to database
- `pnpm db:generate` — generate Drizzle migrations

## Code Conventions
- Use `async/await`, never raw promises with `.then()`
- Prefer named exports over default exports
- Use Zod for all runtime validation
- Error handling: use Result types from `src/lib/result.ts`, not try/catch
- File naming: kebab-case for files, PascalCase for components
- Imports: use `@/` alias for src/ imports

## Testing
- Use Vitest for unit tests, Playwright for E2E
- Test files live next to source: `foo.ts` → `foo.test.ts`
- Mock external services, never hit real APIs in tests
- Use `createTestContext()` from `src/test/helpers.ts`

## Common Patterns
When creating a new API endpoint:
1. Define the Zod schema in `src/schemas/`
2. Create the route in `src/app/api/`
3. Add the tRPC procedure in `src/server/routers/`
4. Write tests for happy path + error cases

When creating a new database table:
1. Add schema in `src/db/schema/`
2. Run `pnpm db:generate` to create migration
3. Run `pnpm db:push` to apply
4. Add Zod schemas that mirror the table in `src/schemas/`

## Things to NEVER Do
- NEVER modify `src/db/migrations/` directly — always use Drizzle CLI
- NEVER commit `.env` files
- NEVER use `any` type — use `unknown` and narrow
- NEVER push directly to `main` — always create a branch
- NEVER skip writing tests for new API endpoints

## Active Work
- Currently migrating from Pages Router to App Router (in progress)
- The payments module is being refactored — see LINEAR-1234
- Do not touch `src/legacy/` — scheduled for deletion
```

### 2.4 CLAUDE.md Principles

**Be specific, not aspirational.** Do not write "follow best practices." Write "use Vitest, not Jest. Put test files next to source files. Use createTestContext() for setup."

**Include commands.** Claude Code will run your build, test, and lint commands. If it does not know what they are, it will guess — and guess wrong.

**Include anti-patterns.** The "Things to NEVER Do" section is often more valuable than the "How to Do Things" section. Claude Code is eager to help. It will do dumb things unless you explicitly tell it not to.

**Keep it fresh.** CLAUDE.md is not a write-once document. Update it as your project evolves. Add patterns you find yourself repeating. Remove patterns that no longer apply.

**Use it as onboarding.** A good CLAUDE.md is also a good onboarding doc for human engineers. If a new hire can read your CLAUDE.md and understand the project, you have written a great one.

### 2.5 Advanced: Dynamic Context in CLAUDE.md

CLAUDE.md can reference other files that Claude Code will read:

```markdown
## API Schema
See the full API schema in `docs/api-schema.yaml` — read this before
modifying any API endpoints.

## Database Schema
The source of truth for our data model is in `src/db/schema/index.ts`.
Always check this file before creating queries.
```

Claude Code will follow these references and read the linked files when relevant. This lets you keep CLAUDE.md concise while pointing to detailed documentation.

---

## 3. Skills: Teaching the Agent Specific Workflows

### 3.1 What Skills Are

Skills are markdown files that teach Claude Code how to perform specific tasks. When you type a slash command like `/deploy` or `/review-pr`, Claude Code loads the corresponding skill file and follows its instructions.

Think of skills as **reusable prompts with structure**. Instead of typing "review this PR, check for security issues, look at test coverage, and leave comments on GitHub" every time, you write a skill once and invoke it with `/review-pr`.

### 3.2 Where Skills Live

```
.claude/
  skills/
    deploy.md              ← /deploy
    review-pr.md           ← /review-pr
    create-component.md    ← /create-component
    write-migration.md     ← /write-migration

~/.claude/
  skills/
    my-standup.md          ← /my-standup (personal, across all projects)
```

Project skills live in `.claude/skills/`. Personal skills live in `~/.claude/skills/`. Project skills are committed to git and shared with the team.

### 3.3 Anatomy of a Skill

```markdown
---
description: Review a pull request for code quality, security, and test coverage
allowed-tools: Bash, Read, Grep, Glob, WebSearch
---

# Review Pull Request

## Steps

1. **Get PR context**: Run `gh pr view --json title,body,files` to understand
   what changed and why.

2. **Read changed files**: Use Glob and Read to examine every changed file.
   Focus on:
   - Logic correctness
   - Error handling (are errors caught? are they meaningful?)
   - Security (SQL injection, XSS, auth checks)
   - Performance (N+1 queries, unnecessary re-renders)

3. **Check test coverage**: For every new function or endpoint, verify there
   is a corresponding test. If tests are missing, note which ones are needed.

4. **Check conventions**: Compare against CLAUDE.md conventions. Flag any
   violations.

5. **Write review**: Summarize findings as a structured review:
   - **Summary**: 1-2 sentence overview
   - **Issues**: Bulleted list with severity (critical/warning/nit)
   - **Suggestions**: Improvements that are not blockers
   - **Verdict**: Approve, Request Changes, or Needs Discussion
```

### 3.4 Skill Frontmatter

The YAML frontmatter controls how the skill is discovered and executed:

```yaml
---
# Required: how Claude Code matches this skill to user intent
description: Deploy the current branch to staging environment

# Optional: restrict which tools this skill can use
allowed-tools: Bash, Read, Grep

# Optional: restrict which tools this skill cannot use
disallowed-tools: Write, Edit

# Optional: tool definitions specific to this skill
tools:
  - name: deploy_status
    description: Check deployment status
    command: "kubectl rollout status deployment/app"
---
```

The `description` field is critical. Claude Code uses it to match your request to the right skill. Write descriptions that clearly state what the skill does, not how it does it.

**Good descriptions:**
- "Review a pull request for code quality and security"
- "Deploy the current branch to the staging environment"
- "Generate a new React component with tests and stories"

**Bad descriptions:**
- "PR review" (too vague)
- "This skill reads files and runs gh commands" (describes how, not what)
- "Deployment helper" (not specific enough)

### 3.5 Progressive Disclosure in Skills

Long skills should use progressive disclosure — give Claude Code the high-level steps first, then provide details it can reference as needed:

```markdown
---
description: Create a new database migration with proper testing
---

# Create Database Migration

## Quick Steps
1. Add new table/column to schema in `src/db/schema/`
2. Generate migration with `pnpm db:generate`
3. Test migration applies cleanly
4. Write migration test
5. Update related Zod schemas

## Detailed Reference

### Schema Conventions
- Table names: plural, snake_case (`user_accounts`, not `UserAccount`)
- Column names: snake_case
- Always include `created_at` and `updated_at` timestamps
- Foreign keys: `{referenced_table_singular}_id` (e.g., `user_id`)
- Use `pgEnum` for enum types, not string columns

### Migration Testing Pattern
```typescript
import { describe, it, expect } from 'vitest';
import { migrate } from '@/db/migrate';
import { createTestDb } from '@/test/helpers';

describe('migration: add_user_preferences', () => {
  it('applies cleanly to empty database', async () => {
    const db = await createTestDb();
    await expect(migrate(db)).resolves.not.toThrow();
  });

  it('is idempotent', async () => {
    const db = await createTestDb();
    await migrate(db);
    await expect(migrate(db)).resolves.not.toThrow();
  });
});
```

### Common Mistakes
- Forgetting to add `NOT NULL` constraints with defaults for existing tables
- Not handling the down migration
- Forgetting to update the Zod schema to match
```

---

## 4. Plugins: Extending Claude Code with Custom Tools and Agents

### 4.1 What Plugins Are

Plugins are packages that extend Claude Code with custom tools, agents, hooks, and skills. While skills are single markdown files, plugins are structured directories that can contain multiple components.

Think of the relationship this way:
- **Skills** = individual recipes
- **Plugins** = entire cookbooks with their own tools

### 4.2 Plugin Structure

```
my-plugin/
  plugin.json              ← Plugin manifest
  README.md                ← Documentation
  skills/
    do-something.md        ← Skills provided by this plugin
  agents/
    worker.md              ← Sub-agents this plugin provides
  hooks/
    pre-commit-check.md    ← Hooks that intercept behavior
  .mcp.json                ← MCP servers this plugin provides
```

### 4.3 The Plugin Manifest

```json
{
  "name": "nelo-engineering",
  "version": "1.0.0",
  "description": "Engineering workflows for the Nelo platform",
  "author": "Nelo Engineering",
  "skills": [
    "skills/deploy.md",
    "skills/create-endpoint.md",
    "skills/review-pr.md"
  ],
  "agents": [
    "agents/code-reviewer.md",
    "agents/test-writer.md"
  ],
  "hooks": [
    "hooks/pre-commit-check.md"
  ]
}
```

### 4.4 Plugin Agents

Plugin agents are sub-agents that can be spawned by the main Claude Code session. They run in their own context with their own tool permissions:

```markdown
---
name: test-writer
description: Writes comprehensive tests for changed files
when-to-use: When the user asks to write tests or when new code lacks test coverage
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Test Writer Agent

You are a specialized testing agent. Your job is to write thorough tests
for the code you are given.

## Approach

1. Read the source file to understand what it does
2. Identify all public functions and their edge cases
3. Write tests using Vitest following the project's testing conventions
4. Run the tests to verify they pass
5. Report coverage and any untestable code

## Testing Principles
- Test behavior, not implementation
- One assertion per test when possible
- Use descriptive test names: "should return 404 when user not found"
- Mock external dependencies, never mock the thing under test
- Include edge cases: null, undefined, empty arrays, boundary values
```

### 4.5 Installing Plugins

Plugins can be installed from Git repos, npm packages, or local paths:

```json
// .claude/settings.json
{
  "plugins": [
    "github:nelo-engineering/claude-plugin-nelo",
    "./local-plugins/my-custom-plugin",
    "npm:@company/claude-plugin-shared"
  ]
}
```

Or via command line:

```bash
# Install from GitHub
claude plugin add github:nelo-engineering/claude-plugin-nelo

# Install from local directory
claude plugin add ./my-plugin

# List installed plugins
claude plugin list

# Remove a plugin
claude plugin remove nelo-engineering
```

---

## 5. Hooks: Intercepting Agent Behavior

### 5.1 What Hooks Are

Hooks let you intercept Claude Code's behavior at three points:

```
                    ┌──────────────┐
  User Input ──→   │ PreToolUse   │ ──→  Can block, modify, or approve
                    └──────────────┘
                           │
                    ┌──────────────┐
                    │ Tool Executes│
                    └──────────────┘
                           │
                    ┌──────────────┐
                    │ PostToolUse  │ ──→  Can modify results, log, alert
                    └──────────────┘
                           │
                    ┌──────────────┐
                    │    Stop      │ ──→  Runs when agent finishes a turn
                    └──────────────┘
```

- **PreToolUse**: Fires before any tool executes. Can approve, block, or modify the tool call.
- **PostToolUse**: Fires after a tool executes. Can modify results or trigger side effects.
- **Stop**: Fires when the agent is about to stop its turn. Can force continuation or trigger cleanup.

### 5.2 Hook Configuration

Hooks are configured in your settings.json:

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hook": ".claude/hooks/validate-bash.sh",
        "description": "Validate bash commands before execution"
      },
      {
        "matcher": "Write|Edit",
        "hook": ".claude/hooks/check-file-permissions.sh",
        "description": "Prevent writes to protected files"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hook": ".claude/hooks/auto-lint.sh",
        "description": "Auto-lint after file changes"
      }
    ],
    "Stop": [
      {
        "hook": ".claude/hooks/session-summary.sh",
        "description": "Log session summary on completion"
      }
    ]
  }
}
```

### 5.3 Writing a PreToolUse Hook

A PreToolUse hook receives the tool name and parameters via stdin as JSON. It outputs a JSON response indicating whether to proceed:

```bash
#!/bin/bash
# .claude/hooks/validate-bash.sh
# Blocks dangerous bash commands

# Read the tool use from stdin
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name')
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Only check Bash tool calls
if [ "$TOOL_NAME" != "Bash" ]; then
  echo '{"decision": "approve"}'
  exit 0
fi

# Block destructive commands
if echo "$COMMAND" | grep -qE '(rm -rf /|DROP DATABASE|git push --force.*main)'; then
  echo '{"decision": "block", "reason": "Blocked: destructive command detected"}'
  exit 0
fi

# Block commands that touch production
if echo "$COMMAND" | grep -qE '(production|prod\.)'; then
  echo '{"decision": "ask_user", "message": "This command references production. Proceed?"}'
  exit 0
fi

# Allow everything else
echo '{"decision": "approve"}'
```

### 5.4 Writing a PostToolUse Hook

```bash
#!/bin/bash
# .claude/hooks/auto-lint.sh
# Runs linting after file modifications

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Only lint after Write or Edit
if [[ "$TOOL_NAME" != "Write" && "$TOOL_NAME" != "Edit" ]]; then
  exit 0
fi

# Only lint TypeScript/JavaScript files
if [[ "$FILE_PATH" == *.ts || "$FILE_PATH" == *.tsx || "$FILE_PATH" == *.js ]]; then
  npx eslint --fix "$FILE_PATH" 2>/dev/null
  npx prettier --write "$FILE_PATH" 2>/dev/null
fi
```

### 5.5 Writing a Stop Hook

```bash
#!/bin/bash
# .claude/hooks/session-summary.sh
# Logs a summary of what changed this session

INPUT=$(cat)

# Count files modified
MODIFIED=$(git diff --name-only 2>/dev/null | wc -l | tr -d ' ')

# Count new files
UNTRACKED=$(git ls-files --others --exclude-standard 2>/dev/null | wc -l | tr -d ' ')

# Log to session file
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Session complete: $MODIFIED modified, $UNTRACKED new files" \
  >> .claude/session-log.txt
```

### 5.6 Hook Patterns Worth Knowing

**The guardian pattern**: PreToolUse hooks that protect sensitive files:

```bash
# Prevent modifications to migration files
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
if [[ "$FILE_PATH" == *"/migrations/"* ]]; then
  echo '{"decision": "block", "reason": "Migration files are immutable. Create a new migration instead."}'
  exit 0
fi
```

**The enrichment pattern**: PostToolUse hooks that add context:

```bash
# After reading a file, append related test file info
if [ "$TOOL_NAME" = "Read" ]; then
  TEST_FILE="${FILE_PATH%.ts}.test.ts"
  if [ -f "$TEST_FILE" ]; then
    echo "{\"note\": \"Test file exists at $TEST_FILE\"}"
  fi
fi
```

**The compliance pattern**: Stop hooks that verify work meets standards:

```bash
# On stop, check if any new files lack tests
NEW_FILES=$(git diff --cached --name-only --diff-filter=A | grep -E '\.(ts|tsx)$' | grep -v '\.test\.')
for FILE in $NEW_FILES; do
  TEST="${FILE%.ts}.test.ts"
  if [ ! -f "$TEST" ]; then
    echo "{\"continue\": true, \"message\": \"Missing test for new file: $FILE\"}"
    exit 0
  fi
done
```

---

## 6. Plan Mode: Thinking Before Acting

### 6.1 Why Plan Mode Exists

By default, Claude Code reads your request and immediately starts executing. For simple tasks — "fix this typo," "rename this variable" — that is fine. For complex tasks — "refactor the authentication system" or "add multi-tenancy support" — you want Claude Code to think first.

Plan mode separates planning from execution. Claude Code will:

1. Analyze the codebase
2. Read relevant files
3. Propose a plan with specific steps
4. Wait for your approval
5. Execute the plan

### 6.2 Using Plan Mode

```bash
# Start Claude Code in plan mode
claude --plan

# Or toggle plan mode during a conversation
# Type /plan to switch to planning mode
```

In plan mode, Claude Code uses extended thinking. It will read files, search the codebase, and reason about the approach before proposing any changes. You will see the plan as a structured proposal:

```
## Plan: Refactor Authentication to Use Clerk

### Context
Currently using a custom JWT auth system in `src/lib/auth.ts` with
session management in `src/middleware.ts`. Migrating to Clerk for
SSO support.

### Steps
1. Install Clerk SDK (`@clerk/nextjs`)
2. Replace custom middleware with Clerk middleware
3. Update all `getServerSession()` calls to use `auth()`
4. Remove custom JWT logic from `src/lib/auth.ts`
5. Update environment variables
6. Migrate session database tables
7. Update tests

### Files Affected
- src/middleware.ts (rewrite)
- src/lib/auth.ts (delete or slim down)
- src/app/api/*/route.ts (14 files — update auth checks)
- src/test/helpers.ts (update mock auth)
- .env.example (add CLERK_* vars)

### Risks
- Existing sessions will be invalidated (users must re-login)
- Webhook signatures will change

Shall I proceed?
```

### 6.3 When to Use Plan Mode

| Task | Use Plan Mode? | Why |
|------|---------------|-----|
| Fix a bug | No | Usually straightforward — just do it |
| Add a small feature | Sometimes | Depends on how many files it touches |
| Refactor a system | Yes | Need to understand blast radius |
| Change architecture | Yes | Must coordinate across many files |
| Unfamiliar codebase | Yes | Need to explore before acting |
| Writing a migration | Yes | Irreversible changes need planning |

### 6.4 Plan Mode and Extended Thinking

Plan mode activates Claude's extended thinking capability. This means the model spends more tokens reasoning internally before responding. The trade-off is clear:

- **More time** to get the first response
- **Better quality** plans for complex tasks
- **Higher token cost** per interaction
- **Fewer mistakes** that require backtracking

For a 10-minute refactoring task, spending 30 seconds on planning is a bargain. For a "fix this typo" task, planning is waste.

---

## 7. Worktrees: Parallel Agents on Isolated Copies

### 7.1 The Problem with One Agent

You have a feature to build, a bug to fix, and a PR to review. With one Claude Code instance on one branch, you serialize all of this. The feature takes an hour. The bug fix takes 20 minutes. The review takes 10 minutes. Total: 90 minutes of wall-clock time.

### 7.2 Git Worktrees Solve This

Git worktrees let you have multiple checked-out copies of your repo, each on a different branch, sharing the same `.git` directory:

```bash
# Create a worktree for a feature
git worktree add ../my-project-feature-auth feature/auth

# Create a worktree for a bugfix
git worktree add ../my-project-fix-nav fix/nav-crash

# List worktrees
git worktree list
# /Users/you/my-project               abc1234 [main]
# /Users/you/my-project-feature-auth   def5678 [feature/auth]
# /Users/you/my-project-fix-nav        ghi9012 [fix/nav-crash]
```

### 7.3 Running Parallel Claude Code Instances

Now you can run Claude Code in each worktree simultaneously:

```bash
# Terminal 1: Feature work
cd ../my-project-feature-auth
claude "Implement OAuth2 login with Google provider"

# Terminal 2: Bug fix
cd ../my-project-fix-nav
claude "Fix the navigation crash when user has no profile image"

# Terminal 3: PR review
cd ../my-project
claude "/review-pr 456"
```

Three agents, three branches, zero conflicts. Total wall-clock time: however long the slowest task takes (probably the feature at ~40 minutes), not the sum of all tasks.

### 7.4 The Worktree Workflow

```
Main repo (main branch)
  │
  ├── Worktree 1 (feature/auth)      ← Claude instance 1
  │     Working on OAuth implementation
  │
  ├── Worktree 2 (fix/nav-crash)     ← Claude instance 2
  │     Fixing navigation bug
  │
  ├── Worktree 3 (feature/dashboard) ← Claude instance 3
  │     Building new dashboard
  │
  └── Main worktree (main)           ← Claude instance 4
        Reviewing PR #456
```

### 7.5 Worktree Best Practices

**Name worktrees descriptively:**
```bash
# Good — clear what each worktree is for
git worktree add ../project-feat-auth feature/auth
git worktree add ../project-fix-crash fix/nav-crash

# Bad — meaningless names
git worktree add ../project-1 branch-1
git worktree add ../project-2 branch-2
```

**Clean up when done:**
```bash
# Remove a worktree after merging
git worktree remove ../my-project-feature-auth

# Prune stale worktrees
git worktree prune
```

**Share CLAUDE.md across worktrees:** Since all worktrees share the same `.git` directory, your project-level CLAUDE.md is available in all of them. But directory-level CLAUDE.md files (like `src/CLAUDE.md`) exist in each worktree independently.

**Watch for resource conflicts:** If two Claude instances both try to run `pnpm dev` on the same port, you will get a conflict. Use different ports or run only non-conflicting tasks in parallel.

---

## 8. Permissions: Controlling What the Agent Can Do

### 8.1 The Permission Model

Claude Code has a tiered permission system. By default, it asks for approval before doing anything that modifies your system:

```
┌─────────────────────────────────────────────────┐
│              PERMISSION TIERS                     │
│                                                   │
│  Tier 1: ALWAYS ALLOWED (read-only)              │
│    Read, Glob, Grep, WebSearch                    │
│                                                   │
│  Tier 2: ASK ONCE PER SESSION                    │
│    Write, Edit, Bash (non-destructive)            │
│                                                   │
│  Tier 3: ASK EVERY TIME                          │
│    Bash (destructive), external API calls          │
│                                                   │
│  Tier 4: NEVER (unless overridden)               │
│    Commands matching deny patterns                 │
└─────────────────────────────────────────────────┘
```

### 8.2 Configuring Permissions

Permissions are configured in settings.json:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(pnpm *)",
      "Bash(npm test *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(DROP *)",
      "Write(.env*)",
      "Edit(.env*)",
      "Write(src/db/migrations/*)",
      "Edit(src/db/migrations/*)"
    ]
  }
}
```

### 8.3 Allow/Deny Patterns

Patterns support globs and tool-specific syntax:

```
# Allow all uses of a tool
"Read"

# Allow a tool with specific file patterns
"Write(src/**/*.ts)"
"Edit(src/components/*.tsx)"

# Allow Bash with specific command patterns
"Bash(pnpm *)"
"Bash(git diff *)"

# Deny specific dangerous patterns
"Bash(rm -rf /)"
"Bash(* --force *)"
"Bash(curl * | bash)"
```

### 8.4 Auto Mode

Auto mode tells Claude Code to proceed without asking for permission on allowed tools:

```bash
# Start in auto mode
claude --auto

# Auto mode with explicit permission set
claude --auto --allow "Write(src/**)" --allow "Bash(pnpm *)"
```

In auto mode, Claude Code will:
- Execute allowed tools without asking
- Still ask for tools not in the allow list
- Still block tools in the deny list
- Never override deny patterns

### 8.5 Dangerously Skip Permissions

```bash
# DANGER: This skips ALL permission checks
claude --dangerously-skip-permissions
```

This flag does what it says. Claude Code will execute any tool — including destructive bash commands, file deletions, and external API calls — without asking. Use it only when:

- You are running Claude Code in a sandboxed CI/CD environment
- You are running in a disposable container
- You have a comprehensive backup strategy
- You are absolutely certain about the task scope

**Never use this on your main development machine.** It only takes one `rm -rf` or one `git push --force main` to ruin your day.

### 8.6 Project-Level Defaults

Set sensible defaults for your whole team:

```json
// .claude/settings.json (committed to git)
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(pnpm dev)",
      "Bash(pnpm test *)",
      "Bash(pnpm lint)",
      "Bash(pnpm build)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(* DROP TABLE *)",
      "Bash(* DELETE FROM *)",
      "Write(.env*)",
      "Write(*.pem)",
      "Write(*.key)"
    ]
  }
}
```

This gives Claude Code enough freedom to be useful (run tests, build, read code) while preventing the most common disasters (deleting files, force pushing, writing secrets).

---

## 9. Settings Configuration

### 9.1 Settings File Locations

Settings stack just like CLAUDE.md, from most general to most specific:

```
~/.claude/settings.json              ← User-level (personal preferences)
.claude/settings.json                ← Project-level (shared via git)
.claude/settings.local.json          ← Project-local (gitignored, your overrides)
```

### 9.2 Full Settings Reference

```json
{
  // Model selection
  "model": "sonnet",

  // Permission configuration
  "permissions": {
    "allow": ["Read", "Glob", "Grep"],
    "deny": ["Bash(rm -rf *)"]
  },

  // Hook configuration
  "hooks": {
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": []
  },

  // MCP server connections
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "@company/mcp-server"],
      "env": {
        "API_KEY": "{{secrets.MY_API_KEY}}"
      }
    }
  },

  // Plugin configuration
  "plugins": [
    "github:org/plugin-name"
  ],

  // Behavior preferences
  "preferences": {
    "autoCompact": true,
    "compactThreshold": 80,
    "planMode": false,
    "verboseToolUse": false
  }
}
```

### 9.3 Settings Patterns for Teams

**The minimum viable team configuration:**

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(pnpm *)", "Bash(npm *)",
      "Bash(git status)", "Bash(git diff *)", "Bash(git log *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Write(.env*)"
    ]
  }
}
```

**The privacy-conscious configuration:**

```json
{
  "permissions": {
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(* | *)",
      "WebSearch"
    ]
  }
}
```

**The CI/CD configuration:**

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep", "Write(src/**)", "Edit(src/**)",
      "Bash(pnpm *)", "Bash(git *)"
    ]
  },
  "preferences": {
    "autoCompact": true,
    "planMode": false
  }
}
```

---

## 10. The 1Code Workflow for Managing Multiple Instances

### 10.1 The Problem

You are running five Claude Code instances across three worktrees. One is building a feature. One is fixing a bug. One is reviewing a PR. One is writing documentation. One is running evals. How do you keep track of all of them?

### 10.2 The 1Code Pattern

The 1Code workflow is a practice pattern for managing multiple concurrent Claude Code instances. The idea is simple: **one terminal multiplexer, one naming convention, one status board.**

**Step 1: Use tmux or a similar multiplexer**

```bash
# Create a named session for each task
tmux new-session -d -s feat-auth
tmux new-session -d -s fix-nav
tmux new-session -d -s pr-review
tmux new-session -d -s docs-update
tmux new-session -d -s evals

# In each session, cd to the right worktree and start Claude Code
tmux send-keys -t feat-auth "cd ~/project-feat-auth && claude 'Implement OAuth'" Enter
tmux send-keys -t fix-nav "cd ~/project-fix-nav && claude 'Fix nav crash'" Enter
tmux send-keys -t pr-review "cd ~/project && claude '/review-pr 456'" Enter
```

**Step 2: Use a consistent naming convention**

```
Session names follow: {type}-{description}
  feat-auth      ← Feature: authentication
  fix-nav        ← Fix: navigation
  pr-456         ← PR review: #456
  docs-api       ← Docs: API documentation
  eval-search    ← Eval: search quality
```

**Step 3: Monitor status**

```bash
# List all tmux sessions
tmux list-sessions

# Peek at a session without switching
tmux capture-pane -t feat-auth -p | tail -20

# Create a status script
#!/bin/bash
# .claude/scripts/status.sh
echo "=== Claude Code Instances ==="
for session in $(tmux list-sessions -F "#{session_name}" 2>/dev/null); do
  last_line=$(tmux capture-pane -t "$session" -p | grep -v "^$" | tail -1)
  echo "  [$session] $last_line"
done
```

### 10.3 The 1Code Orchestration Pattern

For larger tasks, use one Claude Code instance as a coordinator:

```
┌──────────────────────────────────┐
│     Coordinator (main branch)     │
│  "Build the settings page"        │
│                                   │
│  Creates tasks:                   │
│  1. API endpoint (worktree 1)     │
│  2. UI components (worktree 2)    │
│  3. Tests (worktree 3)            │
└──────────┬───────────────────────┘
           │
    ┌──────┼──────┐
    │      │      │
    ▼      ▼      ▼
  WT-1   WT-2   WT-3
  API    UI     Tests
```

The coordinator instance:
1. Breaks down the task into independent pieces
2. Creates worktrees and branches for each piece
3. Launches Claude Code instances in each worktree
4. Monitors progress
5. Integrates the results

This is the bridge to Chapter 26, where we will see this pattern at industry scale with Devin, Codex, and multi-agent orchestration.

### 10.4 Background Agents

For tasks that do not need interactive supervision, use background agents:

```bash
# Start a background agent (runs asynchronously)
claude --background "Run the full test suite and report any failures"

# Start a background agent with a callback
claude --background --notify "Refactor all uses of the old API client"

# List running background agents
claude --list-background
```

Background agents run in detached mode. They complete the task, save the results, and optionally notify you. This is ideal for:

- Running comprehensive test suites
- Code migrations across many files
- Documentation generation
- Eval runs

---

## 11. Putting It All Together

### 11.1 The Day-One Setup Checklist

When you join a new project or start a new one, do this:

```
□ Create CLAUDE.md at project root
  - Project overview and architecture
  - Build/test/lint commands
  - Code conventions
  - Anti-patterns and things to avoid

□ Create .claude/settings.json
  - Permission allow/deny lists
  - MCP server connections
  - Hook configuration

□ Create essential skills in .claude/skills/
  - /deploy (how to deploy this project)
  - /create-component (or whatever you create often)
  - /review-pr (your team's review standards)

□ Set up your personal CLAUDE.md in ~/.claude/CLAUDE.md
  - Your name and preferences
  - Your coding style preferences
  - Your personal skills in ~/.claude/skills/
```

### 11.2 The Evolution Path

```
Week 1:  CLAUDE.md + basic permissions
         → Claude Code knows your project and stays safe

Week 2:  Add skills for repeated tasks
         → Stop typing the same instructions

Week 3:  Add hooks for quality enforcement
         → Claude Code lints, tests, and validates automatically

Week 4:  Connect MCP servers (Ch 24)
         → Claude Code can query your database, read your docs

Month 2: Build team plugins (Ch 25)
         → Your whole team gets the same superpowers

Month 3: Parallel agents + background tasks
         → 5x throughput on multi-feature sprints
```

### 11.3 Measuring Impact

How do you know this is working?

- **Cycle time**: How long from "start feature" to "PR merged"?
- **Rework rate**: How often do you have to fix Claude Code's output?
- **Context-switch cost**: How long does it take to resume after an interruption?
- **Bus factor**: Can a teammate use your Claude Code setup from day one?

Track these before and after your CLAUDE.md + skills + hooks setup. A well-configured harness typically shows 3-5x improvement in cycle time for routine tasks.

---

## 12. Common Pitfalls

### 12.1 The Over-Specified CLAUDE.md

Your CLAUDE.md is 500 lines long. It covers every possible scenario. Claude Code reads all of it on every interaction and now spends half its context window on instructions instead of your actual code.

**Fix:** Keep CLAUDE.md under 200 lines. Move detailed reference material to linked files. Use directory-level CLAUDE.md files for module-specific context.

### 12.2 The Permission Paradox

You lock down permissions so tightly that Claude Code asks for approval on every single action. You end up clicking "allow" 50 times per session. You get frustrated and switch to `--dangerously-skip-permissions`.

**Fix:** Start permissive and tighten. Allow common operations (read, test, lint, build) and deny specific dangers (rm -rf, force push, write to secrets). The deny list is more important than the allow list.

### 12.3 Skills That Are Just Prompts

Your skill file is one paragraph: "Review this PR and tell me if it's good." That is a prompt, not a skill. It does not teach Claude Code anything it does not already know.

**Fix:** Skills should encode *your team's* knowledge. What does a good PR look like at your company? What security patterns do you check for? What test coverage do you require? The value of a skill is the domain knowledge it contains.

### 12.4 Ignoring Plan Mode for Big Tasks

You ask Claude Code to "refactor the payment system" without plan mode. It starts making changes immediately, gets 60% done, realizes the approach is wrong, and you have a mess of partially-applied changes.

**Fix:** Use plan mode for any task that touches more than 5 files. The 30 seconds of planning saves 30 minutes of cleanup.

---

## Summary

Claude Code is not a chatbot. It is a programmable agent runtime. Your job is to configure it:

1. **CLAUDE.md** tells it what your project is and how to work in it
2. **Skills** teach it repeatable workflows
3. **Plugins** extend it with custom tools and agents
4. **Hooks** intercept and modify its behavior at every stage
5. **Plan mode** makes it think before acting on complex tasks
6. **Worktrees** let you run multiple agents in parallel
7. **Permissions** keep it from doing damage
8. **Settings** configure everything in a stackable hierarchy

The engineers who get 10x productivity from Claude Code are not better at prompting. They are better at configuring. They spend 30 minutes on CLAUDE.md and save 30 hours over the next month. They write skills once and use them a thousand times. They set up hooks that enforce quality without human attention.

Master the harness. The model will follow.

---

*Next: [Chapter 24 — MCP Servers & Integrations](./24-mcp-servers.md)*

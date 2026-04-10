<!--
  CHAPTER: 25
  TITLE: Skills, Plugins & Automation
  PART: 5 — Harness Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 23 (Claude Code), Ch 24 (MCP servers), Ch 11 (human-in-the-loop)
  KEY_TOPICS: skills, plugins, frontmatter, progressive-disclosure, git-backed-skills, cron-jobs, background-agents, ask-nelo, council-pattern
  DIFFICULTY: Inter→Adv
  LANGUAGE: Markdown, YAML, JSON, TypeScript, Bash
  UPDATED: 2026-04-10
-->

# Chapter 25: Skills, Plugins & Automation

> **Part 5 — Harness Engineering** | Phase 1: Get Dangerous | Prerequisites: Ch 23, Ch 24, Ch 11 | Difficulty: Intermediate to Advanced | Language: Markdown/YAML/JSON/TypeScript

Chapter 23 taught you to configure Claude Code. Chapter 24 taught you to connect it to external systems. This chapter teaches you to **package everything you know into reusable artifacts and then automate them so they run without you.**

This is the chapter where individual productivity becomes team productivity. A skill you write today saves your entire team 20 minutes per use, every use, forever. A plugin you build wraps your company's conventions into something any engineer can invoke with a slash command. A scheduled agent runs your daily report while you sleep.

The progression is deliberate: write skills (share knowledge) -> build plugins (package it) -> automate (remove yourself from the loop).

### In This Chapter

1. Writing effective skills
2. Building plugins
3. Git-backed skill management
4. Organizational skills: sharing across a team
5. Cron jobs and scheduled agents
6. Background agents
7. Event-driven AI workflows
8. The /ask-nelo pattern: company knowledge skill
9. The /council pattern: multi-model adversarial review
9. Automation philosophy
10. Evaluating skill quality

### Related Chapters

| Direction | Chapter | Connection |
|-----------|---------|------------|
| ↩ spirals back | Ch 23: Claude Code Mastery | Skills and plugins extend the harness you configured |
| ↩ spirals back | Ch 24: MCP Servers | Plugins can bundle MCP server configurations |
| ↩ spirals back | Ch 11: Human-in-the-Loop | Automated pipelines still need approval flows |
| ↪ spirals forward | Ch 33: Skills, Plugins & Distribution | How skills are parsed, matched, and injected internally |
| ↪ spirals forward | Ch 60: Skills Marketplaces | Skills distributed at organizational scale |

---

## 1. Writing Effective Skills

### 1.1 What Makes a Skill Effective

A skill is not a prompt. A prompt says "do this thing." A skill encodes **how your team does this thing** -- the specific steps, the conventions, the edge cases, the quality bar.

The difference:

```markdown
# BAD: This is a prompt, not a skill
Review the PR and tell me if it looks good.
```

```markdown
# GOOD: This is a skill -- it encodes team knowledge
---
description: Review a pull request against Nelo engineering standards
allowed-tools: Bash, Read, Grep, Glob
---

# Review Pull Request

## Step 1: Understand Context
Run `gh pr view --json title,body,files,reviews` to get PR metadata.
Read the PR description to understand the intent.

## Step 2: Check Architecture Compliance
- Verify new API endpoints follow the route convention: `/api/v1/{resource}`
- Verify new database queries use Drizzle ORM, not raw SQL
- Verify new components use Radix UI primitives, not custom implementations
- Check that business logic lives in `src/lib/`, not in API routes or components

## Step 3: Security Review
- Check for SQL injection (parameterized queries only)
- Check for XSS (no raw innerHTML without sanitization)
- Verify auth checks on all new API endpoints
- Check for PII exposure in logs or error messages

## Step 4: Test Coverage
- Every new function must have a corresponding test
- Edge cases: null inputs, empty arrays, boundary values
- Integration tests for new API endpoints
- Check that mocks are for external services only

## Step 5: Output
Format your review as:

### Summary
[1-2 sentences on what this PR does]

### Issues Found
- **Critical**: [must fix before merge]
- **Warning**: [should fix, not blocking]
- **Nit**: [style/preference, optional]

### Verdict
[APPROVE / REQUEST CHANGES / NEEDS DISCUSSION]
```

### 1.2 Frontmatter Deep Dive

The YAML frontmatter is how Claude Code discovers and configures your skill:

```yaml
---
# REQUIRED: How Claude Code matches this skill to user requests
# Write as a clear action description, not a label
description: Generate a new API endpoint with validation, tests, and documentation

# OPTIONAL: Tools this skill is allowed to use
# Restricts the agent to only these tools during skill execution
allowed-tools: Read, Write, Edit, Bash, Glob, Grep

# OPTIONAL: Tools this skill is NOT allowed to use
# Takes precedence over allowed-tools
disallowed-tools: WebSearch

# OPTIONAL: Additional tool definitions for this skill
tools:
  - name: check_api_docs
    description: Validate OpenAPI spec is up to date
    command: "npx @redocly/cli lint openapi.yaml"
---
```

### 1.3 Description Optimization

The `description` field determines whether Claude Code matches your skill to a user's request. It uses semantic matching, not exact keyword matching. Write descriptions that capture intent:

```yaml
# These will match "deploy to staging"
description: Deploy the current branch to the staging environment
description: Push the latest changes to staging for testing

# These will NOT reliably match "deploy to staging"
description: Deployment script runner        # too vague
description: staging                          # not a description
description: Uses kubectl and helm to deploy  # describes HOW, not WHAT
```

**Test your descriptions.** Try invoking the skill with different natural-language requests. If "deploy to staging," "push to staging," and "stage this branch" all find the right skill, your description is good.

### 1.4 Progressive Disclosure

Long skills should front-load the essential steps and back-load reference material. Claude Code reads skills top-to-bottom and can reference later sections as needed:

```markdown
---
description: Create a new database migration for schema changes
---

# Create Database Migration

## Quick Steps
1. Add or modify the schema in `src/db/schema/`
2. Run `pnpm db:generate` to create the migration
3. Run `pnpm db:push` to apply locally
4. Write a migration test (see reference below)
5. Update the Zod schema in `src/schemas/` to match

## Detailed Reference

### Naming Conventions
Tables: plural snake_case (user_accounts)
Columns: snake_case (created_at)
Indexes: idx_{table}_{columns} (idx_users_email)
Foreign keys: {table_singular}_id (account_id)

### Required Columns for All Tables
```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### Migration Test Template
```typescript
import { describe, it, expect } from 'vitest';
import { createTestDb, runMigrations } from '@/test/db-helpers';

describe('migration: {name}', () => {
  it('applies to empty database', async () => {
    const db = await createTestDb();
    await expect(runMigrations(db)).resolves.not.toThrow();
  });

  it('is idempotent', async () => {
    const db = await createTestDb();
    await runMigrations(db);
    await expect(runMigrations(db)).resolves.not.toThrow();
  });

  it('preserves existing data', async () => {
    const db = await createTestDb();
    await runMigrations(db); // run all previous migrations
    await db.insert(testData); // insert test data
    await runMigrations(db); // run new migration
    const rows = await db.select().from(table);
    expect(rows).toHaveLength(testData.length);
  });
});
```

### Common Mistakes
- Adding NOT NULL column without a default to an existing table
- Forgetting to update the Zod schema
- Not testing migration with existing data
- Using SERIAL instead of UUID for primary keys (we use UUID)
```

The agent reads the Quick Steps for the action plan. It references the Detailed Reference only when it needs specifics. This keeps the working context clean while making deep knowledge available.

---

## 2. Building Plugins

### 2.1 When to Graduate from Skill to Plugin

| Signal | Use a Skill | Use a Plugin |
|--------|------------|--------------|
| Single workflow | Yes | Overkill |
| Needs custom tools | No | Yes |
| Multiple related skills | No | Yes |
| Needs sub-agents | No | Yes |
| Needs hooks | No | Yes |
| Needs MCP server | No | Yes |
| Team distribution | Maybe | Yes |
| Cross-project reuse | Maybe | Yes |

### 2.2 Plugin Directory Structure

```
nelo-plugin/
|-- plugin.json                    <- Manifest: name, version, components
|-- README.md                      <- How to use this plugin
|-- CLAUDE.md                      <- Plugin-specific Claude context
|
|-- skills/                        <- Skill files
|   |-- deploy-staging.md
|   |-- create-endpoint.md
|   |-- review-pr.md
|   +-- run-evals.md
|
|-- agents/                        <- Sub-agent definitions
|   |-- code-reviewer.md
|   |-- test-writer.md
|   +-- doc-generator.md
|
|-- hooks/                         <- Behavior interceptors
|   |-- pre-commit-check.md
|   +-- post-edit-lint.md
|
|-- .mcp.json                      <- MCP server configurations
|
+-- scripts/                       <- Helper scripts used by hooks
    |-- lint-check.sh
    +-- test-runner.sh
```

### 2.3 The Plugin Manifest

```json
// plugin.json
{
  "name": "nelo-engineering",
  "version": "2.1.0",
  "description": "Engineering workflows, conventions, and automation for Nelo",
  "author": "Nelo Engineering <eng@nelo.co>",
  "homepage": "https://github.com/nelo-engineering/claude-plugin",
  "license": "MIT",

  "skills": [
    "skills/deploy-staging.md",
    "skills/create-endpoint.md",
    "skills/review-pr.md",
    "skills/run-evals.md"
  ],

  "agents": [
    "agents/code-reviewer.md",
    "agents/test-writer.md",
    "agents/doc-generator.md"
  ],

  "hooks": [
    "hooks/pre-commit-check.md",
    "hooks/post-edit-lint.md"
  ]
}
```

### 2.4 Plugin Agents in Detail

Plugin agents are sub-agents that the main Claude Code session can delegate to. They run in their own context, with their own system prompt and tool permissions:

```markdown
---
name: code-reviewer
description: Reviews code changes for quality, security, and convention compliance
when-to-use: When the user asks for a code review, PR review, or quality check
allowed-tools: Read, Grep, Glob, Bash
color: blue
---

# Code Reviewer Agent

You are a specialized code review agent for the Nelo codebase. Your reviews
are thorough, specific, and actionable.

## Review Priorities (in order)
1. **Security**: Auth checks, injection vulnerabilities, data exposure
2. **Correctness**: Logic errors, edge cases, race conditions
3. **Performance**: N+1 queries, unnecessary re-renders, memory leaks
4. **Conventions**: Code style, naming, file structure
5. **Clarity**: Readability, documentation, naming quality

## Nelo-Specific Rules
- All API routes must check `auth()` before any data access
- Database queries must use Drizzle ORM -- no raw SQL
- Error responses must use the `ApiError` class from `src/lib/errors.ts`
- Components must be memoized if they receive object props
- Feature-gated code must check `featureFlags.isEnabled()`, never env vars

## Output Format
For each issue found:
```
**[SEVERITY]** file.ts:L42 -- Description of issue

Suggestion: How to fix it
```

Severities: CRITICAL (blocks merge), WARNING (should fix), NIT (optional)

End with a summary: total issues by severity, overall verdict, and confidence.
```

### 2.5 Plugin Hooks

Hooks inside plugins use markdown for their configuration:

```markdown
---
event: PreToolUse
matcher: Bash
description: Validate bash commands against project safety rules
---

# Pre-Bash Validation

Before any Bash command runs, check:

1. **No production references**: Block any command containing `production`,
   `prod.`, or production hostnames
2. **No destructive git**: Block `git push --force`, `git reset --hard`,
   `git clean -f`
3. **No secret access**: Block commands that cat or echo .env files
4. **No package installs without lockfile**: Block `npm install` without
   `--save-exact`

If a violation is found, block the command and explain why.
If the command is safe, approve it silently.
```

### 2.6 Plugin MCP Configuration

Plugins can bundle MCP server configurations:

```json
// .mcp.json (at plugin root)
{
  "mcpServers": {
    "nelo-analytics": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/mcp-servers/analytics/dist/index.js"],
      "env": {
        "METABASE_URL": "${METABASE_URL}",
        "METABASE_TOKEN": "${METABASE_TOKEN}"
      }
    },
    "nelo-wiki": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/mcp-servers/wiki/dist/index.js"],
      "env": {
        "WIKI_URL": "${WIKI_URL}",
        "WIKI_TOKEN": "${WIKI_TOKEN}"
      }
    }
  }
}
```

The `${CLAUDE_PLUGIN_ROOT}` variable resolves to the plugin's installation directory. This lets MCP server paths work regardless of where the plugin is installed.

---

## 3. Git-Backed Skill Management

### 3.1 Why Git for Skills

Skills are code. They change, they break, they need review. Git gives you:

- **Version history**: See how a skill evolved
- **Code review**: PRs for skill changes get the same scrutiny as code changes
- **Rollback**: Revert a broken skill update
- **Attribution**: Know who wrote or modified each skill
- **Distribution**: Clone or pull to get the latest skills

### 3.2 Repository Structure

```
your-project/
|-- .claude/
|   |-- settings.json          <- Claude Code configuration
|   |-- settings.local.json    <- Local overrides (gitignored)
|   |-- skills/                <- Team-shared skills
|   |   |-- deploy.md
|   |   |-- review-pr.md
|   |   |-- create-component.md
|   |   +-- run-evals.md
|   +-- hooks/                 <- Team-shared hooks
|       |-- validate-bash.sh
|       +-- auto-lint.sh
|-- CLAUDE.md                  <- Project context
|-- src/
+-- ...
```

### 3.3 Reviewing Skill Changes

Treat skill changes like code changes. Create a PR review checklist:

```markdown
## Skill PR Checklist
- [ ] Description is clear and matches user intent
- [ ] Steps are specific, not vague
- [ ] Tool restrictions (allowed-tools) are appropriate
- [ ] No hardcoded secrets or paths
- [ ] Progressive disclosure: quick steps first, details later
- [ ] Tested with at least 3 different invocation phrases
- [ ] Does not duplicate an existing skill
```

### 3.4 Skill Versioning Patterns

When skills evolve, use clear naming and deprecation:

```
skills/
  deploy-v1.md              <- Original (deprecated)
  deploy.md                 <- Current version
  deploy-canary.md          <- Experimental new version
```

Or use frontmatter versioning:

```yaml
---
description: Deploy to staging environment
version: 2.1.0
deprecated: false
replaces: deploy-v1
changelog:
  - "2.1.0: Added health check verification after deploy"
  - "2.0.0: Switched from kubectl to Helm"
  - "1.0.0: Initial version"
---
```

---

## 4. Organizational Skills

### 4.1 Sharing Across a Team

For teams, skills flow through three channels:

**Channel 1: Repository skills** (most common)
```
project-repo/.claude/skills/    <- Everyone who clones the repo gets these
```

**Channel 2: Plugin distribution**
```json
// .claude/settings.json (in the project repo)
{
  "plugins": [
    "github:nelo-engineering/claude-plugin-nelo"
  ]
}
```
Everyone who uses Claude Code in this project automatically gets the plugin's skills.

**Channel 3: Organization-level distribution**
```
~/.claude/plugins/               <- Per-user plugin installations
```
For skills that should work across all projects, not just one repo.

### 4.2 The Org Admin Model

For companies using Claude Code at scale, the organization admin can set policies:

```json
// Organization settings (managed by admin)
{
  "org": {
    "name": "Nelo",
    "requiredPlugins": [
      "github:nelo-engineering/claude-plugin-nelo"
    ],
    "requiredSettings": {
      "permissions": {
        "deny": [
          "Bash(rm -rf *)",
          "Bash(git push --force *)",
          "Write(.env*)"
        ]
      }
    },
    "approvedMcpServers": [
      "@anthropic/mcp-slack",
      "@anthropic/mcp-linear",
      "@company/mcp-internal-wiki"
    ]
  }
}
```

This ensures every engineer at the company:
- Has the standard skills installed
- Cannot bypass security deny rules
- Only connects to approved MCP servers

### 4.3 Building a Skill Library

As your skill collection grows, organize it:

```
.claude/skills/
|-- README.md                    <- Skill index and descriptions
|
|-- development/                 <- Day-to-day coding
|   |-- create-component.md
|   |-- create-endpoint.md
|   |-- create-migration.md
|   +-- create-test.md
|
|-- review/                      <- Quality and review
|   |-- review-pr.md
|   |-- review-security.md
|   +-- review-performance.md
|
|-- deployment/                  <- Deployment and ops
|   |-- deploy-staging.md
|   |-- deploy-production.md
|   +-- rollback.md
|
|-- analysis/                    <- Data and investigation
|   |-- analyze-error.md
|   |-- analyze-performance.md
|   +-- investigate-incident.md
|
+-- team/                        <- Team workflows
    |-- standup.md
    |-- sprint-review.md
    +-- onboard-engineer.md
```

With a `README.md` index:

```markdown
# Team Skills

## Development
- `/create-component` -- Generate a new React component with tests and story
- `/create-endpoint` -- Create a new API endpoint with validation and docs
- `/create-migration` -- Create a database migration with proper testing
- `/create-test` -- Write tests for an existing file

## Review
- `/review-pr` -- Full PR review against team standards
- `/review-security` -- Security-focused review
- `/review-performance` -- Performance audit

## Deployment
- `/deploy-staging` -- Deploy current branch to staging
- `/deploy-production` -- Production deploy (requires approval)
- `/rollback` -- Rollback to previous deployment

## Analysis
- `/analyze-error` -- Investigate an error from logs/Datadog
- `/analyze-performance` -- Profile and optimize a slow endpoint
- `/investigate-incident` -- Run incident investigation playbook
```

---

## 5. Cron Jobs and Scheduled Agents

### 5.1 What Scheduled Agents Are

A scheduled agent is a Claude Code instance that runs on a cron schedule. It executes a task, produces output, and optionally notifies you. No human is sitting at the keyboard -- the agent runs autonomously.

### 5.2 Setting Up Scheduled Agents

```bash
# Create a scheduled agent that runs daily at 9am
claude schedule create \
  --name "daily-report" \
  --cron "0 9 * * *" \
  --task "Generate a daily engineering report: PR activity, CI status, open incidents" \
  --notify slack:#engineering-daily

# Create a weekly security scan
claude schedule create \
  --name "security-scan" \
  --cron "0 6 * * 1" \
  --task "Run security audit: check dependencies for vulnerabilities, review recent auth changes, scan for hardcoded secrets" \
  --notify slack:#security

# Create a bi-weekly dependency update
claude schedule create \
  --name "dependency-update" \
  --cron "0 10 1,15 * *" \
  --task "Check for outdated dependencies, create a PR updating safe minor/patch versions" \
  --auto-pr true

# List scheduled agents
claude schedule list

# View execution history
claude schedule history daily-report

# Pause/resume
claude schedule pause daily-report
claude schedule resume daily-report

# Delete
claude schedule delete daily-report
```

### 5.3 The Daily Report Agent

Here is a complete skill for a daily engineering report:

```markdown
---
description: Generate daily engineering report with PR activity, CI status, and incidents
allowed-tools: Bash, Read, Grep, Glob
---

# Daily Engineering Report

Generate a daily report covering the last 24 hours of engineering activity.

## Data Collection

### 1. Pull Request Activity
Run the following to get PRs merged in the last 24 hours:
    gh pr list --state merged --json title,author,mergedAt,number

And to get PRs currently open and awaiting review:
    gh pr list --state open --json title,author,createdAt,number,reviewRequests

### 2. CI/CD Status
Get recent workflow runs:
    gh run list --limit 20 --json name,status,conclusion,createdAt

Check for failures in the last 24 hours:
    gh run list --limit 50 --json name,status,conclusion,createdAt

### 3. Open Issues
Get issues created in the last 24 hours:
    gh issue list --state open --json title,author,createdAt,labels

## Report Format

    # Engineering Daily -- {date}

    ## PRs Merged ({count})
    {list with author and PR title}

    ## PRs Awaiting Review ({count})
    {list with author, title, and age}

    ## CI/CD
    - Passing: {count}
    - Failed: {count} -- {list failure names}

    ## New Issues ({count})
    {list with title and labels}

    ## Action Items
    {Any blocked PRs, persistent CI failures, or items needing attention}
```

### 5.4 The Dependency Update Agent

```markdown
---
description: Check for outdated dependencies and create update PRs
allowed-tools: Bash, Read, Write, Edit
---

# Dependency Update

## Step 1: Check for Outdated Packages
Run `pnpm outdated --format json` or `npm outdated --json`

## Step 2: Categorize Updates
- **Safe** (patch versions): Update automatically
- **Minor** (minor versions): Update, run tests, include in PR
- **Major** (major versions): List but do NOT auto-update

## Step 3: Apply Safe Updates
Run `pnpm update --no-major` or `npm update`

## Step 4: Verify
Run:
- `pnpm test` -- all tests must pass
- `pnpm build` -- must build successfully
- `pnpm lint` -- no lint errors

## Step 5: Create PR
If tests pass, create a PR with:
- Title: "chore: update dependencies ({date})"
- Body: list of all updated packages with version changes
- Label: "dependencies"

If tests fail, report which packages caused failures and do not create the PR.
```

---

## 6. Background Agents

### 6.1 What Background Agents Are

Background agents are Claude Code instances that run asynchronously. You start them, walk away, and come back to results. Unlike scheduled agents (which run on a timer), background agents run immediately -- they just do not block your terminal.

### 6.2 Using Background Agents

```bash
# Start a background agent
claude --background "Run the full test suite and identify any flaky tests"

# Start with a specific skill
claude --background "/run-evals search-quality"

# Start multiple background agents
claude --background "Refactor all API routes to use the new error handling pattern"
claude --background "Generate API documentation for all endpoints"
claude --background "Write missing tests for the payments module"

# Check on running agents
claude --list-background
# ID        Started          Status    Task
# bg-a1b2   2026-04-10 09:15 Running   Refactoring API routes...
# bg-c3d4   2026-04-10 09:15 Running   Generating API docs...
# bg-e5f6   2026-04-10 09:16 Complete  Writing payment tests

# Get results from a completed agent
claude --background-result bg-e5f6
```

### 6.3 Background Agent Best Practices

**Give clear, bounded tasks.** "Refactor the entire codebase" is not a good background task. "Refactor all API routes in src/app/api/ to use the new error handling pattern" is.

**Use worktrees.** Background agents modify files. If two background agents modify the same file, you get conflicts. Run them on separate worktrees:

```bash
# Create worktrees for parallel background work
git worktree add ../project-refactor-api feature/refactor-api
git worktree add ../project-add-tests feature/add-tests
git worktree add ../project-update-docs feature/update-docs

# Run background agents on each
(cd ../project-refactor-api && claude --background "Refactor API routes")
(cd ../project-add-tests && claude --background "Write missing tests")
(cd ../project-update-docs && claude --background "Update API docs")
```

**Set expectations for review.** Background agent output should be reviewed, not blindly merged. Treat it like a junior developer's PR -- the code is probably right, but verify.

---

## 7. Event-Driven AI Workflows

Sections 5 and 6 covered agents that run on a schedule or in the background. But real-world AI automation is often *reactive* -- something happens, and the agent responds. A Slack message arrives, a GitHub PR is opened, an email lands in an inbox, a payment fails. The agent needs to wake up, process the event, and take action. This is the "always-on agent" pattern, and it is how systems like OpenClaw work in practice.

### 7.1 The Pattern

```
External Event ──→ Webhook Endpoint ──→ Queue ──→ Agent Worker ──→ Action
                        │
                        ▼
                   Validate + filter
                   (ignore noise, only
                    process relevant events)
```

The key insight: you do not point a webhook directly at an LLM. You point it at a lightweight handler that validates the event, decides if it is worth processing, and enqueues it for an agent worker. The queue gives you reliability (retry on failure), backpressure (don't overwhelm the LLM), and observability (how many events are pending?).

### 7.2 Slack Message Triggers

This is the OpenClaw pattern. A user @-mentions the agent in Slack, and the agent processes the message and responds.

```typescript
// slack-agent-webhook.ts -- Hono handler for Slack events
import { Hono } from "hono";
import { Queue } from "./queue"; // Your queue (SQS, BullMQ, Upstash QStash, etc.)
import crypto from "crypto";

const app = new Hono();

// Slack sends events to this endpoint
app.post("/webhooks/slack", async (c) => {
  const body = await c.req.json();

  // Slack URL verification challenge (required during setup)
  if (body.type === "url_verification") {
    return c.json({ challenge: body.challenge });
  }

  // Verify the request is from Slack (IMPORTANT: prevents spoofed events)
  const signature = c.req.header("x-slack-signature");
  const timestamp = c.req.header("x-slack-request-timestamp");
  const sigBasestring = `v0:${timestamp}:${await c.req.text()}`;
  const expectedSignature = "v0=" + crypto
    .createHmac("sha256", process.env.SLACK_SIGNING_SECRET!)
    .update(sigBasestring)
    .digest("hex");

  if (signature !== expectedSignature) {
    return c.json({ error: "Invalid signature" }, 401);
  }

  // Filter: only process messages that @-mention the bot
  const event = body.event;
  if (event?.type !== "app_mention") {
    return c.json({ ok: true }); // Acknowledge but ignore
  }

  // Enqueue for async processing (don't block the webhook response)
  await Queue.enqueue("agent-tasks", {
    type: "slack_message",
    channel: event.channel,
    threadTs: event.thread_ts ?? event.ts,
    userId: event.user,
    text: event.text,
    timestamp: new Date().toISOString(),
  });

  // Respond to Slack within 3 seconds (required by Slack API)
  return c.json({ ok: true });
});

export default app;
```

### 7.3 GitHub Webhook Triggers

A PR is opened and the agent reviews it, or an issue is created and the agent triages it.

```typescript
// github-agent-webhook.ts
import { Hono } from "hono";
import { Queue } from "./queue";
import crypto from "crypto";

const app = new Hono();

app.post("/webhooks/github", async (c) => {
  // Verify GitHub webhook signature
  const signature = c.req.header("x-hub-signature-256");
  const rawBody = await c.req.text();
  const expectedSignature = "sha256=" + crypto
    .createHmac("sha256", process.env.GITHUB_WEBHOOK_SECRET!)
    .update(rawBody)
    .digest("hex");

  if (signature !== expectedSignature) {
    return c.json({ error: "Invalid signature" }, 401);
  }

  const event = c.req.header("x-github-event");
  const payload = JSON.parse(rawBody);

  // Route different GitHub events to different agent tasks
  switch (event) {
    case "pull_request":
      if (payload.action === "opened" || payload.action === "synchronize") {
        await Queue.enqueue("agent-tasks", {
          type: "pr_review",
          repo: payload.repository.full_name,
          prNumber: payload.pull_request.number,
          prTitle: payload.pull_request.title,
          prBody: payload.pull_request.body,
          author: payload.pull_request.user.login,
        });
      }
      break;

    case "issues":
      if (payload.action === "opened") {
        await Queue.enqueue("agent-tasks", {
          type: "issue_triage",
          repo: payload.repository.full_name,
          issueNumber: payload.issue.number,
          title: payload.issue.title,
          body: payload.issue.body,
          labels: payload.issue.labels.map((l: { name: string }) => l.name),
        });
      }
      break;
  }

  return c.json({ ok: true });
});
```

### 7.4 The Agent Worker

The webhook handlers enqueue tasks. The worker dequeues and processes them. This separation is critical -- it gives you retries, rate limiting, and the ability to scale workers independently.

```typescript
// agent-worker.ts
import Anthropic from "@anthropic-ai/sdk";
import { Queue, type AgentTask } from "./queue";

const anthropic = new Anthropic();

async function processTask(task: AgentTask) {
  switch (task.type) {
    case "slack_message":
      return processSlackMessage(task);
    case "pr_review":
      return processPRReview(task);
    case "issue_triage":
      return processIssueTriage(task);
    default:
      console.warn(`Unknown task type: ${(task as { type: string }).type}`);
  }
}

async function processSlackMessage(task: {
  type: "slack_message";
  channel: string;
  threadTs: string;
  userId: string;
  text: string;
}) {
  // Strip the @mention to get the actual question
  const question = task.text.replace(/<@[A-Z0-9]+>/g, "").trim();

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: "You are a helpful engineering assistant. Answer concisely. Use Slack-compatible markdown.",
    messages: [{ role: "user", content: question }],
  });

  const answer = (response.content[0] as { type: "text"; text: string }).text;

  // Post the response back to the Slack thread
  await fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.SLACK_BOT_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      channel: task.channel,
      thread_ts: task.threadTs, // Reply in thread, not the main channel
      text: answer,
    }),
  });
}

async function processPRReview(task: {
  type: "pr_review";
  repo: string;
  prNumber: number;
  prTitle: string;
  prBody: string;
  author: string;
}) {
  // Fetch the PR diff using GitHub API
  const diffResponse = await fetch(
    `https://api.github.com/repos/${task.repo}/pulls/${task.prNumber}`,
    {
      headers: {
        "Authorization": `Bearer ${process.env.GITHUB_TOKEN}`,
        "Accept": "application/vnd.github.diff",
      },
    }
  );
  const diff = await diffResponse.text();

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: `You are a code reviewer. Review this PR and provide actionable feedback.
Focus on: bugs, security issues, performance problems, and maintainability.
Be specific -- reference line numbers and file names from the diff.
Format as a GitHub-compatible markdown comment.`,
    messages: [{
      role: "user",
      content: `PR #${task.prNumber}: ${task.prTitle}\n\nDescription: ${task.prBody}\n\nDiff:\n${diff}`,
    }],
  });

  const review = (response.content[0] as { type: "text"; text: string }).text;

  // Post the review as a PR comment
  await fetch(
    `https://api.github.com/repos/${task.repo}/issues/${task.prNumber}/comments`,
    {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${process.env.GITHUB_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ body: review }),
    }
  );
}

// Worker loop: pull tasks from queue and process them
async function startWorker() {
  console.log("Agent worker started, waiting for tasks...");

  while (true) {
    const task = await Queue.dequeue("agent-tasks");
    if (!task) {
      // No tasks available -- wait briefly before polling again
      await new Promise(resolve => setTimeout(resolve, 1000));
      continue;
    }

    try {
      await processTask(task);
      await Queue.acknowledge(task); // Mark as successfully processed
    } catch (error) {
      console.error(`Failed to process task:`, error);
      // Queue will automatically retry after visibility timeout
      await Queue.nack(task); // Return to queue for retry
    }
  }
}

startWorker();
```

### 7.5 The n8n Automation Layer

For teams that do not want to write webhook handlers from scratch, n8n is a workflow automation tool that connects triggers to actions with a visual editor. It is self-hostable, has hundreds of integrations, and can spawn agent executions as a workflow step.

The pattern: n8n handles the webhook plumbing (Slack, GitHub, email, calendar, CRM) and your agent handles the intelligence.

```
n8n Workflow:
  Trigger: Slack message in #support channel
  → Filter: Only messages with "help" or "question"
  → HTTP Request: POST to your agent API with the message
  → Slack: Post agent response back to thread
```

This is powerful because non-engineers can create new agent triggers by building n8n workflows without touching code. The agent API stays the same -- only the triggers change.

### 7.6 Error Handling and Retry

Event-driven agents fail. The LLM times out, the Slack API is down, the GitHub token is expired. Your error handling strategy determines whether failures are annoying or catastrophic.

**Retry strategy:**
- **Immediate retry (1x):** Network glitches, transient API errors
- **Exponential backoff (3x):** Rate limits, temporary outages
- **Dead letter queue:** After all retries fail, move to a DLQ for human review
- **Idempotency:** The same event processed twice should produce the same result (or be safely ignored)

```typescript
// retry-config.ts -- example for BullMQ
const queueConfig = {
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: "exponential",
      delay: 5000, // 5s, 10s, 20s
    },
    removeOnComplete: { count: 1000 },  // Keep last 1000 completed jobs
    removeOnFail: { count: 5000 },      // Keep last 5000 failed jobs for debugging
  },
};
```

### 7.7 The Always-On Agent Pattern

The most advanced version of event-driven AI is the "always-on agent" -- an agent that continuously monitors one or more channels and acts autonomously. This is not a webhook that fires on each message. It is a persistent process that maintains context across interactions.

```
Always-On Agent:
  1. Connects to Slack via WebSocket (RTM or Socket Mode)
  2. Listens to configured channels
  3. Maintains conversation context per thread
  4. Responds to @-mentions and keyword triggers
  5. Proactively surfaces insights ("I noticed 3 PRs have been open for 5+ days")
  6. Runs background checks on a timer (CI status, dependency alerts)
```

The trust equation from Section 9 applies here: start with an agent that only responds when @-mentioned. Once you trust it, let it monitor channels passively. Only after extensive testing let it take autonomous actions (creating tickets, posting alerts, merging PRs).

---

## 8. The /ask-nelo Pattern: Company Knowledge Skill

### 8.1 The Problem

Your company has knowledge scattered everywhere: wiki pages, Slack conversations, GitHub issues, Linear tickets, incident postmortems, architecture decision records. When a new engineer asks "how does our payment system work?", the answer is spread across 12 different places.

### 8.2 The Pattern

Build a skill that gives Claude Code access to all of these knowledge sources and lets any engineer ask questions in natural language:

```markdown
---
description: Ask questions about Nelo's codebase, architecture, processes, and conventions
allowed-tools: Read, Grep, Glob, Bash, mcp_wiki_search, mcp_wiki_read, mcp_slack_search
---

# Ask Nelo

Answer questions about Nelo using all available knowledge sources.

## Knowledge Sources (check in order)

### 1. Codebase
- Read CLAUDE.md for project overview
- Search code with Grep/Glob for implementation details
- Read relevant source files

### 2. Documentation
- Search the internal wiki using wiki MCP server
- Check `docs/` directory for architecture docs
- Read ADRs in `docs/adr/`

### 3. Historical Context
- Search Slack for relevant discussions (engineering channels)
- Check Linear for related tickets and their context
- Check GitHub issues for prior art and decisions

## Answer Format
1. **Direct answer** -- answer the question concisely
2. **Sources** -- list where you found the information
3. **Context** -- any relevant caveats, history, or related topics
4. **Confidence** -- HIGH (multiple sources agree), MEDIUM (single source
   or inferred), LOW (mostly guessing, recommend verifying with team)

## Example Interactions
- "How does our payment retry logic work?"
- "What's the process for adding a new microservice?"
- "Why did we choose Drizzle over Prisma?"
- "Who should I talk to about the billing system?"
- "What happened in the last production incident?"
```

### 8.3 Why This Works

The `/ask-nelo` pattern is a force multiplier because:

- **New engineers** become productive faster -- they can ask questions without finding the right person
- **Experienced engineers** save time -- they do not have to answer the same questions repeatedly
- **The knowledge is always current** -- it queries live sources, not stale documentation
- **It crosses silos** -- code, docs, Slack, and tickets are all queried together

### 8.4 Making It Better Over Time

Track which questions get LOW confidence answers. Those are knowledge gaps you need to fill:

```markdown
## When Confidence is LOW
If you cannot find a confident answer:
1. Say so explicitly
2. Suggest who to ask (check git blame, Slack mentions, Linear assignments)
3. Log the question to `.claude/knowledge-gaps.md` for future documentation
```

Over time, the knowledge gaps log tells you exactly what documentation to write.

---

## 9. The /council Pattern: Multi-Model Adversarial Review

### 9.1 Inspiration

Andrej Karpathy described a pattern where you use multiple AI models to review each other's work. The idea: no single model catches all issues. By having multiple models (or the same model with different personas) review code adversarially, you catch more problems.

### 9.2 The Pattern

```markdown
---
description: Run a multi-perspective adversarial review on code changes or designs
allowed-tools: Read, Grep, Glob, Bash
---

# Council Review

Run a council review with three adversarial perspectives on the current changes.

## Perspective 1: The Security Auditor

Review the changes purely from a security perspective:
- Authentication and authorization checks
- Input validation and sanitization
- Data exposure risks
- Injection vulnerabilities (SQL, XSS, command injection)
- Secret management
- Rate limiting and abuse prevention

Be paranoid. Assume every input is malicious. Find the exploit.

## Perspective 2: The Performance Engineer

Review the changes purely from a performance perspective:
- Query efficiency (N+1, missing indexes, full table scans)
- Memory usage (large object allocation, missing cleanup)
- Rendering performance (unnecessary re-renders, missing memoization)
- Network calls (parallelization, caching, request waterfall)
- Concurrency issues (race conditions, deadlocks)

Assume this code will handle 100x current traffic. Find the bottleneck.

## Perspective 3: The Maintainability Advocate

Review the changes from a long-term maintainability perspective:
- Code clarity (is the intent obvious?)
- Abstraction quality (right level? too much? too little?)
- Test coverage (can someone modify this safely in 6 months?)
- Documentation (would a new hire understand this?)
- Consistency with existing patterns (does this fit?)

Assume the author will leave the company tomorrow. Find what will confuse
their successor.

## Synthesis

After running all three perspectives, synthesize:
1. **Critical issues** (things all perspectives agree are problems)
2. **Tension points** (where perspectives disagree -- these are the most
   interesting)
3. **Verdict** (overall assessment with reasoning)
```

### 9.3 Running the Council

You can run the council manually:

```
You: /council
```

Or integrate it as a pre-merge step:

```bash
# In your CI/CD pipeline
claude --auto --dangerously-skip-permissions \
  "/council" \
  --context "$(gh pr diff)" \
  --output council-review.md
```

### 9.4 Why Adversarial Review Works

Single-perspective review has blind spots. A security expert misses performance issues. A performance engineer misses security issues. The council pattern forces **systematic coverage** of different risk dimensions.

The real value is in the tension points -- when the security perspective says "add more validation" but the performance perspective says "that validation is expensive." Those tensions are where the real engineering decisions live.

---

## 10. Automation Philosophy

### 10.1 The Automation Spectrum

```
Manual <----------------------------------------------> Fully Autonomous

  Human types     Human invokes     Agent runs on      Agent runs on
  every command   a skill           schedule, human    schedule, auto-
  every time      (/deploy)         reviews output     merges and deploys

  Week 1          Week 2            Month 1            Month 3+
```

Move right gradually. Trust but verify. Automate the boring stuff first.

### 10.2 What to Automate First

| Task | Automate? | Why |
|------|-----------|-----|
| Code formatting/linting | Yes -- PostToolUse hook | Zero-risk, always correct |
| Dependency updates (patch) | Yes -- scheduled agent | Low-risk, easy to verify |
| Test generation | Semi -- background agent, review output | Medium-risk, needs human check |
| Code review | Semi -- /council as assist, not gatekeeper | Medium-risk, augments human |
| Deployment to staging | Yes -- after tests pass | Low-risk, revertible |
| Deployment to production | No -- human approval required | High-risk |
| Database migrations | No -- human review required | Irreversible |

### 10.3 The Trust Equation

```
Automation Trust = (Consistency x Reversibility) / Impact

High trust:     Linting (consistent, reversible, low impact)
Medium trust:   Test generation (consistent, code is reviewable)
Low trust:      Production deploy (high impact, not easily reversed)
Never auto:     Data deletion (irreversible, catastrophic impact)
```

---

## 11. Evaluating Skill Quality

You've built skills, packaged them into plugins, and shared them with your team. But how do you know a skill is *good*? A skill that works for the author might fail for different users, different inputs, or different codebases. This section builds the evaluation framework for skills, spiraling back from Ch 21 (eval-driven development) and forward to Ch 60 (skills marketplace quality gates).

### 11.1 Testing if a Skill Consistently Produces Good Output

A skill is not a function with a deterministic return value. It's a set of instructions that an LLM interprets. The same skill can produce different results depending on:

- The model's interpretation of ambiguous instructions
- The state of the codebase when the skill runs
- The specificity of the user's invocation
- Context window pressure (what else is loaded)

This means skill testing looks more like eval testing than unit testing.

```typescript
// skill-eval.ts — testing a skill against known scenarios

interface SkillEvalCase {
  id: string;
  invocation: string;             // how the user triggers the skill
  codebaseState: string;          // description or path to fixture repo
  expectedBehaviors: string[];    // what MUST happen
  forbiddenBehaviors: string[];   // what must NOT happen
  outputChecks: {
    type: "contains" | "format" | "file_created" | "file_modified" | "command_ran";
    value: string;
  }[];
}

const reviewPRCases: SkillEvalCase[] = [
  {
    id: "review-pr-sql-injection",
    invocation: "/review-pr",
    codebaseState: "fixtures/pr-with-raw-sql",
    expectedBehaviors: [
      "Identifies the raw SQL query as a security issue",
      "Flags it as CRITICAL severity",
      "Suggests using parameterized queries or Drizzle ORM",
    ],
    forbiddenBehaviors: [
      "Approves the PR without mentioning the SQL issue",
      "Misidentifies the raw SQL as an ORM query",
    ],
    outputChecks: [
      { type: "contains", value: "SQL" },
      { type: "contains", value: "CRITICAL" },
      { type: "contains", value: "REQUEST CHANGES" },
    ],
  },
  {
    id: "review-pr-clean",
    invocation: "/review-pr",
    codebaseState: "fixtures/pr-clean-code",
    expectedBehaviors: [
      "Recognizes the PR follows conventions",
      "Approves or has only minor nits",
    ],
    forbiddenBehaviors: [
      "Flags false positive security issues",
      "Requests changes on compliant code",
    ],
    outputChecks: [
      { type: "contains", value: "APPROVE" },
    ],
  },
];
```

### 11.2 Building Eval Cases for Skills

Every skill should have eval cases that cover:

**Happy path** — the skill works as intended on normal input.

**Edge cases** — unusual but valid inputs (empty PR, single-file change, massive diff).

**Adversarial cases** — inputs designed to confuse the skill (conflicting conventions, ambiguous instructions).

**Regression cases** — inputs that previously produced bad output (add these as you find failures).

```yaml
# evals/skills/deploy-staging.yaml
cases:
  - id: deploy-clean-branch
    invocation: "/deploy-staging"
    context: "Branch is clean, all tests pass, no merge conflicts"
    expected:
      - "Runs the deploy command"
      - "Verifies health check after deploy"
      - "Reports success with deployment URL"
    tags: [smoke, happy-path]

  - id: deploy-dirty-branch
    invocation: "/deploy-staging"
    context: "Branch has uncommitted changes"
    expected:
      - "Warns about uncommitted changes"
      - "Asks for confirmation before proceeding OR refuses to deploy"
    forbidden:
      - "Deploys without warning about uncommitted changes"
    tags: [edge-case]

  - id: deploy-failing-tests
    invocation: "/deploy-staging"
    context: "Branch has 3 failing tests in the payments module"
    expected:
      - "Identifies the failing tests"
      - "Refuses to deploy OR asks for explicit override"
    forbidden:
      - "Deploys without mentioning test failures"
    tags: [edge-case, safety]
```

### 11.3 Automated Skill Testing in CI

Run skill evals in CI whenever a skill file changes. This catches regressions before they reach the team.

```yaml
# .github/workflows/skill-evals.yml
name: Skill Evals

on:
  pull_request:
    paths:
      - '.claude/skills/**'
      - 'evals/skills/**'

jobs:
  eval-skills:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - run: npm ci

      - name: Run skill evals
        run: npx tsx evals/skills/run.ts --changed-only
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          EVAL_BUDGET_CENTS: "500"
        timeout-minutes: 15

      - name: Post eval results to PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('skill-eval-results.json', 'utf-8'));

            let body = `## Skill Eval Results\n\n`;
            body += `| Skill | Cases | Passed | Score |\n|-------|-------|--------|-------|\n`;
            for (const skill of results.skills) {
              const icon = skill.passRate >= 0.8 ? '✓' : '✗';
              body += `| ${icon} ${skill.name} | ${skill.total} | ${skill.passed} | ${(skill.avgScore * 100).toFixed(0)}% |\n`;
            }

            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
```

### 11.4 Quality Metrics for Skills

Track these metrics for every skill:

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| **Task completion rate** | Does the skill accomplish its stated goal? | > 90% |
| **Output format compliance** | Does the output match the expected format? | > 95% |
| **Instruction adherence** | Does the skill follow all its own steps? | > 85% |
| **Safety compliance** | Does the skill respect tool restrictions and deny rules? | 100% |
| **User satisfaction** | Do users invoke the skill again (retention)? | > 70% |
| **Invocation match rate** | Does the right skill activate for the right request? | > 90% |

```typescript
// skill-metrics.ts — track skill quality over time

interface SkillMetrics {
  skillId: string;
  period: string;           // "2026-W15"
  invocations: number;
  completions: number;       // successfully finished
  errors: number;            // crashed or timed out
  userRetries: number;       // user re-invoked after seeing output (signal of dissatisfaction)
  avgDurationMs: number;
  taskCompletionRate: number; // completions / invocations
  formatComplianceRate: number;
}

function assessSkillHealth(metrics: SkillMetrics): {
  status: "healthy" | "degraded" | "broken";
  issues: string[];
} {
  const issues: string[] = [];

  if (metrics.taskCompletionRate < 0.9) {
    issues.push(`Task completion rate ${(metrics.taskCompletionRate * 100).toFixed(0)}% (target: 90%)`);
  }
  if (metrics.errors / metrics.invocations > 0.05) {
    issues.push(`Error rate ${((metrics.errors / metrics.invocations) * 100).toFixed(1)}% (target: < 5%)`);
  }
  if (metrics.userRetries / metrics.invocations > 0.3) {
    issues.push(`High retry rate ${((metrics.userRetries / metrics.invocations) * 100).toFixed(0)}% — users may be dissatisfied`);
  }

  if (issues.length === 0) return { status: "healthy", issues };
  if (issues.length <= 2) return { status: "degraded", issues };
  return { status: "broken", issues };
}
```

### 11.5 The Skill Review Checklist

Before publishing a skill to a marketplace (Ch 60) or sharing it with a team, run through this checklist:

```markdown
## Skill Quality Checklist

### Description & Discovery
- [ ] Description clearly states what the skill does (not how)
- [ ] Tested with 3+ different invocation phrases — all match correctly
- [ ] Does not collide with existing skills in the same namespace

### Content & Instructions
- [ ] Steps are specific and actionable — no vague instructions
- [ ] Progressive disclosure: quick steps first, reference material later
- [ ] No hardcoded paths, secrets, or environment-specific values
- [ ] Tool restrictions (allowed-tools) are appropriately scoped

### Quality & Testing
- [ ] Eval cases exist for happy path, edge cases, and at least one adversarial case
- [ ] Eval pass rate >= 80% across all cases
- [ ] Tested on a clean codebase (not just the author's environment)
- [ ] Output format is consistent across multiple runs

### Safety & Security
- [ ] Does not expose secrets or PII in output
- [ ] Respects deny rules and permission boundaries
- [ ] Cannot be tricked into running destructive commands
- [ ] Uses minimum necessary tool permissions

### Documentation
- [ ] CHANGELOG entry for version changes
- [ ] Known limitations documented
- [ ] Example invocations included
```

---

## 12. Putting It All Together

### 12.1 The Complete Setup

Here is a complete `.claude/` directory that implements everything in this chapter:

```
.claude/
|-- settings.json
|-- settings.local.json          (gitignored)
|
|-- skills/
|   |-- development/
|   |   |-- create-component.md
|   |   |-- create-endpoint.md
|   |   +-- create-migration.md
|   |
|   |-- review/
|   |   |-- review-pr.md
|   |   |-- review-security.md
|   |   +-- council.md
|   |
|   |-- ops/
|   |   |-- deploy-staging.md
|   |   |-- daily-report.md
|   |   +-- investigate-incident.md
|   |
|   +-- knowledge/
|       +-- ask-nelo.md
|
|-- hooks/
|   |-- validate-bash.sh
|   |-- auto-lint.sh
|   +-- session-summary.sh
|
+-- plugins/
    +-- nelo-engineering/         (symlink or git submodule)
```

### 12.2 The Team Adoption Path

```
Week 1:   One engineer sets up CLAUDE.md and 2-3 skills
Week 2:   Team tries it, gives feedback, skills get refined
Week 3:   Add hooks for code quality automation
Week 4:   Connect MCP servers for data access
Month 2:  Build a plugin, distribute to team
Month 3:  Add scheduled agents for routine reports
Month 6:  Review automation -- what worked, what didn't, what to add next
```

### 12.3 Measuring Success

Track these metrics for your skills and automations:

- **Skill usage**: How often is each skill invoked? Low-use skills should be improved or removed.
- **Time saved**: Estimate time per manual invocation vs. skill invocation. Multiply by frequency.
- **Error reduction**: Compare defect rates before/after council reviews.
- **Coverage**: What percentage of your team's workflows have a corresponding skill?
- **Freshness**: When was each skill last updated? Stale skills drift from reality.

---

## Key Takeaways

This chapter covered four progressively powerful layers:

1. **Skills** encode your team's knowledge into reusable, shareable workflows. The key is writing descriptions that match intent, using progressive disclosure, and treating skills as code (git, review, version).

2. **Plugins** package related skills, agents, hooks, and MCP configurations into distributable units. Build a plugin when you have a coherent set of workflows that should travel together.

3. **Automation** removes you from the loop. Scheduled agents run reports. Background agents do bulk work. The trust equation tells you what to automate: high consistency, high reversibility, low impact.

4. **Event-driven workflows** make agents reactive. Slack messages, GitHub webhooks, and external events trigger agent execution through a queue-based architecture that gives you reliability, retry, and observability.

The patterns that make this real -- `/ask-nelo` for knowledge, `/council` for review, daily reports for visibility, event-driven agents for always-on responsiveness -- are not theoretical. They are working systems at companies using Claude Code today.

The next chapter (Ch 26) zooms out from your personal and team workflow to the industry level: how companies like Stripe, Ramp, and Anthropic are building coding agent systems that make entire organizations more productive.

---

*Previous: [Chapter 24 -- MCP Servers & Integrations](./24-mcp-servers.md)* | *Next: [Chapter 26 -- AI-Augmented Development](./26-ai-augmented-dev.md)*

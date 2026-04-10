<!--
  CHAPTER: 24
  TITLE: MCP Servers & Integrations
  PART: 5 — Harness Engineering
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 8 (tool calling), Ch 23 (Claude Code basics)
  KEY_TOPICS: mcp, model-context-protocol, mcp-servers, tools, resources, prompts, security, typescript
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript, JSON
  UPDATED: 2026-04-10
-->

# Chapter 24: MCP Servers & Integrations

> Part 5: Harness Engineering · Phase 1: Get Dangerous · Prereqs: Ch 8, 23 · Difficulty: Intermediate · Language: TypeScript/JSON

In Chapter 8 you learned how to give an LLM tools — functions it can call to interact with the world. In Chapter 23 you learned that Claude Code is an agent loop built around those tools. Now the question becomes: **what if you could give Claude Code access to every system your company uses?**

Your database. Your analytics dashboard. Your feature flag system. Your CI/CD pipeline. Your Slack workspace. Your Linear board. Your documentation wiki.

That is what MCP does. The Model Context Protocol is a standard for connecting AI agents to external systems. Instead of every AI tool building its own bespoke integrations, MCP defines a common protocol — and any MCP server can work with any MCP client. Build one server for your Metabase instance and it works with Claude Code, Cursor, Windsurf, and any other tool that speaks MCP.

### In This Chapter

1. What MCP is and why it exists
2. How MCP works: the protocol
3. Connecting existing MCP servers
4. Building your own MCP server from scratch
5. MCP server security
6. The MCP ecosystem

### Related Chapters

| Direction | Chapter | Connection |
|-----------|---------|------------|
| ↩ spirals back | Ch 8: Tool Calling | MCP servers ARE tools — same concept, standardized protocol |
| ↩ spirals back | Ch 15: The RAG Pipeline | MCP can serve as knowledge retrieval layer |
| ↩ spirals back | Ch 23: Claude Code Mastery | MCP servers plug into Claude Code via settings |
| ↪ spirals forward | Ch 25: Skills & Automation | Skills can orchestrate MCP server calls |
| ↪ spirals forward | Ch 30: Tool Execution Internals | How MCP tools are resolved and executed |
| ↪ spirals forward | Ch 31: Web Search Pipelines | Web search implemented as MCP |
| ↪ spirals forward | Ch 59: Internal AI Tools | MCP servers as platform building blocks |

---

## 1. What MCP Is and Why It Exists

### 1.1 The Integration Problem

Before MCP, every AI tool built its own integrations:

```
Claude Code ──→ custom Slack integration
Cursor      ──→ different Slack integration
Windsurf    ──→ yet another Slack integration
Copilot     ──→ still another Slack integration
```

Four AI tools, four Slack integrations, all doing the same thing slightly differently. Now multiply by every system your company uses: Slack, Linear, Jira, GitHub, Datadog, Metabase, Notion, Google Docs, your internal APIs. The combinatorial explosion is unsustainable.

### 1.2 MCP: One Protocol to Connect Them All

MCP inverts the relationship:

```
                    ┌─────────────────┐
                    │   MCP Server:   │
Claude Code  ──→   │     Slack       │   ←── Any MCP Client
Cursor       ──→   │                 │
Windsurf     ──→   └─────────────────┘
Copilot      ──→
                    ┌─────────────────┐
                    │   MCP Server:   │
Claude Code  ──→   │    Metabase     │   ←── Any MCP Client
Cursor       ──→   │                 │
Windsurf     ──→   └─────────────────┘
```

One Slack MCP server. All AI tools can use it. One Metabase MCP server. All AI tools can use it. Build once, use everywhere.

### 1.3 The Mental Model

If you understand REST APIs, you understand MCP:

| REST API | MCP Equivalent | What It Does |
|----------|---------------|--------------|
| Endpoint | Tool | An action the AI can take |
| GET endpoint | Resource | Data the AI can read |
| API docs | Prompt | Templates the AI can use |
| Auth header | Server config | Credentials and connection |

An MCP server is essentially a specialized API that speaks a protocol AI tools understand natively. Instead of the AI needing to figure out how to call a REST API (construct URLs, handle auth, parse responses), it calls an MCP tool just like any other tool — same interface it uses for reading files or running bash commands.

---

## 2. How MCP Works: The Protocol

### 2.1 Architecture

```
┌────────────────┐         ┌──────────────────┐
│   MCP Client   │  stdio  │   MCP Server     │
│  (Claude Code) │ ──────→ │  (your server)   │
│                │ ←────── │                   │
│  Discovers:    │         │  Provides:        │
│  - tools       │         │  - tools          │
│  - resources   │         │  - resources      │
│  - prompts     │         │  - prompts        │
└────────────────┘         └──────────────────┘
                                    │
                                    │ HTTP/SDK
                                    ▼
                           ┌──────────────────┐
                           │  External System  │
                           │  (Slack, DB, etc) │
                           └──────────────────┘
```

MCP uses a client-server architecture with JSON-RPC over stdio (for local servers) or HTTP with Server-Sent Events (for remote servers). The client (Claude Code) discovers what the server offers and presents those capabilities to the model as available tools.

### 2.2 The Three Primitives

MCP servers expose three types of capabilities:

**Tools** — Actions the AI can take:
```json
{
  "name": "query_database",
  "description": "Run a read-only SQL query against the analytics database",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "SQL SELECT query to execute"
      }
    },
    "required": ["query"]
  }
}
```

**Resources** — Data the AI can read:
```json
{
  "uri": "metabase://dashboards/revenue",
  "name": "Revenue Dashboard",
  "description": "Current month revenue metrics and trends",
  "mimeType": "application/json"
}
```

**Prompts** — Templates the AI can use:
```json
{
  "name": "analyze_metrics",
  "description": "Analyze a specific business metric with historical context",
  "arguments": [
    {
      "name": "metric_name",
      "description": "The metric to analyze (e.g., 'MRR', 'churn', 'DAU')",
      "required": true
    }
  ]
}
```

### 2.3 The Lifecycle

```
1. CLIENT starts SERVER process
   └── claude code runs: npx @company/mcp-metabase

2. CLIENT sends "initialize" request
   └── server responds with capabilities and protocol version

3. CLIENT sends "tools/list" request
   └── server responds with available tools and their schemas

4. CLIENT sends "resources/list" request
   └── server responds with available resources

5. CLIENT sends "prompts/list" request
   └── server responds with available prompt templates

6. MODEL decides to use a tool
   └── client sends "tools/call" with tool name and arguments
   └── server executes and returns results

7. Repeat step 6 as needed

8. CLIENT shuts down SERVER process
```

### 2.4 Transport Mechanisms

**stdio (local servers):**
```
Claude Code ──stdin──→ MCP Server Process
Claude Code ←stdout── MCP Server Process
```

The server runs as a child process. Communication happens over stdin/stdout using JSON-RPC. This is the most common transport for local development.

**HTTP + SSE (remote servers):**
```
Claude Code ──HTTP POST──→ Remote MCP Server
Claude Code ←───SSE────── Remote MCP Server
```

For servers that run on a remote host. The client sends requests via HTTP POST and receives streaming responses via Server-Sent Events.

---

## 3. Connecting Existing MCP Servers

### 3.1 Configuration in Claude Code

MCP servers are configured in your settings.json:

```json
// .claude/settings.json
{
  "mcpServers": {
    "metabase": {
      "command": "npx",
      "args": ["-y", "@company/mcp-metabase"],
      "env": {
        "METABASE_URL": "https://metabase.internal.company.com",
        "METABASE_TOKEN": "{{secrets.METABASE_TOKEN}}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "{{secrets.SLACK_BOT_TOKEN}}"
      }
    },
    "linear": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-linear"],
      "env": {
        "LINEAR_API_KEY": "{{secrets.LINEAR_API_KEY}}"
      }
    }
  }
}
```

### 3.2 Popular MCP Servers

Here is a practical selection of MCP servers that cover the most common engineering workflows:

**Analytics & Data:**

```json
{
  "metabase": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-metabase"],
    "env": {
      "METABASE_URL": "https://metabase.company.com",
      "METABASE_TOKEN": "{{secrets.METABASE_TOKEN}}"
    }
  }
}
```

With Metabase connected, you can ask Claude Code: "What was our revenue trend last quarter?" and it will query Metabase dashboards directly, interpret the data, and give you an answer with context.

**Feature Flags:**

```json
{
  "growthbook": {
    "command": "npx",
    "args": ["-y", "@company/mcp-growthbook"],
    "env": {
      "GROWTHBOOK_API_KEY": "{{secrets.GROWTHBOOK_KEY}}",
      "GROWTHBOOK_URL": "https://growthbook.company.com"
    }
  }
}
```

"Which feature flags are enabled in production?" "What's the rollout percentage for the new checkout flow?" Claude Code queries GrowthBook and tells you.

**Monitoring:**

```json
{
  "datadog": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-datadog"],
    "env": {
      "DD_API_KEY": "{{secrets.DD_API_KEY}}",
      "DD_APP_KEY": "{{secrets.DD_APP_KEY}}"
    }
  }
}
```

"Are there any elevated error rates in the payments service?" "Show me the P99 latency for the search endpoint over the last hour." Claude Code checks Datadog and gives you operational intelligence.

**Project Management:**

```json
{
  "linear": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-linear"],
    "env": {
      "LINEAR_API_KEY": "{{secrets.LINEAR_API_KEY}}"
    }
  }
}
```

"What's assigned to me this sprint?" "Create a bug ticket for the checkout race condition." Claude Code reads from and writes to Linear.

**Communication:**

```json
{
  "slack": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-slack"],
    "env": {
      "SLACK_BOT_TOKEN": "{{secrets.SLACK_BOT_TOKEN}}"
    }
  }
}
```

"Summarize what happened in #engineering today." "Post a deployment notification to #releases." Claude Code reads and sends Slack messages.

**Search:**

```json
{
  "google-cli": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-google"],
    "env": {
      "GOOGLE_API_KEY": "{{secrets.GOOGLE_API_KEY}}",
      "GOOGLE_CX": "{{secrets.GOOGLE_CX}}"
    }
  }
}
```

### 3.3 Using MCP Tools in Practice

Once configured, MCP tools appear alongside built-in tools. You do not need special syntax — just ask for what you need:

```
You: "What's the error rate for the checkout service right now?"

Claude Code: Let me check Datadog for the current checkout service metrics.

[Tool: datadog.get_metrics]
  query: "avg:http.error_rate{service:checkout}"
  timeframe: "1h"

The checkout service error rate is currently 0.3%, which is within
normal range. The P99 latency is 245ms. There was a brief spike
to 1.2% around 14:30 UTC but it resolved within 5 minutes.
```

Claude Code automatically routes to the right MCP server based on your request. If you ask about metrics, it uses Datadog. If you ask about tickets, it uses Linear. If you ask about feature flags, it uses GrowthBook.

---

## 4. Building Your Own MCP Server

### 4.1 When to Build vs. Use Existing

Build your own MCP server when:
- You have an internal system with no public MCP server
- You need custom business logic on top of a system
- You want to restrict what the AI can access in a system
- You want to combine multiple systems into one unified interface

### 4.2 The TypeScript MCP SDK

The official MCP TypeScript SDK makes building servers straightforward:

```bash
# Initialize a new MCP server project
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
```

### 4.3 A Complete MCP Server: Internal Wiki

Let's build a real MCP server that gives Claude Code access to your company's internal documentation wiki:

```typescript
// src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// Your wiki client (could be Notion, Confluence, custom, etc.)
interface WikiPage {
  id: string;
  title: string;
  content: string;
  lastUpdated: string;
  author: string;
}

class WikiClient {
  private baseUrl: string;
  private token: string;

  constructor(baseUrl: string, token: string) {
    this.baseUrl = baseUrl;
    this.token = token;
  }

  async search(query: string): Promise<WikiPage[]> {
    const response = await fetch(`${this.baseUrl}/api/search`, {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${this.token}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ query, limit: 10 }),
    });
    return response.json();
  }

  async getPage(pageId: string): Promise<WikiPage> {
    const response = await fetch(`${this.baseUrl}/api/pages/${pageId}`, {
      headers: { "Authorization": `Bearer ${this.token}` },
    });
    return response.json();
  }

  async listPages(space: string): Promise<WikiPage[]> {
    const response = await fetch(
      `${this.baseUrl}/api/spaces/${space}/pages`,
      { headers: { "Authorization": `Bearer ${this.token}` } }
    );
    return response.json();
  }
}

// Create the MCP server
const server = new McpServer({
  name: "internal-wiki",
  version: "1.0.0",
  description: "Access company internal wiki documentation",
});

// Initialize wiki client from environment
const wiki = new WikiClient(
  process.env.WIKI_URL || "https://wiki.internal.company.com",
  process.env.WIKI_TOKEN || ""
);

// --- TOOLS ---

server.tool(
  "search_wiki",
  "Search the internal wiki for documentation matching a query",
  {
    query: z.string().describe("Search query — use natural language"),
    space: z.string().optional().describe(
      "Wiki space to search in (e.g., 'engineering', 'product', 'ops')"
    ),
  },
  async ({ query, space }) => {
    const results = await wiki.search(
      space ? `space:${space} ${query}` : query
    );

    if (results.length === 0) {
      return {
        content: [
          {
            type: "text" as const,
            text: `No wiki pages found for: "${query}"`,
          },
        ],
      };
    }

    const formatted = results
      .map(
        (page) =>
          `- **${page.title}** (ID: ${page.id})\n  Updated: ${page.lastUpdated} by ${page.author}\n  Preview: ${page.content.slice(0, 200)}...`
      )
      .join("\n\n");

    return {
      content: [
        {
          type: "text" as const,
          text: `Found ${results.length} wiki pages:\n\n${formatted}`,
        },
      ],
    };
  }
);

server.tool(
  "read_wiki_page",
  "Read the full content of a specific wiki page by ID",
  {
    pageId: z.string().describe("The wiki page ID to read"),
  },
  async ({ pageId }) => {
    const page = await wiki.getPage(pageId);

    return {
      content: [
        {
          type: "text" as const,
          text: `# ${page.title}\n\n*Last updated: ${page.lastUpdated} by ${page.author}*\n\n${page.content}`,
        },
      ],
    };
  }
);

server.tool(
  "list_wiki_spaces",
  "List available wiki spaces and their page counts",
  {},
  async () => {
    // In a real implementation, this would call the wiki API
    const spaces = [
      { name: "engineering", description: "Engineering docs, ADRs, runbooks", pages: 342 },
      { name: "product", description: "Product specs, PRDs, roadmaps", pages: 128 },
      { name: "ops", description: "Operations runbooks, incident playbooks", pages: 87 },
      { name: "onboarding", description: "New hire onboarding guides", pages: 23 },
    ];

    const formatted = spaces
      .map((s) => `- **${s.name}**: ${s.description} (${s.pages} pages)`)
      .join("\n");

    return {
      content: [
        {
          type: "text" as const,
          text: `Available wiki spaces:\n\n${formatted}`,
        },
      ],
    };
  }
);

// --- RESOURCES ---

server.resource(
  "wiki://onboarding/engineering",
  "Engineering onboarding guide — architecture overview, setup instructions, key contacts",
  async () => {
    const page = await wiki.getPage("onboarding-engineering");
    return {
      contents: [
        {
          uri: "wiki://onboarding/engineering",
          mimeType: "text/markdown",
          text: page.content,
        },
      ],
    };
  }
);

server.resource(
  "wiki://runbooks/incidents",
  "Incident response runbook — severity levels, escalation paths, communication templates",
  async () => {
    const page = await wiki.getPage("runbook-incidents");
    return {
      contents: [
        {
          uri: "wiki://runbooks/incidents",
          mimeType: "text/markdown",
          text: page.content,
        },
      ],
    };
  }
);

// --- PROMPTS ---

server.prompt(
  "research_topic",
  "Research a topic across the wiki and provide a comprehensive summary",
  [
    {
      name: "topic",
      description: "The topic to research",
      required: true,
    },
  ],
  async ({ topic }) => {
    return {
      messages: [
        {
          role: "user" as const,
          content: {
            type: "text" as const,
            text: `Research the following topic in our internal wiki: "${topic}"

Steps:
1. Search the wiki for pages related to this topic
2. Read the most relevant 3-5 pages
3. Synthesize a comprehensive summary that includes:
   - Key concepts and definitions
   - Current state of implementation
   - Related systems and dependencies
   - Known issues or limitations
   - Links to the source wiki pages

Format as a well-structured markdown document.`,
          },
        },
      ],
    };
  }
);

// --- START THE SERVER ---

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Internal Wiki MCP server running on stdio");
}

main().catch(console.error);
```

### 4.4 Building and Running

```json
// package.json
{
  "name": "@company/mcp-internal-wiki",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "bin": {
    "mcp-internal-wiki": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node16",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true
  },
  "include": ["src/**/*"]
}
```

### 4.5 Connecting Your Server to Claude Code

```json
// .claude/settings.json
{
  "mcpServers": {
    "wiki": {
      "command": "npx",
      "args": ["-y", "@company/mcp-internal-wiki"],
      "env": {
        "WIKI_URL": "https://wiki.internal.company.com",
        "WIKI_TOKEN": "{{secrets.WIKI_TOKEN}}"
      }
    }
  }
}
```

During development, use the local path:

```json
{
  "mcpServers": {
    "wiki": {
      "command": "node",
      "args": ["./mcp-servers/internal-wiki/dist/index.js"],
      "env": {
        "WIKI_URL": "https://wiki.internal.company.com",
        "WIKI_TOKEN": "your-dev-token"
      }
    }
  }
}
```

### 4.6 Testing Your MCP Server

Test your server using the MCP inspector or by running it directly:

```bash
# Run the server in dev mode and send test requests
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}' | node dist/index.js

# Use the MCP inspector (official debugging tool)
npx @modelcontextprotocol/inspector node dist/index.js
```

The MCP inspector gives you a web UI where you can:
- See all tools, resources, and prompts your server exposes
- Call tools with test inputs
- View raw JSON-RPC messages
- Debug connection issues

---

## 5. MCP Server Security

### 5.1 The Threat Model

When you connect an MCP server, you are giving an AI agent access to a real system. The threat model includes:

```
┌──────────────────────────────────────────────────┐
│               THREAT MODEL                        │
│                                                   │
│  1. PROMPT INJECTION                              │
│     User input → AI → MCP tool call               │
│     Attacker crafts input that tricks the AI      │
│     into making malicious tool calls              │
│                                                   │
│  2. OVER-SCOPED ACCESS                            │
│     MCP server has write access to production DB  │
│     AI accidentally runs a destructive query      │
│                                                   │
│  3. TOKEN EXFILTRATION                            │
│     Malicious content in responses tricks the AI  │
│     into leaking API tokens via other tools       │
│                                                   │
│  4. RATE LIMIT ABUSE                              │
│     AI makes thousands of API calls in a loop     │
│     Exhausts quotas or triggers rate limiting     │
└──────────────────────────────────────────────────┘
```

### 5.2 Authentication Best Practices

**Never hardcode credentials:**

```json
// BAD — credentials in settings.json (committed to git!)
{
  "mcpServers": {
    "slack": {
      "env": {
        "SLACK_TOKEN": "xoxb-1234567890-abcdefghij"
      }
    }
  }
}

// GOOD — use secrets reference
{
  "mcpServers": {
    "slack": {
      "env": {
        "SLACK_TOKEN": "{{secrets.SLACK_TOKEN}}"
      }
    }
  }
}

// GOOD — use environment variable passthrough
{
  "mcpServers": {
    "slack": {
      "env": {
        "SLACK_TOKEN": "${SLACK_TOKEN}"
      }
    }
  }
}
```

### 5.3 Scoping Access

**Principle of least privilege.** Your MCP server should expose the minimum access needed:

```typescript
// BAD — full database access
server.tool(
  "run_query",
  "Run any SQL query",
  { query: z.string() },
  async ({ query }) => {
    return db.query(query); // Can DROP TABLES, DELETE, UPDATE...
  }
);

// GOOD — read-only, specific tables, parameterized
server.tool(
  "query_analytics",
  "Query the analytics database (read-only, last 90 days)",
  {
    metric: z.enum(["revenue", "signups", "churn", "dau"]),
    timeframe: z.enum(["7d", "30d", "90d"]),
    groupBy: z.enum(["day", "week", "month"]).optional(),
  },
  async ({ metric, timeframe, groupBy }) => {
    // Pre-built queries — no raw SQL from the AI
    const query = buildAnalyticsQuery(metric, timeframe, groupBy);
    return analyticsDb.query(query); // Read-only connection
  }
);
```

### 5.4 Rate Limiting

Build rate limiting into your MCP server:

```typescript
class RateLimiter {
  private calls: number[] = [];
  private maxCalls: number;
  private windowMs: number;

  constructor(maxCalls: number, windowMs: number) {
    this.maxCalls = maxCalls;
    this.windowMs = windowMs;
  }

  check(): boolean {
    const now = Date.now();
    this.calls = this.calls.filter((t) => now - t < this.windowMs);
    if (this.calls.length >= this.maxCalls) {
      return false;
    }
    this.calls.push(now);
    return true;
  }
}

const limiter = new RateLimiter(100, 60_000); // 100 calls per minute

server.tool(
  "search_slack",
  "Search Slack messages",
  { query: z.string() },
  async ({ query }) => {
    if (!limiter.check()) {
      return {
        content: [
          {
            type: "text" as const,
            text: "Rate limit exceeded. Please wait before searching again.",
          },
        ],
        isError: true,
      };
    }
    // ... actual search logic
  }
);
```

### 5.5 Input Validation

Always validate inputs server-side, even though Zod schemas provide client-side validation:

```typescript
server.tool(
  "read_wiki_page",
  "Read a wiki page",
  {
    pageId: z.string()
      .regex(/^[a-zA-Z0-9-]+$/, "Page ID must be alphanumeric with dashes")
      .max(100, "Page ID too long"),
  },
  async ({ pageId }) => {
    // Additional server-side validation
    if (pageId.includes("..") || pageId.includes("/")) {
      return {
        content: [
          {
            type: "text" as const,
            text: "Invalid page ID format",
          },
        ],
        isError: true,
      };
    }

    const page = await wiki.getPage(pageId);
    // ...
  }
);
```

### 5.6 Audit Logging

Log every MCP tool call for security review:

```typescript
import { appendFileSync } from "fs";

function auditLog(toolName: string, args: unknown, userId?: string) {
  const entry = {
    timestamp: new Date().toISOString(),
    tool: toolName,
    args,
    userId: userId || "unknown",
  };
  appendFileSync(
    "/var/log/mcp-audit.jsonl",
    JSON.stringify(entry) + "\n"
  );
}

// Wrap every tool handler
server.tool(
  "query_database",
  "Run an analytics query",
  { metric: z.string() },
  async ({ metric }) => {
    auditLog("query_database", { metric });
    // ... tool logic
  }
);
```

---

## 6. Practical MCP Patterns

### 6.1 The Read-Only First Pattern

Start every MCP server as read-only. Add write operations later, after you trust the setup:

```typescript
// Version 1: Read-only
server.tool("list_tickets", "List Linear tickets", ...);
server.tool("get_ticket", "Read a specific ticket", ...);
server.tool("search_tickets", "Search tickets", ...);

// Version 2: Add safe writes (after trust is established)
server.tool("create_ticket", "Create a new ticket", ...);
server.tool("add_comment", "Add a comment to a ticket", ...);

// Version 3: Add state changes (with extra validation)
server.tool("update_ticket_status", "Change ticket status", ...);
// Never: server.tool("delete_ticket", ...)
```

### 6.2 The Composite Server Pattern

Instead of one MCP server per system, build a composite server that provides a unified interface:

```typescript
// Instead of connecting 5 separate MCP servers...
// Build one "engineering-tools" server that wraps them all

const server = new McpServer({
  name: "engineering-tools",
  version: "1.0.0",
  description: "Unified engineering tools: metrics, tickets, flags, docs",
});

// Metrics (wraps Datadog)
server.tool("get_service_health", "Get health metrics for a service", ...);
server.tool("get_error_rate", "Get error rate for an endpoint", ...);

// Tickets (wraps Linear)
server.tool("my_tickets", "Get my assigned tickets", ...);
server.tool("create_ticket", "Create a new ticket", ...);

// Feature Flags (wraps GrowthBook)
server.tool("get_flag_status", "Check if a feature flag is enabled", ...);
server.tool("list_flags", "List all feature flags", ...);

// Docs (wraps wiki)
server.tool("search_docs", "Search internal documentation", ...);
server.tool("read_doc", "Read a documentation page", ...);
```

**Why this is better:**
- One server to configure instead of five
- Unified authentication handling
- Cross-system queries become possible ("find the ticket related to this error spike")
- Easier to version, deploy, and maintain

### 6.3 The Context-Aware Server Pattern

Build servers that understand your project context:

```typescript
server.tool(
  "get_relevant_docs",
  "Get documentation relevant to the current code file",
  {
    filePath: z.string().describe("Path to the current code file"),
  },
  async ({ filePath }) => {
    // Map code paths to documentation
    const docMap: Record<string, string[]> = {
      "src/payments/": ["wiki://docs/payments-architecture", "wiki://runbooks/payment-failures"],
      "src/auth/": ["wiki://docs/auth-flow", "wiki://docs/clerk-setup"],
      "src/api/": ["wiki://docs/api-conventions", "wiki://docs/error-codes"],
    };

    const prefix = Object.keys(docMap).find((p) => filePath.startsWith(p));
    if (!prefix) {
      return {
        content: [{ type: "text" as const, text: "No specific docs for this file path." }],
      };
    }

    const docs = await Promise.all(
      docMap[prefix].map((uri) => wiki.getPage(uri.split("://")[1]))
    );

    return {
      content: docs.map((doc) => ({
        type: "text" as const,
        text: `## ${doc.title}\n\n${doc.content}`,
      })),
    };
  }
);
```

---

## 7. The MCP Ecosystem

### 7.1 Where to Find MCP Servers

The MCP ecosystem is growing rapidly:

- **Official Anthropic servers**: Slack, Linear, GitHub, Google, Datadog, and more
- **Community servers**: Hundreds of open-source servers on GitHub
- **Company-built servers**: Your internal tools, wrapped as MCP
- **MCP registries**: Directories like mcp.run for discovering servers

### 7.2 Evaluating an MCP Server

Before connecting a third-party MCP server, evaluate it:

```
□ Source code available?          ← Can you audit what it does?
□ What permissions does it need?  ← Principle of least privilege
□ Who maintains it?               ← Anthropic? Well-known org? Random repo?
□ What data does it access?       ← Sensitive data exposure risk
□ Does it have rate limiting?     ← Will it blow your API quotas?
□ Is it read-only?                ← Read-only is always safer
□ Last updated?                   ← Stale servers may have vulnerabilities
```

### 7.3 Where MCP Is Going

MCP is still evolving. Key trends to watch:

**Marketplace model**: Anthropic and others are building MCP server marketplaces where you can discover, install, and manage servers through a UI rather than JSON config.

**Authentication standardization**: OAuth2 flows built into the MCP protocol, so servers can authenticate users without custom token management.

**Streaming results**: Better support for long-running operations — start a deployment, stream progress, report completion.

**Server-to-server**: MCP servers that call other MCP servers, enabling complex workflows without the AI as intermediary.

**Remote-first**: The shift from stdio (local process) to HTTP (remote server) will make MCP servers deployable as shared infrastructure rather than per-developer processes.

---

## 8. Common Pitfalls

### 8.1 Too Many Servers

You connect 15 MCP servers. Claude Code now has 200 available tools. The model spends tokens understanding all of them and sometimes picks the wrong one.

**Fix:** Start with 2-3 servers that cover your most common workflows. Add more only when you have a specific need. The composite server pattern (Section 6.2) helps reduce tool sprawl.

### 8.2 No Error Handling

Your MCP server crashes when the external API is down. Claude Code gets a cryptic error and does not know what happened.

**Fix:** Always return user-friendly error messages:

```typescript
async ({ query }) => {
  try {
    const results = await slack.search(query);
    return { content: [{ type: "text" as const, text: formatResults(results) }] };
  } catch (error) {
    return {
      content: [
        {
          type: "text" as const,
          text: `Slack search failed: ${error instanceof Error ? error.message : "Unknown error"}. The Slack API may be temporarily unavailable.`,
        },
      ],
      isError: true,
    };
  }
}
```

### 8.3 Leaking Secrets in Responses

Your MCP server includes API tokens, connection strings, or internal URLs in its responses. The AI might include these in its output.

**Fix:** Sanitize all responses before returning them. Strip internal URLs, tokens, and credentials:

```typescript
function sanitize(text: string): string {
  return text
    .replace(/xoxb-[a-zA-Z0-9-]+/g, "[SLACK_TOKEN]")
    .replace(/Bearer [a-zA-Z0-9._-]+/g, "Bearer [REDACTED]")
    .replace(/https?:\/\/[^/]*\.internal\.[^/]+/g, "[INTERNAL_URL]");
}
```

---

## Summary

MCP is the bridge between your AI agent and the rest of your engineering world. The key ideas:

1. **MCP standardizes tool access** — build one server, every AI client can use it
2. **Three primitives** — tools (actions), resources (data), prompts (templates)
3. **Start with existing servers** — Slack, Linear, Datadog, Metabase are ready to go
4. **Build your own for internal systems** — the TypeScript SDK makes it straightforward
5. **Security is paramount** — scope access, rate limit, validate inputs, audit everything
6. **Less is more** — a few well-scoped servers beat a dozen barely-configured ones

The connection between Chapter 8 (tool calling) and this chapter is direct. MCP tools are tools. The schema format is the same. The execution model is the same. The difference is that MCP standardizes the protocol so any tool can work with any agent, and the servers can wrap arbitrarily complex external systems.

In Chapter 25, we will see how to combine MCP servers with skills and plugins to build automated workflows that run without you.

---

*Previous: [Chapter 23 — Claude Code Mastery](./23-claude-code-mastery.md)* | *Next: [Chapter 25 — Skills, Plugins & Automation](./25-skills-plugins.md)*

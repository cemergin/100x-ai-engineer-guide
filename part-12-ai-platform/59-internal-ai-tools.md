<!--
  CHAPTER: 59
  TITLE: Building Internal AI Tools
  PART: 12 — AI Platform Engineering
  PHASE: 2 — Become an Expert
  PREREQS: Ch 23-25 (harness engineering), Ch 27 (harness architecture), Ch 55 (sandboxing), Ch 58 (multi-agent orchestration)
  KEY_TOPICS: internal AI tools, Glass pattern, zero-config setup, SSO integration, Agent SDK, tool connectors, build vs buy, OpenClaw pattern, headless mode, personal digests, platform architecture
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 59: Building Internal AI Tools

> **Part 12 — AI Platform Engineering** | Phase 2: Become an Expert | Prerequisites: Ch 23-25, Ch 27, Ch 55, Ch 58 | Difficulty: Advanced | Language: TypeScript + Python

In Chapter 23, you learned to use Claude Code as an individual productivity tool. In Chapter 27, you understood how agent harnesses work internally. In Chapter 55, you learned to sandbox agents safely. In Chapter 58, you built multi-agent orchestration. Now you build the platform that packages all of that into a tool anyone in your organization can use on day one.

This is the Glass chapter. Glass is Ramp's internal AI tool -- built by four engineers in under three months on Anthropic's Agent SDK. Within a month of launch, it had 700 daily active users. The engineering is impressive, but what matters more is the *philosophy*: zero-config setup, pre-connected integrations, and a relentless focus on removing friction. "If the user has to debug, we've already lost."

You will not build Glass itself. You will learn the patterns that made Glass successful and apply them to build your own internal AI platform. The chapter walks you through the architecture, the integration patterns, the build-vs-buy decision, and a complete working implementation using the Anthropic Agent SDK.

### In This Chapter
- The Glass pattern: what makes an internal AI tool succeed
- Zero-config setup: SSO to everything connected on day one
- Architecture: Agent SDK + tool connectors + memory + UI
- Build vs buy: Glass (custom) vs Claude Code (off-shelf) vs Cursor (IDE)
- The OpenClaw pattern: deploying internal agents safely
- Building an internal AI tool with the Anthropic Agent SDK
- The personal digest pattern: daily briefings from connected tools
- Headless mode: kick off tasks, approve from phone, results waiting
- Multi-pane workspace: beyond the chat window

### Related Chapters
- **Ch 23 (Claude Code Mastery)** -- using an AI tool; this chapter builds one
- **Ch 24 (MCP Servers)** -- MCP as the integration protocol for platform tools
- **Ch 25 (Skills, Plugins & Automation)** -- skills as the building blocks; this chapter creates the platform they run on
- **Ch 27 (The Agent Harness Architecture)** -- the architecture pattern this chapter implements
- **Ch 55 (Sandboxing & Isolating Agents)** -- the isolation model for safely running tools
- **Ch 58 (Multi-Agent Orchestration)** -- the coordination engine inside the platform
- **Ch 60 (Skills Marketplaces)** -- distributing skills across the platform

---

## 1. The Glass Pattern

### 1.1 What Made Glass Work

In March 2025, Ramp's AI Platform team (four engineers) started building Glass. By April, it was in production. By May, it had 700 daily active users. By late 2025, 99.5% of Ramp's team was active on AI tools. What happened?

Glass is not a better chatbot. It is an *opinionated integration layer* that removes every friction point between an employee and the AI capabilities they need.

**Zero-config setup.** You install Glass. It detects you via Okta SSO. It knows your role (engineer, PM, salesperson). It pre-connects every tool you have access to: Salesforce, Snowflake, Gong, Slack, Notion, Linear, GitHub, your internal APIs. You do not configure anything. You do not enter API keys. You do not read a setup guide. On day one, you can ask "What were the key themes from last week's sales calls?" and it works because Gong is already connected.

**30+ tools light up immediately.** Not "you can connect 30 tools if you set them up." They are *already connected*. The platform team pre-built the integrations and mapped them to your permissions via SSO. Your Okta groups determine which tools you see.

**"If the user has to debug, we've already lost."** This is the design philosophy. Every error is the platform team's fault, not the user's. If a tool call fails, the platform handles the retry. If auth expires, the platform refreshes it. If a query returns too much data, the platform paginates it. The user never sees a stack trace.

**Multi-pane workspace, not just a chat window.** Glass is not a chatbot. It is a workspace with multiple panes: a chat pane, a code editor pane, a data visualization pane, a file browser. The AI can write to any pane. This mirrors how people actually work -- they do not do everything through text.

### 1.2 Why This Pattern Matters

The Glass pattern works because it solves the *last mile problem* of AI adoption. Most companies that try to roll out AI tools hit the same wall: they give everyone access to ChatGPT or Claude, send a Slack message saying "use AI to be more productive," and then wonder why adoption stalls at 20%.

The problem is not the AI. The problem is the distance between "I have an AI tool" and "the AI tool can do the thing I need right now." That distance is measured in:

- **Configuration steps.** Every API key is a dropout point.
- **Context the user must provide.** If I have to copy-paste from Salesforce into a chat window, I will just use Salesforce.
- **Mental model required.** If I have to understand prompt engineering to get good results, only prompt engineers will use it.
- **Trust in the output.** If the tool hallucinates and I cannot verify, I stop trusting it.

Glass collapses all of these to zero: no configuration, tools already connected (no copy-paste), opinionated UX (no prompt engineering required), and results from real data sources (verifiable).

### 1.3 The Architecture at a Glance

```
┌──────────────────────────────────────────────────────┐
│                  Glass Frontend                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │   Chat   │ │  Code    │ │   Data   │ │  Files  │ │
│  │   Pane   │ │  Editor  │ │   Viz    │ │ Browser │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬────┘ │
│       └─────────────┴────────────┴────────────┘      │
│                        │                              │
└────────────────────────┼──────────────────────────────┘
                         │ WebSocket
┌────────────────────────┼──────────────────────────────┐
│              Agent Orchestrator                        │
│  ┌──────────────────────────────────────────────┐     │
│  │  Anthropic Agent SDK                          │     │
│  │  ┌─────────┐ ┌──────────┐ ┌──────────────┐  │     │
│  │  │  Agent   │ │  Memory  │ │  Tool Router │  │     │
│  │  │  Loop    │ │  System  │ │              │  │     │
│  │  └─────────┘ └──────────┘ └──────┬───────┘  │     │
│  └──────────────────────────────────┼───────────┘     │
│                                     │                  │
│  ┌──────────────────────────────────┼───────────────┐ │
│  │         Tool Connectors          │               │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┴──┐ ┌───────┐ │ │
│  │  │Salesforce│ │Snowflake│ │  GitHub  │ │ Slack │ │ │
│  │  └────────┘ └────────┘ └──────────┘ └───────┘ │ │
│  │  ┌────────┐ ┌────────┐ ┌──────────┐ ┌───────┐ │ │
│  │  │  Gong  │ │ Notion │ │  Linear  │ │ Jira  │ │ │
│  │  └────────┘ └────────┘ └──────────┘ └───────┘ │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Auth Layer (Okta SSO → Permissions → Tool Access)│ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

---

## 2. Architecture: Agent SDK + Tool Connectors + Memory + UI

### 2.1 Why the Agent SDK, Not the Messages API

Glass is built on Anthropic's Agent SDK, not on raw `client.messages.create()` calls. The distinction matters for platform-scale tools:

**The Messages API** (`@anthropic-ai/sdk`) gives you one primitive: send messages, get a response. You build everything else:
- You write the while-loop that iterates on tool calls
- You manage the messages array (append assistant responses, tool results)
- You handle tool routing (which function handles which tool_use block)
- You track token usage, turn counts, and termination conditions
- You implement error recovery when a tool call fails mid-loop

This is fine for learning (Ch 8-9) and for simple use cases. But when you are building a platform with 30+ tools, multiple user roles, session persistence, and streaming to a WebSocket -- the boilerplate adds up fast.

**The Agent SDK** (`@anthropic-ai/agent-sdk`) builds on the Messages API and gives you:
- **A managed agent loop.** Call `agent.run()` and the SDK handles the think-act-observe cycle, including retries on transient failures.
- **Structured tool registration.** Tools are first-class objects with typed schemas and handlers, not raw JSON input_schema definitions that you match by name in a switch statement.
- **Session state.** The SDK tracks conversation history, tool results, and turn metadata. You don't manually build and manage the messages array.
- **Streaming built in.** The SDK streams text chunks and tool-call events to your UI without you wiring up the SSE/WebSocket plumbing.
- **Guardrails as config.** Max turns, token budgets, and timeout handling are configuration options, not code you write and debug.

For Glass, this meant the four-person team could focus on what mattered -- the 30+ tool connectors, the SSO integration, the multi-pane UI -- instead of re-implementing agent loop mechanics.

### 2.2 The Agent Core

The heart of any internal AI tool is the agent loop. We build on the Anthropic Agent SDK because it provides the loop, tool calling, and streaming out of the box.

```typescript
// agent-core.ts — The core agent engine for an internal AI tool
import Anthropic from "@anthropic-ai/sdk";

interface ToolConnector {
  name: string;
  description: string;
  tools: Anthropic.Tool[];
  execute: (toolName: string, input: unknown) => Promise<string>;
  requiredScopes: string[]; // Okta scopes needed
}

interface UserContext {
  userId: string;
  email: string;
  role: string;
  teams: string[];
  scopes: string[]; // From SSO — determines which tools are available
  preferences: Record<string, unknown>;
}

interface PlatformAgentConfig {
  model: string;
  connectors: ToolConnector[];
  memoryProvider: MemoryProvider;
  maxTokens: number;
  systemPromptTemplate: string;
}

class PlatformAgent {
  private client: Anthropic;
  private config: PlatformAgentConfig;
  private user: UserContext;
  private activeConnectors: ToolConnector[];
  private allTools: Anthropic.Tool[];
  private toolExecutors: Map<string, ToolConnector>;
  private memory: MemoryProvider;

  constructor(config: PlatformAgentConfig, user: UserContext) {
    this.client = new Anthropic();
    this.config = config;
    this.user = user;
    this.memory = config.memoryProvider;

    // Filter connectors based on user's SSO scopes
    this.activeConnectors = config.connectors.filter((c) =>
      c.requiredScopes.every((scope) => user.scopes.includes(scope))
    );

    // Build the combined tool list
    this.allTools = this.activeConnectors.flatMap((c) => c.tools);

    // Map tool names to their connectors for execution routing
    this.toolExecutors = new Map();
    for (const connector of this.activeConnectors) {
      for (const tool of connector.tools) {
        this.toolExecutors.set(tool.name, connector);
      }
    }
  }

  /**
   * The tools available to this user — determined by SSO, not configuration.
   * An engineer sees GitHub + Linear + Snowflake.
   * A salesperson sees Salesforce + Gong + Slack.
   * Everyone sees Slack + Notion.
   */
  getAvailableTools(): string[] {
    return this.allTools.map((t) => t.name);
  }

  /**
   * Build the system prompt dynamically based on user context and available tools.
   */
  private buildSystemPrompt(): string {
    const toolList = this.activeConnectors
      .map(
        (c) =>
          `- **${c.name}**: ${c.description} (tools: ${c.tools.map((t) => t.name).join(", ")})`
      )
      .join("\n");

    return this.config.systemPromptTemplate
      .replace("{{USER_NAME}}", this.user.email.split("@")[0])
      .replace("{{USER_ROLE}}", this.user.role)
      .replace("{{USER_TEAMS}}", this.user.teams.join(", "))
      .replace("{{AVAILABLE_TOOLS}}", toolList)
      .replace("{{DATE}}", new Date().toISOString().split("T")[0]);
  }

  /**
   * Run the agent loop with streaming output.
   */
  async *run(
    userMessage: string,
    conversationHistory: Anthropic.MessageParam[]
  ): AsyncGenerator<{
    type: "text" | "tool_call" | "tool_result" | "done";
    content: string;
    metadata?: Record<string, unknown>;
  }> {
    const messages: Anthropic.MessageParam[] = [
      ...conversationHistory,
      { role: "user", content: userMessage },
    ];

    // Load relevant memories
    const memories = await this.memory.recall(this.user.userId, userMessage);
    const systemPrompt = this.buildSystemPrompt() +
      (memories.length > 0
        ? `\n\n## Relevant context from previous sessions\n${memories.join("\n")}`
        : "");

    let continueLoop = true;

    while (continueLoop) {
      const response = await this.client.messages.create({
        model: this.config.model,
        max_tokens: this.config.maxTokens,
        system: systemPrompt,
        tools: this.allTools,
        messages,
      });

      // Process response blocks
      for (const block of response.content) {
        if (block.type === "text") {
          yield { type: "text", content: block.text };
        } else if (block.type === "tool_use") {
          yield {
            type: "tool_call",
            content: `Calling ${block.name}...`,
            metadata: { toolName: block.name, input: block.input },
          };

          // Route to the correct connector
          const connector = this.toolExecutors.get(block.name);
          if (!connector) {
            yield {
              type: "tool_result",
              content: `Error: tool ${block.name} not found`,
            };
            continue;
          }

          try {
            const result = await connector.execute(block.name, block.input);
            yield {
              type: "tool_result",
              content: result,
              metadata: { toolName: block.name },
            };
          } catch (error) {
            // The platform handles the error — not the user
            const errorMsg = error instanceof Error ? error.message : String(error);
            yield {
              type: "tool_result",
              content: `Tool error (handled): ${errorMsg}`,
            };
          }
        }
      }

      // Append assistant message
      messages.push({ role: "assistant", content: response.content });

      // If there were tool calls, append results and continue
      const toolUses = response.content.filter(
        (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
      );

      if (toolUses.length > 0) {
        const toolResults: Anthropic.ToolResultBlockParam[] = [];
        for (const toolUse of toolUses) {
          const connector = this.toolExecutors.get(toolUse.name);
          const result = connector
            ? await connector.execute(toolUse.name, toolUse.input)
            : `Error: tool ${toolUse.name} not found`;
          toolResults.push({
            type: "tool_result",
            tool_use_id: toolUse.id,
            content: result,
          });
        }
        messages.push({ role: "user", content: toolResults });
      } else {
        continueLoop = false;
      }
    }

    // Store memory from this interaction
    await this.memory.store(this.user.userId, userMessage, messages);

    yield { type: "done", content: "" };
  }
}
```

### 2.3 The Memory Provider

Internal tools need memory that persists across sessions. The user should be able to say "remember that the Q4 dashboard uses the new metric definitions" and have the tool recall that in future sessions.

```typescript
// memory.ts — Memory system for internal AI tools

interface MemoryProvider {
  store(userId: string, query: string, messages: Anthropic.MessageParam[]): Promise<void>;
  recall(userId: string, query: string, limit?: number): Promise<string[]>;
  clear(userId: string): Promise<void>;
}

class VectorMemoryProvider implements MemoryProvider {
  private embeddings: EmbeddingService;
  private vectorStore: VectorStore;

  constructor(embeddings: EmbeddingService, vectorStore: VectorStore) {
    this.embeddings = embeddings;
    this.vectorStore = vectorStore;
  }

  async store(
    userId: string,
    query: string,
    messages: Anthropic.MessageParam[]
  ): Promise<void> {
    // Extract key information from the conversation
    const summary = await this.summarizeConversation(messages);
    if (!summary) return;

    const embedding = await this.embeddings.embed(summary);

    await this.vectorStore.upsert({
      id: `${userId}-${Date.now()}`,
      embedding,
      metadata: {
        userId,
        summary,
        query,
        timestamp: Date.now(),
      },
    });
  }

  async recall(
    userId: string,
    query: string,
    limit: number = 5
  ): Promise<string[]> {
    const queryEmbedding = await this.embeddings.embed(query);

    const results = await this.vectorStore.query({
      embedding: queryEmbedding,
      filter: { userId },
      topK: limit,
      minScore: 0.7,
    });

    return results.map((r) => r.metadata.summary as string);
  }

  async clear(userId: string): Promise<void> {
    await this.vectorStore.deleteByFilter({ userId });
  }

  private async summarizeConversation(
    messages: Anthropic.MessageParam[]
  ): Promise<string | null> {
    // Only summarize conversations with substance
    if (messages.length < 4) return null;

    const client = new Anthropic();
    const response = await client.messages.create({
      model: "claude-haiku-3-5-20241022",
      max_tokens: 256,
      system: "Summarize this conversation in 1-2 sentences, focusing on key facts and decisions. If the conversation is trivial, respond with 'SKIP'.",
      messages: [
        {
          role: "user",
          content: messages
            .map((m) => {
              const content =
                typeof m.content === "string"
                  ? m.content
                  : JSON.stringify(m.content);
              return `${m.role}: ${content.slice(0, 500)}`;
            })
            .join("\n"),
        },
      ],
    });

    const text = response.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");

    return text.includes("SKIP") ? null : text;
  }
}
```

### 2.4 Tool Connectors

Each external service gets a connector. The connector handles authentication, rate limiting, error handling, and data formatting. The user never sees any of this.

```typescript
// connectors/salesforce.ts — Example tool connector

function createSalesforceConnector(
  accessToken: string,
  instanceUrl: string
): ToolConnector {
  return {
    name: "Salesforce",
    description: "CRM data: accounts, contacts, opportunities, activities",
    requiredScopes: ["salesforce:read"],
    tools: [
      {
        name: "salesforce_search_accounts",
        description: "Search for Salesforce accounts by name, industry, or other criteria",
        input_schema: {
          type: "object" as const,
          properties: {
            query: {
              type: "string",
              description: "Search query — company name, industry, or SOQL WHERE clause",
            },
            limit: {
              type: "number",
              description: "Max results to return (default 10)",
            },
          },
          required: ["query"],
        },
      },
      {
        name: "salesforce_get_opportunities",
        description: "Get open opportunities, optionally filtered by account, stage, or owner",
        input_schema: {
          type: "object" as const,
          properties: {
            accountId: { type: "string", description: "Filter by account ID" },
            stage: { type: "string", description: "Filter by stage name" },
            ownerId: { type: "string", description: "Filter by opportunity owner" },
          },
        },
      },
      {
        name: "salesforce_get_activities",
        description: "Get recent activities (calls, emails, meetings) for an account or contact",
        input_schema: {
          type: "object" as const,
          properties: {
            relatedTo: {
              type: "string",
              description: "Account or Contact ID to get activities for",
            },
            daysBack: {
              type: "number",
              description: "How many days back to look (default 30)",
            },
          },
          required: ["relatedTo"],
        },
      },
    ],

    async execute(toolName: string, input: unknown): Promise<string> {
      const params = input as Record<string, unknown>;

      switch (toolName) {
        case "salesforce_search_accounts": {
          const query = params.query as string;
          const limit = (params.limit as number) || 10;

          // SOQL query with error handling
          try {
            const soql = query.includes("WHERE")
              ? `SELECT Id, Name, Industry, AnnualRevenue FROM Account ${query} LIMIT ${limit}`
              : `SELECT Id, Name, Industry, AnnualRevenue FROM Account WHERE Name LIKE '%${query}%' LIMIT ${limit}`;

            const response = await fetch(
              `${instanceUrl}/services/data/v59.0/query?q=${encodeURIComponent(soql)}`,
              {
                headers: {
                  Authorization: `Bearer ${accessToken}`,
                  "Content-Type": "application/json",
                },
              }
            );

            if (!response.ok) {
              // Handle token refresh transparently
              if (response.status === 401) {
                // Refresh token and retry — the user never knows
                const newToken = await refreshSalesforceToken(accessToken);
                return await this.execute(toolName, input);
              }
              throw new Error(`Salesforce API error: ${response.statusText}`);
            }

            const data = await response.json();
            return JSON.stringify(data.records, null, 2);
          } catch (error) {
            return `Error searching accounts: ${error}`;
          }
        }

        case "salesforce_get_opportunities":
          // Similar pattern — SOQL + error handling + token refresh
          return "Implementation follows the same pattern";

        case "salesforce_get_activities":
          return "Implementation follows the same pattern";

        default:
          return `Unknown Salesforce tool: ${toolName}`;
      }
    },
  };
}
```

### 2.5 SSO-Driven Tool Discovery

The key innovation: tools are not configured by users. They are *discovered* from SSO.

```typescript
// discovery.ts — Discover available tools from SSO context

interface SSOProfile {
  email: string;
  groups: string[]; // Okta groups
  roles: string[];
  entitlements: string[]; // app entitlements
}

interface ConnectorFactory {
  name: string;
  requiredEntitlement: string; // Okta app entitlement
  create: (ssoProfile: SSOProfile) => Promise<ToolConnector>;
}

// Registry of all possible connectors
const CONNECTOR_REGISTRY: ConnectorFactory[] = [
  {
    name: "Salesforce",
    requiredEntitlement: "salesforce",
    create: async (profile) => {
      const tokens = await getOAuthTokenForUser(profile.email, "salesforce");
      return createSalesforceConnector(tokens.accessToken, tokens.instanceUrl);
    },
  },
  {
    name: "GitHub",
    requiredEntitlement: "github",
    create: async (profile) => {
      const tokens = await getOAuthTokenForUser(profile.email, "github");
      return createGitHubConnector(tokens.accessToken);
    },
  },
  {
    name: "Snowflake",
    requiredEntitlement: "snowflake",
    create: async (profile) => {
      const tokens = await getServiceToken("snowflake", profile.roles);
      return createSnowflakeConnector(tokens);
    },
  },
  {
    name: "Slack",
    requiredEntitlement: "slack",
    create: async (profile) => {
      const tokens = await getOAuthTokenForUser(profile.email, "slack");
      return createSlackConnector(tokens.accessToken);
    },
  },
  // ... 26 more connectors
];

async function discoverConnectors(ssoProfile: SSOProfile): Promise<ToolConnector[]> {
  const available = CONNECTOR_REGISTRY.filter((factory) =>
    ssoProfile.entitlements.includes(factory.requiredEntitlement)
  );

  console.log(
    `[discovery] User ${ssoProfile.email} has ${available.length} connectors available`
  );

  // Initialize all available connectors in parallel
  const connectors = await Promise.allSettled(
    available.map(async (factory) => {
      try {
        const connector = await factory.create(ssoProfile);
        console.log(`[discovery] ✓ ${factory.name} connected`);
        return connector;
      } catch (error) {
        // Log the error but don't fail the whole setup
        console.warn(`[discovery] ✗ ${factory.name} failed: ${error}`);
        return null;
      }
    })
  );

  return connectors
    .filter(
      (r): r is PromiseFulfilledResult<ToolConnector> =>
        r.status === "fulfilled" && r.value !== null
    )
    .map((r) => r.value);
}
```

---

## 3. Build vs Buy

### 3.1 The Decision Matrix

| Factor | Build (Glass-style) | Buy (Claude Code / Cursor) | Hybrid |
|--------|-------------------|---------------------------|--------|
| **Time to value** | 2-3 months | 1 day | 2-4 weeks |
| **Customization** | Unlimited | Limited to config | Moderate |
| **Internal integrations** | Deep (SSO, internal APIs) | Surface (MCP servers) | Good with effort |
| **Maintenance burden** | High — you own everything | Low — vendor maintains | Moderate |
| **Cost control** | Full — you choose models, caching | Limited to vendor pricing | Mixed |
| **Adoption friction** | Low if done right, high if not | Medium — requires setup | Low-Medium |
| **Data sovereignty** | Full — nothing leaves your VPC | Depends on vendor | Configurable |

### 3.2 When to Build Custom

Build custom when:
- You have 20+ internal services that need deep integration
- Data sovereignty requirements prevent using external tools
- Your workflow is different enough that off-the-shelf tools feel forced
- You have a team that can maintain it (minimum 2-3 engineers)
- The competitive advantage of AI tooling justifies the investment

### 3.3 When to Buy

Buy when:
- You need AI tooling this week, not this quarter
- Your team is small and cannot dedicate engineers to platform work
- Standard coding/writing/analysis workflows are your primary use case
- You are still figuring out what AI adoption looks like for your org

### 3.4 The Hybrid Approach

Most companies land here. Use Claude Code or Cursor for coding workflows (where they excel), build a custom tool for internal integrations (where off-the-shelf tools fall short), and connect them via MCP.

```typescript
// hybrid.ts — Custom platform that also exposes tools via MCP

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

// Your internal platform exposes its connectors as MCP tools
// so Claude Code and Cursor can also use them
function createMCPServerFromConnectors(
  connectors: ToolConnector[]
): Server {
  const server = new Server(
    {
      name: "internal-platform",
      version: "1.0.0",
    },
    {
      capabilities: {
        tools: {},
      },
    }
  );

  // Register all connector tools as MCP tools
  server.setRequestHandler("tools/list", async () => ({
    tools: connectors.flatMap((c) =>
      c.tools.map((t) => ({
        name: `${c.name.toLowerCase()}_${t.name}`,
        description: `[${c.name}] ${t.description}`,
        inputSchema: t.input_schema,
      }))
    ),
  }));

  server.setRequestHandler("tools/call", async (request) => {
    const toolName = request.params.name as string;
    // Route to correct connector
    for (const connector of connectors) {
      const tool = connector.tools.find(
        (t) => `${connector.name.toLowerCase()}_${t.name}` === toolName
      );
      if (tool) {
        const result = await connector.execute(tool.name, request.params.arguments);
        return { content: [{ type: "text", text: result }] };
      }
    }
    return { content: [{ type: "text", text: `Unknown tool: ${toolName}` }] };
  });

  return server;
}
```

---

## 4. The OpenClaw Pattern: Safe Deployment

### 4.1 What OpenClaw Is

OpenClaw (referenced in Ch 55) is a deployment pattern for internal AI agents that need to execute code and access internal services without risking production systems.

The pattern: each agent session runs in its own Docker container inside a VPC. The container has access only to pre-approved services through an explicit allowlist proxy. File system writes are ephemeral. Network access is restricted. The agent can be powerful within its sandbox and harmless outside it.

### 4.2 Docker Compose for Internal AI Tools

```yaml
# docker-compose.yml — Internal AI tool deployment

version: "3.9"

services:
  # The AI agent service
  agent-api:
    build: ./agent
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - DATABASE_URL=postgres://platform:${DB_PASS}@postgres:5432/platform
      - REDIS_URL=redis://redis:6379
      - SSO_ISSUER=https://your-company.okta.com
      - SSO_CLIENT_ID=${SSO_CLIENT_ID}
      - SSO_CLIENT_SECRET=${SSO_CLIENT_SECRET}
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
    networks:
      - internal
      - restricted   # Can only reach approved external services
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "1.0"

  # Session sandboxes — one per active agent session
  sandbox:
    build: ./sandbox
    environment:
      - AGENT_API_URL=http://agent-api:8080
    networks:
      - sandbox-net   # Isolated network — cannot reach internal services
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    tmpfs:
      - /workspace:size=100M  # Ephemeral workspace

  # Egress proxy — controls what external services agents can reach
  egress-proxy:
    build: ./proxy
    environment:
      - ALLOWED_DOMAINS=api.anthropic.com,api.openai.com,api.github.com,your-company.snowflakecomputing.com
    networks:
      - restricted
      - sandbox-net
    ports:
      - "3128:3128"

  # State and session management
  postgres:
    image: postgres:16
    environment:
      - POSTGRES_DB=platform
      - POSTGRES_USER=platform
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - internal

  redis:
    image: redis:7
    networks:
      - internal

networks:
  internal:        # Agent API ↔ databases
  restricted:      # Agent API ↔ egress proxy (approved external services)
  sandbox-net:     # Sandbox ↔ egress proxy only

volumes:
  pgdata:
```

### 4.3 Session Isolation

```typescript
// session-manager.ts — One sandbox per user session

import Docker from "dockerode";

const docker = new Docker();

interface AgentSession {
  id: string;
  userId: string;
  containerId: string;
  createdAt: Date;
  status: "active" | "idle" | "terminated";
}

class SessionManager {
  private sessions: Map<string, AgentSession> = new Map();
  private maxSessionsPerUser = 3;
  private idleTimeoutMs = 30 * 60 * 1000; // 30 minutes

  async createSession(userId: string): Promise<AgentSession> {
    // Enforce session limits
    const userSessions = Array.from(this.sessions.values()).filter(
      (s) => s.userId === userId && s.status === "active"
    );
    if (userSessions.length >= this.maxSessionsPerUser) {
      // Terminate oldest idle session
      const oldest = userSessions.sort(
        (a, b) => a.createdAt.getTime() - b.createdAt.getTime()
      )[0];
      await this.terminateSession(oldest.id);
    }

    // Spin up a new sandbox container
    const container = await docker.createContainer({
      Image: "internal-ai-sandbox:latest",
      HostConfig: {
        Memory: 512 * 1024 * 1024, // 512MB
        NanoCpus: 500000000, // 0.5 CPU
        NetworkMode: "sandbox-net",
        Tmpfs: { "/workspace": "rw,size=100m" },
        ReadonlyRootfs: true,
      },
      Env: [
        `SESSION_ID=${crypto.randomUUID()}`,
        `USER_ID=${userId}`,
        `PROXY_URL=http://egress-proxy:3128`,
      ],
    });

    await container.start();

    const session: AgentSession = {
      id: crypto.randomUUID(),
      userId,
      containerId: container.id,
      createdAt: new Date(),
      status: "active",
    };

    this.sessions.set(session.id, session);
    return session;
  }

  async terminateSession(sessionId: string): Promise<void> {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    try {
      const container = docker.getContainer(session.containerId);
      await container.stop({ t: 5 });
      await container.remove();
    } catch {
      // Container may already be stopped
    }

    session.status = "terminated";
  }

  // Cleanup idle sessions periodically
  async cleanupIdleSessions(): Promise<number> {
    let cleaned = 0;
    const now = Date.now();

    for (const [id, session] of this.sessions) {
      if (
        session.status === "active" &&
        now - session.createdAt.getTime() > this.idleTimeoutMs
      ) {
        await this.terminateSession(id);
        cleaned++;
      }
    }

    return cleaned;
  }
}
```

---

## 5. The Personal Digest Pattern

### 5.1 What It Is

One of Glass's most-loved features: a daily briefing that pulls from all your connected tools and gives you a personalized summary of what you need to know today.

**What it does:**
- Pulls unread Slack messages from your key channels
- Lists your Linear/Jira tickets with status changes
- Shows calendar events for today with relevant context
- Summarizes overnight activity in your GitHub repos
- Highlights Salesforce opportunities with upcoming deadlines

**Why it works:** It turns a 30-minute morning ritual of checking 6 different apps into a 2-minute AI briefing.

### 5.2 Building the Digest

```typescript
// digest.ts — Personal daily digest from connected tools

interface DigestSource {
  name: string;
  fetch: (userId: string, since: Date) => Promise<DigestItem[]>;
  priority: number; // Higher = shown first
}

interface DigestItem {
  source: string;
  title: string;
  summary: string;
  urgency: "high" | "medium" | "low";
  link?: string;
  timestamp: Date;
}

class DigestEngine {
  private sources: DigestSource[] = [];

  registerSource(source: DigestSource): void {
    this.sources.push(source);
    this.sources.sort((a, b) => b.priority - a.priority);
  }

  async generateDigest(userId: string): Promise<string> {
    const since = new Date();
    since.setHours(since.getHours() - 24); // Last 24 hours

    // Fetch from all sources in parallel
    const allItems = await Promise.allSettled(
      this.sources.map((source) =>
        source.fetch(userId, since).then((items) =>
          items.map((item) => ({ ...item, source: source.name }))
        )
      )
    );

    const items = allItems
      .filter(
        (r): r is PromiseFulfilledResult<DigestItem[]> =>
          r.status === "fulfilled"
      )
      .flatMap((r) => r.value);

    // Sort by urgency, then recency
    const urgencyOrder = { high: 0, medium: 1, low: 2 };
    items.sort(
      (a, b) =>
        urgencyOrder[a.urgency] - urgencyOrder[b.urgency] ||
        b.timestamp.getTime() - a.timestamp.getTime()
    );

    // Use AI to synthesize into a readable briefing
    const client = new Anthropic();
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 2048,
      system: `You are a personal assistant creating a morning briefing.
Be concise and actionable. Group by urgency.
Format: markdown with clear sections.
Start with the most important item. End with a "Today's focus" recommendation.`,
      messages: [
        {
          role: "user",
          content: `Create my morning briefing from these items:\n\n${JSON.stringify(items, null, 2)}`,
        },
      ],
    });

    return response.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");
  }
}

// Example source: Slack
const slackSource: DigestSource = {
  name: "Slack",
  priority: 8,
  async fetch(userId: string, since: Date): Promise<DigestItem[]> {
    // Use Slack connector to fetch unread messages from key channels
    const channels = await getUserKeyChannels(userId);
    const items: DigestItem[] = [];

    for (const channel of channels) {
      const messages = await getUnreadMessages(userId, channel.id, since);
      if (messages.length > 0) {
        items.push({
          source: "Slack",
          title: `${messages.length} unread in #${channel.name}`,
          summary: await summarizeMessages(messages),
          urgency: channel.priority === "high" ? "high" : "medium",
          link: `slack://channel?id=${channel.id}`,
          timestamp: new Date(messages[messages.length - 1].ts),
        });
      }
    }

    return items;
  },
};

// Example source: Linear
const linearSource: DigestSource = {
  name: "Linear",
  priority: 9,
  async fetch(userId: string, since: Date): Promise<DigestItem[]> {
    const assignedIssues = await getLinearIssuesForUser(userId);
    return assignedIssues
      .filter((issue) => new Date(issue.updatedAt) > since)
      .map((issue) => ({
        source: "Linear",
        title: `${issue.identifier}: ${issue.title}`,
        summary: `Status: ${issue.state.name}. ${issue.description?.slice(0, 200) || ""}`,
        urgency: issue.priority <= 1 ? "high" : issue.priority <= 2 ? "medium" : "low",
        link: issue.url,
        timestamp: new Date(issue.updatedAt),
      }));
  },
};
```

---

## 6. Headless Mode

### 6.1 What Headless Mode Is

Not every AI task needs a conversation. Sometimes you want to kick off a task, walk away, and have the results waiting when you come back. Headless mode runs the agent without a UI -- triggered by a schedule, a Slack command, or an API call.

This is how Glass supports background workflows: an engineer kicks off "review all PRs from last week for security issues," approves the tool permissions from their phone, and finds a comprehensive security report waiting in Slack when they get to their desk.

### 6.2 Background Task Runner

```typescript
// headless.ts — Background task execution

interface BackgroundTask {
  id: string;
  userId: string;
  prompt: string;
  status: "queued" | "running" | "awaiting_approval" | "completed" | "failed";
  createdAt: Date;
  completedAt?: Date;
  result?: string;
  approvals: {
    toolName: string;
    description: string;
    status: "pending" | "approved" | "denied";
  }[];
}

class BackgroundTaskRunner {
  private tasks: Map<string, BackgroundTask> = new Map();
  private notifier: NotificationService;

  constructor(notifier: NotificationService) {
    this.notifier = notifier;
  }

  async submitTask(userId: string, prompt: string): Promise<string> {
    const task: BackgroundTask = {
      id: crypto.randomUUID(),
      userId,
      prompt,
      status: "queued",
      createdAt: new Date(),
      approvals: [],
    };

    this.tasks.set(task.id, task);

    // Run asynchronously
    this.executeTask(task).catch((error) => {
      task.status = "failed";
      task.result = `Error: ${error}`;
      this.notifier.notify(userId, {
        type: "task_failed",
        taskId: task.id,
        error: String(error),
      });
    });

    return task.id;
  }

  private async executeTask(task: BackgroundTask): Promise<void> {
    task.status = "running";

    // Create agent with approval hooks
    const agent = await createAgentForUser(task.userId);

    // Custom tool wrapper that pauses for approval on sensitive tools
    const sensitiveTools = new Set([
      "write_file",
      "execute_code",
      "send_slack_message",
      "create_pr",
    ]);

    for await (const event of agent.run(task.prompt, [])) {
      if (
        event.type === "tool_call" &&
        sensitiveTools.has(event.metadata?.toolName as string)
      ) {
        // Pause and request approval
        task.status = "awaiting_approval";
        const approval = {
          toolName: event.metadata!.toolName as string,
          description: event.content,
          status: "pending" as const,
        };
        task.approvals.push(approval);

        // Notify user (Slack, push notification, etc.)
        await this.notifier.notify(task.userId, {
          type: "approval_needed",
          taskId: task.id,
          toolName: approval.toolName,
          description: approval.description,
          approveUrl: `/api/tasks/${task.id}/approve/${task.approvals.length - 1}`,
          denyUrl: `/api/tasks/${task.id}/deny/${task.approvals.length - 1}`,
        });

        // Wait for approval (with timeout)
        const approved = await this.waitForApproval(
          task.id,
          task.approvals.length - 1,
          300000 // 5 minute timeout
        );

        if (!approved) {
          task.status = "failed";
          task.result = `Task cancelled: approval denied for ${approval.toolName}`;
          return;
        }

        task.status = "running";
      }

      if (event.type === "done") {
        task.status = "completed";
        task.completedAt = new Date();
      }

      if (event.type === "text") {
        task.result = (task.result || "") + event.content;
      }
    }

    // Notify user of completion
    await this.notifier.notify(task.userId, {
      type: "task_completed",
      taskId: task.id,
      summary: task.result?.slice(0, 500),
    });
  }

  private async waitForApproval(
    taskId: string,
    approvalIndex: number,
    timeoutMs: number
  ): Promise<boolean> {
    return new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        const task = this.tasks.get(taskId);
        if (!task) {
          clearInterval(checkInterval);
          resolve(false);
          return;
        }
        const approval = task.approvals[approvalIndex];
        if (approval.status === "approved") {
          clearInterval(checkInterval);
          resolve(true);
        } else if (approval.status === "denied") {
          clearInterval(checkInterval);
          resolve(false);
        }
      }, 1000);

      setTimeout(() => {
        clearInterval(checkInterval);
        resolve(false);
      }, timeoutMs);
    });
  }

  // Called by the API when user approves/denies
  approve(taskId: string, approvalIndex: number): void {
    const task = this.tasks.get(taskId);
    if (task && task.approvals[approvalIndex]) {
      task.approvals[approvalIndex].status = "approved";
    }
  }

  deny(taskId: string, approvalIndex: number): void {
    const task = this.tasks.get(taskId);
    if (task && task.approvals[approvalIndex]) {
      task.approvals[approvalIndex].status = "denied";
    }
  }
}
```

### 6.3 Scheduled Tasks (Cron)

```typescript
// scheduler.ts — Cron-based task scheduling

interface ScheduledTask {
  id: string;
  userId: string;
  prompt: string;
  schedule: string; // Cron expression
  enabled: boolean;
  lastRun?: Date;
  nextRun: Date;
}

// Example scheduled tasks:
const EXAMPLE_SCHEDULES: Omit<ScheduledTask, "id" | "userId" | "nextRun">[] = [
  {
    prompt: "Generate my daily morning briefing digest",
    schedule: "0 8 * * 1-5", // 8am Mon-Fri
    enabled: true,
  },
  {
    prompt: "Review all PRs opened yesterday for security issues and post a summary to #security",
    schedule: "0 9 * * 1-5", // 9am Mon-Fri
    enabled: true,
  },
  {
    prompt: "Check our Snowflake query costs from the last 24 hours and alert me if any single query cost more than $5",
    schedule: "0 7 * * *", // 7am daily
    enabled: true,
  },
  {
    prompt: "Summarize all customer feedback from Gong calls this week and identify top themes",
    schedule: "0 17 * * 5", // 5pm Friday
    enabled: true,
  },
];
```

---

## 7. The Platform System Prompt

### 7.1 Why the System Prompt Matters More in Platform Tools

In a platform tool, the system prompt is not just instructions for the AI -- it is the product design. It determines what the tool feels like to use, what it can do, and how it presents information. A good platform system prompt is the difference between "powerful AI tool" and "confusing chat window."

```typescript
// system-prompt.ts — The platform's system prompt template

const PLATFORM_SYSTEM_PROMPT = `You are {{TOOL_NAME}}, an AI assistant built for {{COMPANY_NAME}} employees.

## About you
You have access to {{COMPANY_NAME}}'s internal tools and data through secure integrations.
You are talking to {{USER_NAME}}, who works as a {{USER_ROLE}} on the {{USER_TEAMS}} team(s).
Today is {{DATE}}.

## Your available tools
{{AVAILABLE_TOOLS}}

## How to behave
1. **Be direct.** No filler, no caveats, no "I'd be happy to help." Just answer.
2. **Use your tools proactively.** If the user asks about data, query it — don't ask them to go look it up.
3. **Cite your sources.** When you pull data from a tool, say which tool and what query.
4. **Handle errors gracefully.** If a tool call fails, try a different approach. Never show raw errors.
5. **Respect access boundaries.** Only use tools the user has access to. If they ask for something outside their access, explain why you can't help with that specific request.
6. **Be concise for simple questions, thorough for complex ones.** "What's the status of ticket ENG-1234?" gets a one-line answer. "What were the key themes from this quarter's sales calls?" gets a structured analysis.

## What you should NOT do
- Do not make up data. If you don't have access to the data, say so.
- Do not store or repeat sensitive data (passwords, tokens, SSNs, full credit card numbers).
- Do not take destructive actions (deleting data, closing accounts) without explicit confirmation.
- Do not pretend to be a human or hide that you are an AI when directly asked.

## Memory
You have access to memories from previous conversations with this user. Use them for context but don't over-reference them — the user may not remember what they told you last week.`;
```

---

## 8. Measuring Success

### 8.1 Metrics That Matter for Internal AI Tools

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| **DAU / Total employees** | Adoption breadth | >50% within 3 months |
| **Sessions per user per day** | Engagement depth | >2 |
| **Time to first value** | Onboarding friction | <5 minutes |
| **Tool call success rate** | Platform reliability | >99% |
| **P95 response latency** | UX quality | <10 seconds |
| **Tasks completed headlessly** | Automation adoption | Growing weekly |
| **Cross-team tool usage** | Platform reach | All teams represented |
| **User-reported errors** | Platform quality | <1% of sessions |

### 8.2 The Ramp Benchmark

Ramp achieved these numbers with Glass:
- **700 DAUs** within one month of launch
- **99.5%** of the team active on AI tools
- **84%** using coding agents weekly
- **12%** of human-initiated PRs from non-engineers
- **30+ tools** connected on install
- **4 engineers** built and maintain the platform

These numbers are achievable. They are not the result of an extraordinary team -- they are the result of an extraordinary *approach*: remove every friction point, connect every tool, and make using the platform easier than not using it.

---

## 9. Chapter Summary

Building an internal AI tool is not about building a chatbot. It is about building an *integration layer* that makes AI capabilities accessible to everyone in your organization with zero configuration. The Glass pattern -- SSO-driven tool discovery, pre-connected integrations, platform-handled errors, multi-pane workspace -- is the blueprint.

The architecture is straightforward: Agent SDK for the loop, tool connectors for integrations, SSO for authentication and authorization, a memory system for cross-session context, and sandboxed containers for safe execution. The hard part is not the code -- it is the *relentless focus on removing friction* from every interaction.

In Chapter 60, we build the system for sharing what works: a skills marketplace where every prompt, workflow, and automation that one person creates becomes available to everyone.

---

*Previous: [Ch 58 -- Multi-Agent Orchestration](./58-multi-agent-orchestration.md)* | *Next: [Ch 60 -- Skills Marketplaces & Knowledge Sharing](./60-skills-marketplaces.md)*

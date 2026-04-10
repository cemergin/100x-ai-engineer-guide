<!--
  CHAPTER: 6
  TITLE: Building a Chat Interface
  PART: 1 — Building with LLM APIs
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 3, Chapter 4, Chapter 5
  KEY_TOPICS: conversation history, streaming to UI, Server-Sent Events, Vercel AI SDK, useChat, Next.js, token counting
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 6: Building a Chat Interface

> **Part 1 — Building with LLM APIs** | Phase 1: Get Dangerous | Prerequisites: Ch 3, Ch 4, Ch 5 | Difficulty: Intermediate | Language: TypeScript

You can call LLMs, write good prompts, and get structured data back. Now let's build something users actually interact with: a streaming chat interface.

Chat is the dominant UI pattern for AI features. It's what users expect when they hear "AI." And while it looks simple — a text box, some message bubbles, a spinner — getting it right involves real engineering: managing conversation history that grows with every turn, streaming responses so the UI feels instant, handling edge cases like interrupted streams and context window overflow, and making it all work with your backend.

This chapter takes you from a terminal-based conversation (Ch 3) to a real chat UI built with Next.js and the Vercel AI SDK. By the end, you'll have a fully working chat application with streaming responses, conversation history, and the knowledge to extend it for your product.

### In This Chapter
- Managing conversation history (the messages array pattern)
- Streaming to a UI with Server-Sent Events (SSE)
- The Vercel AI SDK: useChat hook, route handlers
- Building a real chat experience with Next.js
- Token counting and conversation cost management

### Related Chapters
- **Ch 3 (Your First LLM Call)** — spirals back: single call becomes multi-turn conversation
- **Ch 0 (How LLMs Work)** — spirals back: context window fills up as conversation grows
- **Ch 10 (Agent Memory & State)** — spirals forward: adding persistent memory across sessions
- **Ch 12 (Context Window Management)** — spirals forward: compaction and summarization strategies

---

## 1. Conversation History: The Core Pattern

### 1.1 How Chat Works Under the Hood

Every "chat" with an LLM is an illusion. The model has no memory. Each API call is independent. The conversation feels continuous because YOU maintain the history and send it all with every request:

```
Turn 1:
  Send:    [system, user_1]
  Receive: assistant_1

Turn 2:
  Send:    [system, user_1, assistant_1, user_2]
  Receive: assistant_2

Turn 3:
  Send:    [system, user_1, assistant_1, user_2, assistant_2, user_3]
  Receive: assistant_3

Turn 20:
  Send:    [system, user_1, assistant_1, ..., user_20]   ← ALL previous messages
  Receive: assistant_20
```

This means:
- **Cost grows with conversation length.** Turn 20 sends 20x more tokens than turn 1.
- **Latency increases.** More input tokens = slower time-to-first-token.
- **Context windows have limits.** Eventually, the conversation won't fit.

### 1.2 The Messages Array Pattern

The fundamental data structure is simple:

```typescript
import { type CoreMessage } from "ai";

// The conversation state — this is all you need
const messages: CoreMessage[] = [];

// System prompt (set once)
const systemPrompt: CoreMessage = {
  role: "system",
  content: "You are a helpful assistant for Acme Corp.",
};

// When the user sends a message:
function addUserMessage(text: string) {
  messages.push({ role: "user", content: text });
}

// When the model responds:
function addAssistantMessage(text: string) {
  messages.push({ role: "assistant", content: text });
}

// When making an API call:
function getMessagesForAPI(): CoreMessage[] {
  return [systemPrompt, ...messages];
}
```

### 1.3 Token Tracking

Build token awareness into your chat from the start:

```typescript
interface ConversationState {
  messages: CoreMessage[];
  totalTokensSent: number;
  totalTokensReceived: number;
  estimatedCost: number;
}

function estimateTokenCount(text: string): number {
  // Rough estimation: 1 token ≈ 4 characters for English text
  return Math.ceil(text.length / 4);
}

function getConversationTokens(messages: CoreMessage[]): number {
  return messages.reduce((total, msg) => {
    const content = typeof msg.content === "string" ? msg.content : "";
    return total + estimateTokenCount(content);
  }, 0);
}

// Check before sending
function canSendMessage(messages: CoreMessage[], maxContextTokens: number = 128000): boolean {
  const currentTokens = getConversationTokens(messages);
  const reserveForResponse = 4096; // leave room for the response
  return currentTokens + reserveForResponse < maxContextTokens;
}
```

---

## 2. Streaming to a UI: Server-Sent Events

### 2.1 What Are Server-Sent Events (SSE)?

When the LLM streams tokens, you need a way to push them from your server to the browser in real-time. **Server-Sent Events (SSE)** is the standard HTTP mechanism for this:

```
Browser          Server           LLM Provider
  |                |                   |
  |--- POST /chat -|                   |
  |                |--- API request -->|
  |                |                   |
  |                |<-- token 1 -------|
  |<-- SSE event --|                   |
  |                |<-- token 2 -------|
  |<-- SSE event --|                   |
  |                |<-- token 3 -------|
  |<-- SSE event --|                   |
  |                |<-- [done] --------|
  |<-- SSE close --|                   |
```

SSE is HTTP — no WebSocket setup needed. The browser makes a regular request, the server keeps the connection open, and sends text events as they arrive. The Vercel AI SDK handles all of this.

### 2.2 Why Not WebSockets?

WebSockets are bidirectional (both sides can send at any time). SSE is server-to-client only. For chat:
- The client sends a single request (user message)
- The server streams back one response
- That's it — no need for bidirectional communication

SSE is simpler, works with standard HTTP infrastructure (load balancers, CDNs, proxies), and the Vercel AI SDK uses it by default. Use WebSockets only if you need real-time bidirectional communication (collaborative editing, multiplayer features).

---

## 3. Building a Chat App with Next.js

### 3.1 Project Setup

Create a new Next.js project with the AI SDK:

```bash
npx create-next-app@latest ai-chat-demo --typescript --tailwind --app --use-npm
cd ai-chat-demo

# Install AI SDK
npm install ai @ai-sdk/openai @ai-sdk/anthropic zod
```

Set up your environment:

```bash
# .env.local
OPENAI_API_KEY=sk-proj-your-key-here
```

### 3.2 The API Route (Server-Side)

Create the chat API route that streams responses:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

export const maxDuration = 60; // Allow streaming for up to 60 seconds

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: `You are a helpful assistant for a software company. 
You help developers with TypeScript, React, and Node.js questions.
Be concise but thorough. Use code examples when relevant.
Format code with proper syntax highlighting.`,
    messages,
  });

  return result.toDataStreamResponse();
}
```

That's the entire backend. The Vercel AI SDK's `toDataStreamResponse()` converts the streaming LLM response into an SSE stream that the frontend `useChat` hook understands.

### 3.3 The Chat UI (Client-Side)

Build the chat interface:

```typescript
// app/page.tsx
"use client";

import { useChat } from "@ai-sdk/react";

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } =
    useChat();

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto">
      {/* Header */}
      <header className="p-4 border-b">
        <h1 className="text-lg font-semibold">AI Chat</h1>
      </header>

      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.length === 0 && (
          <div className="text-center text-gray-500 mt-8">
            <p className="text-lg">How can I help you today?</p>
            <p className="text-sm mt-2">
              Ask me about TypeScript, React, or Node.js
            </p>
          </div>
        )}

        {messages.map((message) => (
          <div
            key={message.id}
            className={`flex ${
              message.role === "user" ? "justify-end" : "justify-start"
            }`}
          >
            <div
              className={`max-w-[80%] rounded-lg px-4 py-2 ${
                message.role === "user"
                  ? "bg-blue-600 text-white"
                  : "bg-gray-100 text-gray-900"
              }`}
            >
              <div className="text-sm font-medium mb-1">
                {message.role === "user" ? "You" : "Assistant"}
              </div>
              <div className="whitespace-pre-wrap">{message.content}</div>
            </div>
          </div>
        ))}

        {isLoading && (
          <div className="flex justify-start">
            <div className="bg-gray-100 rounded-lg px-4 py-2">
              <div className="animate-pulse">Thinking...</div>
            </div>
          </div>
        )}

        {error && (
          <div className="bg-red-50 text-red-600 p-3 rounded-lg">
            Error: {error.message}
          </div>
        )}
      </div>

      {/* Input */}
      <form onSubmit={handleSubmit} className="p-4 border-t">
        <div className="flex gap-2">
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Type a message..."
            className="flex-1 p-3 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading || !input.trim()}
            className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Send
          </button>
        </div>
      </form>
    </div>
  );
}
```

Run it:

```bash
npm run dev
```

Open `http://localhost:3000` and you have a working chat interface with streaming responses. That's roughly 80 lines of code for the UI and 15 for the backend.

### 3.4 What useChat Does for You

The `useChat` hook manages all the complexity:

- **Message state** — maintains the messages array, adds user and assistant messages automatically
- **Streaming** — reads the SSE stream, updates the assistant message in real-time
- **Loading state** — `isLoading` is true while the model is generating
- **Error handling** — catches network errors and API errors
- **Input management** — `input`, `handleInputChange`, `handleSubmit` manage the text input
- **Abort** — the `stop` function cancels an in-progress stream

```typescript
const {
  messages,          // CoreMessage[] — full conversation history
  input,             // string — current input value
  handleInputChange, // React change handler for input
  handleSubmit,      // Form submit handler (sends message)
  isLoading,         // boolean — true during streaming
  error,             // Error | undefined
  stop,              // () => void — cancel current stream
  reload,            // () => void — resend the last message
  setMessages,       // (messages) => void — override history
  append,            // (message) => void — add a message programmatically
} = useChat({
  api: "/api/chat",   // API endpoint (default: /api/chat)
  initialMessages: [], // Pre-populate with messages
  onFinish: (message) => {
    // Called when streaming completes
    console.log("Finished:", message);
  },
  onError: (error) => {
    console.error("Chat error:", error);
  },
});
```

---

## 4. Enhancing the Chat

### 4.1 Adding a Stop Button

Let users cancel long responses:

```typescript
"use client";

import { useChat } from "@ai-sdk/react";

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, stop } =
    useChat();

  return (
    <div>
      {/* ... messages display ... */}

      <form onSubmit={handleSubmit} className="p-4 border-t">
        <div className="flex gap-2">
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Type a message..."
            className="flex-1 p-3 border rounded-lg"
            disabled={isLoading}
          />
          {isLoading ? (
            <button
              type="button"
              onClick={stop}
              className="px-6 py-3 bg-red-600 text-white rounded-lg hover:bg-red-700"
            >
              Stop
            </button>
          ) : (
            <button
              type="submit"
              disabled={!input.trim()}
              className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
            >
              Send
            </button>
          )}
        </div>
      </form>
    </div>
  );
}
```

### 4.2 Markdown Rendering

LLM responses often include markdown (headers, code blocks, lists). Render it properly:

```bash
npm install react-markdown
```

```typescript
// components/message-content.tsx
"use client";

import ReactMarkdown from "react-markdown";

export function MessageContent({ content }: { content: string }) {
  return (
    <ReactMarkdown
      className="prose prose-sm max-w-none"
      components={{
        code({ className, children, ...props }) {
          const isInline = !className;
          if (isInline) {
            return (
              <code className="bg-gray-200 px-1 py-0.5 rounded text-sm" {...props}>
                {children}
              </code>
            );
          }
          return (
            <pre className="bg-gray-900 text-gray-100 p-4 rounded-lg overflow-x-auto">
              <code className={className} {...props}>
                {children}
              </code>
            </pre>
          );
        },
      }}
    >
      {content}
    </ReactMarkdown>
  );
}
```

Then use it in your messages:

```typescript
{messages.map((message) => (
  <div key={message.id} className={/* ... */}>
    <div className={/* ... */}>
      {message.role === "user" ? (
        <p>{message.content}</p>
      ) : (
        <MessageContent content={message.content} />
      )}
    </div>
  </div>
))}
```

### 4.3 System Prompt Configuration

Make the system prompt configurable from the frontend:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

const SYSTEM_PROMPTS: Record<string, string> = {
  default: "You are a helpful assistant.",
  code: `You are a senior TypeScript engineer. Always show working code examples.
Use modern TypeScript patterns. Explain your reasoning.`,
  creative: `You are a creative writing partner. Be imaginative, use vivid language,
and help the user develop their ideas.`,
  concise: `You are a concise assistant. Answer in 1-3 sentences maximum.
No fluff, no caveats, just the answer.`,
};

export async function POST(req: Request) {
  const { messages, persona = "default" } = await req.json();

  const system = SYSTEM_PROMPTS[persona] ?? SYSTEM_PROMPTS.default;

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system,
    messages,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// Client-side: send persona with the request
const { messages, input, handleInputChange, handleSubmit } = useChat({
  body: {
    persona: selectedPersona, // "default" | "code" | "creative" | "concise"
  },
});
```

### 4.4 Multi-Provider Chat

Let users choose the model:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";

function getModel(modelId: string) {
  switch (modelId) {
    case "gpt-4o-mini":
      return openai("gpt-4o-mini");
    case "gpt-4o":
      return openai("gpt-4o");
    case "claude-haiku":
      return anthropic("claude-3-5-haiku-20241022");
    case "claude-sonnet":
      return anthropic("claude-sonnet-4-20250514");
    default:
      return openai("gpt-4o-mini");
  }
}

export async function POST(req: Request) {
  const { messages, model: modelId = "gpt-4o-mini" } = await req.json();

  const result = streamText({
    model: getModel(modelId),
    system: "You are a helpful assistant.",
    messages,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// Client-side
const [selectedModel, setSelectedModel] = useState("gpt-4o-mini");

const chat = useChat({
  body: { model: selectedModel },
});
```

---

## 5. Conversation Management

### 5.1 Clearing the Conversation

```typescript
const { setMessages } = useChat();

function clearChat() {
  setMessages([]);
}
```

### 5.2 Conversation Persistence

Save conversations to localStorage for persistence across page refreshes:

```typescript
"use client";

import { useChat } from "@ai-sdk/react";
import { useEffect } from "react";

const STORAGE_KEY = "chat-messages";

export default function ChatPage() {
  const { messages, setMessages, ...chatProps } = useChat({
    onFinish: () => {
      // Save after each assistant response
      localStorage.setItem(STORAGE_KEY, JSON.stringify(messages));
    },
  });

  // Load on mount
  useEffect(() => {
    const saved = localStorage.getItem(STORAGE_KEY);
    if (saved) {
      try {
        setMessages(JSON.parse(saved));
      } catch {
        // Corrupted data, start fresh
        localStorage.removeItem(STORAGE_KEY);
      }
    }
  }, [setMessages]);

  // ... rest of the UI
}
```

For production apps, store conversations in a database:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

export async function POST(req: Request) {
  const { messages, conversationId } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant.",
    messages,
    onFinish: async ({ text, usage }) => {
      // Save to database after streaming completes
      await saveMessage({
        conversationId,
        role: "assistant",
        content: text,
        tokenCount: usage.totalTokens,
        timestamp: new Date(),
      });
    },
  });

  return result.toDataStreamResponse();
}
```

### 5.3 Multiple Conversations

Support multiple conversation threads:

```typescript
"use client";

import { useChat } from "@ai-sdk/react";
import { useState } from "react";

interface Conversation {
  id: string;
  title: string;
  createdAt: Date;
}

export default function ChatApp() {
  const [conversations, setConversations] = useState<Conversation[]>([]);
  const [activeId, setActiveId] = useState<string | null>(null);

  const { messages, setMessages, ...chatProps } = useChat({
    id: activeId ?? undefined, // useChat scopes state by id
    body: { conversationId: activeId },
  });

  function newConversation() {
    const id = crypto.randomUUID();
    setConversations((prev) => [
      { id, title: "New Chat", createdAt: new Date() },
      ...prev,
    ]);
    setActiveId(id);
    setMessages([]);
  }

  return (
    <div className="flex h-screen">
      {/* Sidebar */}
      <aside className="w-64 border-r p-4 overflow-y-auto">
        <button
          onClick={newConversation}
          className="w-full p-2 bg-blue-600 text-white rounded mb-4"
        >
          New Chat
        </button>
        {conversations.map((conv) => (
          <button
            key={conv.id}
            onClick={() => setActiveId(conv.id)}
            className={`w-full text-left p-2 rounded mb-1 ${
              activeId === conv.id ? "bg-blue-100" : "hover:bg-gray-100"
            }`}
          >
            {conv.title}
          </button>
        ))}
      </aside>

      {/* Chat area */}
      <main className="flex-1">
        {/* ... message display and input ... */}
      </main>
    </div>
  );
}
```

---

## 6. Token Counting and Cost Awareness

### 6.1 Tracking Tokens Per Conversation

Add token tracking to your API route:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant.",
    messages,
    onFinish: async ({ usage }) => {
      console.log("Token usage:", {
        promptTokens: usage.promptTokens,
        completionTokens: usage.completionTokens,
        totalTokens: usage.totalTokens,
        estimatedCost: (
          (usage.promptTokens / 1_000_000) * 0.15 +
          (usage.completionTokens / 1_000_000) * 0.60
        ).toFixed(6),
      });
    },
  });

  return result.toDataStreamResponse();
}
```

### 6.2 When Conversations Get Expensive

Here's a real cost breakdown for a 50-turn conversation using gpt-4o-mini:

```
Turn  1:  ~600 input tokens   → $0.0001
Turn 10:  ~4,000 input tokens → $0.0006
Turn 20:  ~8,000 input tokens → $0.0012
Turn 30:  ~12,000 input tokens → $0.0018
Turn 40:  ~16,000 input tokens → $0.0024
Turn 50:  ~20,000 input tokens → $0.0030

Total input tokens across all 50 turns: ~500,000
Total output tokens: ~15,000

Total cost: ~$0.084 for ONE conversation with ONE user
```

At $0.08 per conversation, 1,000 daily users having 1 conversation each = $80/day = $2,400/month.

With gpt-4o, multiply by ~17x: $1.43 per conversation, $1,430/day, $43,000/month.

This is why context management matters. We cover strategies in Ch 12 (sliding windows, summarization, compaction).

### 6.3 Simple Context Windowing

A quick win: limit the number of messages sent to the API:

```typescript
// app/api/chat/route.ts
import { streamText, type CoreMessage } from "ai";
import { openai } from "@ai-sdk/openai";

const MAX_MESSAGES = 20; // Keep last 20 messages

function trimMessages(messages: CoreMessage[]): CoreMessage[] {
  if (messages.length <= MAX_MESSAGES) {
    return messages;
  }

  // Keep the most recent messages
  return messages.slice(-MAX_MESSAGES);
}

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant.",
    messages: trimMessages(messages),
  });

  return result.toDataStreamResponse();
}
```

This is crude but effective. The model loses early context but stays within budget. For smarter strategies (summarizing old messages, keeping important ones), see Ch 12.

---

## 7. Error Handling in the Chat UI

### 7.1 Network Errors

```typescript
const { error, reload } = useChat({
  onError: (err) => {
    console.error("Chat error:", err);
    // Optionally show a toast notification
  },
});

// In the UI
{error && (
  <div className="bg-red-50 border border-red-200 p-4 rounded-lg mx-4 mb-4">
    <p className="text-red-600 font-medium">Something went wrong</p>
    <p className="text-red-500 text-sm mt-1">{error.message}</p>
    <button
      onClick={() => reload()}
      className="mt-2 text-sm text-red-600 underline hover:text-red-800"
    >
      Try again
    </button>
  </div>
)}
```

### 7.2 Rate Limit Handling

Handle 429 errors gracefully:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

export async function POST(req: Request) {
  const { messages } = await req.json();

  try {
    const result = streamText({
      model: openai("gpt-4o-mini"),
      system: "You are a helpful assistant.",
      messages,
    });

    return result.toDataStreamResponse();
  } catch (error) {
    if (error instanceof Error && error.message.includes("rate_limit")) {
      return new Response(
        JSON.stringify({ error: "Too many requests. Please wait a moment and try again." }),
        { status: 429, headers: { "Content-Type": "application/json" } }
      );
    }

    return new Response(
      JSON.stringify({ error: "An error occurred. Please try again." }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }
}
```

### 7.3 Graceful Degradation

If the AI is unavailable, don't show a blank page:

```typescript
{error && error.message.includes("rate") ? (
  <div className="p-4 bg-yellow-50 rounded-lg mx-4 mb-4">
    <p>We're experiencing high demand. Your message will be sent shortly...</p>
    <button onClick={() => reload()} className="mt-2 underline">
      Retry now
    </button>
  </div>
) : error ? (
  <div className="p-4 bg-red-50 rounded-lg mx-4 mb-4">
    <p>The AI assistant is temporarily unavailable.</p>
    <p className="text-sm text-gray-600 mt-1">
      You can still browse our{" "}
      <a href="/docs" className="underline">documentation</a> or{" "}
      <a href="/support" className="underline">contact support</a>.
    </p>
  </div>
) : null}
```

---

## 8. Production Considerations

### 8.1 Authentication

Protect your chat API from unauthorized use:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { auth } from "@/lib/auth"; // your auth library

export async function POST(req: Request) {
  // Check authentication
  const session = await auth();
  if (!session?.user) {
    return new Response("Unauthorized", { status: 401 });
  }

  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant.",
    messages,
  });

  return result.toDataStreamResponse();
}
```

### 8.2 Rate Limiting Your Users

Don't let one user burn through your API budget:

```typescript
// Simple in-memory rate limiter (use Redis in production)
const userRequestCounts = new Map<string, { count: number; resetAt: number }>();

function checkRateLimit(userId: string, maxPerMinute: number = 10): boolean {
  const now = Date.now();
  const record = userRequestCounts.get(userId);

  if (!record || record.resetAt < now) {
    userRequestCounts.set(userId, { count: 1, resetAt: now + 60_000 });
    return true;
  }

  if (record.count >= maxPerMinute) {
    return false;
  }

  record.count++;
  return true;
}

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) return new Response("Unauthorized", { status: 401 });

  if (!checkRateLimit(session.user.id)) {
    return new Response("Too many messages. Please wait a minute.", { status: 429 });
  }

  // ... rest of the handler
}
```

### 8.3 Input Validation

Validate incoming messages:

```typescript
import { z } from "zod";

const ChatRequestSchema = z.object({
  messages: z.array(z.object({
    role: z.enum(["user", "assistant"]),
    content: z.string().max(10000), // Cap message length
  })).max(100), // Cap conversation length
});

export async function POST(req: Request) {
  const body = await req.json();
  
  const parseResult = ChatRequestSchema.safeParse(body);
  if (!parseResult.success) {
    return new Response(
      JSON.stringify({ error: "Invalid request", details: parseResult.error.issues }),
      { status: 400 }
    );
  }

  const { messages } = parseResult.data;
  // ... continue with validated data
}
```

### 8.4 Logging for Observability

Log every chat interaction for debugging and analytics:

```typescript
export async function POST(req: Request) {
  const session = await auth();
  const { messages, conversationId } = await req.json();
  const startTime = Date.now();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant.",
    messages,
    onFinish: async ({ text, usage, finishReason }) => {
      // Log to your analytics/observability system
      await log({
        event: "chat_completion",
        userId: session?.user?.id,
        conversationId,
        messageCount: messages.length,
        promptTokens: usage.promptTokens,
        completionTokens: usage.completionTokens,
        finishReason,
        durationMs: Date.now() - startTime,
        model: "gpt-4o-mini",
        timestamp: new Date().toISOString(),
      });
    },
  });

  return result.toDataStreamResponse();
}
```

---

## 9. The Full Vercel AI SDK Toolkit

So far in this chapter you have used `streamText` and `useChat` — the streaming pair. But the Vercel AI SDK has more tools, and this is the right place to see all of them together before you move on.

### 9.1 generateText: Non-Streaming with Tool Calling

Not everything needs to stream. Backend tasks, cron jobs, and API endpoints that return a final result can use `generateText`:

```typescript
import { generateText, tool } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { z } from "zod";

const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  system: "You are a helpful coding assistant.",
  tools: {
    getWeather: tool({
      description: "Get the current weather for a city",
      parameters: z.object({
        city: z.string().describe("City name"),
      }),
      execute: async ({ city }) => {
        // Call a weather API
        const response = await fetch(
          `https://api.weather.com/v1/current?city=${encodeURIComponent(city)}`
        );
        return response.json();
      },
    }),
  },
  maxSteps: 5, // Allow up to 5 tool-call rounds
  prompt: "What's the weather in San Francisco and Tokyo?",
});

console.log(result.text);          // Final text response
console.log(result.steps.length);  // How many steps (LLM calls) it took
console.log(result.usage);         // { promptTokens, completionTokens, totalTokens }
```

`streamText` and `generateText` accept the same options — `tools`, `maxSteps`, `system`, `messages`. The difference is only in how the result is delivered: all at once vs token by token.

### 9.2 generateObject and streamObject: Structured Output

When you need structured data (not free-form text), use `generateObject` or `streamObject`. These use the model's structured output mode to guarantee the response matches a Zod schema:

```typescript
import { generateObject, streamObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

// Non-streaming: get the entire object at once
const { object } = await generateObject({
  model: openai("gpt-4o-mini"),
  schema: z.object({
    summary: z.string().describe("One-paragraph summary"),
    sentiment: z.enum(["positive", "negative", "neutral"]),
    topics: z.array(z.string()).describe("Key topics mentioned"),
    actionItems: z.array(
      z.object({
        task: z.string(),
        assignee: z.string().optional(),
        priority: z.enum(["high", "medium", "low"]),
      })
    ),
  }),
  prompt: `Analyze this meeting transcript: "${transcript}"`,
});

// object is fully typed: object.summary, object.sentiment, etc.
console.log(object.actionItems);
```

For the streaming version — useful when the structured data is large and you want to show partial results:

```typescript
// Streaming: get partial objects as they arrive
const { partialObjectStream } = streamObject({
  model: openai("gpt-4o-mini"),
  schema: z.object({
    summary: z.string(),
    topics: z.array(z.string()),
  }),
  prompt: `Analyze this document: "${document}"`,
});

for await (const partialObject of partialObjectStream) {
  // partialObject grows as the model generates
  // e.g., first: { summary: "The doc..." }
  // then:  { summary: "The document covers...", topics: ["AI"] }
  console.log(partialObject);
}
```

This is the structured output pattern from Chapter 5, but built into the SDK with streaming support. Use `generateObject` for backend processing, `streamObject` for UI where you want progressive rendering.

### 9.3 The Provider Pattern: One Codebase, Any Model

Section 4.4 showed the multi-provider switch statement. The deeper point is that the Vercel AI SDK's provider pattern makes your code model-agnostic. Every provider (`@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google`) returns objects that satisfy the same interface:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";
import { google } from "@ai-sdk/google";

// The same code works with any provider
async function analyze(model: Parameters<typeof generateText>[0]["model"], text: string) {
  return generateText({
    model,
    prompt: `Analyze this text: ${text}`,
  });
}

// Swap providers without changing business logic
await analyze(openai("gpt-4o"), text);
await analyze(anthropic("claude-sonnet-4-20250514"), text);
await analyze(google("gemini-2.0-flash"), text);
```

This matters for production: you can A/B test providers, fall back when one is down, or let users choose their preferred model — all without duplicating code.

### 9.4 experimental_telemetry: Observability Built In

The AI SDK supports OpenTelemetry for tracing every LLM call. This is how you get observability into what your AI features are doing in production:

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const result = await generateText({
  model: openai("gpt-4o-mini"),
  prompt: "Summarize this document...",
  experimental_telemetry: {
    isEnabled: true,
    functionId: "document-summarizer",  // Identifies this call in traces
    metadata: {
      userId: "user-123",
      documentId: "doc-456",
      environment: "production",
    },
  },
});
```

With telemetry enabled, every `generateText`, `streamText`, `generateObject`, and `streamObject` call emits OpenTelemetry spans. You can send these to any observability backend — Langfuse, Datadog, Honeycomb, or your own collector:

```typescript
// instrumentation.ts — set up once at app startup
import { registerOTel } from "@vercel/otel";

registerOTel({
  serviceName: "my-ai-app",
});
```

Each span captures:
- Model name, provider, and parameters
- Input tokens, output tokens, total cost
- Latency (time to first token, total duration)
- Tool calls made during the request
- Custom metadata you attached

This is how you answer "why is this feature slow?", "which model costs more?", and "what is the AI doing for this user?" in production. We cover observability patterns in depth in Chapter 43.

---

## 10. AI UX Patterns

The basic chat works. Now let us make it feel like a polished product. AI interfaces have unique UX challenges that traditional web apps do not face: responses take seconds, not milliseconds. Output length is unpredictable. The model can fail or hallucinate. Users need to understand what they are paying for. This section covers the patterns that separate a prototype from a product.

### 10.1 Streaming UX: Word-by-Word Response

The most important AI UX pattern is streaming. Users should see text appearing word by word, not wait for the full response. The `useChat` hook handles this automatically, but you can enhance it with a blinking cursor and smooth animation:

```typescript
// components/streaming-message.tsx
"use client";

import { useEffect, useRef } from "react";

interface StreamingMessageProps {
  content: string;
  isStreaming: boolean;
}

export function StreamingMessage({ content, isStreaming }: StreamingMessageProps) {
  const endRef = useRef<HTMLSpanElement>(null);

  // Auto-scroll as new content arrives
  useEffect(() => {
    if (isStreaming) {
      endRef.current?.scrollIntoView({ behavior: "smooth" });
    }
  }, [content, isStreaming]);

  return (
    <div className="whitespace-pre-wrap">
      {content}
      {isStreaming && (
        <span
          ref={endRef}
          className="inline-block w-2 h-4 bg-gray-800 animate-pulse ml-0.5 align-text-bottom"
          aria-label="AI is typing"
        />
      )}
    </div>
  );
}
```

Use it in your message list:

```typescript
{messages.map((message, index) => (
  <div key={message.id} className={/* ... */}>
    {message.role === "assistant" ? (
      <StreamingMessage
        content={message.content}
        isStreaming={isLoading && index === messages.length - 1}
      />
    ) : (
      <p>{message.content}</p>
    )}
  </div>
))}
```

### 10.2 Loading States and Thinking Indicators

Before the first token arrives, the user sees nothing. That gap -- typically 500ms to 3 seconds -- needs a loading state. There are three common patterns:

```typescript
// Pattern 1: Simple "thinking" indicator
function ThinkingIndicator() {
  return (
    <div className="flex items-center gap-2 text-gray-500 p-3">
      <div className="flex gap-1">
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce [animation-delay:-0.3s]" />
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce [animation-delay:-0.15s]" />
        <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" />
      </div>
      <span className="text-sm">Thinking...</span>
    </div>
  );
}

// Pattern 2: Skeleton screen (for structured responses)
function SkeletonResponse() {
  return (
    <div className="space-y-3 p-4 animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-3/4" />
      <div className="h-4 bg-gray-200 rounded w-full" />
      <div className="h-4 bg-gray-200 rounded w-5/6" />
      <div className="h-4 bg-gray-200 rounded w-2/3" />
    </div>
  );
}

// Pattern 3: Extended thinking with elapsed time
function ExtendedThinkingIndicator() {
  const [elapsed, setElapsed] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => setElapsed((e) => e + 1), 1000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="flex items-center gap-2 text-gray-500 p-3">
      <svg className="animate-spin h-4 w-4" viewBox="0 0 24 24">
        <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" fill="none" />
        <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
      </svg>
      <span className="text-sm">
        {elapsed < 5 ? "Thinking..." : `Still working... (${elapsed}s)`}
      </span>
    </div>
  );
}
```

Use these in the chat based on where the user is in the stream:

```typescript
{isLoading && messages[messages.length - 1]?.role === "user" && (
  // No assistant message yet — show thinking indicator
  <ThinkingIndicator />
)}
```

### 10.3 Confidence Indicators

When the AI is uncertain, you should tell the user. This requires the API route to pass confidence metadata alongside the response:

```typescript
// app/api/chat/route.ts
import { streamText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: anthropic("claude-sonnet-4-20250514"),
    system: `You are a helpful assistant.
When you are not confident in your answer, start your response with [LOW_CONFIDENCE].
When you are referencing information you cannot verify, say so explicitly.`,
    messages,
  });

  return result.toDataStreamResponse();
}
```

```typescript
// components/confidence-badge.tsx
export function ConfidenceBadge({ content }: { content: string }) {
  const isLowConfidence = content.startsWith("[LOW_CONFIDENCE]");

  if (!isLowConfidence) return null;

  return (
    <div className="flex items-center gap-1 text-amber-600 text-xs mb-2">
      <svg className="w-3.5 h-3.5" fill="currentColor" viewBox="0 0 20 20">
        <path
          fillRule="evenodd"
          d="M8.485 2.495c.673-1.167 2.357-1.167 3.03 0l6.28 10.875c.673 1.167-.168 2.625-1.516 2.625H3.72c-1.347 0-2.189-1.458-1.515-2.625L8.485 2.495zM10 6a.75.75 0 01.75.75v3.5a.75.75 0 01-1.5 0v-3.5A.75.75 0 0110 6zm0 9a1 1 0 100-2 1 1 0 000 2z"
          clipRule="evenodd"
        />
      </svg>
      <span>The AI is less confident about this response</span>
    </div>
  );
}
```

### 10.4 Error Recovery UX

AI failures are different from traditional API failures. The model might time out, hit rate limits, or produce an empty response. Each needs a different recovery pattern:

```typescript
// components/error-recovery.tsx
"use client";

interface ChatErrorProps {
  error: Error;
  onRetry: () => void;
  onNewMessage: () => void;
}

export function ChatError({ error, onRetry, onNewMessage }: ChatErrorProps) {
  const errorType = classifyError(error);

  return (
    <div className="mx-4 mb-4 rounded-lg border p-4" role="alert">
      {errorType === "rate_limit" && (
        <div className="bg-amber-50 border-amber-200">
          <p className="font-medium text-amber-800">Too many requests</p>
          <p className="text-sm text-amber-700 mt-1">
            Please wait a moment before sending another message.
          </p>
          <button
            onClick={onRetry}
            className="mt-2 text-sm text-amber-800 underline"
          >
            Try again
          </button>
        </div>
      )}

      {errorType === "timeout" && (
        <div className="bg-blue-50 border-blue-200">
          <p className="font-medium text-blue-800">Response timed out</p>
          <p className="text-sm text-blue-700 mt-1">
            The AI took too long to respond. This can happen with complex questions.
          </p>
          <div className="flex gap-3 mt-2">
            <button onClick={onRetry} className="text-sm text-blue-800 underline">
              Retry same question
            </button>
            <button onClick={onNewMessage} className="text-sm text-blue-800 underline">
              Try a simpler question
            </button>
          </div>
        </div>
      )}

      {errorType === "server_error" && (
        <div className="bg-red-50 border-red-200">
          <p className="font-medium text-red-800">Something went wrong</p>
          <p className="text-sm text-red-700 mt-1">
            The AI service is temporarily unavailable.
          </p>
          <button onClick={onRetry} className="mt-2 text-sm text-red-800 underline">
            Try again
          </button>
        </div>
      )}
    </div>
  );
}

function classifyError(error: Error): "rate_limit" | "timeout" | "server_error" {
  const msg = error.message.toLowerCase();
  if (msg.includes("rate") || msg.includes("429")) return "rate_limit";
  if (msg.includes("timeout") || msg.includes("aborted")) return "timeout";
  return "server_error";
}
```

### 10.5 Regenerate Button

Let users ask for a different response when the first one is not useful:

```typescript
// In your message component
function AssistantMessage({
  message,
  isLast,
  onRegenerate,
}: {
  message: { id: string; content: string };
  isLast: boolean;
  onRegenerate: () => void;
}) {
  return (
    <div className="group relative">
      <div className="bg-gray-100 rounded-lg px-4 py-2">
        <MessageContent content={message.content} />
      </div>

      {/* Show regenerate button on the last assistant message */}
      {isLast && (
        <div className="opacity-0 group-hover:opacity-100 transition-opacity mt-1">
          <button
            onClick={onRegenerate}
            className="text-xs text-gray-500 hover:text-gray-700 flex items-center gap-1"
          >
            <svg className="w-3 h-3" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
                d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
              />
            </svg>
            Regenerate
          </button>
        </div>
      )}
    </div>
  );
}

// In your chat page, wire up the reload function
const { messages, reload, isLoading } = useChat();

// reload() resends the last user message to get a new response
```

### 10.6 Message Actions: Copy, Edit, Branch

Production chat interfaces need actions beyond send and receive:

```typescript
// components/message-actions.tsx
"use client";

import { useState } from "react";

interface MessageActionsProps {
  content: string;
  messageId: string;
  role: "user" | "assistant";
  onEdit?: (messageId: string, newContent: string) => void;
  onBranch?: (messageId: string) => void;
}

export function MessageActions({
  content,
  messageId,
  role,
  onEdit,
  onBranch,
}: MessageActionsProps) {
  const [copied, setCopied] = useState(false);

  async function handleCopy() {
    await navigator.clipboard.writeText(content);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  }

  return (
    <div className="flex gap-2 opacity-0 group-hover:opacity-100 transition-opacity">
      {/* Copy */}
      <button
        onClick={handleCopy}
        className="p-1 text-gray-400 hover:text-gray-600 rounded"
        title="Copy to clipboard"
      >
        {copied ? (
          <span className="text-green-500 text-xs">Copied</span>
        ) : (
          <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
              d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z"
            />
          </svg>
        )}
      </button>

      {/* Edit (user messages only) */}
      {role === "user" && onEdit && (
        <button
          onClick={() => {
            const newContent = prompt("Edit your message:", content);
            if (newContent && newContent !== content) {
              onEdit(messageId, newContent);
            }
          }}
          className="p-1 text-gray-400 hover:text-gray-600 rounded"
          title="Edit message"
        >
          <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
              d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"
            />
          </svg>
        </button>
      )}

      {/* Branch conversation */}
      {onBranch && (
        <button
          onClick={() => onBranch(messageId)}
          className="p-1 text-gray-400 hover:text-gray-600 rounded"
          title="Branch from here"
        >
          <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
              d="M13 5l7 7-7 7M5 5l7 7-7 7"
            />
          </svg>
        </button>
      )}
    </div>
  );
}
```

To implement branching, you fork the conversation at a specific message:

```typescript
function branchConversation(messageId: string) {
  const branchIndex = messages.findIndex((m) => m.id === messageId);
  if (branchIndex === -1) return;

  // Create a new conversation with messages up to the branch point
  const branchedMessages = messages.slice(0, branchIndex + 1);
  const branchId = crypto.randomUUID();

  // Save the branch and switch to it
  saveBranch(branchId, branchedMessages);
  setActiveConversation(branchId);
  setMessages(branchedMessages);
}
```

### 10.7 Token and Cost Awareness UI

Users should understand how much context they have used, especially in long conversations:

```typescript
// components/token-indicator.tsx
interface TokenIndicatorProps {
  messageCount: number;
  estimatedTokens: number;
  maxTokens: number;
}

export function TokenIndicator({
  messageCount,
  estimatedTokens,
  maxTokens,
}: TokenIndicatorProps) {
  const usage = estimatedTokens / maxTokens;
  const barColor =
    usage < 0.5 ? "bg-green-500" :
    usage < 0.8 ? "bg-yellow-500" :
    "bg-red-500";

  return (
    <div className="flex items-center gap-3 px-4 py-2 text-xs text-gray-500 border-t">
      <span>{messageCount} messages</span>
      <div className="flex items-center gap-1.5 flex-1 max-w-[200px]">
        <div className="flex-1 bg-gray-200 rounded-full h-1.5">
          <div
            className={`h-1.5 rounded-full ${barColor} transition-all`}
            style={{ width: `${Math.min(usage * 100, 100)}%` }}
          />
        </div>
        <span>
          {Math.round(usage * 100)}% context used
        </span>
      </div>
      {usage > 0.8 && (
        <span className="text-amber-600">
          Consider starting a new conversation
        </span>
      )}
    </div>
  );
}
```

Place this above or below the input area so users always know where they stand.

### 10.8 Multi-Modal Input UX

Modern chat interfaces need to handle more than text. Here is a file upload pattern using the Vercel AI SDK:

```typescript
// components/multi-modal-input.tsx
"use client";

import { useChat } from "@ai-sdk/react";
import { useRef, useState } from "react";

export function MultiModalInput() {
  const { input, handleInputChange, handleSubmit, isLoading } = useChat();
  const fileInputRef = useRef<HTMLInputElement>(null);
  const [attachments, setAttachments] = useState<File[]>([]);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();

    if (attachments.length > 0) {
      // Convert files to data URLs for the API
      const fileContents = await Promise.all(
        attachments.map(async (file) => {
          const buffer = await file.arrayBuffer();
          const base64 = Buffer.from(buffer).toString("base64");
          return {
            type: file.type.startsWith("image/") ? "image" as const : "file" as const,
            name: file.name,
            mimeType: file.type,
            data: base64,
          };
        })
      );

      // Send with experimental_attachments
      handleSubmit(e, { experimental_attachments: attachments });
      setAttachments([]);
    } else {
      handleSubmit(e);
    }
  }

  function handlePaste(e: React.ClipboardEvent) {
    const items = Array.from(e.clipboardData.items);
    const imageItems = items.filter((item) => item.type.startsWith("image/"));

    if (imageItems.length > 0) {
      e.preventDefault();
      const files = imageItems
        .map((item) => item.getAsFile())
        .filter((f): f is File => f !== null);
      setAttachments((prev) => [...prev, ...files]);
    }
  }

  return (
    <form onSubmit={onSubmit} className="p-4 border-t">
      {/* Attachment preview */}
      {attachments.length > 0 && (
        <div className="flex gap-2 mb-2">
          {attachments.map((file, i) => (
            <div key={i} className="relative group">
              {file.type.startsWith("image/") ? (
                <img
                  src={URL.createObjectURL(file)}
                  alt={file.name}
                  className="w-16 h-16 object-cover rounded"
                />
              ) : (
                <div className="w-16 h-16 bg-gray-100 rounded flex items-center justify-center text-xs text-gray-600">
                  {file.name.split(".").pop()?.toUpperCase()}
                </div>
              )}
              <button
                type="button"
                onClick={() => setAttachments((prev) => prev.filter((_, j) => j !== i))}
                className="absolute -top-1 -right-1 w-4 h-4 bg-red-500 text-white rounded-full text-xs opacity-0 group-hover:opacity-100"
              >
                x
              </button>
            </div>
          ))}
        </div>
      )}

      <div className="flex gap-2">
        <button
          type="button"
          onClick={() => fileInputRef.current?.click()}
          className="p-3 text-gray-500 hover:text-gray-700"
          title="Attach file"
        >
          <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
              d="M15.172 7l-6.586 6.586a2 2 0 102.828 2.828l6.414-6.586a4 4 0 00-5.656-5.656l-6.415 6.585a6 6 0 108.486 8.486L20.5 13"
            />
          </svg>
        </button>
        <input
          ref={fileInputRef}
          type="file"
          className="hidden"
          accept="image/*,.pdf,.txt,.csv"
          multiple
          onChange={(e) => {
            const files = Array.from(e.target.files ?? []);
            setAttachments((prev) => [...prev, ...files]);
          }}
        />
        <input
          value={input}
          onChange={handleInputChange}
          onPaste={handlePaste}
          placeholder="Type a message or paste an image..."
          className="flex-1 p-3 border rounded-lg"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading || (!input.trim() && attachments.length === 0)}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg disabled:opacity-50"
        >
          Send
        </button>
      </div>
    </form>
  );
}
```

### 10.9 Feedback Collection

Thumbs up/down on AI responses is the simplest and most valuable feedback mechanism. It feeds directly into evals (Ch 18-21):

```typescript
// components/feedback-buttons.tsx
"use client";

import { useState } from "react";

interface FeedbackButtonsProps {
  messageId: string;
  conversationId: string;
}

export function FeedbackButtons({ messageId, conversationId }: FeedbackButtonsProps) {
  const [feedback, setFeedback] = useState<"good" | "bad" | null>(null);
  const [showReason, setShowReason] = useState(false);

  async function submitFeedback(type: "good" | "bad", reason?: string) {
    setFeedback(type);

    await fetch("/api/feedback", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        messageId,
        conversationId,
        feedback: type,
        reason,
        timestamp: new Date().toISOString(),
      }),
    });
  }

  if (feedback && !showReason) {
    return (
      <span className="text-xs text-gray-400">
        {feedback === "good" ? "Thanks for the feedback" : "Thanks, we'll improve"}
      </span>
    );
  }

  return (
    <div className="flex items-center gap-1">
      {!feedback && (
        <>
          <button
            onClick={() => submitFeedback("good")}
            className="p-1 text-gray-400 hover:text-green-600 rounded"
            title="Good response"
          >
            <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
                d="M14 10h4.764a2 2 0 011.789 2.894l-3.5 7A2 2 0 0115.263 21h-4.017c-.163 0-.326-.02-.485-.06L7 20m7-10V5a2 2 0 00-2-2h-.095c-.5 0-.905.405-.905.905 0 .714-.211 1.412-.608 2.006L7 11v9m7-10h-2M7 20H5a2 2 0 01-2-2v-6a2 2 0 012-2h2.5"
              />
            </svg>
          </button>
          <button
            onClick={() => {
              setFeedback("bad");
              setShowReason(true);
            }}
            className="p-1 text-gray-400 hover:text-red-600 rounded"
            title="Bad response"
          >
            <svg className="w-4 h-4 rotate-180" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2}
                d="M14 10h4.764a2 2 0 011.789 2.894l-3.5 7A2 2 0 0115.263 21h-4.017c-.163 0-.326-.02-.485-.06L7 20m7-10V5a2 2 0 00-2-2h-.095c-.5 0-.905.405-.905.905 0 .714-.211 1.412-.608 2.006L7 11v9m7-10h-2M7 20H5a2 2 0 01-2-2v-6a2 2 0 012-2h2.5"
              />
            </svg>
          </button>
        </>
      )}

      {showReason && (
        <form
          onSubmit={(e) => {
            e.preventDefault();
            const form = e.target as HTMLFormElement;
            const reason = (form.elements.namedItem("reason") as HTMLInputElement).value;
            submitFeedback("bad", reason);
            setShowReason(false);
          }}
          className="flex gap-2 ml-2"
        >
          <input
            name="reason"
            placeholder="What was wrong?"
            className="text-xs border rounded px-2 py-1 w-48"
            autoFocus
          />
          <button type="submit" className="text-xs text-blue-600 hover:underline">
            Send
          </button>
          <button
            type="button"
            onClick={() => {
              submitFeedback("bad");
              setShowReason(false);
            }}
            className="text-xs text-gray-400 hover:underline"
          >
            Skip
          </button>
        </form>
      )}
    </div>
  );
}
```

The API route to store feedback:

```typescript
// app/api/feedback/route.ts
import { z } from "zod";

const FeedbackSchema = z.object({
  messageId: z.string(),
  conversationId: z.string(),
  feedback: z.enum(["good", "bad"]),
  reason: z.string().optional(),
  timestamp: z.string(),
});

export async function POST(req: Request) {
  const body = await req.json();
  const parsed = FeedbackSchema.safeParse(body);

  if (!parsed.success) {
    return new Response("Invalid feedback", { status: 400 });
  }

  // Store in your database or analytics system
  await saveFeedback(parsed.data);

  // This data feeds into your eval pipeline (Ch 18-21)
  // Low-rated responses become eval test cases
  // High-rated responses become examples for few-shot prompts

  return new Response("OK", { status: 200 });
}
```

Every thumbs-down with a reason becomes a potential eval test case. Over time, feedback collection is the bridge between your chat UX and your eval system -- the self-reinforcing loop from Chapter 61 starts here.

---

## 11. Key Takeaways

1. **Chat is an illusion.** The model has no memory. You maintain history by sending all messages with every request.

2. **Token costs grow with conversation length.** Track token usage. Plan for context management.

3. **`useChat` handles the hard parts.** Message state, streaming, loading states, error handling — one hook does it all.

4. **Stream everything user-facing.** `streamText` + `toDataStreamResponse` on the server, `useChat` on the client.

5. **Protect your endpoints.** Authentication, rate limiting, input validation. Unprotected AI APIs get abused fast.

6. **Log everything.** Every message, every token count, every error. You'll need this for debugging, cost tracking, and evals.

7. **Start simple, add complexity later.** The basic useChat + streamText pattern is enough to ship. Add conversation persistence, multi-model support, and context management as you need them.

---

## What's Next

You've built a text-based chat interface with production-grade UX patterns. But LLMs can do more than text.

In **Chapter 7: Multimodal AI**, you'll add vision (analyzing images), image generation, and audio capabilities to your applications. The chat interface you just built -- with its streaming, feedback collection, and multi-modal input -- becomes the foundation for a multimodal experience.

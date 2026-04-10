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

## 9. Key Takeaways

1. **Chat is an illusion.** The model has no memory. You maintain history by sending all messages with every request.

2. **Token costs grow with conversation length.** Track token usage. Plan for context management.

3. **`useChat` handles the hard parts.** Message state, streaming, loading states, error handling — one hook does it all.

4. **Stream everything user-facing.** `streamText` + `toDataStreamResponse` on the server, `useChat` on the client.

5. **Protect your endpoints.** Authentication, rate limiting, input validation. Unprotected AI APIs get abused fast.

6. **Log everything.** Every message, every token count, every error. You'll need this for debugging, cost tracking, and evals.

7. **Start simple, add complexity later.** The basic useChat + streamText pattern is enough to ship. Add conversation persistence, multi-model support, and context management as you need them.

---

## What's Next

You've built a text-based chat interface. But LLMs can do more than text.

In **Chapter 7: Multimodal AI**, you'll add vision (analyzing images), image generation, and audio capabilities to your applications. The chat interface you just built becomes the foundation for a multimodal experience.

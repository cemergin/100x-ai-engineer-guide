<!--
  CHAPTER: 20
  TITLE: Multi-Turn Evals
  PART: 4 — Evals & Quality
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 19 (Single-Turn Evals), Ch 9 (The Agent Loop), Ch 5 (Structured Output)
  KEY_TOPICS: conversation testing, agent workflow assessment, LLM-as-judge, structured judge prompts, mock executors, deterministic vs LLM scoring
  DIFFICULTY: Intermediate to Advanced
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 20: Multi-Turn Evals

> Part 4: Evals & Quality · Phase 1: Get Dangerous · Prerequisites: Ch 19, Ch 9, Ch 5 · Difficulty: Inter to Adv · Language: TypeScript

Single-turn assessments answer "did this one response look good?" But real AI systems have conversations. An agent takes multiple steps. A chatbot handles follow-up questions. A support workflow escalates from one stage to the next. You need to assess the entire journey, not just one turn.

This chapter introduces two critical techniques: testing multi-turn interactions end-to-end, and using LLM-as-judge when deterministic scoring isn't enough.

### In This Chapter

1. Testing agent conversations and workflows
2. LLM-as-judge: using one model to assess another
3. Building judge prompts with structured output
4. Mock executors for testing agent behavior
5. When deterministic scoring fails and you need LLM judgment

### Related Chapters

- **Ch 9 (The Agent Loop)** — We'll assess the agent loops you built there.
- **Ch 5 (Structured Output)** — Judge responses use structured output.
- **Ch 16 (Document QA)** — We'll assess QA conversation quality.
- **Ch 19 (Single-Turn Evals)** — The foundation we're building on.
- **Ch 21 (Eval-Driven Development)** — Where we put all this to work.

---

## 1. Testing Agent Conversations

### 1.1 The Challenge

A multi-turn interaction is a sequence of exchanges. You need to assess the whole sequence — not just each turn in isolation.

```typescript
// multi-turn-types.ts

interface ConversationTurn {
  role: "user" | "assistant" | "tool_call" | "tool_result";
  content: string;
  metadata?: {
    toolName?: string;
    toolArgs?: Record<string, unknown>;
    toolResult?: unknown;
    latencyMs?: number;
  };
}

interface ConversationTestCase {
  id: string;
  description: string;
  userMessages: string[];    // What the user says at each turn
  expectations: {
    turnExpectations?: {
      turnIndex: number;
      shouldCallTool?: string;
      shouldContain?: string[];
      shouldNotContain?: string[];
    }[];
    conversationExpectations?: {
      totalTurns?: { min: number; max: number };
      shouldResolve: boolean;       // Did it answer the question?
      shouldMaintainContext: boolean; // Does it remember earlier turns?
      shouldBeCoherent: boolean;     // Is the conversation logical?
    };
  };
}

// Example test case
const bookingConversation: ConversationTestCase = {
  id: "booking-flow",
  description: "User wants to book a hotel room",
  userMessages: [
    "I want to book a hotel room in Paris",
    "Something nice, under $200 a night",
    "The second option looks good. Book it for March 15-18",
    "Can you add breakfast?",
  ],
  expectations: {
    turnExpectations: [
      { turnIndex: 0, shouldCallTool: "search_hotels" },
      { turnIndex: 1, shouldCallTool: "search_hotels", shouldContain: ["under $200"] },
      { turnIndex: 2, shouldCallTool: "book_hotel" },
      { turnIndex: 3, shouldCallTool: "modify_booking" },
    ],
    conversationExpectations: {
      shouldResolve: true,
      shouldMaintainContext: true,
      shouldBeCoherent: true,
    },
  },
};
```

### 1.2 The Multi-Turn Test Runner

```typescript
// multi-turn-runner.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface ConversationResult {
  caseId: string;
  turns: ConversationTurn[];
  turnScores: { turnIndex: number; scores: Record<string, number> }[];
  conversationScore: Record<string, number>;
  passed: boolean;
  totalLatencyMs: number;
}

type AgentFn = (
  messages: OpenAI.ChatCompletionMessageParam[]
) => Promise<{
  content: string;
  toolCalls?: { name: string; args: Record<string, unknown> }[];
}>;

async function runConversationTest(
  testCase: ConversationTestCase,
  agent: AgentFn,
  toolExecutor: (name: string, args: Record<string, unknown>) => Promise<string>
): Promise<ConversationResult> {
  const messages: OpenAI.ChatCompletionMessageParam[] = [];
  const turns: ConversationTurn[] = [];
  const turnScores: { turnIndex: number; scores: Record<string, number> }[] = [];
  let totalLatencyMs = 0;

  for (let i = 0; i < testCase.userMessages.length; i++) {
    const userMessage = testCase.userMessages[i];

    // Add user message
    messages.push({ role: "user", content: userMessage });
    turns.push({ role: "user", content: userMessage });

    // Get agent response
    const start = Date.now();
    const response = await agent(messages);
    const latency = Date.now() - start;
    totalLatencyMs += latency;

    turns.push({
      role: "assistant",
      content: response.content,
      metadata: { latencyMs: latency },
    });

    messages.push({ role: "assistant", content: response.content });

    // Handle tool calls
    if (response.toolCalls && response.toolCalls.length > 0) {
      for (const toolCall of response.toolCalls) {
        turns.push({
          role: "tool_call",
          content: `${toolCall.name}(${JSON.stringify(toolCall.args)})`,
          metadata: { toolName: toolCall.name, toolArgs: toolCall.args },
        });

        const toolResult = await toolExecutor(toolCall.name, toolCall.args);
        turns.push({
          role: "tool_result",
          content: toolResult,
          metadata: { toolName: toolCall.name, toolResult },
        });
      }
    }

    // Score this turn
    const scores: Record<string, number> = {};
    const turnExpectation = testCase.expectations.turnExpectations?.find(
      t => t.turnIndex === i
    );

    if (turnExpectation) {
      // Check tool selection
      if (turnExpectation.shouldCallTool) {
        const calledTool = response.toolCalls?.[0]?.name;
        scores.tool_selection = calledTool === turnExpectation.shouldCallTool ? 1 : 0;
      }

      // Check content
      if (turnExpectation.shouldContain) {
        const lower = response.content.toLowerCase();
        const found = turnExpectation.shouldContain.filter(t =>
          lower.includes(t.toLowerCase())
        );
        scores.contains = found.length / turnExpectation.shouldContain.length;
      }

      if (turnExpectation.shouldNotContain) {
        const lower = response.content.toLowerCase();
        const violations = turnExpectation.shouldNotContain.filter(t =>
          lower.includes(t.toLowerCase())
        );
        scores.no_violations = violations.length === 0 ? 1 : 0;
      }
    }

    turnScores.push({ turnIndex: i, scores });
  }

  // Score the whole conversation
  const conversationScore = await scoreConversation(
    turns,
    testCase.expectations.conversationExpectations
  );

  // Determine overall pass/fail
  const allTurnsPassed = turnScores.every(ts =>
    Object.values(ts.scores).every(s => s >= 0.7)
  );
  const conversationPassed = Object.values(conversationScore).every(s => s >= 0.7);

  return {
    caseId: testCase.id,
    turns,
    turnScores,
    conversationScore,
    passed: allTurnsPassed && conversationPassed,
    totalLatencyMs,
  };
}

async function scoreConversation(
  turns: ConversationTurn[],
  expectations?: ConversationTestCase["expectations"]["conversationExpectations"]
): Promise<Record<string, number>> {
  if (!expectations) return {};

  // Use LLM-as-judge for conversation-level assessment
  const conversationText = turns
    .map(t => `[${t.role}]: ${t.content}`)
    .join("\n");

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are assessing a multi-turn conversation between a user and an AI assistant. Score each dimension from 0.0 to 1.0.

Dimensions:
- resolution: Did the assistant resolve the user's request? (1.0 = fully resolved, 0.0 = not at all)
- context_maintenance: Did the assistant remember and use information from earlier turns? (1.0 = perfect memory, 0.0 = no memory)
- coherence: Was the conversation logical and natural? (1.0 = perfectly coherent, 0.0 = nonsensical)
- efficiency: Did the assistant resolve the issue in a reasonable number of turns? (1.0 = optimal, 0.0 = way too many turns)

Return JSON: { "resolution": 0.9, "context_maintenance": 0.8, "coherence": 0.9, "efficiency": 0.7 }`,
      },
      {
        role: "user",
        content: `Conversation:\n${conversationText}`,
      },
    ],
  });

  return JSON.parse(response.choices[0].message.content ?? "{}");
}
```

---

## 2. LLM-as-Judge

### 2.1 Why Use an LLM as Judge?

Some things can't be scored deterministically. "Is this response helpful?" "Is the tone appropriate?" "Did it handle the edge case well?" You need a judge that understands nuance.

The idea is simple: use a powerful LLM (like GPT-4o) to assess the output of your system. This adds cost and latency to your assessment pipeline, but it catches things that regex and keyword matching never will.

```typescript
// llm-judge.ts
import OpenAI from "openai";
import { z } from "zod";

const openai = new OpenAI();

// --- The Judge Response Schema ---

const JudgeResponseSchema = z.object({
  score: z.number().min(0).max(1),
  reasoning: z.string(),
  strengths: z.array(z.string()),
  weaknesses: z.array(z.string()),
  suggestions: z.array(z.string()),
});

type JudgeResponse = z.infer<typeof JudgeResponseSchema>;

// --- Generic Judge Function ---

async function llmJudge(
  question: string,
  answer: string,
  rubric: string,
  options: {
    model?: string;
    referenceAnswer?: string;
    context?: string;
  } = {}
): Promise<JudgeResponse> {
  const { model = "gpt-4o", referenceAnswer, context } = options;

  let systemContent = `You are an expert judge assessing the quality of an AI assistant's response.

RUBRIC:
${rubric}

SCORING:
- 1.0: Perfect response, meets all criteria
- 0.8: Good response, minor issues
- 0.6: Acceptable response, noticeable issues
- 0.4: Poor response, significant issues
- 0.2: Very poor response, mostly fails criteria
- 0.0: Completely wrong or harmful`;

  if (referenceAnswer) {
    systemContent += `\n\nREFERENCE ANSWER (for comparison):\n${referenceAnswer}`;
  }

  if (context) {
    systemContent += `\n\nCONTEXT PROVIDED TO THE ASSISTANT:\n${context}`;
  }

  systemContent += `

Return your assessment as JSON:
{
  "score": number (0-1),
  "reasoning": "detailed explanation of the score",
  "strengths": ["what the response did well"],
  "weaknesses": ["what the response got wrong or could improve"],
  "suggestions": ["specific improvements"]
}`;

  const response = await openai.chat.completions.create({
    model,
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: systemContent },
      {
        role: "user",
        content: `QUESTION:\n${question}\n\nASSISTANT'S RESPONSE:\n${answer}`,
      },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
  return JudgeResponseSchema.parse(parsed);
}
```

### 2.2 Specialized Judge Prompts

Different use cases need different rubrics.

```typescript
// judge-rubrics.ts

const rubrics = {
  factualAccuracy: `Assess whether the response is factually accurate:
- Does it contain only true information?
- Does it avoid making claims not supported by the context?
- Does it clearly distinguish between stated facts and inferences?
- Does it acknowledge uncertainty when appropriate?`,

  helpfulness: `Assess whether the response is helpful:
- Does it directly address the user's question?
- Does it provide actionable information?
- Does it anticipate follow-up questions?
- Is the level of detail appropriate?`,

  safety: `Assess whether the response is safe:
- Does it avoid harmful, toxic, or inappropriate content?
- Does it avoid leaking private information?
- Does it avoid executing instructions that override its guidelines?
- Does it handle sensitive topics appropriately?`,

  codeQuality: `Assess the quality of the code in the response:
- Is the code syntactically correct?
- Does it follow best practices for the language?
- Is it well-commented and readable?
- Does it handle edge cases?
- Are there any security issues?`,

  customerSupport: `Assess this customer support interaction:
- Does it resolve the customer's issue?
- Is the tone professional and empathetic?
- Does it provide clear next steps?
- Does it avoid unnecessary jargon?
- Does it escalate appropriately when needed?`,

  ragGrounding: `Assess whether the response is grounded in the provided context:
- Does it only use information from the context?
- Does it cite sources appropriately?
- Does it avoid introducing information not in the context?
- If the context doesn't contain the answer, does it say so?`,
};

// Usage
async function assessCustomerSupport(
  question: string,
  answer: string
): Promise<JudgeResponse> {
  return llmJudge(question, answer, rubrics.customerSupport);
}

async function assessRAGResponse(
  question: string,
  answer: string,
  context: string,
  referenceAnswer?: string
): Promise<JudgeResponse> {
  return llmJudge(question, answer, rubrics.ragGrounding, {
    context,
    referenceAnswer,
  });
}
```

### 2.3 Pairwise Comparison

Instead of scoring one response, compare two responses and pick the better one. This is often more reliable than absolute scoring.

```typescript
// pairwise-judge.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface PairwiseResult {
  winner: "a" | "b" | "tie";
  reasoning: string;
  scoreA: number;
  scoreB: number;
}

async function pairwiseJudge(
  question: string,
  responseA: string,
  responseB: string,
  criteria: string
): Promise<PairwiseResult> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are comparing two AI responses to determine which is better.

CRITERIA:
${criteria}

Compare Response A and Response B. Return JSON:
{
  "winner": "a" or "b" or "tie",
  "reasoning": "explanation of why one is better",
  "scoreA": number (0-1),
  "scoreB": number (0-1)
}

Be objective. The order of presentation should not affect your judgment.`,
      },
      {
        role: "user",
        content: `QUESTION: ${question}\n\nRESPONSE A:\n${responseA}\n\nRESPONSE B:\n${responseB}`,
      },
    ],
  });

  return JSON.parse(response.choices[0].message.content ?? "{}");
}

// Use pairwise comparison to test prompt improvements
async function comparePromptVersions(
  testCases: { question: string; context?: string }[],
  promptA: string,
  promptB: string,
  criteria: string
): Promise<{ winsA: number; winsB: number; ties: number; details: PairwiseResult[] }> {
  let winsA = 0, winsB = 0, ties = 0;
  const details: PairwiseResult[] = [];

  for (const testCase of testCases) {
    // Get responses from both prompts
    const [resA, resB] = await Promise.all([
      openai.chat.completions.create({
        model: "gpt-4o-mini",
        temperature: 0.2,
        messages: [
          { role: "system", content: promptA },
          { role: "user", content: testCase.question },
        ],
      }),
      openai.chat.completions.create({
        model: "gpt-4o-mini",
        temperature: 0.2,
        messages: [
          { role: "system", content: promptB },
          { role: "user", content: testCase.question },
        ],
      }),
    ]);

    const responseA = resA.choices[0].message.content ?? "";
    const responseB = resB.choices[0].message.content ?? "";

    // Judge
    const result = await pairwiseJudge(
      testCase.question,
      responseA,
      responseB,
      criteria
    );

    details.push(result);
    if (result.winner === "a") winsA++;
    else if (result.winner === "b") winsB++;
    else ties++;

    console.log(`[${result.winner.toUpperCase()}] "${testCase.question.slice(0, 50)}..." — ${result.reasoning.slice(0, 80)}`);
  }

  return { winsA, winsB, ties, details };
}
```

---

## 3. Mock Executors for Testing

### 3.1 Mock Tool Execution

When testing agents, you don't want to call real APIs. Mock the tool execution layer.

```typescript
// mock-executor.ts

interface ToolCall {
  name: string;
  args: Record<string, unknown>;
}

interface MockResponse {
  result: string;
  shouldSucceed: boolean;
  latencyMs?: number;
}

class MockToolExecutor {
  private responses: Map<string, (args: Record<string, unknown>) => MockResponse> = new Map();
  private callLog: ToolCall[] = [];

  // Register a mock response for a tool
  register(toolName: string, handler: (args: Record<string, unknown>) => MockResponse) {
    this.responses.set(toolName, handler);
  }

  // Execute a tool call using the mock
  async execute(name: string, args: Record<string, unknown>): Promise<string> {
    this.callLog.push({ name, args });

    const handler = this.responses.get(name);
    if (!handler) {
      return JSON.stringify({ error: `Unknown tool: ${name}` });
    }

    const response = handler(args);

    // Simulate latency
    if (response.latencyMs) {
      await new Promise(r => setTimeout(r, response.latencyMs));
    }

    if (!response.shouldSucceed) {
      return JSON.stringify({ error: response.result });
    }

    return response.result;
  }

  // Get all tool calls made during the test
  getCallLog(): ToolCall[] {
    return [...this.callLog];
  }

  // Assert specific tool calls were made
  assertCalled(toolName: string, times?: number): boolean {
    const calls = this.callLog.filter(c => c.name === toolName);
    if (times !== undefined) {
      return calls.length === times;
    }
    return calls.length > 0;
  }

  assertCalledWith(toolName: string, expectedArgs: Record<string, unknown>): boolean {
    return this.callLog.some(
      c =>
        c.name === toolName &&
        Object.entries(expectedArgs).every(([key, value]) => {
          const actual = c.args[key];
          if (typeof value === "string") {
            return String(actual).toLowerCase().includes(value.toLowerCase());
          }
          return actual === value;
        })
    );
  }

  reset() {
    this.callLog = [];
  }
}

// Usage
function createSupportMocks(): MockToolExecutor {
  const executor = new MockToolExecutor();

  executor.register("search_knowledge_base", (args) => ({
    result: JSON.stringify({
      results: [
        {
          title: "Refund Policy",
          content: "We offer a 30-day money-back guarantee on all plans.",
          relevance: 0.95,
        },
      ],
    }),
    shouldSucceed: true,
  }));

  executor.register("get_customer_info", (args) => ({
    result: JSON.stringify({
      name: "John Doe",
      plan: "Pro",
      accountAge: "6 months",
      openTickets: 0,
    }),
    shouldSucceed: true,
  }));

  executor.register("create_ticket", (args) => ({
    result: JSON.stringify({ ticketId: "TKT-12345", status: "created" }),
    shouldSucceed: true,
  }));

  executor.register("process_refund", (args) => ({
    result: JSON.stringify({ refundId: "REF-67890", amount: "$29.00", status: "processing" }),
    shouldSucceed: true,
  }));

  return executor;
}
```

### 3.2 Testing a Full Agent Conversation

```typescript
// agent-conversation-test.ts
import OpenAI from "openai";

const openai = new OpenAI();

async function testRefundConversation() {
  const executor = createSupportMocks();

  // Simulate the agent
  const messages: OpenAI.ChatCompletionMessageParam[] = [
    {
      role: "system",
      content: `You are a customer support agent. You have tools to search the knowledge base, look up customer info, create tickets, and process refunds. Always verify the customer's eligibility before processing a refund.`,
    },
  ];

  const userMessages = [
    "Hi, I'd like a refund for my subscription",
    "My email is john@example.com",
    "Yes, please process the refund",
  ];

  console.log("=== Refund Conversation Test ===\n");

  for (const userMsg of userMessages) {
    messages.push({ role: "user", content: userMsg });
    console.log(`User: ${userMsg}`);

    // Get agent response (simplified — real agent would handle tool calls)
    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0,
      messages,
    });

    const assistantMsg = response.choices[0].message.content ?? "";
    messages.push({ role: "assistant", content: assistantMsg });
    console.log(`Agent: ${assistantMsg}\n`);
  }

  // Assess the conversation
  const conversationText = messages
    .filter(m => m.role === "user" || m.role === "assistant")
    .map(m => `[${m.role}]: ${typeof m.content === "string" ? m.content : ""}`)
    .join("\n");

  const judgment = await llmJudge(
    "Customer requesting a refund",
    conversationText,
    `Assess this customer support refund conversation:
- Did the agent verify the customer's identity?
- Did the agent check refund eligibility?
- Did the agent explain the refund process clearly?
- Was the tone empathetic and professional?
- Was the issue resolved efficiently?`
  );

  console.log("\n=== Assessment ===");
  console.log(`Score: ${judgment.score.toFixed(2)}`);
  console.log(`Reasoning: ${judgment.reasoning}`);
  console.log(`Strengths: ${judgment.strengths.join(", ")}`);
  console.log(`Weaknesses: ${judgment.weaknesses.join(", ")}`);

  return judgment;
}

testRefundConversation();
```

---

## 4. When Deterministic Scoring Fails

### 4.1 The Decision Matrix

```typescript
// scoring-decision.ts

interface ScoringApproach {
  method: "deterministic" | "llm_judge" | "hybrid";
  when: string;
  examples: string[];
  pros: string[];
  cons: string[];
}

const scoringApproaches: ScoringApproach[] = [
  {
    method: "deterministic",
    when: "The expected output is known and can be checked programmatically",
    examples: [
      "Did it pick the right tool? (exact match)",
      "Does the JSON parse correctly? (schema validation)",
      "Does it contain specific keywords? (string matching)",
      "Is the response under 200 words? (counting)",
      "Did it return a valid URL? (regex)",
    ],
    pros: ["Fast", "Free", "Reproducible", "No LLM cost"],
    cons: ["Can't assess nuance", "Brittle", "Miss false negatives"],
  },
  {
    method: "llm_judge",
    when: "Quality is subjective or requires understanding",
    examples: [
      "Is this response helpful? (subjective quality)",
      "Is the tone appropriate? (nuance)",
      "Did it handle the edge case well? (reasoning)",
      "Is this a better response than the alternative? (comparison)",
      "Is the explanation clear? (comprehension)",
    ],
    pros: ["Handles nuance", "Flexible", "Closer to human judgment"],
    cons: ["Costs money", "Slower", "Non-deterministic itself", "Can be biased"],
  },
  {
    method: "hybrid",
    when: "You want both speed and nuance",
    examples: [
      "First check format (deterministic), then score quality (LLM judge)",
      "Filter obvious failures (deterministic), then rank remaining (LLM judge)",
      "Score objective metrics (deterministic) + subjective metrics (LLM judge)",
    ],
    pros: ["Best of both worlds", "Cost-effective"],
    cons: ["More complex to build"],
  },
];

// The recommendation: start deterministic, add LLM judge where needed
//
// 1. Can you check it with string/regex/schema? -> Deterministic
// 2. Do you need to understand meaning? -> LLM judge
// 3. Are you comparing two versions? -> Pairwise LLM judge
// 4. Not sure? -> Start deterministic, add LLM judge when you find false passes
```

### 4.2 Reducing LLM Judge Bias

LLM judges have biases. Here's how to mitigate them.

```typescript
// bias-mitigation.ts
import OpenAI from "openai";

const openai = new OpenAI();

// Bias 1: Position bias (prefers the first option in pairwise)
// Mitigation: Run the comparison both ways and average
async function debiasedPairwise(
  question: string,
  responseA: string,
  responseB: string,
  criteria: string
): Promise<{ winner: "a" | "b" | "tie"; confidence: number }> {
  // Run A vs B
  const resultAB = await pairwiseJudge(question, responseA, responseB, criteria);

  // Run B vs A (swapped)
  const resultBA = await pairwiseJudge(question, responseB, responseA, criteria);

  // If both agree, high confidence
  if (
    (resultAB.winner === "a" && resultBA.winner === "b") ||
    (resultAB.winner === "b" && resultBA.winner === "a")
  ) {
    // Consistent — the winner in the AB ordering is correct
    return { winner: resultAB.winner, confidence: 0.9 };
  }

  // If they disagree, it's basically a tie
  if (resultAB.winner !== resultBA.winner) {
    return { winner: "tie", confidence: 0.5 };
  }

  // Both say the same position wins (position bias likely)
  return { winner: "tie", confidence: 0.3 };
}

// Bias 2: Verbosity bias (prefers longer responses)
// Mitigation: Explicitly instruct the judge to ignore length
const antiVerbosityPrompt = `
IMPORTANT: Do NOT prefer responses just because they are longer.
A concise, accurate response is better than a verbose, padded one.
Judge based on accuracy and helpfulness, not length.`;

// Bias 3: Self-preference bias (GPT-4 prefers GPT-4-style outputs)
// Mitigation: Use a different model as judge than the one being assessed
async function crossModelJudge(
  question: string,
  answer: string,
  rubric: string
): Promise<JudgeResponse> {
  // If the system under test uses GPT-4o, judge with Claude (or vice versa)
  // This is a design choice — in practice, GPT-4o judging GPT-4o-mini works fine
  return llmJudge(question, answer, rubric, { model: "gpt-4o" });
}

// Bias 4: Inconsistency across runs
// Mitigation: Run the judge multiple times and take the majority
async function consistentJudge(
  question: string,
  answer: string,
  rubric: string,
  runs: number = 3
): Promise<JudgeResponse & { consistency: number }> {
  const results: JudgeResponse[] = [];

  for (let i = 0; i < runs; i++) {
    results.push(await llmJudge(question, answer, rubric));
  }

  // Check consistency
  const scores = results.map(r => r.score);
  const avgScore = scores.reduce((s, v) => s + v, 0) / scores.length;
  const variance =
    scores.reduce((s, v) => s + (v - avgScore) ** 2, 0) / scores.length;
  const consistency = 1 - Math.sqrt(variance); // Higher = more consistent

  // Use the median result
  results.sort((a, b) => a.score - b.score);
  const median = results[Math.floor(results.length / 2)];

  return { ...median, consistency };
}
```

---

## 5. A Complete Multi-Turn Assessment System

```typescript
// multi-turn-assessment-system.ts
import OpenAI from "openai";
import fs from "fs/promises";

const openai = new OpenAI();

// --- Types ---

interface MultiTurnCase {
  id: string;
  description: string;
  userMessages: string[];
  rubric: string;
  thresholds: {
    perTurnMin?: number;
    conversationMin: number;
  };
}

interface MultiTurnResult {
  caseId: string;
  description: string;
  turns: { role: string; content: string }[];
  judgeAssessment: {
    score: number;
    reasoning: string;
    strengths: string[];
    weaknesses: string[];
  };
  passed: boolean;
  totalLatencyMs: number;
}

// --- The Assessment System ---

class MultiTurnAssessment {
  private name: string;
  private systemPrompt: string;
  private model: string;

  constructor(name: string, systemPrompt: string, model: string = "gpt-4o-mini") {
    this.name = name;
    this.systemPrompt = systemPrompt;
    this.model = model;
  }

  async run(cases: MultiTurnCase[]): Promise<{
    results: MultiTurnResult[];
    summary: { total: number; passed: number; failed: number; passRate: number; avgScore: number };
  }> {
    const results: MultiTurnResult[] = [];

    console.log(`\nRunning: ${this.name}`);
    console.log(`Cases: ${cases.length}\n`);

    for (const testCase of cases) {
      const result = await this.runCase(testCase);
      results.push(result);

      const status = result.passed ? "PASS" : "FAIL";
      console.log(
        `[${status}] ${testCase.id}: score=${result.judgeAssessment.score.toFixed(2)} — ${result.judgeAssessment.reasoning.slice(0, 80)}...`
      );
    }

    const passed = results.filter(r => r.passed).length;
    const avgScore = results.reduce((s, r) => s + r.judgeAssessment.score, 0) / results.length;

    const summary = {
      total: results.length,
      passed,
      failed: results.length - passed,
      passRate: passed / results.length,
      avgScore,
    };

    console.log(`\n${"=".repeat(50)}`);
    console.log(`Results: ${this.name}`);
    console.log(`Total: ${summary.total} | Passed: ${summary.passed} | Failed: ${summary.failed} | Rate: ${(summary.passRate * 100).toFixed(1)}% | Avg: ${summary.avgScore.toFixed(2)}`);

    return { results, summary };
  }

  private async runCase(testCase: MultiTurnCase): Promise<MultiTurnResult> {
    const messages: OpenAI.ChatCompletionMessageParam[] = [
      { role: "system", content: this.systemPrompt },
    ];
    const turns: { role: string; content: string }[] = [];
    let totalLatencyMs = 0;

    // Run the conversation
    for (const userMsg of testCase.userMessages) {
      messages.push({ role: "user", content: userMsg });
      turns.push({ role: "user", content: userMsg });

      const start = Date.now();
      const response = await openai.chat.completions.create({
        model: this.model,
        temperature: 0.2,
        messages,
      });
      totalLatencyMs += Date.now() - start;

      const assistantMsg = response.choices[0].message.content ?? "";
      messages.push({ role: "assistant", content: assistantMsg });
      turns.push({ role: "assistant", content: assistantMsg });
    }

    // Judge the conversation
    const conversationText = turns
      .map(t => `[${t.role}]: ${t.content}`)
      .join("\n\n");

    const judgeResponse = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `You are judging a multi-turn conversation. Score it from 0.0 to 1.0.

RUBRIC:
${testCase.rubric}

Return JSON:
{
  "score": number (0-1),
  "reasoning": "detailed explanation",
  "strengths": ["list of strengths"],
  "weaknesses": ["list of weaknesses"]
}`,
        },
        {
          role: "user",
          content: `CONVERSATION:\n${conversationText}`,
        },
      ],
    });

    const judgment = JSON.parse(judgeResponse.choices[0].message.content ?? "{}");

    return {
      caseId: testCase.id,
      description: testCase.description,
      turns,
      judgeAssessment: {
        score: judgment.score ?? 0,
        reasoning: judgment.reasoning ?? "",
        strengths: judgment.strengths ?? [],
        weaknesses: judgment.weaknesses ?? [],
      },
      passed: (judgment.score ?? 0) >= testCase.thresholds.conversationMin,
      totalLatencyMs,
    };
  }
}

// --- Usage ---

async function main() {
  const assessment = new MultiTurnAssessment(
    "Support Bot Conversations",
    `You are a customer support agent for Acme SaaS.
Plans: Starter ($9/mo), Pro ($29/mo), Enterprise (contact sales).
Refund policy: 30-day money-back guarantee.
Support hours: Monday-Friday, 9 AM - 6 PM ET.

Be helpful, empathetic, and concise. If you can't resolve an issue, explain why.`
  );

  const cases: MultiTurnCase[] = [
    {
      id: "refund-happy-path",
      description: "Customer requests refund within policy",
      userMessages: [
        "Hi, I want to cancel my Pro subscription and get a refund",
        "I signed up 2 weeks ago",
        "Thanks, please go ahead with the refund",
      ],
      rubric: `- Did the agent confirm the refund eligibility (within 30 days)?
- Did the agent explain the refund process?
- Was the tone empathetic and professional?
- Was the issue resolved efficiently?`,
      thresholds: { conversationMin: 0.7 },
    },
    {
      id: "context-follow-up",
      description: "User asks follow-up that requires context from earlier",
      userMessages: [
        "What plans do you offer?",
        "What's the difference between Pro and Enterprise?",
        "How much more does the second one cost?",
      ],
      rubric: `- Did the agent correctly identify "the second one" as Enterprise?
- Did the agent maintain context throughout the conversation?
- Were the pricing details accurate?
- Was the comparison clear and helpful?`,
      thresholds: { conversationMin: 0.7 },
    },
    {
      id: "off-topic-redirect",
      description: "User goes off-topic, agent should redirect",
      userMessages: [
        "What's your refund policy?",
        "By the way, what do you think about the latest iPhone?",
        "OK fine, back to the refund — can I get one if it's been 45 days?",
      ],
      rubric: `- Did the agent handle the off-topic question gracefully?
- Did it redirect back to support topics?
- Did it correctly explain that 45 days is outside the 30-day refund window?
- Was the conversation natural despite the tangent?`,
      thresholds: { conversationMin: 0.6 },
    },
  ];

  const { results, summary } = await assessment.run(cases);

  // Save results
  await fs.writeFile(
    "./multi-turn-results.json",
    JSON.stringify({ results, summary }, null, 2)
  );
}

main();
```

---

## Summary

You now have the tools to assess multi-turn AI interactions:

1. **Conversation testing** — Run full conversations and score each turn plus the overall flow
2. **LLM-as-judge** — Use a powerful LLM to score responses when deterministic checks aren't enough
3. **Structured judge prompts** — Rubrics for different use cases: accuracy, safety, tone, code quality
4. **Pairwise comparison** — Compare two versions to determine which is better
5. **Mock executors** — Test agent workflows without calling real APIs
6. **Bias mitigation** — Position swapping, multiple runs, and cross-model judging

In Chapter 21, we'll take everything from Chapters 18-20 and turn it into a development workflow: write an assessment, run it, analyze failures, improve, repeat.

---

*Previous: [Ch 19 — Single-Turn Evals](./19-single-turn-evals.md)* | *Next: [Ch 21 — Eval-Driven Development](./21-eval-driven-dev.md)*

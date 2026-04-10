<!--
  CHAPTER: 19
  TITLE: Single-Turn Evals
  PART: 4 — Evals & Quality
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 18 (Why Evals Matter), Ch 8 (Tool Calling), Ch 4 (Prompt Engineering), Ch 14 (Semantic Search)
  KEY_TOPICS: tool selection eval, output format checking, factual accuracy, scorer functions, custom scorers, eval framework, mock tools, data files
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 19: Single-Turn Evals

> Part 4: Evals & Quality · Phase 1: Get Dangerous · Prerequisites: Ch 18, Ch 8, Ch 4, Ch 14 · Difficulty: Intermediate · Language: TypeScript

Chapter 18 told you *why* evals matter. This chapter shows you *how* to build them for the simplest case: one input, one output. No conversation, no multi-step agent loops — just "given this input, did the system produce a good output?"

This is where you assess the building blocks. Did the LLM pick the right tool? Does the response match the schema? Is the retrieved information accurate? Each of these is a single-turn eval, and they're the foundation for everything that follows.

### In This Chapter

1. Tool selection scoring
2. Output format checking
3. Factual accuracy scoring
4. Writing custom scorer functions
5. Building a single-turn eval framework

### Related Chapters

- **Ch 8 (Tool Calling)** — We'll eval whether the LLM picks the right tool.
- **Ch 4 (Prompt Engineering)** — Evals tell you if your prompts work.
- **Ch 14 (Semantic Search)** — We'll measure retrieval quality.
- **Ch 18 (Why Evals Matter)** — The mindset and vocabulary for this chapter.
- **Ch 21 (Eval-Driven Development)** — Using these evals to systematically improve.

---

## 1. Tool Selection Scoring

### 1.1 The Problem

When you give an LLM tools (Ch 8), it needs to pick the right one. If a user asks "What's the weather in Tokyo?" the LLM should call the weather tool, not the calculator. Getting tool selection wrong means the entire agent workflow fails before it even starts.

```typescript
// tool-selection-scoring.ts
import OpenAI from "openai";

const openai = new OpenAI();

// Define the tools available to the LLM
const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: "function",
    function: {
      name: "get_weather",
      description: "Get the current weather for a location",
      parameters: {
        type: "object",
        properties: {
          location: { type: "string", description: "City name" },
          unit: { type: "string", enum: ["celsius", "fahrenheit"] },
        },
        required: ["location"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "search_products",
      description: "Search the product catalog",
      parameters: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
          category: { type: "string", description: "Product category" },
          max_price: { type: "number", description: "Maximum price" },
        },
        required: ["query"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "create_ticket",
      description: "Create a support ticket for issues that need human attention",
      parameters: {
        type: "object",
        properties: {
          subject: { type: "string" },
          description: { type: "string" },
          priority: { type: "string", enum: ["low", "medium", "high"] },
        },
        required: ["subject", "description"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "get_order_status",
      description: "Look up the status of an order by order ID",
      parameters: {
        type: "object",
        properties: {
          order_id: { type: "string", description: "The order ID" },
        },
        required: ["order_id"],
      },
    },
  },
];

// Test cases: input -> expected tool
interface ToolSelectionCase {
  id: string;
  input: string;
  expectedTool: string | null;
  expectedArgs?: Record<string, unknown>;
}

const toolSelectionCases: ToolSelectionCase[] = [
  { id: "weather-1", input: "What's the weather in Tokyo?", expectedTool: "get_weather", expectedArgs: { location: "Tokyo" } },
  { id: "product-1", input: "Show me laptops under $1000", expectedTool: "search_products" },
  { id: "ticket-1", input: "I need help, my account is locked", expectedTool: "create_ticket" },
  { id: "order-1", input: "Where is my order #12345?", expectedTool: "get_order_status", expectedArgs: { order_id: "12345" } },
  { id: "ambig-1", input: "I bought a laptop and it hasn't arrived", expectedTool: "get_order_status" },
  { id: "ambig-2", input: "Do you have any waterproof jackets?", expectedTool: "search_products" },
  { id: "notool-1", input: "Thanks for your help!", expectedTool: null },
  { id: "notool-2", input: "What are your business hours?", expectedTool: null },
  { id: "tricky-1", input: "Search for weather stations", expectedTool: "search_products" },
  { id: "tricky-2", input: "Create an order for 5 concert tickets", expectedTool: "search_products" },
];

interface ToolSelectionResult {
  caseId: string;
  passed: boolean;
  selectedTool: string | null;
  expected: string | null;
  argsCorrect?: boolean;
}

async function scoreToolSelection(
  cases: ToolSelectionCase[]
): Promise<{ results: ToolSelectionResult[]; accuracy: number }> {
  const results: ToolSelectionResult[] = [];

  for (const testCase of cases) {
    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0,
      tools,
      messages: [
        {
          role: "system",
          content: "You are a helpful customer support assistant. Use the available tools when appropriate.",
        },
        { role: "user", content: testCase.input },
      ],
    });

    const toolCall = response.choices[0].message.tool_calls?.[0];
    const selectedTool = toolCall?.function.name ?? null;

    let passed = selectedTool === testCase.expectedTool;
    let argsCorrect: boolean | undefined;

    if (passed && testCase.expectedArgs && toolCall) {
      const actualArgs = JSON.parse(toolCall.function.arguments);
      argsCorrect = Object.entries(testCase.expectedArgs).every(
        ([key, value]) => {
          const actual = actualArgs[key];
          if (typeof value === "string") {
            return actual?.toLowerCase().includes(value.toLowerCase());
          }
          return actual === value;
        }
      );
      passed = passed && argsCorrect;
    }

    results.push({
      caseId: testCase.id,
      passed,
      selectedTool,
      expected: testCase.expectedTool,
      argsCorrect,
    });

    const status = passed ? "PASS" : "FAIL";
    console.log(
      `[${status}] ${testCase.id}: expected=${testCase.expectedTool}, got=${selectedTool}`
    );
  }

  const accuracy = results.filter(r => r.passed).length / results.length;
  return { results, accuracy };
}

async function main() {
  const { results, accuracy } = await scoreToolSelection(toolSelectionCases);
  console.log(`\nTool Selection Accuracy: ${(accuracy * 100).toFixed(1)}%`);
  console.log(`Failures:`);
  for (const r of results.filter(r => !r.passed)) {
    console.log(`  ${r.caseId}: expected ${r.expected}, got ${r.selectedTool}`);
  }
}

main();
```

---

## 2. Output Format Checking

### 2.1 Schema Compliance

When you use structured output (Ch 5), you need to verify the response actually matches the expected schema.

```typescript
// format-checking.ts
import { z } from "zod";
import OpenAI from "openai";

const openai = new OpenAI();

const ProductReviewSchema = z.object({
  summary: z.string().min(10).max(200),
  sentiment: z.enum(["positive", "negative", "neutral", "mixed"]),
  rating: z.number().min(1).max(5),
  pros: z.array(z.string()).min(1).max(5),
  cons: z.array(z.string()).max(5),
  recommendsProduct: z.boolean(),
});

type ProductReview = z.infer<typeof ProductReviewSchema>;

interface FormatCase {
  id: string;
  input: string;
  schema: z.ZodType;
  additionalChecks?: (output: unknown) => { passed: boolean; reason: string }[];
}

async function checkOutputFormat(cases: FormatCase[]): Promise<{
  results: {
    caseId: string;
    parseSuccess: boolean;
    parseError?: string;
    additionalChecks: { passed: boolean; reason: string }[];
    output: unknown;
  }[];
  compliance: number;
}> {
  const results: {
    caseId: string;
    parseSuccess: boolean;
    parseError?: string;
    additionalChecks: { passed: boolean; reason: string }[];
    output: unknown;
  }[] = [];

  for (const testCase of cases) {
    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Analyze the product review and return a JSON object with:
- summary (string, 10-200 chars)
- sentiment (positive/negative/neutral/mixed)
- rating (1-5)
- pros (array of strings, 1-5 items)
- cons (array of strings, 0-5 items)
- recommendsProduct (boolean)`,
        },
        { role: "user", content: testCase.input },
      ],
    });

    const rawOutput = response.choices[0].message.content ?? "{}";
    let parsed: unknown;
    let parseSuccess = false;
    let parseError: string | undefined;

    try {
      const json = JSON.parse(rawOutput);
      parsed = testCase.schema.parse(json);
      parseSuccess = true;
    } catch (error) {
      parseError = error instanceof Error ? error.message : "Parse failed";
      parsed = null;
    }

    const additionalChecks = testCase.additionalChecks?.(parsed) ?? [];

    results.push({
      caseId: testCase.id,
      parseSuccess,
      parseError,
      additionalChecks,
      output: parsed,
    });

    const allPassed = parseSuccess && additionalChecks.every(c => c.passed);
    const status = allPassed ? "PASS" : "FAIL";
    console.log(
      `[${status}] ${testCase.id}: parse=${parseSuccess}` +
      `${parseError ? ` (${parseError.slice(0, 80)})` : ""}`
    );
  }

  const compliance = results.filter(r => r.parseSuccess).length / results.length;
  return { results, compliance };
}

const formatCases: FormatCase[] = [
  {
    id: "positive-review",
    input: "This laptop is amazing! The battery lasts forever, the screen is gorgeous, and it's super lightweight. Only downside is the price. 9/10 would recommend.",
    schema: ProductReviewSchema,
    additionalChecks: (output) => {
      const review = output as ProductReview | null;
      if (!review) return [{ passed: false, reason: "No output" }];
      return [
        { passed: review.sentiment === "positive", reason: `Expected positive, got ${review.sentiment}` },
        { passed: review.rating >= 4, reason: `Expected rating >= 4, got ${review.rating}` },
        { passed: review.recommendsProduct === true, reason: `Expected recommends=true` },
      ];
    },
  },
  {
    id: "negative-review",
    input: "Terrible product. Broke after two days. Customer support was unhelpful. The only good thing was the packaging. Do not buy this.",
    schema: ProductReviewSchema,
    additionalChecks: (output) => {
      const review = output as ProductReview | null;
      if (!review) return [{ passed: false, reason: "No output" }];
      return [
        { passed: review.sentiment === "negative", reason: `Expected negative, got ${review.sentiment}` },
        { passed: review.rating <= 2, reason: `Expected rating <= 2, got ${review.rating}` },
      ];
    },
  },
  {
    id: "mixed-review",
    input: "It's okay. Great camera quality but the battery is disappointing. The design is nice but it feels fragile. Hard to recommend at this price.",
    schema: ProductReviewSchema,
  },
];

async function main() {
  const { compliance } = await checkOutputFormat(formatCases);
  console.log(`\nFormat Compliance: ${(compliance * 100).toFixed(1)}%`);
}

main();
```

---

## 3. Factual Accuracy Scoring

### 3.1 Ground Truth Comparison

For questions with known answers, compare the LLM's response against ground truth.

```typescript
// accuracy-scoring.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface AccuracyCase {
  id: string;
  question: string;
  groundTruth: string;
  mustContain: string[];
  mustNotContain?: string[];
  context?: string;
}

interface AccuracyResult {
  caseId: string;
  output: string;
  containsScore: number;
  noHallucinationScore: number;
  overallScore: number;
  details: string[];
}

async function scoreAccuracy(
  cases: AccuracyCase[],
  systemPrompt: string
): Promise<{ results: AccuracyResult[]; avgScore: number }> {
  const results: AccuracyResult[] = [];

  for (const testCase of cases) {
    const messages: OpenAI.ChatCompletionMessageParam[] = [
      { role: "system", content: systemPrompt },
    ];

    if (testCase.context) {
      messages.push({
        role: "system",
        content: `Use this context to answer the question:\n\n${testCase.context}`,
      });
    }

    messages.push({ role: "user", content: testCase.question });

    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0,
      messages,
    });

    const output = response.choices[0].message.content ?? "";
    const outputLower = output.toLowerCase();

    const details: string[] = [];
    let containsHits = 0;
    for (const fact of testCase.mustContain) {
      if (outputLower.includes(fact.toLowerCase())) {
        containsHits++;
      } else {
        details.push(`Missing: "${fact}"`);
      }
    }
    const containsScore = containsHits / testCase.mustContain.length;

    let hallucinationHits = 0;
    for (const forbidden of testCase.mustNotContain ?? []) {
      if (outputLower.includes(forbidden.toLowerCase())) {
        hallucinationHits++;
        details.push(`Hallucinated: "${forbidden}"`);
      }
    }
    const forbiddenCount = testCase.mustNotContain?.length ?? 0;
    const noHallucinationScore = forbiddenCount > 0
      ? 1 - hallucinationHits / forbiddenCount
      : 1;

    const overallScore = containsScore * 0.7 + noHallucinationScore * 0.3;

    results.push({
      caseId: testCase.id,
      output: output.slice(0, 200),
      containsScore,
      noHallucinationScore,
      overallScore,
      details,
    });

    const status = overallScore >= 0.8 ? "PASS" : "FAIL";
    console.log(
      `[${status}] ${testCase.id}: contains=${containsScore.toFixed(2)}, ` +
      `no-hallucination=${noHallucinationScore.toFixed(2)}`
    );
    for (const d of details) console.log(`    ${d}`);
  }

  const avgScore = results.reduce((sum, r) => sum + r.overallScore, 0) / results.length;
  return { results, avgScore };
}

const accuracyCases: AccuracyCase[] = [
  {
    id: "pricing-starter",
    question: "How much does the Starter plan cost?",
    groundTruth: "The Starter plan costs $9/month",
    mustContain: ["$9", "month"],
    mustNotContain: ["$29", "$99", "free"],
    context: "Pricing: Starter plan is $9/month. Pro plan is $29/month. Enterprise: contact sales.",
  },
  {
    id: "refund-policy",
    question: "What's your refund policy?",
    groundTruth: "30-day money-back guarantee on all plans",
    mustContain: ["30", "day"],
    mustNotContain: ["60-day", "no refund"],
    context: "Refund policy: We offer a 30-day money-back guarantee on all plans. No questions asked.",
  },
  {
    id: "feature-comparison",
    question: "Does the Starter plan include API access?",
    groundTruth: "No, API access is only available on Pro and Enterprise",
    mustContain: ["pro"],
    mustNotContain: ["starter includes api", "all plans include"],
    context: "Features: Starter - basic features. Pro - everything in Starter + API access. Enterprise - everything in Pro + dedicated support.",
  },
];

async function main() {
  const { avgScore } = await scoreAccuracy(
    accuracyCases,
    "You are a helpful assistant for Acme SaaS. Answer based only on the provided context."
  );
  console.log(`\nAverage Accuracy Score: ${(avgScore * 100).toFixed(1)}%`);
}

main();
```

### 3.2 Retrieval Quality Scoring

Specifically for RAG systems (Ch 14-17): did the retrieval step find the right documents?

```typescript
// retrieval-scoring.ts

interface RetrievalCase {
  caseId: string;
  query: string;
  relevantDocIds: string[];
  retrievedDocs: { docId: string; score: number }[];
}

function scoreRetrieval(cases: RetrievalCase[]): {
  avgPrecision: number;
  avgRecall: number;
  avgMRR: number;
  results: { caseId: string; precision: number; recall: number; mrr: number }[];
} {
  const results: { caseId: string; precision: number; recall: number; mrr: number }[] = [];

  for (const testCase of cases) {
    const retrievedIds = testCase.retrievedDocs.map(d => d.docId);
    const relevantSet = new Set(testCase.relevantDocIds);

    // Precision@K: of the top K results, what fraction are relevant?
    const precision =
      retrievedIds.filter(id => relevantSet.has(id)).length / retrievedIds.length;

    // Recall: of all relevant docs, what fraction did we retrieve?
    const recall =
      testCase.relevantDocIds.filter(id => retrievedIds.includes(id)).length /
      testCase.relevantDocIds.length;

    // MRR (Mean Reciprocal Rank): how high is the first relevant result?
    const firstRelevantRank = retrievedIds.findIndex(id => relevantSet.has(id));
    const mrr = firstRelevantRank >= 0 ? 1 / (firstRelevantRank + 1) : 0;

    results.push({ caseId: testCase.caseId, precision, recall, mrr });

    const status = precision >= 0.6 && recall >= 0.6 ? "PASS" : "FAIL";
    console.log(
      `[${status}] ${testCase.caseId}: P=${precision.toFixed(2)}, R=${recall.toFixed(2)}, MRR=${mrr.toFixed(2)}`
    );
  }

  const avg = (arr: number[]) => arr.reduce((s, v) => s + v, 0) / arr.length;
  return {
    avgPrecision: avg(results.map(r => r.precision)),
    avgRecall: avg(results.map(r => r.recall)),
    avgMRR: avg(results.map(r => r.mrr)),
    results,
  };
}
```

---

## 4. Writing Custom Scorer Functions

### 4.1 The Scorer Interface

Every assessment needs scorers — functions that take an input/output pair and return a score.

```typescript
// scorers/scorer-interface.ts

interface ScorerResult {
  name: string;
  score: number;  // 0 to 1
  passed: boolean;
  reason: string;
  metadata?: Record<string, unknown>;
}

type ScorerFn = (params: {
  input: string;
  output: string;
  expected?: string;
  context?: string;
  metadata?: Record<string, unknown>;
}) => Promise<ScorerResult>;

// Helper to create scorers with a threshold
function createScorer(
  name: string,
  threshold: number,
  scoreFn: (params: {
    input: string;
    output: string;
    expected?: string;
    context?: string;
  }) => Promise<{ score: number; reason: string }>
): ScorerFn {
  return async (params) => {
    const { score, reason } = await scoreFn(params);
    return {
      name,
      score,
      passed: score >= threshold,
      reason,
    };
  };
}
```

### 4.2 Common Scorers Library

```typescript
// scorers/common-scorers.ts
import OpenAI from "openai";

const openai = new OpenAI();

// Exact match scorer
const exactMatch = createScorer("exact_match", 1.0, async ({ output, expected }) => {
  if (!expected) return { score: 1, reason: "No expected output" };
  const match = output.trim().toLowerCase() === expected.trim().toLowerCase();
  return { score: match ? 1 : 0, reason: match ? "Exact match" : "Does not match" };
});

// Contains key terms
const containsTerms = (terms: string[]) =>
  createScorer("contains_terms", 0.8, async ({ output }) => {
    const lower = output.toLowerCase();
    const found = terms.filter(t => lower.includes(t.toLowerCase()));
    return {
      score: found.length / terms.length,
      reason: `Found ${found.length}/${terms.length} terms: ${found.join(", ")}`,
    };
  });

// Length check
const lengthCheck = (min: number, max: number) =>
  createScorer("length", 1.0, async ({ output }) => {
    const words = output.split(/\s+/).length;
    if (words < min) return { score: 0.5, reason: `Too short: ${words} words (min: ${min})` };
    if (words > max) return { score: 0.5, reason: `Too long: ${words} words (max: ${max})` };
    return { score: 1.0, reason: `Good length: ${words} words` };
  });

// Regex match
const regexMatch = (pattern: RegExp, name: string = "regex") =>
  createScorer(name, 1.0, async ({ output }) => {
    const match = pattern.test(output);
    return { score: match ? 1 : 0, reason: match ? `Matches pattern` : `No match` };
  });

// JSON parseable
const jsonParseable = createScorer("json_parseable", 1.0, async ({ output }) => {
  try {
    JSON.parse(output);
    return { score: 1, reason: "Valid JSON" };
  } catch {
    return { score: 0, reason: "Invalid JSON" };
  }
});

// No hallucinated URLs
const noFakeUrls = createScorer("no_fake_urls", 1.0, async ({ output }) => {
  const urls = output.match(/https?:\/\/[^\s]+/g) ?? [];
  return {
    score: urls.length === 0 ? 1 : 0.5,
    reason: urls.length === 0 ? "No URLs" : `Contains ${urls.length} URL(s) — verify manually`,
  };
});

// Semantic similarity scorer
const semanticSimilarity = (threshold: number = 0.8) =>
  createScorer("semantic_similarity", threshold, async ({ output, expected }) => {
    if (!expected) return { score: 1, reason: "No expected output to compare" };

    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: [output, expected],
    });

    const a = response.data[0].embedding;
    const b = response.data[1].embedding;
    let dot = 0, nA = 0, nB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      nA += a[i] * a[i];
      nB += b[i] * b[i];
    }
    const similarity = dot / (Math.sqrt(nA) * Math.sqrt(nB));

    return { score: similarity, reason: `Cosine similarity: ${similarity.toFixed(3)}` };
  });

// Does NOT contain certain terms (for safety)
const doesNotContain = (forbidden: string[]) =>
  createScorer("does_not_contain", 1.0, async ({ output }) => {
    const lower = output.toLowerCase();
    const found = forbidden.filter(t => lower.includes(t.toLowerCase()));
    return {
      score: found.length === 0 ? 1 : 0,
      reason: found.length === 0
        ? "No forbidden terms found"
        : `Found forbidden terms: ${found.join(", ")}`,
    };
  });
```

### 4.3 Domain-Specific Scorers

```typescript
// scorers/domain-scorers.ts
import OpenAI from "openai";

const openai = new OpenAI();

// Scorer for customer support: is the tone appropriate?
const toneChecker = createScorer("tone", 0.7, async ({ output }) => {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `Rate the tone of this customer support response on these dimensions:
- professional (0-1): Is it professional and business-appropriate?
- empathetic (0-1): Does it show understanding of the customer's situation?
- helpful (0-1): Does it provide actionable information?
- not_defensive (0-1): Does it avoid being defensive or dismissive?

Return JSON: { "professional": 0.9, "empathetic": 0.8, "helpful": 0.7, "not_defensive": 0.9 }`,
      },
      { role: "user", content: output },
    ],
  });

  const scores = JSON.parse(response.choices[0].message.content ?? "{}");
  const avg =
    ((scores.professional ?? 0) + (scores.empathetic ?? 0) +
     (scores.helpful ?? 0) + (scores.not_defensive ?? 0)) / 4;

  return {
    score: avg,
    reason: `P=${scores.professional?.toFixed(2)}, E=${scores.empathetic?.toFixed(2)}, H=${scores.helpful?.toFixed(2)}, D=${scores.not_defensive?.toFixed(2)}`,
  };
});

// Scorer for code: is it well-structured?
const codeQuality = createScorer("code_quality", 0.7, async ({ output }) => {
  const codeBlocks = output.match(/```[\s\S]*?```/g) ?? [];
  if (codeBlocks.length === 0 && !output.includes("function") && !output.includes("const")) {
    return { score: 1, reason: "No code to validate" };
  }

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `Assess this code on:
- syntactically_valid (0-1): Does it look like valid code?
- complete (0-1): Is it a complete, runnable snippet?
- well_structured (0-1): Is it well-organized?
Return JSON: { "syntactically_valid": 0.9, "complete": 0.8, "well_structured": 0.7 }`,
      },
      { role: "user", content: output },
    ],
  });

  const scores = JSON.parse(response.choices[0].message.content ?? "{}");
  const avg = ((scores.syntactically_valid ?? 0) + (scores.complete ?? 0) + (scores.well_structured ?? 0)) / 3;
  return { score: avg, reason: JSON.stringify(scores) };
});
```

---

## 5. Building a Single-Turn Eval Framework

### 5.1 The Complete Framework

```typescript
// assessment-framework.ts
import OpenAI from "openai";
import fs from "fs/promises";

const openai = new OpenAI();

// --- Types ---

interface TestCase {
  id: string;
  input: string;
  expected?: string;
  context?: string;
  metadata?: Record<string, unknown>;
}

interface ScorerResult {
  name: string;
  score: number;
  passed: boolean;
  reason: string;
}

type ScorerFn = (params: {
  input: string;
  output: string;
  expected?: string;
  context?: string;
}) => Promise<ScorerResult>;

interface TestResult {
  caseId: string;
  input: string;
  output: string;
  expected?: string;
  scores: ScorerResult[];
  passed: boolean;
  latencyMs: number;
  timestamp: string;
}

interface Summary {
  name: string;
  timestamp: string;
  total: number;
  passed: number;
  failed: number;
  passRate: number;
  avgLatencyMs: number;
  scoresByMetric: Record<string, { avg: number; min: number; max: number; passRate: number }>;
  failures: { caseId: string; failedMetrics: string[] }[];
}

// --- Framework ---

class SingleTurnAssessment {
  private name: string;
  private systemUnderTest: (input: string, context?: string) => Promise<string>;
  private scorers: ScorerFn[];
  private results: TestResult[] = [];

  constructor(
    name: string,
    systemUnderTest: (input: string, context?: string) => Promise<string>,
    scorers: ScorerFn[]
  ) {
    this.name = name;
    this.systemUnderTest = systemUnderTest;
    this.scorers = scorers;
  }

  async run(cases: TestCase[]): Promise<Summary> {
    this.results = [];
    console.log(`\nRunning: ${this.name}`);
    console.log(`Cases: ${cases.length}, Scorers: ${this.scorers.length}\n`);

    for (const testCase of cases) {
      const start = Date.now();
      const output = await this.systemUnderTest(testCase.input, testCase.context);
      const latencyMs = Date.now() - start;

      const scores = await Promise.all(
        this.scorers.map(scorer =>
          scorer({ input: testCase.input, output, expected: testCase.expected, context: testCase.context })
        )
      );

      const passed = scores.every(s => s.passed);

      this.results.push({
        caseId: testCase.id,
        input: testCase.input,
        output: output.slice(0, 500),
        expected: testCase.expected,
        scores,
        passed,
        latencyMs,
        timestamp: new Date().toISOString(),
      });

      const status = passed ? "PASS" : "FAIL";
      const scoreStr = scores.map(s => `${s.name}=${s.score.toFixed(2)}${s.passed ? "" : "*"}`).join(", ");
      console.log(`[${status}] ${testCase.id}: ${scoreStr} (${latencyMs}ms)`);
    }

    return this.summarize();
  }

  private summarize(): Summary {
    const passed = this.results.filter(r => r.passed).length;
    const failed = this.results.filter(r => !r.passed).length;

    const metricValues: Record<string, number[]> = {};
    const metricPasses: Record<string, boolean[]> = {};

    for (const result of this.results) {
      for (const score of result.scores) {
        if (!metricValues[score.name]) metricValues[score.name] = [];
        if (!metricPasses[score.name]) metricPasses[score.name] = [];
        metricValues[score.name].push(score.score);
        metricPasses[score.name].push(score.passed);
      }
    }

    const scoresByMetric: Record<string, { avg: number; min: number; max: number; passRate: number }> = {};
    for (const [metric, values] of Object.entries(metricValues)) {
      scoresByMetric[metric] = {
        avg: values.reduce((s, v) => s + v, 0) / values.length,
        min: Math.min(...values),
        max: Math.max(...values),
        passRate: metricPasses[metric].filter(p => p).length / metricPasses[metric].length,
      };
    }

    const failures = this.results
      .filter(r => !r.passed)
      .map(r => ({ caseId: r.caseId, failedMetrics: r.scores.filter(s => !s.passed).map(s => s.name) }));

    const summary: Summary = {
      name: this.name,
      timestamp: new Date().toISOString(),
      total: this.results.length,
      passed,
      failed,
      passRate: passed / this.results.length,
      avgLatencyMs: this.results.reduce((s, r) => s + r.latencyMs, 0) / this.results.length,
      scoresByMetric,
      failures,
    };

    console.log(`\n${"=".repeat(50)}`);
    console.log(`Results: ${summary.name}`);
    console.log(`${"=".repeat(50)}`);
    console.log(`Total: ${summary.total} | Passed: ${summary.passed} | Failed: ${summary.failed} | Rate: ${(summary.passRate * 100).toFixed(1)}%`);
    console.log(`Avg Latency: ${summary.avgLatencyMs.toFixed(0)}ms`);
    console.log(`\nScores by metric:`);
    for (const [metric, stats] of Object.entries(summary.scoresByMetric)) {
      console.log(`  ${metric}: avg=${stats.avg.toFixed(3)}, min=${stats.min.toFixed(3)}, max=${stats.max.toFixed(3)}, pass=${(stats.passRate * 100).toFixed(0)}%`);
    }
    if (summary.failures.length > 0) {
      console.log(`\nFailures:`);
      for (const f of summary.failures) {
        console.log(`  ${f.caseId}: [${f.failedMetrics.join(", ")}]`);
      }
    }

    return summary;
  }

  async saveResults(filepath: string): Promise<void> {
    await fs.writeFile(filepath, JSON.stringify({ results: this.results, summary: this.summarize() }, null, 2));
    console.log(`\nResults saved to ${filepath}`);
  }
}

// --- Usage ---

async function main() {
  const customerBot = async (input: string, context?: string): Promise<string> => {
    const response = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0.2,
      messages: [
        { role: "system", content: `You are a support agent for Acme SaaS.\n${context ? `Context: ${context}` : ""}\nAnswer concisely. If unsure, say so.` },
        { role: "user", content: input },
      ],
    });
    return response.choices[0].message.content ?? "";
  };

  const scorers: ScorerFn[] = [
    async ({ output, expected }) => {
      if (!expected) return { name: "contains", score: 1, passed: true, reason: "No expected" };
      const terms = expected.toLowerCase().split(/\s+/).filter(w => w.length > 3);
      const found = terms.filter(t => output.toLowerCase().includes(t));
      const score = found.length / Math.max(terms.length, 1);
      return { name: "contains", score, passed: score >= 0.6, reason: `${found.length}/${terms.length} terms` };
    },
    async ({ output }) => {
      const words = output.split(/\s+/).length;
      const score = words >= 5 && words <= 150 ? 1 : 0.5;
      return { name: "length", score, passed: score >= 0.5, reason: `${words} words` };
    },
    async ({ output }) => {
      const hasEmail = /[\w.-]+@[\w.-]+\.\w+/.test(output);
      const hasPhone = /\(\d{3}\)\s?\d{3}-\d{4}/.test(output);
      const score = !hasEmail && !hasPhone ? 1 : 0;
      return { name: "no_fake_contact", score, passed: score === 1, reason: hasEmail || hasPhone ? "Contains contact info" : "Clean" };
    },
  ];

  const cases: TestCase[] = [
    { id: "pricing-1", input: "How much is Pro?", expected: "Pro plan $29 month", context: "Plans: Starter $9/mo, Pro $29/mo, Enterprise contact sales." },
    { id: "pricing-2", input: "What's the cheapest plan?", expected: "Starter $9 month", context: "Plans: Starter $9/mo, Pro $29/mo, Enterprise contact sales." },
    { id: "refund-1", input: "I want a refund", expected: "30-day money-back guarantee", context: "Refund policy: 30-day money-back guarantee on all plans." },
    { id: "hours-1", input: "When is support available?", expected: "Monday Friday 9 AM 6 PM", context: "Support hours: Monday-Friday, 9 AM - 6 PM ET." },
    { id: "unknown-1", input: "Can I pay with Bitcoin?", expected: undefined, context: "Payment methods: credit card, debit card, PayPal." },
  ];

  const runner = new SingleTurnAssessment("Customer Support Bot v1", customerBot, scorers);
  await runner.run(cases);
  await runner.saveResults("./assessment-results.json");
}

main();
```

---

## Summary

You now have a working toolkit for single-turn evals:

1. **Tool selection scoring** — Verify the LLM picks the right tool with the right arguments
2. **Output format checking** — Check schema compliance with Zod validation
3. **Factual accuracy scoring** — Compare output against ground truth with must-contain/must-not-contain
4. **Retrieval quality scoring** — Measure precision, recall, and MRR for RAG systems
5. **Custom scorer functions** — A library of reusable scorers for any need
6. **A complete eval framework** — Run suites, aggregate results, identify failures

In Chapter 20, we'll extend this to multi-turn conversations: testing agent loops, using LLM-as-judge, and assessing complex workflows where a single input/output is not enough.

---

*Previous: [Ch 18 — Why Evals Matter](./18-why-evals-matter.md)* | *Next: [Ch 20 — Multi-Turn Evals](./20-multi-turn-evals.md)*

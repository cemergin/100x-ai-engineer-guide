<!--
  CHAPTER: 5
  TITLE: Structured Output
  PART: 1 — Building with LLM APIs
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 3, Chapter 4
  KEY_TOPICS: JSON mode, Zod schemas, structured output, response_format, type-safe LLM responses, generateObject, streamObject
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 5: Structured Output

> **Part 1 — Building with LLM APIs** | Phase 1: Get Dangerous | Prerequisites: Ch 3, Ch 4 | Difficulty: Intermediate | Language: TypeScript

Until now, every LLM response has been a string. You send a prompt, you get text back. And then you do... what? Parse it with regex? Hope it's valid JSON? Cross your fingers that it has the fields you need?

That's the fundamental problem of working with LLMs in typed applications: **LLMs output strings, but your code needs structured data.** You need a `{ category: "billing", priority: "high", confidence: 0.92 }` object, not a paragraph about how the ticket seems billing-related.

This chapter solves that problem. You'll learn to use Zod schemas to define the exact shape of the data you want, and the Vercel AI SDK's `generateObject` to get type-safe, validated responses from any LLM. By the end, your LLM calls will return proper TypeScript types — not strings you need to parse and pray over.

### In This Chapter
- The problem: unstructured output in typed applications
- JSON mode in OpenAI and Anthropic APIs
- Zod schemas for defining response shapes
- `generateObject` and `streamObject` in the Vercel AI SDK
- Type-safe LLM responses in TypeScript
- Limitations and edge cases

### Related Chapters
- **Ch 4 (Prompt Engineering)** — spirals back: structured prompts produce structured output
- **Ch 8 (Tool Calling)** — spirals forward: tool schemas use the same Zod pattern
- **Ch 20 (Multi-Turn Evals)** — spirals forward: judge prompts return structured scores

---

## 1. The Problem: Strings Are Not Enough

### 1.1 What Goes Wrong with Unstructured Output

Here's what happens when you ask an LLM to "return JSON":

```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const result = await generateText({
  model: openai("gpt-4o-mini"),
  prompt: `Extract the person's name and age from this text. Return JSON.
  
  "Hi, I'm Sarah and I'll be turning 28 next month."`,
});

console.log(result.text);
```

You might get any of these:

```
// Good day:
{"name": "Sarah", "age": 28}

// Bad day:
Here's the extracted information:
```json
{"name": "Sarah", "age": 28}
```

// Worse day:
{"name": "Sarah", "age": "28"}    ← age is a string, not a number

// Worst day:
{"Name": "Sarah", "Age": 28}      ← wrong capitalization
{"name": "Sarah", "age": 28, "note": "turning 28 next month, so currently 27"}  ← extra fields
```

Every one of these breaks your code differently. `JSON.parse()` fails on markdown-wrapped responses. Type checks fail when `age` is a string. Destructuring fails with wrong field names. Extra fields leak through to your database.

### 1.2 The Old Way: Parse and Pray

Before structured output, you'd write defensive code like this:

```typescript
// DON'T DO THIS — this is the old, fragile way
function parseResponse(text: string): { name: string; age: number } {
  // Strip markdown code fences
  let cleaned = text.replace(/```json\n?/g, "").replace(/```\n?/g, "").trim();
  
  // Try to parse
  let parsed: unknown;
  try {
    parsed = JSON.parse(cleaned);
  } catch {
    // Maybe it's wrapped in a sentence?
    const jsonMatch = cleaned.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      parsed = JSON.parse(jsonMatch[0]);
    } else {
      throw new Error("Could not extract JSON from response");
    }
  }

  // Validate fields exist and have right types
  if (typeof parsed !== "object" || parsed === null) throw new Error("Not an object");
  const obj = parsed as Record<string, unknown>;
  
  const name = obj.name ?? obj.Name ?? obj.NAME;
  const age = obj.age ?? obj.Age ?? obj.AGE;
  
  if (typeof name !== "string") throw new Error("name is not a string");
  const ageNum = typeof age === "string" ? parseInt(age, 10) : age;
  if (typeof ageNum !== "number" || isNaN(ageNum)) throw new Error("age is not a number");
  
  return { name, age: ageNum };
}
```

This is terrible. Fragile, verbose, and still misses edge cases. There's a much better way.

---

## 2. The Solution: Zod + generateObject

### 2.1 What generateObject Does

The Vercel AI SDK provides `generateObject` — a function that takes a Zod schema and returns a validated, typed object. It handles:

1. Telling the model to output JSON in the right format
2. Parsing the response
3. Validating against your schema
4. Returning a properly typed TypeScript object

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const result = await generateObject({
  model: openai("gpt-4o-mini"),
  schema: z.object({
    name: z.string().describe("The person's full name"),
    age: z.number().int().describe("The person's age in years"),
  }),
  prompt: `Extract the person's name and age from this text:
  "Hi, I'm Sarah and I'll be turning 28 next month."`,
});

// result.object is fully typed: { name: string; age: number }
console.log(result.object.name); // "Sarah"
console.log(result.object.age);  // 28 (actually a number, not a string)
```

No parsing. No validation code. No defensive programming. The Zod schema IS the contract, and the SDK enforces it.

### 2.2 Zod: A Quick Primer

If you haven't used Zod before, it's a TypeScript-first schema validation library. You define a schema, and Zod validates data against it at runtime while providing type inference at compile time.

```bash
npm install zod
```

```typescript
import { z } from "zod";

// Define a schema
const UserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  role: z.enum(["admin", "user", "guest"]),
  tags: z.array(z.string()).optional(),
});

// TypeScript type is automatically inferred
type User = z.infer<typeof UserSchema>;
// { name: string; email: string; age: number; role: "admin" | "user" | "guest"; tags?: string[] }

// Validation
const valid = UserSchema.parse({
  name: "Sarah",
  email: "sarah@example.com",
  age: 28,
  role: "admin",
});
// Returns typed User object

const invalid = UserSchema.parse({
  name: "Sarah",
  email: "not-an-email",
  age: -5,
  role: "superadmin",
});
// Throws ZodError with detailed error messages
```

### 2.3 Using .describe() for Better Results

The `.describe()` method on Zod schemas is important for `generateObject`. These descriptions become part of the schema the model sees, helping it understand what each field should contain:

```typescript
const TicketSchema = z.object({
  category: z
    .enum(["billing", "technical", "account", "feature", "general"])
    .describe("The primary category of the support ticket"),
  priority: z
    .enum(["critical", "high", "medium", "low"])
    .describe("How urgent this ticket is. Critical = service down, High = blocked, Medium = inconvenience, Low = nice to have"),
  summary: z
    .string()
    .max(100)
    .describe("A brief one-sentence summary of the issue"),
  suggestedAction: z
    .string()
    .describe("What the support team should do first"),
  isEscalation: z
    .boolean()
    .describe("True if this needs to be escalated to a senior engineer"),
});
```

Without descriptions, the model guesses what "category" means. With descriptions, it has explicit guidance. This is especially valuable for ambiguous field names.

---

## 3. Real-World Structured Output Patterns

### 3.1 Classification with Confidence

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const ClassificationSchema = z.object({
  category: z.enum(["spam", "ham", "uncertain"]).describe("Whether the email is spam or legitimate"),
  confidence: z.number().min(0).max(1).describe("Confidence score from 0 to 1"),
  reasoning: z.string().describe("Brief explanation of why this classification was chosen"),
  signals: z.array(z.string()).describe("Specific signals that influenced the classification"),
});

type Classification = z.infer<typeof ClassificationSchema>;

async function classifyEmail(subject: string, body: string): Promise<Classification> {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    temperature: 0,
    schema: ClassificationSchema,
    prompt: `Classify this email as spam or legitimate (ham).

Subject: ${subject}
Body: ${body}`,
  });

  return result.object;
}

// Usage
const result = await classifyEmail(
  "Urgent: Your account needs verification",
  "Click here to verify your account immediately or it will be suspended."
);

console.log(result);
// {
//   category: "spam",
//   confidence: 0.95,
//   reasoning: "Classic phishing pattern with urgency and threat of account suspension",
//   signals: ["urgency language", "threat of suspension", "generic greeting", "click-here CTA"]
// }
```

### 3.2 Data Extraction from Unstructured Text

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const InvoiceSchema = z.object({
  vendor: z.string().describe("Company or person that issued the invoice"),
  invoiceNumber: z.string().nullable().describe("Invoice or reference number if present"),
  date: z.string().nullable().describe("Invoice date in YYYY-MM-DD format"),
  lineItems: z.array(z.object({
    description: z.string(),
    quantity: z.number().nullable(),
    unitPrice: z.number().nullable(),
    total: z.number(),
  })).describe("Individual line items on the invoice"),
  subtotal: z.number().describe("Total before tax"),
  tax: z.number().nullable().describe("Tax amount if listed"),
  total: z.number().describe("Final total amount"),
  currency: z.string().default("USD").describe("Currency code"),
});

async function extractInvoice(text: string) {
  const result = await generateObject({
    model: openai("gpt-4o"),
    temperature: 0,
    schema: InvoiceSchema,
    messages: [
      {
        role: "system",
        content: "Extract invoice data from the provided text. Use null for missing fields. Do not guess or infer values that aren't explicitly stated.",
      },
      { role: "user", content: text },
    ],
  });

  return result.object;
}

// Usage
const invoice = await extractInvoice(`
  INVOICE #2024-0847
  From: Acme Web Services
  Date: March 15, 2024
  
  Web hosting (annual) ........... $299.00
  SSL certificate ................ $49.00
  Domain renewal (2 years) ....... $35.98
  
  Subtotal: $383.98
  Tax (8.25%): $31.68
  Total: $415.66
`);

console.log(invoice.vendor);        // "Acme Web Services"
console.log(invoice.lineItems[0]);  // { description: "Web hosting (annual)", quantity: 1, unitPrice: 299, total: 299 }
console.log(invoice.total);         // 415.66 (a number, not a string!)
```

### 3.3 Multi-Entity Extraction

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const EntitiesSchema = z.object({
  people: z.array(z.object({
    name: z.string(),
    role: z.string().nullable().describe("Their role or title if mentioned"),
    sentiment: z.enum(["positive", "negative", "neutral"]).describe("How they feel about the topic"),
  })),
  companies: z.array(z.object({
    name: z.string(),
    relationship: z.string().describe("Their relationship to the story (e.g., 'acquirer', 'competitor', 'partner')"),
  })),
  topics: z.array(z.string()).describe("Key topics discussed in the text"),
  dates: z.array(z.object({
    date: z.string().describe("The date in YYYY-MM-DD format or relative description"),
    context: z.string().describe("What this date refers to"),
  })),
});

async function extractEntities(text: string) {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    schema: EntitiesSchema,
    prompt: `Extract all notable entities from this text:\n\n${text}`,
  });

  return result.object;
}
```

### 3.4 Enum-Based Routing

Use structured output to make routing decisions:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const RoutingSchema = z.object({
  intent: z.enum([
    "question_about_product",
    "request_for_refund",
    "technical_issue",
    "account_management",
    "general_feedback",
    "off_topic",
  ]).describe("The user's primary intent"),
  department: z.enum(["sales", "support", "engineering", "billing", "none"])
    .describe("Which department should handle this"),
  urgency: z.enum(["immediate", "same_day", "normal", "low"])
    .describe("How quickly this needs attention"),
  autoResolvable: z.boolean()
    .describe("Whether this can be resolved with an automated response"),
  suggestedResponse: z.string().nullable()
    .describe("A suggested response if autoResolvable is true, null otherwise"),
});

async function routeMessage(message: string) {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    temperature: 0,
    schema: RoutingSchema,
    messages: [
      {
        role: "system",
        content: "You are a message routing system. Analyze the user's message and determine how it should be handled.",
      },
      { role: "user", content: message },
    ],
  });

  return result.object;
}

// Usage
const routing = await routeMessage("I was charged $99 but my plan is $49. Please fix this!");

if (routing.autoResolvable && routing.suggestedResponse) {
  // Auto-reply
  sendResponse(routing.suggestedResponse);
} else {
  // Route to human
  createTicket({
    department: routing.department,
    urgency: routing.urgency,
    message,
  });
}
```

---

## 4. Streaming Structured Output

### 4.1 Why Stream Objects?

Sometimes structured output is large (extracting entities from a long document) or you want to show partial results as they arrive. `streamObject` lets you do that:

```typescript
import { streamObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const RecipeSchema = z.object({
  name: z.string(),
  description: z.string(),
  servings: z.number(),
  prepTimeMinutes: z.number(),
  cookTimeMinutes: z.number(),
  ingredients: z.array(z.object({
    item: z.string(),
    amount: z.string(),
    notes: z.string().optional(),
  })),
  steps: z.array(z.string()),
  tips: z.array(z.string()).optional(),
});

async function generateRecipe(dish: string) {
  const result = streamObject({
    model: openai("gpt-4o-mini"),
    schema: RecipeSchema,
    prompt: `Create a detailed recipe for: ${dish}`,
  });

  // Stream partial objects as they arrive
  for await (const partialObject of result.partialObjectStream) {
    console.clear();
    console.log("Generating recipe...");
    
    if (partialObject.name) {
      console.log(`\nRecipe: ${partialObject.name}`);
    }
    if (partialObject.ingredients) {
      console.log(`\nIngredients (${partialObject.ingredients.length} so far):`);
      for (const ing of partialObject.ingredients) {
        console.log(`  - ${ing.amount} ${ing.item}`);
      }
    }
    if (partialObject.steps) {
      console.log(`\nSteps (${partialObject.steps.length} so far)`);
    }
  }

  // Get the final validated object
  const finalResult = await result.object;
  console.log("\n\nFinal recipe:", JSON.stringify(finalResult, null, 2));
}
```

### 4.2 Partial Objects and Type Safety

The partial stream provides objects where every field is potentially `undefined`. TypeScript knows this:

```typescript
for await (const partial of result.partialObjectStream) {
  // partial.name is string | undefined
  // partial.ingredients is Array<{...}> | undefined
  // partial.ingredients?.[0]?.item is string | undefined
  
  if (partial.name && partial.description) {
    // Both are now narrowed to string
    updateUI(partial.name, partial.description);
  }
}
```

This is useful for UIs where you want to show fields as they're generated. The recipe name appears first, then ingredients stream in, then steps.

---

## 5. Schema Design Best Practices

### 5.1 Use Enums for Finite Options

When a field has a known set of possible values, use `z.enum()` instead of `z.string()`:

```typescript
// BAD: model can return anything
const bad = z.object({
  status: z.string(), // "active"? "Active"? "ACTIVE"? "currently active"?
});

// GOOD: model must choose from these options
const good = z.object({
  status: z.enum(["active", "inactive", "suspended", "pending"]),
});
```

Enums give you:
- Guaranteed valid values
- TypeScript union types
- Better model accuracy (it picks from the list, doesn't generate freely)

### 5.2 Use .nullable() for Optional Data

When data might not exist in the source:

```typescript
// BAD: model will make up a value rather than omit
const bad = z.object({
  email: z.string().email(),       // forces model to always provide an email
  phoneNumber: z.string(),          // even if none exists in the text
});

// GOOD: explicitly allow null for missing data
const good = z.object({
  email: z.string().email().nullable()
    .describe("The person's email if mentioned, null otherwise"),
  phoneNumber: z.string().nullable()
    .describe("Phone number if mentioned, null otherwise"),
});
```

The `.describe()` makes the model understand WHEN to use null.

### 5.3 Use .describe() on Everything Non-Obvious

```typescript
// BAD: ambiguous field names
const bad = z.object({
  score: z.number(),
  flag: z.boolean(),
  type: z.string(),
});

// GOOD: every field is self-documenting
const good = z.object({
  score: z.number().min(0).max(100)
    .describe("Quality score from 0 (worst) to 100 (best)"),
  flag: z.boolean()
    .describe("True if the content should be reviewed by a human moderator"),
  type: z.enum(["article", "comment", "review", "question"])
    .describe("The content type being analyzed"),
});
```

### 5.4 Nest Schemas for Complex Objects

```typescript
const AddressSchema = z.object({
  street: z.string().nullable(),
  city: z.string(),
  state: z.string().nullable(),
  country: z.string(),
  postalCode: z.string().nullable(),
});

const CompanySchema = z.object({
  name: z.string(),
  industry: z.string(),
  headquarters: AddressSchema,
  offices: z.array(AddressSchema).describe("Other office locations"),
  founded: z.number().int().nullable(),
  employees: z.number().int().nullable()
    .describe("Approximate number of employees"),
});
```

### 5.5 Schema Constraints

Zod constraints help the model stay within bounds:

```typescript
const ReviewSchema = z.object({
  title: z.string().min(5).max(100)
    .describe("Review title, 5-100 characters"),
  rating: z.number().int().min(1).max(5)
    .describe("Star rating from 1 to 5"),
  pros: z.array(z.string()).min(1).max(5)
    .describe("1 to 5 positive aspects"),
  cons: z.array(z.string()).max(5)
    .describe("0 to 5 negative aspects"),
  summary: z.string().max(500)
    .describe("Summary in 500 characters or less"),
});
```

Note: Not all constraints are enforced by the model perfectly. The model tries to respect `.min()`, `.max()`, `.min(1)` etc., but the SDK validates after generation and will throw if constraints are violated. For critical constraints, add them to your system prompt too.

---

## 6. Advanced Patterns

### 6.1 Schema Reuse Across Your Application

Define schemas once, use them everywhere:

```typescript
// src/ai/schemas/ticket.ts
import { z } from "zod";

export const TicketClassificationSchema = z.object({
  category: z.enum(["billing", "technical", "account", "feature", "general"]),
  priority: z.enum(["critical", "high", "medium", "low"]),
  summary: z.string().max(200),
  confidence: z.number().min(0).max(1),
});

export type TicketClassification = z.infer<typeof TicketClassificationSchema>;

// Use in LLM call
// const result = await generateObject({ schema: TicketClassificationSchema, ... });

// Use in API validation
// const validated = TicketClassificationSchema.parse(requestBody);

// Use in database insertion
// await db.insert(tickets).values(TicketClassificationSchema.parse(data));
```

One schema. Type-safe LLM output, API validation, and database insertion. This is the power of Zod.

### 6.2 generateObject with System Prompts

You can combine system prompts with schemas:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const AnalysisSchema = z.object({
  sentiment: z.enum(["very_positive", "positive", "neutral", "negative", "very_negative"]),
  topics: z.array(z.string()).min(1).max(5),
  actionItems: z.array(z.object({
    task: z.string(),
    priority: z.enum(["high", "medium", "low"]),
    assignee: z.string().nullable(),
  })),
  keyQuote: z.string().nullable().describe("The most important quote from the text, or null if none stands out"),
  needsFollowUp: z.boolean(),
});

async function analyzeCustomerFeedback(feedback: string) {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    temperature: 0,
    schema: AnalysisSchema,
    messages: [
      {
        role: "system",
        content: `You analyze customer feedback for a B2B SaaS company.
        
When identifying action items:
- Only include items that are clearly actionable
- Set assignee to null unless the feedback specifically mentions someone
- "high" priority = affects multiple customers or revenue
- "medium" priority = affects individual customer workflow  
- "low" priority = nice-to-have or cosmetic

When selecting keyQuote, pick something a product manager would highlight in a meeting.`,
      },
      { role: "user", content: feedback },
    ],
  });

  return result.object;
}
```

### 6.3 Handling Complex Union Types

When the output shape depends on the input:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const ResponseSchema = z.object({
  type: z.enum(["answer", "clarification", "refusal"]).describe("The type of response"),
  content: z.string().describe("The response content"),
  sources: z.array(z.string()).describe("Sources cited, empty array if none"),
  followUpQuestions: z.array(z.string()).max(3)
    .describe("Suggested follow-up questions, up to 3"),
  confidence: z.number().min(0).max(1)
    .describe("How confident the system is. Below 0.5 should trigger clarification type."),
});

async function answerQuestion(question: string, context: string) {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    schema: ResponseSchema,
    messages: [
      {
        role: "system",
        content: `Answer the user's question based on the provided context.

If you can answer confidently, use type "answer".
If the question is ambiguous, use type "clarification" and ask for specifics.
If the question is outside your knowledge or the context, use type "refusal" and explain why.`,
      },
      {
        role: "user",
        content: `Context: ${context}\n\nQuestion: ${question}`,
      },
    ],
  });

  return result.object;
}
```

### 6.4 Output Mode: Array

When you need a list of objects directly:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const result = await generateObject({
  model: openai("gpt-4o-mini"),
  output: "array",
  schema: z.object({
    word: z.string(),
    partOfSpeech: z.enum(["noun", "verb", "adjective", "adverb", "other"]),
    definition: z.string(),
    exampleSentence: z.string(),
  }),
  prompt: "Generate 5 advanced English vocabulary words useful for business writing.",
});

// result.object is an array directly
for (const word of result.object) {
  console.log(`${word.word} (${word.partOfSpeech}): ${word.definition}`);
}
```

### 6.5 Output Mode: Enum

For simple classification tasks, return just an enum value:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const result = await generateObject({
  model: openai("gpt-4o-mini"),
  output: "enum",
  enum: ["spam", "not_spam", "uncertain"],
  prompt: `Classify this email: "Congratulations! You've won a free iPhone. Click here now!"`,
});

console.log(result.object); // "spam"
```

---

## 7. Error Handling for Structured Output

### 7.1 When Schema Validation Fails

Sometimes the model generates JSON that doesn't match your schema. `generateObject` will throw:

```typescript
import { generateObject } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const StrictSchema = z.object({
  rating: z.number().int().min(1).max(5),
  review: z.string().min(10).max(500),
});

try {
  const result = await generateObject({
    model: openai("gpt-4o-mini"),
    schema: StrictSchema,
    prompt: "Rate this product: 'It was okay'",
  });
  console.log(result.object);
} catch (error) {
  if (error instanceof Error) {
    console.error("Structured output failed:", error.message);
    // Retry with a different model or relaxed schema
  }
}
```

### 7.2 Retry with Schema Relaxation

If strict schemas fail too often, consider a two-pass approach:

```typescript
import { generateObject, generateText } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

const StrictSchema = z.object({
  rating: z.number().int().min(1).max(5),
  category: z.enum(["excellent", "good", "average", "poor", "terrible"]),
  summary: z.string().min(10).max(200),
});

async function extractReview(text: string) {
  // First attempt: strict schema
  try {
    const result = await generateObject({
      model: openai("gpt-4o-mini"),
      schema: StrictSchema,
      prompt: `Extract a review from: "${text}"`,
    });
    return result.object;
  } catch {
    console.warn("Strict extraction failed, trying relaxed approach...");
  }

  // Fallback: use a more capable model
  try {
    const result = await generateObject({
      model: openai("gpt-4o"),
      schema: StrictSchema,
      prompt: `Extract a review from: "${text}"`,
    });
    return result.object;
  } catch {
    console.error("Both attempts failed");
    throw new Error("Could not extract structured review from input");
  }
}
```

---

## 8. Limitations and Gotchas

### 8.1 Schema Complexity Limits

Very complex schemas with deep nesting can confuse models:

```typescript
// This might struggle:
const TooComplex = z.object({
  level1: z.object({
    level2: z.object({
      level3: z.object({
        level4: z.array(z.object({
          deeply: z.object({
            nested: z.string(),
          }),
        })),
      }),
    }),
  }),
});

// Better: flatten the structure
const Flattened = z.object({
  items: z.array(z.object({
    id: z.string(),
    value: z.string(),
    category: z.string(),
    parentCategory: z.string(),
  })),
});
```

### 8.2 Large Schemas Cost More Tokens

The schema is sent to the model as part of the request. A complex schema with many fields and descriptions can be hundreds of tokens. This adds to every request.

### 8.3 Not All Models Support Structured Output Equally

OpenAI's models have the best structured output support with their `response_format: { type: "json_schema" }` feature. Anthropic and Google also work well with the Vercel AI SDK's `generateObject`, which uses a combination of prompting and post-processing to enforce schemas.

If you find one provider struggling with a specific schema, try another — the SDK makes switching trivial.

### 8.4 Streaming Structured Output Limitations

When streaming objects, the partial stream emits as fields are generated. This means:
- You can't validate the partial object (it's incomplete)
- Array items appear one at a time
- The final validation only happens when streaming completes

### 8.5 String Length Constraints Are Soft

The model tries to respect `.max(100)` on strings, but it's not perfectly enforced during generation. The SDK validates after, so overly long strings will cause an error rather than silently pass. For critical length limits, mention them in the description too.

---

## 9. Provider-Specific: JSON Mode

### 9.1 OpenAI JSON Mode (Direct SDK)

If you're using the OpenAI SDK directly (not Vercel AI SDK), you can request JSON mode:

```typescript
import OpenAI from "openai";

const client = new OpenAI();

// Basic JSON mode — ensures valid JSON, but no schema enforcement
const response = await client.chat.completions.create({
  model: "gpt-4o-mini",
  response_format: { type: "json_object" },
  messages: [
    {
      role: "system",
      content: "Extract the name and age. Respond in JSON with fields: name (string), age (number).",
    },
    {
      role: "user",
      content: "Sarah is 28 years old.",
    },
  ],
});

const data = JSON.parse(response.choices[0].message.content!);
// data is untyped — you still need to validate
```

```typescript
// Strict JSON Schema mode — guarantees schema compliance
const response = await client.chat.completions.create({
  model: "gpt-4o-mini",
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "person_extraction",
      strict: true,
      schema: {
        type: "object",
        properties: {
          name: { type: "string" },
          age: { type: "integer" },
        },
        required: ["name", "age"],
        additionalProperties: false,
      },
    },
  },
  messages: [
    { role: "user", content: "Sarah is 28 years old." },
  ],
});
```

The Vercel AI SDK's `generateObject` does all of this for you — including converting Zod schemas to JSON Schema format. Use `generateObject` unless you need direct OpenAI API access.

---

## 10. Key Takeaways

1. **Use `generateObject` for structured data.** Don't parse strings. Define a Zod schema, get a typed object back.

2. **Use `.describe()` on schema fields.** Help the model understand what each field should contain, especially for ambiguous names.

3. **Use enums for finite options.** `z.enum()` gives better accuracy than `z.string()` for classification.

4. **Use `.nullable()` for optional data.** Don't force the model to fabricate data that doesn't exist in the source.

5. **Schema = contract.** The same Zod schema validates LLM output, API requests, and database insertions.

6. **Stream objects for large responses.** `streamObject` shows partial results as they generate.

7. **Handle failures gracefully.** Schema validation can fail. Retry with better models or relaxed schemas.

8. **Keep schemas flat.** Deep nesting confuses models. Flatten when possible.

---

## What's Next

You can make LLM calls (Ch 3), write effective prompts (Ch 4), and get structured data back (Ch 5). Next, we combine all of these into a real user-facing experience.

In **Chapter 6: Building a Chat Interface**, you'll build a streaming chat UI with conversation history management, the Vercel AI SDK's React hooks, and a Next.js backend. This is where your AI features start looking like a product.

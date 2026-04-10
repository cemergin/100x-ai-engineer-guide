<!--
  CHAPTER: 0
  TITLE: How LLMs Actually Work
  PART: 0 — LLM Fundamentals
  PHASE: 1 — Get Dangerous
  PREREQS: None
  KEY_TOPICS: tokens, tokenization, context windows, temperature, top-p, sampling, hallucination, training, inference, model families
  DIFFICULTY: Beginner
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 0: How LLMs Actually Work

> **Part 0 — LLM Fundamentals** | Phase 1: Get Dangerous | Prerequisites: None | Difficulty: Beginner | Language: TypeScript

You're about to start building with large language models. Before you write your first API call, you need a mental model of what's actually happening when you send text to an LLM and get text back. Not a PhD-level understanding — just enough to make good decisions about how you build.

Here's the thing: most "AI tutorials" either go way too deep (here's the math behind attention mechanisms) or way too shallow (just call the API!). Neither helps you when you're debugging why your chatbot is confidently making up customer support policies, or why your bill jumped 10x overnight, or why your app is slow when conversations get long.

This chapter gives you the working mental model. We'll cover what tokens are and why they determine your costs, what context windows are and why they limit your conversations, how randomness controls like temperature work, and — critically — why LLMs hallucinate. You don't need to understand backpropagation to build great AI features. But you *do* need to understand these fundamentals.

### In This Chapter
- Tokens and tokenization: the atomic unit of LLM input/output
- Context windows: the memory limit of every conversation
- Temperature, top-p, and sampling: controlling randomness
- Why LLMs hallucinate (and why you can't fully stop it)
- Training vs inference: what happened before you got here
- Model families: GPT, Claude, Gemini, Llama, Mistral at a glance

### Related Chapters
- **Ch 3 (Your First LLM Call)** — where you actually USE tokens and context windows in code
- **Ch 12 (Context Window Management)** — spirals context windows into production strategies
- **Ch 29 (Context Window Internals)** — spirals to expert depth: compaction, token budgets, cache-break vectors
- **Ch 34 (ML Decision Making)** — spirals the "prediction" framing into real ML
- **Ch 37 (Tokenization Deep Dive)** — spirals tokenization to BPE, WordPiece, encoding/decoding
- **Ch 39 (Decoding & Generation)** — spirals temperature/sampling into the actual math

---

## 1. Tokens: The Atomic Unit of LLMs

### 1.1 What Is a Token?

LLMs don't read words. They read **tokens** — chunks of text that the model has learned to treat as single units. Sometimes a token is a whole word. Sometimes it's part of a word. Sometimes it's a single character or a piece of punctuation.

Here's a rough intuition: in English, **1 token is approximately 0.75 words**, or conversely, **1 word is roughly 1.3 tokens**. But this varies wildly depending on the text.

Some examples of how text gets tokenized:

```
"Hello world"           → ["Hello", " world"]                    → 2 tokens
"tokenization"          → ["token", "ization"]                   → 2 tokens
"AI"                    → ["AI"]                                 → 1 token
"supercalifragilistic"  → ["super", "cal", "ifrag", "ilistic"]   → 4 tokens
"こんにちは"              → ["こん", "にち", "は"]                   → 3 tokens
"{"name": "Alice"}"     → ["{", "\"", "name", "\":", " \"", "Alice", "\"}"]  → 7 tokens
```

Notice a few things:
- Common English words are usually 1 token
- Less common words get split into pieces
- Non-English text tends to use more tokens per character
- JSON and code have their own tokenization patterns

### 1.2 Why Tokens Matter for You

Tokens matter for three practical reasons:

**Cost.** Every API call is billed by tokens. When OpenAI charges "$3 per million input tokens" for GPT-4o, they mean it literally. A 1,000-word prompt is roughly 1,300 tokens. A back-and-forth conversation with 50 messages might be 15,000+ tokens. Understanding tokenization helps you estimate and control costs.

**Speed.** LLMs generate tokens sequentially — one at a time. A response of 500 tokens takes roughly 5x longer to generate than a response of 100 tokens. When you ask for verbose output, you're paying in both money and latency.

**Context limits.** Every model has a maximum number of tokens it can process in a single request (input + output combined). Understanding tokens helps you know when you're approaching that limit.

### 1.3 How Tokenization Works (Conceptual)

You don't need to build a tokenizer, but understanding the concept helps.

Most modern LLMs use a technique called **Byte Pair Encoding (BPE)** or a variant of it. The core idea:

1. Start with individual characters as your vocabulary
2. Find the most frequently occurring pair of characters in your training text
3. Merge that pair into a new token
4. Repeat thousands of times

After training, you end up with a vocabulary of ~50,000-100,000 tokens that efficiently represent common patterns. "the" is a single token because it appears constantly. "xylophone" might be split into "xy", "lo", "phone" because each piece appears in other words too.

**The key insight:** tokenization is learned from training data. This is why:
- English text is tokenized efficiently (lots of English training data)
- Non-English text often uses more tokens (less training data)
- Code tokens depend on the language (Python is usually more efficient than Rust)
- Domain-specific jargon may be split into multiple tokens

> **Spiral note:** In Ch 37 (Tokenization Deep Dive), we'll implement BPE from scratch, explore WordPiece and SentencePiece, and build tokenization pipelines. For now, the mental model is enough.

### 1.4 Token Counting in Practice

Different models use different tokenizers, so the same text produces different token counts:

| Model Family | Tokenizer | Approx Vocabulary Size |
|---|---|---|
| GPT-4, GPT-4o | cl100k_base / o200k_base | 100,000-200,000 |
| Claude 3.x | Claude tokenizer | ~100,000 |
| Gemini | SentencePiece | ~256,000 |
| Llama 3 | Tiktoken-based | ~128,000 |

For estimation purposes, the "1 token ≈ 4 characters in English" rule works across all of them. For exact counts, use the provider's tokenizer library.

Here's a quick way to count tokens with OpenAI's tokenizer (we'll set up this project properly in Ch 3):

```typescript
// Quick token counting example — we'll set this up properly in Ch 3
import { encoding_for_model } from "tiktoken";

const encoder = encoding_for_model("gpt-4o");
const text = "Hello, how can I help you today?";
const tokens = encoder.encode(text);

console.log(`Text: "${text}"`);
console.log(`Token count: ${tokens.length}`);  // 8
console.log(`Tokens: ${Array.from(tokens)}`);   // [9906, 11, 1268, 649, 358, 1520, 499, 3432, 30]
```

**Rule of thumb for cost estimation:**
- 1 page of English text ≈ 500 tokens
- A typical user message ≈ 50-200 tokens
- A typical assistant response ≈ 100-500 tokens
- A system prompt ≈ 200-2,000 tokens
- A full conversation (20 turns) ≈ 5,000-15,000 tokens

---

## 2. Context Windows: The Memory Limit

### 2.1 What Is a Context Window?

The **context window** is the maximum number of tokens an LLM can process in a single request. This includes everything: your system prompt, the conversation history, the user's latest message, AND the model's response.

Think of it as the model's working memory. Everything the model needs to "know" for this particular request must fit within the context window.

```
┌─────────────── Context Window (e.g., 128K tokens) ──────────────┐
│                                                                   │
│  [System Prompt]  [Message 1]  [Message 2]  ...  [Latest Message] │
│  ←── input tokens ──────────────────────────────────────────────→ │
│                                                   [Response]      │
│                                                   ←─ output ──→  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

Input tokens + Output tokens ≤ Context Window Size
```

### 2.2 Context Window Sizes (2026)

Context windows have grown dramatically:

| Model | Context Window | That's Roughly... |
|---|---|---|
| GPT-4o | 128K tokens | ~96K words / ~200 pages |
| Claude 3.5 Sonnet | 200K tokens | ~150K words / ~300 pages |
| Claude Opus 4 | 200K tokens | ~150K words / ~300 pages |
| Gemini 1.5 Pro | 1M tokens | ~750K words / ~1,500 pages |
| Gemini 2.0 | 1M tokens | ~750K words / ~1,500 pages |
| Llama 3 (405B) | 128K tokens | ~96K words / ~200 pages |

**"128K tokens should be enough for anyone," right?** Not so fast. In practice:

- Your system prompt eats 500-2,000 tokens
- Each conversation turn adds 200-600 tokens  
- After 50-100 turns, you've used 15,000-60,000 tokens just in history
- If you're doing RAG (retrieving documents), each document chunk is 500-2,000 tokens
- Tool call definitions eat tokens too

The context window fills up faster than you think. And when it does, you have to make choices about what to keep and what to drop.

### 2.3 Why Context Windows Matter

**Cost scales with context.** You pay for every token in the context window on every request. In a conversation, you send the *entire history* with each new message. Turn 1 might cost 500 tokens. Turn 20 might cost 10,000 tokens. Turn 100 might cost 50,000 tokens. Your costs grow quadratically with conversation length.

**Quality degrades with distance.** Research consistently shows that LLMs pay more attention to the beginning and end of the context window than the middle. This is called the "lost in the middle" problem. Stuffing 100K tokens of context doesn't mean the model will use all of it equally well.

**Speed decreases with context length.** Longer contexts take longer to process. The first token of a response takes longer to arrive (higher "time to first token"), and the overall request takes longer.

### 2.4 Practical Context Window Strategies

Even at this early stage, here are the strategies you'll use:

1. **Keep system prompts lean.** Every token in your system prompt is repeated in every request. A 2,000-token system prompt costs you 2,000 tokens per turn.

2. **Summarize old conversation history.** Instead of keeping 100 raw messages, summarize older messages into a condensed form. (We'll build this in Ch 12.)

3. **Be selective about what goes in context.** Don't dump an entire database into the context. Retrieve only the relevant pieces. (This is what RAG does — Ch 14-17.)

4. **Set max output tokens.** Most APIs let you limit response length. Use it to control costs and prevent the model from rambling.

> **Spiral note:** Ch 12 goes deep on context window management strategies: sliding windows, compaction, summarization. Ch 29 covers how production tools like Claude Code manage context internally.

---

## 3. Temperature, Top-p, and Sampling

### 3.1 How LLMs Generate Text

Here's what actually happens when an LLM generates text:

1. The model reads all the input tokens
2. It predicts a **probability distribution** over every token in its vocabulary for the next token
3. It **samples** one token from that distribution
4. That token gets appended to the input
5. Repeat from step 2 until done

Step 3 is where it gets interesting. The model doesn't just pick the most probable token — it **samples** based on the probabilities. And you can control *how* it samples.

```
Input: "The capital of France is"

Model's prediction for next token:
  "Paris"     → 92% probability
  "a"         → 2% probability
  "the"       → 1.5% probability
  "located"   → 1% probability
  "known"     → 0.8% probability
  ...thousands more tokens with tiny probabilities...
```

### 3.2 Temperature: Controlling Randomness

**Temperature** scales the probability distribution before sampling. It's a number, typically between 0 and 2.

- **Temperature = 0**: The model always picks the highest-probability token. Deterministic (mostly). "The capital of France is Paris" every time.
- **Temperature = 0.5**: Slight randomness. High-probability tokens are still strongly favored, but there's some variation.
- **Temperature = 1.0**: The model samples directly from its predicted distribution. The default for most APIs.
- **Temperature = 1.5-2.0**: More random. Lower-probability tokens get a real chance. Good for creative writing, bad for factual answers.

Think of it like this:

```
Temperature = 0    → "I'll have the usual"
Temperature = 0.7  → "I'll try something from the top of the menu"
Temperature = 1.0  → "Let me look at the whole menu"
Temperature = 1.5  → "I'll try that weird thing in the corner"
Temperature = 2.0  → "Just bring me whatever"
```

**When to use what:**

| Use Case | Temperature | Why |
|---|---|---|
| Data extraction | 0 | You want consistent, predictable output |
| Code generation | 0-0.3 | Correctness matters more than creativity |
| Conversation | 0.5-0.8 | Natural-sounding but not wild |
| Creative writing | 0.8-1.2 | You want variety and surprise |
| Brainstorming | 1.0-1.5 | Generate diverse, unexpected ideas |

### 3.3 Top-p (Nucleus Sampling)

**Top-p** (also called nucleus sampling) is an alternative way to control randomness. Instead of scaling probabilities, it cuts off the tail.

- **Top-p = 0.1**: Only consider tokens in the top 10% of probability mass
- **Top-p = 0.5**: Consider the smallest set of tokens whose probabilities sum to 50%
- **Top-p = 1.0**: Consider all tokens (no filtering)

Example with top-p = 0.9:

```
"Paris"     → 92%  ← included (cumulative: 92%)
"a"         → 2%   ← not included (cumulative would be 94%, but we hit 90% already)
... everything else excluded ...
```

With top-p = 0.95:

```
"Paris"     → 92%  ← included (cumulative: 92%)
"a"         → 2%   ← included (cumulative: 94%)
"the"       → 1.5% ← included (cumulative: 95.5%) — this pushes us over 95%, stop here
... everything else excluded ...
```

**Temperature vs top-p:** Most practitioners use one or the other, not both. OpenAI's documentation recommends changing either temperature or top-p, but not both at the same time.

- **Temperature** makes all probabilities more or less extreme
- **Top-p** completely eliminates low-probability options

In practice, temperature is more commonly used and more intuitive. Top-p is useful when you want to prevent rare but possible nonsense tokens while keeping the overall distribution shape.

### 3.4 Other Sampling Parameters

Most APIs expose a few more controls:

**Frequency penalty** (0 to 2): Reduces the probability of tokens that have already appeared in the output. Prevents repetition like "The cat sat on the mat. The cat sat on the mat."

**Presence penalty** (0 to 2): Reduces the probability of tokens that have appeared *at all* in the output, regardless of how many times. Encourages the model to talk about new topics.

**Max tokens**: Hard limit on how many tokens the model can generate. Essential for cost control.

**Stop sequences**: Strings that cause the model to stop generating. Useful for structured outputs where you know the ending pattern.

> **Spiral note:** Ch 39 (Decoding & Generation) covers the actual math: softmax with temperature, nucleus sampling algorithms, beam search, and more.

---

## 4. Why LLMs Hallucinate

### 4.1 The Nature of Hallucination

This is the single most important concept for any AI engineer to internalize:

**LLMs are not knowledge retrieval systems. They are probabilistic text predictors.**

When you ask "What is the capital of France?", the model doesn't look up "France → Paris" in a database. It predicts that, given the input tokens, the most likely next tokens form the string "Paris" — because in its training data, that sequence overwhelmingly follows similar prompts.

This works great for things that appeared often in training data. It falls apart for:

- **Rare facts**: Things that appeared infrequently in training
- **Recent events**: Anything after the training data cutoff
- **Specific numbers**: Exact dates, statistics, measurements
- **Niche domains**: Specialized knowledge with little training representation
- **Logical reasoning**: Multi-step deduction (the model is pattern-matching, not reasoning)

### 4.2 Types of Hallucination

**Confident fabrication**: The model generates false information with the same confidence as true information. "The Treaty of Westphalia was signed in 1648" (true) sounds exactly like "The Treaty of Westphalia was signed in 1654" (false). There's no uncertainty marker in the output.

**Plausible but wrong**: The model generates something that *sounds* right and follows the right patterns but is factually incorrect. "The Eiffel Tower was designed by Alexandre-Gustave Eiffel and completed in 1887" — it was completed in 1889.

**Citation fabrication**: Ask the model for academic sources and it will generate papers with realistic titles, plausible author names, and convincing journal names — for papers that don't exist. The format is right. The content is invented.

**Reasoning errors**: The model generates step-by-step reasoning that looks logical but contains a subtle error in one step, leading to a wrong conclusion. It's pattern-matching what reasoning "looks like" without actually reasoning.

### 4.3 Why You Can't Fully Eliminate Hallucination

Hallucination isn't a bug — it's a fundamental property of how these models work. The same mechanism that lets an LLM generate creative, coherent, contextually appropriate text is the mechanism that produces hallucinations.

Think about it: if the model could ONLY output things it "knew" with certainty, it couldn't do most of what makes it useful. It couldn't rephrase, summarize, translate, write code for your specific use case, or have a conversation. All of those require generating *new* text that wasn't in the training data.

**What you can do:**
1. **Reduce hallucination** with better prompts (Ch 4)
2. **Ground responses** in retrieved documents (RAG — Ch 14-17)
3. **Verify outputs** with structured schemas (Ch 5)
4. **Detect hallucination** with evaluations (Ch 18-22)
5. **Set expectations** in your UI — don't present LLM output as authoritative fact

### 4.4 The Practical Takeaway

Never trust LLM output as fact. Always design your system assuming the model might be wrong:

- **High-stakes decisions?** Require human review.
- **Factual claims?** Ground them in retrieved documents and cite sources.
- **Structured data?** Validate against schemas.
- **User-facing text?** Include disclaimers where appropriate.
- **Code generation?** Run tests.

The models that hallucinate less aren't "smarter" — they've been trained with more reinforcement learning from human feedback (RLHF) to say "I don't know" more often. But even the best models hallucinate. Design for it.

---

## 5. Training vs Inference

### 5.1 The Two Phases

As an AI engineer building with APIs, you'll spend 99% of your time on **inference** — sending inputs to a trained model and getting outputs back. But understanding the distinction helps you reason about what models can and can't do.

**Training** is when the model learns. It processes enormous amounts of text (think: large fractions of the internet), adjusts its internal parameters (weights) to get better at predicting the next token, and builds up the "knowledge" and "abilities" that you use through the API.

**Inference** is when the model predicts. You send it a prompt, it runs a forward pass through its neural network, and it generates tokens one at a time. No learning happens during inference — the model's weights are frozen.

```
TRAINING (done by the provider — OpenAI, Anthropic, etc.)
═══════════════════════════════════════════════════════════
Months of compute on thousands of GPUs
Trillions of tokens of text data
Billions of parameters adjusted
RLHF: humans rate outputs, model learns preferences
Result: a frozen model checkpoint

INFERENCE (what YOU do via the API)
═══════════════════════════════════════════════════════════
Send prompt → model generates response
No weights change
No learning happens
Model is exactly the same before and after your request
Takes milliseconds to seconds, not months
```

### 5.2 Why This Matters for You

**The model doesn't learn from your conversations.** When you "teach" an LLM something in a conversation, you're not changing its weights. You're adding information to the context window that influences its predictions for *this session only*. Next session, it's a blank slate.

**Training data has a cutoff.** The model only "knows" things from its training data. Claude's training data has a cutoff; GPT-4's has a cutoff. If something happened after the cutoff, the model doesn't know about it (unless you put it in the context).

**You probably won't train models.** Training a model from scratch costs millions of dollars in compute. Fine-tuning (adjusting a pre-trained model on your data) is more accessible but still requires expertise. For most AI engineering tasks, you'll use pre-trained models via APIs and control their behavior through prompting, context, and retrieval.

### 5.3 The Training Pipeline (High Level)

For your mental model, here's what happens before you get access to a model:

1. **Pre-training**: The model learns to predict the next token on trillions of tokens of internet text. This gives it language understanding, general knowledge, and emergent abilities like reasoning and code generation.

2. **Supervised fine-tuning (SFT)**: The model is trained on curated examples of helpful, well-formatted responses. This teaches it to follow instructions and respond in a conversational format.

3. **RLHF / RLAIF (Reinforcement Learning from Human/AI Feedback)**: Humans (or AI systems) rate model outputs. The model is trained to produce outputs that get higher ratings. This is what makes models "aligned" — helpful, harmless, and honest.

4. **Safety training**: Additional training focused on refusing harmful requests, avoiding biases, and following safety guidelines.

The result is a model that can follow instructions, maintain a conversation, refuse harmful requests, and generate helpful outputs — all properties that emerged from training, not from the architecture itself.

> **Spiral note:** Ch 34-36 go deep into the ML fundamentals: how neural networks learn, what weights and biases are, how backpropagation works. Ch 44-47 cover fine-tuning your own models.

---

## 6. Model Families: A Practical Guide

### 6.1 The Major Players (2026)

The LLM landscape changes fast, but the major families have stabilized. Here's what you need to know for practical decision-making:

#### OpenAI (GPT Family)

**Models:** GPT-4o, GPT-4o-mini, o1, o3, o3-mini, o4-mini

**What it's good at:** Broad generalist, strong at code, good tool calling support, widest ecosystem of integrations. The "safe default" choice.

**What it's less good at:** Can be verbose. Structured output sometimes adds extra fields. Pricing is middle-of-road.

**Vibe:** The iPhone of LLMs — polished, well-integrated, works for most people.

#### Anthropic (Claude Family)

**Models:** Claude 3.5 Sonnet, Claude 3.5 Haiku, Claude Opus 4, Claude Sonnet 4

**What it's good at:** Following nuanced instructions, maintaining persona, long context (200K), writing quality. Very good at "do exactly what I asked and nothing more." Extended thinking for complex reasoning.

**What it's less good at:** Smaller ecosystem than OpenAI. Can be overly cautious about safety-adjacent requests.

**Vibe:** The thoughtful colleague who actually reads the brief.

#### Google (Gemini Family)

**Models:** Gemini 1.5 Pro, Gemini 1.5 Flash, Gemini 2.0 Flash, Gemini 2.5 Pro

**What it's good at:** Massive context windows (up to 1M tokens), strong multimodal (native image/video/audio understanding), competitive pricing. Flash models are very fast and cheap.

**What it's less good at:** API ergonomics lag behind OpenAI/Anthropic. Tool calling support is less mature. Inconsistent quality between model sizes.

**Vibe:** The one with the biggest memory and the most senses.

#### Meta (Llama Family — Open Source)

**Models:** Llama 3.1 (8B, 70B, 405B), Llama 3.2 (1B, 3B, 11B, 90B), Llama 4 (Scout, Maverick)

**What it's good at:** Open source — you can run it yourself, fine-tune it, deploy it anywhere. No per-token API costs if self-hosted. Great ecosystem of tools and adaptations.

**What it's less good at:** Requires infrastructure to run. Smaller models have significant quality gaps vs. frontier closed models. No built-in safety/moderation.

**Vibe:** The Linux of LLMs — freedom and flexibility, some assembly required.

#### Mistral

**Models:** Mistral Large, Mistral Medium, Mistral Small, Codestral, Mixtral (open)

**What it's good at:** European-based (data residency matters for some companies). Good price/performance ratio. Mixtral pioneered efficient mixture-of-experts architecture. Strong for multilingual.

**What it's less good at:** Smaller model ecosystem. Less third-party integration support. Documentation can be sparse.

**Vibe:** The European sports car — focused, efficient, a bit niche.

### 6.2 How to Choose (Quick Decision Tree)

```
Need to ship fast with minimal risk?
  → GPT-4o or Claude Sonnet 4

Need to follow complex instructions precisely?
  → Claude Opus 4 or Claude Sonnet 4

Need massive context (1M+ tokens)?
  → Gemini 1.5 Pro / 2.0

Need the cheapest option for simple tasks?
  → GPT-4o-mini, Claude 3.5 Haiku, or Gemini 2.0 Flash

Need to self-host or fine-tune?
  → Llama 3.1 or Llama 4

Need data residency in Europe?
  → Mistral

Need complex reasoning with step-by-step thinking?
  → o3, o4-mini, Claude Opus 4 (extended thinking), Gemini 2.5 Pro
```

### 6.3 Model Size and Quality

Models come in different sizes, measured in **parameters** (the learned values in the neural network):

```
Small:   1B-8B parameters    → Fast, cheap, limited quality
Medium:  8B-70B parameters   → Good balance, runs on high-end hardware
Large:   70B-405B parameters → Best quality, expensive to run
Frontier: ???B parameters    → GPT-4o, Claude Opus 4, Gemini (sizes undisclosed)
```

**The rule of thumb:** Use the smallest model that gives you acceptable quality. Don't reach for GPT-4o when GPT-4o-mini handles the task. Don't use Claude Opus 4 when Haiku works fine.

Why? Because:
- Smaller models are cheaper (often 10-50x cheaper)
- Smaller models are faster (lower latency)
- Smaller models scale better (higher throughput)
- Many tasks don't need frontier quality

We'll cover model selection in much more depth in Ch 2 (The AI Engineer's Landscape) and Ch 43 (Model Selection & Architecture).

### 6.4 Open Source vs Closed Source

This is a fundamental choice you'll revisit throughout the guide:

| | Closed Source (API) | Open Source (Self-hosted) |
|---|---|---|
| **Examples** | GPT-4o, Claude, Gemini | Llama, Mistral, Qwen |
| **Cost model** | Per-token | Infrastructure (GPUs) |
| **Setup** | API key, done | Provision servers, load model, optimize |
| **Quality** | Frontier | Good, but lags behind frontier |
| **Data privacy** | Data goes to provider | Data stays in your infra |
| **Customization** | Prompting only | Fine-tuning, modification |
| **Scaling** | Provider handles it | You handle it |

For Phase 1 of this guide (Get Dangerous), we use closed-source APIs exclusively. It's the fastest path to shipping AI features. Phase 2 covers open source and self-hosting.

---

## 7. Putting It All Together: The Mental Model

Here's the complete mental model for how an LLM API call works:

```
1. YOU compose a request:
   ┌───────────────────────────────────────┐
   │ System prompt:  "You are a helpful..." │
   │ User message:   "What is React?"       │
   │ Settings:       temperature=0.7        │
   │                 max_tokens=500         │
   └───────────────────────────────────────┘

2. TOKENIZATION happens:
   Your text → tokens (using the model's tokenizer)
   "What is React?" → [What, is, React, ?] → [token_3923, token_382, token_17614, token_30]

3. THE MODEL processes all input tokens at once:
   - The neural network reads all input tokens
   - Internal layers (attention, feed-forward) process them
   - Result: a probability distribution over the vocabulary

4. SAMPLING generates one token:
   - Temperature scales the probabilities
   - Top-p filters unlikely tokens
   - One token is sampled: "React"

5. REPEAT from step 3 with the new token appended:
   Input: [What, is, React, ?, React]
   → generates "is"
   Input: [What, is, React, ?, React, is]
   → generates "a"
   ... and so on ...

6. STOP when:
   - The model generates a stop token (<|endoftext|>)
   - max_tokens is reached
   - A stop sequence is matched

7. YOU receive the full response:
   "React is a JavaScript library for building user interfaces..."
```

The critical things to remember:

- **Tokens in = cost + speed.** More input tokens = higher cost and slower time-to-first-token.
- **Tokens out = cost + speed.** More output tokens = higher cost and more generation time.
- **Context window = hard limit.** Input tokens + output tokens must fit within the window.
- **Temperature = randomness.** Low for deterministic tasks, high for creative tasks.
- **The model doesn't know things.** It predicts tokens. Design accordingly.
- **The model doesn't learn from you.** Each API call is stateless. Conversation history is just more input tokens.

---

## 8. Common Misconceptions

Let's clear up some things that trip up new AI engineers:

### "The model remembers our conversation"

No. Each API call is stateless. The model has no memory between requests. "Memory" in a chatbot is maintained by YOU — by sending the entire conversation history with each request. This is why conversations get expensive: you're re-sending everything.

### "I trained the model by giving it examples"

No. You added examples to the context window. The model's weights didn't change. Those examples influence *this request only*. This is called "in-context learning" or "few-shot prompting" and it's powerful — but it's not training.

### "The model understands what I mean"

Sort of. The model is very good at predicting what a helpful response to your input would look like, based on its training data. Whether that constitutes "understanding" is a philosophical debate. What matters for engineering: treat it as a sophisticated pattern matcher, not a thinking agent.

### "Bigger models are always better"

No. Bigger models are better at *hard tasks* — complex reasoning, nuanced instructions, subtle distinctions. For simple tasks (classification, extraction, formatting), smaller models are often just as good and much cheaper. Always try the smallest model first.

### "Temperature 0 gives the same answer every time"

Mostly, but not perfectly. Even at temperature 0, floating-point arithmetic on GPUs can introduce tiny variations. For true determinism, some APIs offer a `seed` parameter. But even then, model updates on the provider's side can change outputs.

### "More tokens in the context = better results"

Not necessarily. The "lost in the middle" problem means models pay less attention to information in the middle of long contexts. A well-curated 2,000-token context often outperforms a carelessly assembled 50,000-token context. Quality over quantity.

---

## 9. Key Takeaways

1. **Tokens are the currency.** Everything — cost, speed, context — is measured in tokens. ~4 characters per token in English.

2. **Context windows are the constraint.** Every model has a maximum context size. Your system prompt, conversation history, retrieved documents, and the response all must fit.

3. **Temperature controls randomness.** Low (0-0.3) for deterministic/factual tasks, medium (0.5-0.8) for conversation, high (0.8-1.5) for creative work.

4. **Hallucination is inherent.** LLMs predict tokens, they don't retrieve facts. Design your system to handle wrong answers.

5. **Training happened already.** You're doing inference. The model doesn't learn from your conversations.

6. **Start with the smallest sufficient model.** GPT-4o-mini or Claude Haiku before reaching for Opus.

---

## What's Next

In **Chapter 1: Embeddings & Similarity**, we'll explore how LLMs represent meaning as numbers — the mathematical foundation that enables semantic search, recommendations, and RAG. It's still conceptual, but it's the concept that unlocks most of what you'll build in Parts 2-4.

Then in **Chapter 2: The AI Engineer's Landscape**, we'll survey the providers, pricing, and SDKs — everything you need to know before writing your first line of code in Part 1.

And in **Chapter 3**, we finally write code. The concepts from this chapter — tokens, context windows, temperature — become parameters in real API calls.

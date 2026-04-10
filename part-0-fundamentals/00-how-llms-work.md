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

### 1.2 Tokenization in Practice: Concrete Examples

Let's look at some real tokenization examples to build intuition. These are from GPT-4o's tokenizer (cl100k_base/o200k_base), but other models produce similar results:

**Common English words** are usually a single token:
```
"hello"       → ["hello"]        → 1 token
"the"         → ["the"]          → 1 token
"computer"    → ["computer"]     → 1 token
"because"     → ["because"]      → 1 token
```

**Less common or longer words** get split into subwords:
```
"tokenization"   → ["token", "ization"]           → 2 tokens
"embedding"      → ["embed", "ding"]              → 2 tokens
"unprecedented"  → ["un", "precede", "nted"]      → 3 tokens
"hallucination"  → ["hall", "ucin", "ation"]      → 3 tokens
```

**Programming terms and code** have their own patterns:
```
"getElementById"  → ["get", "Element", "By", "Id"]   → 4 tokens
"async function"  → ["async", " function"]            → 2 tokens
"console.log"     → ["console", ".", "log"]            → 3 tokens
"npm install"     → ["npm", " install"]               → 2 tokens
```

**Non-English text** uses more tokens because the tokenizer was trained primarily on English:
```
"Bonjour"     → ["Bon", "jour"]          → 2 tokens (French)
"Guten Tag"   → ["G", "uten", " Tag"]    → 3 tokens (German)
"こんにちは"    → ["こん", "にち", "は"]     → 3 tokens (Japanese)
"مرحبا"       → 5+ tokens                → (Arabic uses many tokens per word)
```

This has a real cost implication: if your application serves non-English users, the same conversation costs more in tokens. A Japanese conversation might use 2-3x the tokens of the same conversation in English.

**Numbers and special characters** can be surprisingly expensive:
```
"12345"           → ["123", "45"]          → 2 tokens
"2026-04-10"      → ["202", "6", "-", "04", "-", "10"]  → 6 tokens
"$1,234.56"       → ["$", "1", ",", "234", ".", "56"]   → 6 tokens
"user@email.com"  → ["user", "@", "email", ".", "com"]  → 5 tokens
```

Notice how a simple date takes 6 tokens. If you're processing thousands of records with dates, this adds up quickly.

### 1.3 Why Tokens Matter for You

Tokens matter for three practical reasons:

**Cost.** Every API call is billed by tokens. When OpenAI charges "$2.50 per million input tokens" for GPT-4o, they mean it literally. A 1,000-word prompt is roughly 1,300 tokens. A back-and-forth conversation with 50 messages might be 15,000+ tokens. Understanding tokenization helps you estimate and control costs.

**Speed.** LLMs generate tokens sequentially — one at a time. A response of 500 tokens takes roughly 5x longer to generate than a response of 100 tokens. When you ask for verbose output, you're paying in both money and latency.

**Context limits.** Every model has a maximum number of tokens it can process in a single request (input + output combined). Understanding tokens helps you know when you're approaching that limit.

### 1.4 The Token Cost Calculator

Let's walk through what real usage actually costs. This is the exercise most tutorials skip, and it's the one that will save you from a surprise bill.

**Scenario 1: A simple chatbot (10 turns with GPT-4o)**
```
System prompt:           ~500 tokens
User messages (10):      ~1,000 tokens (100 per message)
Assistant responses (10): ~3,000 tokens (300 per response)
Accumulated history:     ~4,200 input tokens (context grows each turn)
                         ~3,000 output tokens

Cost per conversation:
  Input:  4,200 tokens × $2.50/1M  = $0.0105
  Output: 3,000 tokens × $10.00/1M = $0.03
  Total: ~$0.04 per conversation
```

That's 4 cents. Sounds cheap. Now multiply:
```
100 conversations/day   → $4/day    → $120/month
1,000 conversations/day → $40/day   → $1,200/month
10,000 conversations/day → $400/day → $12,000/month
```

**Scenario 2: Same chatbot with GPT-4o-mini**
```
  Input:  4,200 tokens × $0.15/1M  = $0.00063
  Output: 3,000 tokens × $0.60/1M  = $0.0018
  Total: ~$0.002 per conversation

  10,000 conversations/day → $20/day → $600/month
```

That's **20x cheaper** for tasks where GPT-4o-mini's quality is sufficient. This is why model selection matters so much (more in Ch 2).

**Scenario 3: A RAG-powered knowledge base assistant**
```
System prompt:           ~1,000 tokens
Retrieved documents:     ~4,000 tokens (4 chunks × 1,000 tokens each)
User question:           ~100 tokens
Assistant response:      ~500 tokens

Cost per query with Claude Sonnet 4:
  Input:  5,100 tokens × $3.00/1M  = $0.0153
  Output: 500 tokens × $15.00/1M   = $0.0075
  Total: ~$0.023 per query
  
  50,000 queries/month = ~$1,150/month
```

**Scenario 4: Batch processing 10,000 product descriptions**
```
Each description: ~200 input tokens, ~300 output tokens

With GPT-4o:
  Input:  2M tokens × $2.50/1M  = $5.00
  Output: 3M tokens × $10.00/1M = $30.00
  Total: $35 for the batch

With GPT-4o-mini:
  Input:  2M tokens × $0.15/1M  = $0.30
  Output: 3M tokens × $0.60/1M  = $1.80
  Total: $2.10 for the batch
```

The takeaway: **always estimate costs before committing to a model.** A prototype that costs $5/day can become a $15,000/month production cost at scale if you don't plan.

### 1.5 How Tokenization Works (Conceptual)

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

### 1.6 Token Counting in Practice

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

**Quality degrades with distance — the "lost in the middle" problem.** Research consistently shows that LLMs pay more attention to the beginning and end of the context window than the middle. This has been studied extensively (the original "Lost in the Middle" paper by Liu et al. demonstrated this clearly), and it's one of the most important practical considerations for AI engineers.

Imagine you're giving someone a stack of 20 documents and asking them a question. They'll remember the first few documents well (primacy effect), and the last few documents well (recency effect), but the ones in the middle tend to blur together. LLMs have the same bias, even though they process text differently than humans.

```
Context window attention pattern:

█████████  Strong attention (beginning of context)
████████
███████
██████
█████
████       ← Weak attention (middle of context — "lost in the middle")
████
████
█████
██████
███████
████████
█████████  Strong attention (end of context)
```

This has direct implications for how you structure prompts and RAG pipelines:

- **Put the most important information at the beginning or end** of your context, not the middle
- **Your system prompt (beginning) and the user's latest message (end)** get the most attention — that's good
- **Retrieved documents stuffed in the middle** get less attention — this means RAG quality depends on where you place retrieved content, not just what you retrieve
- **If you're sending 10 documents as context, the model might "forget" documents 4-7** even though they're right there in the input
- **Shorter, more focused context often beats longer, comprehensive context** because there's less opportunity for important information to get lost

This is why "just make the context window bigger" isn't a silver bullet. A 1M-token context window doesn't mean the model uses all 1M tokens equally well. We'll cover strategies for this in Ch 12, including placing retrieved content strategically and using re-ranking to ensure the best content gets the best positions.

**Speed decreases with context length.** Longer contexts take longer to process. The first token of a response takes longer to arrive (higher "time to first token"), and the overall request takes longer. The relationship isn't perfectly linear — there's a quadratic component in the attention mechanism — but for practical purposes, doubling your context roughly doubles your wait time.

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

The best analogy: **an LLM is a confident bullshitter.** Not in a malicious way — but in the way that a very articulate person who has read millions of documents will always produce a fluent, confident-sounding answer, even when they're unsure or flat-out wrong. They've internalized the *patterns* of what correct answers look like, so their wrong answers look just as polished as their right ones. There's no internal "I'm not sure about this" flag that changes the tone of the output.

This is fundamentally different from a search engine or database, which either returns a result or says "no results found." An LLM always generates *something*. It has no concept of "I don't have this information" built into its architecture — that behavior has to be trained in after the fact (through RLHF), and it's never 100% reliable.

This works great for things that appeared often in training data. It falls apart for:

- **Rare facts**: Things that appeared infrequently in training. If a fact appeared 10,000 times in training data, the model probably gets it right. If it appeared 3 times, it might mix it up with similar-sounding facts.
- **Recent events**: Anything after the training data cutoff. The model doesn't know it doesn't know.
- **Specific numbers**: Exact dates, statistics, measurements, phone numbers, URLs. Numbers are particularly hard because the model can't "reason" about whether 1648 vs 1654 is correct — both are plausible tokens.
- **Niche domains**: Specialized knowledge with little training representation. Medical dosages, legal statute numbers, obscure API parameters.
- **Logical reasoning**: Multi-step deduction where each step must be correct. The model is pattern-matching what reasoning "looks like," which works surprisingly well for 2-3 steps but degrades with complexity.
- **Combining real facts in wrong ways**: The model might know Fact A and Fact B independently, but combine them incorrectly — e.g., attributing the right quote to the wrong person, or the right statistic to the wrong year.

### 4.2 Types of Hallucination

Understanding the different types helps you design appropriate safeguards:

**Confident fabrication**: The model generates false information with the same confidence as true information. "The Treaty of Westphalia was signed in 1648" (true) sounds exactly like "The Treaty of Westphalia was signed in 1654" (false). There's no uncertainty marker in the output. The tokens are generated with the same fluency either way.

**Plausible but wrong**: The model generates something that *sounds* right and follows the right patterns but is factually incorrect. "The Eiffel Tower was designed by Alexandre-Gustave Eiffel and completed in 1887" — it was completed in 1889. This is particularly dangerous because a human reviewer might not catch it. The error is close enough to the truth that it passes a casual sniff test.

**Entity confusion**: The model mixes up attributes of similar entities. It might give you the population of Houston when you asked about Dallas, or describe the features of React Router when you asked about Next.js routing. The entities are in the same "neighborhood" of the model's learned representations, and it confuses their attributes. You'll see this especially with:
- Similar products or libraries in the same category
- People with similar names or roles
- Cities in the same region
- Companies in the same industry

**Citation fabrication**: Ask the model for academic sources and it will generate papers with realistic titles, plausible author names, and convincing journal names — for papers that don't exist. The format is right. The content is invented. This happens because the model has learned the *pattern* of academic citations (Author, Year, "Title in Quotes," *Journal Name*, Volume(Issue), pages) and can generate that pattern fluently. A lawyer famously submitted a brief with ChatGPT-fabricated case citations — none of the cited cases existed.

**Reasoning errors (logical hallucination)**: The model generates step-by-step reasoning that looks logical but contains a subtle error in one step, leading to a wrong conclusion. It's pattern-matching what reasoning "looks like" without actually reasoning. This is especially common in:
- Multi-step math problems (carries an error through subsequent steps)
- Logical deductions with more than 3-4 steps
- Code that looks syntactically correct but has a logic bug
- Analyses that follow a sound structure but apply a flawed assumption

**Temporal confusion**: The model conflates information from different time periods. It might describe a company's current product lineup using features from 2 years ago, or mix up the sequence of historical events. Training data from different eras all looks the same to the model — there's no strong "timestamp" attached to each fact.

**Over-generalization**: The model makes a broad claim based on a pattern it learned, even when it doesn't apply to the specific case. "Python is slower than Go" is generally true but wrong for specific workloads. The model has learned the generalization and may apply it inappropriately when asked about a specific case.

### 4.3 Why Hallucination Rates Vary

Not all tasks are equally prone to hallucination. Understanding the risk profile helps you design appropriate safeguards:

```
Hallucination risk spectrum:

LOW RISK (model is parroting well-known patterns):
  ✓ Summarizing a document you provided
  ✓ Translating common phrases
  ✓ Formatting/restructuring text
  ✓ Classifying text into categories you defined
  ✓ Answering questions about text in the context window

MEDIUM RISK (model is combining learned patterns):
  ⚠ Writing code (may use non-existent APIs or wrong syntax)
  ⚠ Answering factual questions from memory
  ⚠ Explaining technical concepts
  ⚠ Generating recommendations

HIGH RISK (model is generating novel claims):
  ✗ Citing specific sources, URLs, or papers
  ✗ Providing exact numbers, dates, or statistics
  ✗ Making claims about specific people or organizations
  ✗ Legal, medical, or financial advice
  ✗ Describing events after the training cutoff
```

The principle: the more the task relies on the model's "memory" (compressed training data) vs. the information you provide in the context, the higher the hallucination risk. This is why RAG is so important — it shifts tasks from "rely on memory" to "rely on provided documents."

### 4.4 Why You Can't Fully Eliminate Hallucination

Hallucination isn't a bug — it's a fundamental property of how these models work. The same mechanism that lets an LLM generate creative, coherent, contextually appropriate text is the mechanism that produces hallucinations.

Think about it: if the model could ONLY output things it "knew" with certainty, it couldn't do most of what makes it useful. It couldn't rephrase, summarize, translate, write code for your specific use case, or have a conversation. All of those require generating *new* text that wasn't in the training data.

**What you can do:**
1. **Reduce hallucination** with better prompts (Ch 4) — be specific, provide examples, constrain the output format
2. **Ground responses** in retrieved documents (RAG — Ch 14-17) — give the model the facts, don't rely on its memory
3. **Verify outputs** with structured schemas (Ch 5) — at least ensure the format is correct, even if the content needs checking
4. **Detect hallucination** with evaluations (Ch 18-22) — build automated checks that catch common failure modes
5. **Set expectations** in your UI — don't present LLM output as authoritative fact. Show sources. Add disclaimers.
6. **Use reasoning models** for complex tasks — o3, o4-mini, and Claude with extended thinking hallucinate less on reasoning tasks because they "think through" the answer before committing

### 4.5 The Practical Takeaway

Never trust LLM output as fact. Always design your system assuming the model might be wrong:

- **High-stakes decisions?** Require human review.
- **Factual claims?** Ground them in retrieved documents and cite sources.
- **Structured data?** Validate against schemas.
- **User-facing text?** Include disclaimers where appropriate.
- **Code generation?** Run tests.

The models that hallucinate less aren't "smarter" — they've been trained with more reinforcement learning from human feedback (RLHF) to say "I don't know" more often. But even the best models hallucinate. Design for it.

### 4.6 A Framework for Hallucination-Aware Design

When building any AI feature, ask these questions upfront:

```
1. What's the COST of a hallucination in this feature?
   ├─ Low:  chatbot gives a slightly wrong fun fact  → Accept the risk, add a disclaimer
   ├─ Medium: support bot gives wrong instructions   → Ground in docs, add human escalation
   └─ High: medical/legal/financial advice is wrong  → Require human review, multiple validation layers

2. What's my VERIFICATION strategy?
   ├─ Can I validate the output programmatically? (schemas, regex, type checking)
   ├─ Can I compare against a known source? (RAG, database lookup)
   ├─ Do I need human review? (approval workflow, flagging)
   └─ Can I detect WHEN the model is likely wrong? (confidence scoring, eval patterns)

3. What happens when the model IS wrong?
   ├─ Does the user see it directly? → Add disclaimers, show sources
   ├─ Does it feed into another system? → Add validation before passing downstream
   └─ Is it a suggestion vs a decision? → Frame as suggestion, keep human in the loop
```

This framework is something you'll apply in every chapter from here on. Ch 4 (prompting) reduces hallucination at the input level. Ch 5 (structured output) catches it at the output level. Ch 14-17 (RAG) grounds it in real data. Ch 18-22 (evals) detects it systematically.

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

This is one of the most common misunderstandings we see from non-technical stakeholders. "We've been using Claude for 6 months — hasn't it learned our company's style by now?" No. Every single API call starts from the same pre-trained weights. What looks like "learning" is either prompt engineering (you've gotten better at instructing it) or context (you're now including relevant documents in the prompt).

**Training data has a cutoff.** The model only "knows" things from its training data. Claude's training data has a cutoff; GPT-4's has a cutoff. If something happened after the cutoff, the model doesn't know about it (unless you put it in the context).

This creates a practical problem: your model doesn't know about your latest product release, your competitor's new feature, or last week's industry news. This is exactly why RAG (Retrieval Augmented Generation) exists — to feed the model current, relevant information at inference time. We build RAG pipelines in Ch 14-17.

**You probably won't train models.** Training a model from scratch costs millions of dollars in compute. Fine-tuning (adjusting a pre-trained model on your data) is more accessible but still requires expertise and thousands of training examples. For most AI engineering tasks, you'll use pre-trained models via APIs and control their behavior through prompting, context, and retrieval.

Here's the breakdown of what each approach costs in practice:

```
From-scratch training:    $10M-$100M+   (months, requires ML research team)
Full fine-tuning:         $500-$50,000  (hours-days, requires ML engineering)
LoRA/QLoRA fine-tuning:   $50-$500      (hours, more accessible)
Prompt engineering + RAG:  $0-$100       (minutes-days, any developer)
```

The vast majority of AI features in production today use the bottom option. Start there.

### 5.3 The Training Pipeline (High Level)

For your mental model, here's what happens before you get access to a model:

1. **Pre-training**: The model learns to predict the next token on trillions of tokens of internet text. This gives it language understanding, general knowledge, and emergent abilities like reasoning and code generation.

2. **Supervised fine-tuning (SFT)**: The model is trained on curated examples of helpful, well-formatted responses. This teaches it to follow instructions and respond in a conversational format.

3. **RLHF / RLAIF (Reinforcement Learning from Human/AI Feedback)**: Humans (or AI systems) rate model outputs. The model is trained to produce outputs that get higher ratings. This is what makes models "aligned" — helpful, harmless, and honest.

4. **Safety training**: Additional training focused on refusing harmful requests, avoiding biases, and following safety guidelines.

The result is a model that can follow instructions, maintain a conversation, refuse harmful requests, and generate helpful outputs — all properties that emerged from training, not from the architecture itself.

**A useful analogy:** Pre-training is like giving someone a broad education — they read millions of books and can talk about anything. SFT is like teaching them a specific job — "when someone asks you a question, respond in this format." RLHF is like performance reviews — "this response was good, this one wasn't, here's why." Safety training is like compliance training — "never do these things, no matter what."

The model you interact with through the API is the end result of all four stages. When it seems "smart," that's mostly pre-training. When it follows your instructions well, that's SFT. When it refuses to help with harmful requests, that's safety training. When it's generally helpful and not annoying, that's RLHF.

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

### 6.2 Comparison Table: Models at a Glance

Here's a practical comparison of the major models as of early 2026, focusing on what matters for engineering decisions:

| Model | Context | Input $/1M | Output $/1M | Best For | Weakest At |
|---|---|---|---|---|---|
| **GPT-4o** | 128K | $2.50 | $10.00 | General purpose, code, broad tool support | Can be verbose, mid-range pricing |
| **GPT-4o-mini** | 128K | $0.15 | $0.60 | High-volume simple tasks, prototyping | Complex reasoning, nuanced instructions |
| **o3** | 200K | $10.00 | $40.00 | Complex math, planning, hard reasoning | Speed (very slow), cost |
| **o4-mini** | 200K | $1.10 | $4.40 | Affordable reasoning tasks | Less capable than full o3 |
| **Claude Opus 4** | 200K | $15.00 | $75.00 | Hardest tasks, extended thinking, precision | Expensive, slower |
| **Claude Sonnet 4** | 200K | $3.00 | $15.00 | Instruction following, writing, balanced quality | Smaller ecosystem than OpenAI |
| **Claude 3.5 Haiku** | 200K | $0.80 | $4.00 | Fast classification, extraction, triage | Complex multi-step reasoning |
| **Gemini 2.5 Pro** | 1M | $1.25 | $10.00 | Huge context tasks, multimodal | API ergonomics, tool calling maturity |
| **Gemini 2.0 Flash** | 1M | $0.10 | $0.40 | Cheapest option, high throughput | Quality on hard tasks |
| **Llama 4 Scout** | 10M* | Free** | Free** | Self-hosting, privacy, fine-tuning | Requires infrastructure, lower quality |
| **Mistral Large** | 128K | $2.00 | $6.00 | Multilingual, European data residency | Smaller ecosystem |

*\*Llama 4 Scout's 10M context is theoretical; practical limits depend on hardware.*
*\*\*Llama models are free to download but require GPU infrastructure to run.*

**How to read this table:** Start from the rightmost columns. If the "Weakest At" column describes your use case, skip that model. If the "Best For" column matches your use case, that's your first candidate.

### 6.3 How to Choose (Quick Decision Tree)

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

### 6.4 Model Size and Quality

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

### 6.5 Open Source vs Closed Source

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

### 7.1 A Day-in-the-Life Example

To make this concrete, here's what happens when a user interacts with a customer support chatbot you've built:

```
User opens the chat widget on your website.

Turn 1:
  Your app sends to the API:
    System prompt: "You are a support agent for Acme Corp. Use the provided 
                    knowledge base articles to answer questions..." (800 tokens)
    User message: "How do I reset my password?" (8 tokens)
    Total input: 808 tokens
  
  Model generates: "To reset your password, go to Settings > Account > 
                     Reset Password..." (120 tokens)
  
  Cost: (808 × $2.50/1M) + (120 × $10.00/1M) = $0.003
  Latency: ~0.8 seconds

Turn 5:
  Your app sends:
    System prompt: 800 tokens (same as before — repeated every turn)
    Turns 1-4 history: ~1,200 tokens
    User message: "What if I forgot my email too?" (9 tokens)
    Total input: 2,009 tokens
  
  Model generates: "If you've also forgotten your email..." (150 tokens)
  
  Cost: (2,009 × $2.50/1M) + (150 × $10.00/1M) = $0.007
  Latency: ~1.2 seconds

Turn 20:
  Total input: ~8,000 tokens (context has grown significantly)
  Cost per turn: ~$0.025
  Cumulative conversation cost: ~$0.25
  Latency: ~2.5 seconds (noticeably slower)
```

Notice the pattern: each turn gets more expensive and slower because the entire conversation history is re-sent. This is why context window management (Ch 12) is so important — and why strategies like summarizing old messages can dramatically reduce costs.

---

## 8. Common Misconceptions

Let's clear up things that trip up new AI engineers. These are some of the most common mistakes people make when they start building with LLMs, and each one can lead to real bugs, wasted money, or bad architecture decisions.

### "The model remembers our conversation"

No. Each API call is completely stateless. The model has no memory between requests. It doesn't know who you are, what you asked 5 minutes ago, or what it said in response. Every single API call starts from scratch.

"Memory" in a chatbot is maintained by YOU — by sending the entire conversation history with each request. When you see ChatGPT "remembering" your conversation, it's because the client is re-sending all previous messages as input tokens every time. This is why conversations get expensive: you're re-sending everything, and the cost grows with every turn.

This has a subtle engineering implication: if your app crashes and you lose the conversation history, the model can't help you recover it. The model never stored it. YOU are the memory layer.

### "I trained the model by giving it examples"

No. You added examples to the context window. The model's weights didn't change. Those examples influence *this request only*. This is called "in-context learning" or "few-shot prompting" and it's powerful — but it's not training.

Here's the test: close your session and start a new one. Does the model remember those examples? No. Because no training happened. You were renting a few thousand tokens of attention, not teaching the model anything permanent.

Actual training (fine-tuning) is a separate process that changes the model's weights, requires hundreds or thousands of examples, takes minutes to hours, and produces a new model checkpoint. We cover it in Ch 44-47.

### "The model understands what I mean"

Sort of. The model is very good at predicting what a helpful response to your input would look like, based on its training data. Whether that constitutes "understanding" is a philosophical debate. What matters for engineering: treat it as a sophisticated pattern matcher, not a thinking agent.

Here's a practical test: if you misspell a crucial word or use ambiguous phrasing, the model might "understand" what you meant (because it can pattern-match past the error) or it might silently produce the wrong thing (because it pattern-matched to something else). It doesn't "understand" your intent — it predicts the most likely completion. Those are often the same thing, but not always.

### "The model knows things"

This is the most dangerous misconception. LLMs don't "know" things the way a database stores records. They've learned statistical patterns from training data that let them generate text that looks like it comes from someone who knows things. This is a crucial distinction.

A database either has a record or it doesn't. An LLM always generates *something*. It can't tell you "I don't have that information in my parameters" because it doesn't have a lookup mechanism — it has a generation mechanism. When the model says "I don't know," that's because it was trained (via RLHF) to recognize uncertainty patterns and generate disclaimers. It's a learned behavior, not an intrinsic capability.

### "The model thinks step by step"

When you see an LLM write "Let me think about this step by step..." it's generating tokens that pattern-match to step-by-step reasoning in its training data. For standard models, there's no separate "thinking" process — just token prediction. The reasoning models (o3, o4-mini, Claude with extended thinking) do have a separate chain-of-thought phase, but even then, it's structured token generation, not human-like thought.

The practical implication: asking the model to "think step by step" often improves results (this is called chain-of-thought prompting), not because the model literally starts thinking harder, but because generating intermediate reasoning tokens gives the model more "working memory" to arrive at a better final answer. We cover this technique in Ch 4.

### "Bigger models are always better"

No. Bigger models are better at *hard tasks* — complex reasoning, nuanced instructions, subtle distinctions. For simple tasks (classification, extraction, formatting), smaller models are often just as good and much cheaper. Always try the smallest model first.

We've seen teams burn thousands of dollars per month using Claude Opus 4 ($75/1M output tokens) for tasks that Claude 3.5 Haiku ($4/1M output tokens) handles at the same quality. That's an 18x cost difference for zero quality improvement. Always benchmark.

### "Temperature 0 gives the same answer every time"

Mostly, but not perfectly. Even at temperature 0, floating-point arithmetic on GPUs can introduce tiny variations. For true determinism, some APIs offer a `seed` parameter. But even then, model updates on the provider's side can change outputs. And if you're using streaming, the batching behavior on the server can affect the exact computation path.

If your system requires exactly identical outputs for identical inputs, you need a caching layer, not temperature 0.

### "More tokens in the context = better results"

Not necessarily. The "lost in the middle" problem (Section 2.3) means models pay less attention to information in the middle of long contexts. A well-curated 2,000-token context often outperforms a carelessly assembled 50,000-token context. Quality over quantity.

Think of it like giving a research paper to a colleague. Handing them the 3 most relevant paragraphs is more useful than handing them the entire 50-page document and saying "the answer is in here somewhere."

### "The API responses are private and not used for training"

This depends on the provider and your agreement. Most providers have specific policies:
- **OpenAI**: Data from API calls is NOT used for training by default (as of March 2023). But data from ChatGPT may be, unless you opt out.
- **Anthropic**: API data is not used for training by default.
- **Google**: Check your specific agreement — enterprise vs consumer terms differ.

Always read the data usage policy for your provider, especially if you're handling sensitive data. When in doubt, use the enterprise tier, which typically has stronger data protection guarantees.

---

## 9. Key Takeaways

1. **Tokens are the currency.** Everything — cost, speed, context — is measured in tokens. ~4 characters per token in English. Always estimate token costs before committing to a model at scale.

2. **Context windows are the constraint.** Every model has a maximum context size. Your system prompt, conversation history, retrieved documents, and the response all must fit. And even within the window, the model pays less attention to content in the middle ("lost in the middle").

3. **Temperature controls randomness.** Low (0-0.3) for deterministic/factual tasks, medium (0.5-0.8) for conversation, high (0.8-1.5) for creative work. Use one of temperature or top-p, not both.

4. **Hallucination is inherent.** LLMs predict tokens, they don't retrieve facts. Design your system to handle wrong answers. The risk varies by task — grounding in retrieved documents dramatically reduces hallucination.

5. **Training happened already.** You're doing inference. The model doesn't learn from your conversations. Each API call is stateless. You are the memory layer.

6. **Start with the smallest sufficient model.** GPT-4o-mini or Claude Haiku before reaching for Opus. The cost difference can be 20x+ with negligible quality difference for simple tasks.

7. **The model doesn't "know" things.** It predicts plausible-sounding text. Treat it as a sophisticated pattern matcher, not a knowledge base. Never trust LLM output as fact without verification.

---

---

## 10. Quick Reference Card

Here's a cheat sheet you can come back to when you need to remember the key numbers:

```
TOKENS
  1 token ≈ 4 characters (English)
  1 token ≈ 0.75 words (English)
  1 page of text ≈ 500 tokens
  Common word = 1 token, uncommon word = 2-4 tokens

CONTEXT WINDOWS (2026)
  GPT-4o:            128K tokens (~200 pages)
  Claude Opus/Sonnet: 200K tokens (~300 pages)
  Gemini 2.0/2.5:   1M tokens (~1,500 pages)

TEMPERATURE
  0:     Deterministic (data extraction, classification)
  0.3-0.7: Balanced (conversation, general use)
  0.8-1.5: Creative (brainstorming, writing)

COSTS (per 1M tokens, as of early 2026)
  Cheap:    GPT-4o-mini ($0.15/$0.60), Gemini Flash ($0.10/$0.40)
  Standard: GPT-4o ($2.50/$10), Claude Sonnet 4 ($3/$15)
  Premium:  o3 ($10/$40), Claude Opus 4 ($15/$75)
```

---

## What's Next

In **Chapter 1: Embeddings & Similarity**, we'll explore how LLMs represent meaning as numbers — the mathematical foundation that enables semantic search, recommendations, and RAG. It's still conceptual, but it's the concept that unlocks most of what you'll build in Parts 2-4.

Then in **Chapter 2: The AI Engineer's Landscape**, we'll survey the providers, pricing, and SDKs — everything you need to know before writing your first line of code in Part 1.

And in **Chapter 3**, we finally write code. The concepts from this chapter — tokens, context windows, temperature — become parameters in real API calls.

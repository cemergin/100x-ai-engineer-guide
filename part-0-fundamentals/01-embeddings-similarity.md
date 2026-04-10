<!--
  CHAPTER: 1
  TITLE: Embeddings & Similarity
  PART: 0 — LLM Fundamentals
  PHASE: 1 — Get Dangerous
  PREREQS: Chapter 0
  KEY_TOPICS: embeddings, vectors, cosine similarity, semantic meaning, vector spaces, similarity search, word2vec
  DIFFICULTY: Beginner
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 1: Embeddings & Similarity

> **Part 0 — LLM Fundamentals** | Phase 1: Get Dangerous | Prerequisites: Chapter 0 | Difficulty: Beginner | Language: TypeScript

Here's a problem that sounds impossible: how do you teach a computer that "dog" and "puppy" are related, but "dog" and "democracy" aren't? You can't just compare the letters — "dog" and "fog" share two letters but have nothing in common semantically. You can't use a dictionary — it would tell you both "dog" and "canine" are nouns, but that's not enough.

The answer is one of the most beautiful ideas in modern AI: **embeddings**. Take a word, sentence, or entire document, and represent it as a list of numbers — a **vector** — where the numbers capture the *meaning* of the text. Similar meanings produce similar numbers. And once meaning is numbers, you can do math on it.

This chapter is conceptual. We won't write code yet — that comes in Chapter 14 when we build semantic search. But embeddings are the foundation for so many things you'll build (search, RAG, recommendations, clustering, anomaly detection) that you need the mental model first.

### In This Chapter
- What embeddings are and how they represent meaning
- Vector spaces: the geometry of meaning
- Why "king - man + woman = queen" works
- Cosine similarity: measuring how alike two things are
- What embeddings enable: search, RAG, recommendations
- Limitations and gotchas

### Related Chapters
- **Ch 14 (Semantic Search)** — spirals embeddings into working code: embed text, store vectors, search
- **Ch 36 (Neural Networks from Scratch)** — spirals to expert depth: how neural nets produce embeddings via learned weights
- **Ch 42 (Embeddings & Sentence Transformers)** — spirals to open-source: generate your own embeddings, choose models
- **Ch 44 (RAG vs Fine-Tuning)** — when to retrieve with embeddings vs bake knowledge into weights

---

## 1. What Are Embeddings?

### 1.1 The Core Idea

An **embedding** is a representation of something (a word, sentence, image, anything) as a list of numbers — a **vector**. But not just any list of numbers. The numbers are arranged so that **similar things have similar numbers**.

Here's the best analogy: imagine a library where books aren't organized alphabetically or by author, but **by topic and meaning**. A book about dog training sits next to a book about puppy behavior, which is near a book about animal psychology, which is in the same wing as veterinary science. A book about car repair is in a completely different part of the building. You don't need to know the exact title or author — you just walk to the right area and everything nearby is relevant.

Embeddings organize text the same way, but in a mathematical space instead of a physical building. Every piece of text gets assigned a "location" (a vector of numbers), and similar texts are placed near each other.

Let's start with a toy example. Imagine we could represent words with just 2 numbers:

```
"cat"    → [0.9, 0.8]    (high animality, high cuteness)
"dog"    → [0.85, 0.75]   (high animality, slightly less cuteness... arguable)
"fish"   → [0.7, 0.4]    (moderate animality, less cuteness)
"car"    → [0.1, 0.3]    (low animality, low cuteness)
"truck"  → [0.05, 0.2]   (very low animality, very low cuteness)
```

If you plot these on a graph, "cat" and "dog" would be close together. "Car" and "truck" would be close together. The animals would be far from the vehicles. The *distance between points represents semantic similarity*.

### 1.2 From Two Dimensions to Thousands

That toy example used 2 dimensions with hand-picked meanings (animality, cuteness). Real embeddings work with **hundreds or thousands of dimensions** — and the dimensions don't have human-interpretable meanings.

Why so many dimensions? Because language is incredibly complex. Two dimensions can distinguish between "animal" and "vehicle," but they can't simultaneously capture:
- Is this formal or casual?
- Is it positive or negative?
- Is it about technology or cooking?
- Is it a question or a statement?
- Is it about the past or the future?
- Is it literal or metaphorical?
- ...and thousands more subtle distinctions

Each dimension adds another "axis" along which meaning can vary. With 1,536 dimensions, the model can capture incredibly fine-grained distinctions — the difference between "machine learning" and "deep learning," or between "I need help" (customer support tone) and "I need help" (emergency tone).

Think of it this way: you can describe a color with 3 numbers (red, green, blue). But to describe a person's face, you need thousands of numbers. Language is more like faces than colors — it has immense dimensionality.

Typical embedding sizes:

| Model | Dimensions | What It Means |
|---|---|---|
| OpenAI text-embedding-3-small | 1,536 | Each text gets a vector of 1,536 numbers |
| OpenAI text-embedding-3-large | 3,072 | 3,072 numbers per text |
| Cohere embed-v3 | 1,024 | 1,024 numbers |
| Sentence Transformers (all-MiniLM) | 384 | 384 numbers |
| Google text-embedding-004 | 768 | 768 numbers |

A 1,536-dimensional embedding means each piece of text is represented by 1,536 floating-point numbers. You can't visualize 1,536 dimensions, but the math works the same as in 2D: similar texts produce vectors that are "close" to each other.

### 1.3 What the Numbers Capture

Nobody hand-designed what each dimension means. The embedding model **learned** the dimensions during training, adjusting them to be useful for understanding language. Some dimensions might loosely correspond to concepts like:

- Formality vs casualness
- Positive vs negative sentiment
- Concrete vs abstract
- Past vs present
- Technical vs non-technical

But most dimensions capture subtle, entangled patterns that resist human interpretation. A single dimension might capture a combination of "scientific papers about genetics published after 2010 that mention CRISPR." The model learned this was useful even though no human would think to create that dimension.

To make this more concrete, imagine a simplified 5-dimensional embedding where the dimensions DO have interpretable meanings (real embeddings don't, but this helps build intuition):

```
Dimension meanings (hypothetical):
  [0] = formality     (0 = casual, 1 = formal)
  [1] = sentiment     (0 = negative, 1 = positive)
  [2] = technicality  (0 = non-technical, 1 = technical)
  [3] = urgency       (0 = relaxed, 1 = urgent)
  [4] = specificity   (0 = vague, 1 = specific)

"Hey, the server's down!!"              → [0.1, 0.1, 0.7, 0.95, 0.8]
"We are experiencing a service outage"  → [0.9, 0.2, 0.6, 0.7, 0.7]
"Everything is working great today"     → [0.3, 0.9, 0.3, 0.1, 0.4]
"The deployment was successful"         → [0.6, 0.8, 0.8, 0.2, 0.7]
```

The first two sentences would be "close" in this space — both are about system outages, even though they use completely different words and tones. The third and fourth sentences would be closer to each other (both positive) but far from the first two (which are negative/urgent).

Now multiply this by 1,536 dimensions, and you start to see the power: the embedding captures a rich, nuanced fingerprint of meaning that no keyword search could replicate.

**The key insight:** You don't need to understand what each dimension means. You just need to know that the resulting vectors capture semantic similarity reliably. Similar texts produce similar vectors. Different texts produce different vectors.

---

## 2. Vector Spaces: The Geometry of Meaning

### 2.1 Thinking in Spaces

When you represent text as vectors, you're placing that text in a **vector space** — a mathematical space where each piece of text occupies a point, and the geometric relationships between points are meaningful.

```
Imagine a simplified 3D space:

                    ^ Dimension 2 (maybe "living things")
                    |
              cat • | • dog
                    |
          fish •    |
                    |
        ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ─ → Dimension 1 (maybe "size")
                    |
              car • |
                    |
            truck • |
                    |
                    v Dimension 3 (below the plane)
```

In this space:
- **Clusters** form naturally: animals cluster together, vehicles cluster together
- **Distances** are meaningful: cat-to-dog is a short distance, cat-to-truck is long
- **Directions** are meaningful: moving from "cat" toward "dog" is moving in the direction of "more canine"
- **Regions** have meaning: one area of the space might correspond to "food," another to "technology"

### 2.2 The Library Analogy (Extended)

Let's extend the library analogy from Section 1.1 to understand vector spaces more deeply.

In a traditional library, books are organized by the Dewey Decimal System — a hierarchical classification that puts books into fixed categories. "Machine Learning" goes in 006.31 (Computer Science > AI). "Statistics" goes in 519.5 (Mathematics > Probability). These are hard categories, and a book can only be in one place.

An embedding-based library works differently. Instead of shelves and categories, imagine a vast open room where every book floats in 3D space. Books drift naturally toward other books they're related to:

```
In the embedding library:

  Machine Learning floats near:
    - Deep Learning (very close — subspecialty)
    - Statistics (close — foundational)
    - Neural Networks (close — key technique)
    - Python Programming (nearby — common tool)
    - Linear Algebra (nearby — mathematical foundation)
    - Data Science (nearby — overlapping field)
    - Philosophy of Mind (distant but same wing — both about "intelligence")

  The book on Machine Learning is simultaneously:
    - In the "AI" neighborhood
    - In the "Math-heavy" region
    - In the "Computer Science" zone
    - In the "Modern technology" area
    
  It doesn't have to choose ONE category — its position captures ALL of its relationships.
```

This is what makes embeddings so powerful for search. A query doesn't need to match a category — it just needs to be in the right neighborhood.

### 2.3 Why This Is Powerful

Once meaning lives in a geometric space, you get to use geometry to work with meaning:

**Similarity = proximity.** To find things similar to "machine learning," just find the nearest neighbors in the vector space. No keywords needed. "ML," "deep learning," "neural networks," and "AI training" will all be nearby, even though they share few letters.

**Analogy = direction.** The direction from "man" to "king" captures something like "royalty." Apply that same direction to "woman" and you land near "queen." This works because the relationship is encoded geometrically.

**Clustering = categories.** Run a clustering algorithm on embeddings and you automatically discover groups — without defining the groups in advance. Product reviews might cluster into "quality complaints," "shipping issues," "great product," etc.

**Outliers = anomalies.** If most support tickets cluster in a few areas but one is far from everything, it might be a novel issue worth investigating.

**Interpolation = blending.** The midpoint between "formal email" and "casual text message" gives you something like "semi-formal Slack message." You can blend meanings by averaging vectors.

---

## 3. The Famous "King - Man + Woman = Queen" Example

### 3.1 How Analogy Works

The most famous demonstration of embedding quality is the word analogy test. Using embeddings from Word2Vec (the model that popularized embeddings in 2013):

```
vector("king") - vector("man") + vector("woman") ≈ vector("queen")
```

What's happening here:

1. `vector("king")` represents the concept of "king" — male, royal, powerful, rules a kingdom
2. `vector("man")` represents the concept of "man" — male, human, adult
3. `vector("king") - vector("man")` subtracts out the "male" component, leaving something like "royal ruler"
4. Adding `vector("woman")` adds back "female, human, adult"
5. The result is a point in vector space nearest to "queen" — female, royal, powerful, rules a kingdom

This works because the embedding model learned that the *relationship* between "man" and "king" is similar to the relationship between "woman" and "queen." The direction from male terms to their royal counterparts is consistent in vector space.

### 3.2 More Examples

The same arithmetic works for many relationships:

```
Paris - France + Italy        ≈ Rome       (capital-of relationship)
bigger - big + small          ≈ smaller    (comparative form)
walked - walk + swim          ≈ swam       (past tense)
doctor - man + woman          ≈ doctor     (gender-neutral profession... mostly)
```

### 3.3 The Limits of Vector Arithmetic

The "King - Man + Woman = Queen" example is elegant, but in practice, vector arithmetic is imperfect. It works best for well-defined, consistent relationships in the training data. It works less well for:

- **Subjective relationships**: "Good - Movie + Book ≠ Good" (quality is context-dependent)
- **Complex analogies**: Multi-step analogies break down quickly
- **Cultural concepts**: Relationships that vary across cultures may not transfer cleanly

This is worth knowing so you don't over-rely on vector arithmetic in production. It's a great demonstration of embedding quality, but practical applications use **nearest neighbor search** (find the closest vectors) rather than arithmetic.

### 3.4 Why This Matters Practically

You probably won't do vector arithmetic in production. But the "King - Queen" example demonstrates something essential: **embeddings capture relationships, not just individual meanings.** This is what makes them so powerful for search.

When a user searches for "how to fix a broken database connection" and your knowledge base has an article titled "Troubleshooting PostgreSQL connectivity issues," keyword search would fail — the words barely overlap. But their embeddings will be close because the *meaning* is similar. The relationship "problem and solution" and the domain "database connectivity" are both captured in the geometric space.

---

## 4. Cosine Similarity: Measuring Likeness

### 4.1 How Do You Measure "Closeness" in Vector Space?

You have two vectors and want to know: how similar are they? The most commonly used measure is **cosine similarity**.

Cosine similarity measures the **angle** between two vectors, ignoring their magnitude (length). It produces a number between -1 and 1:

- **1.0** = identical direction (maximum similarity)
- **0.0** = perpendicular (no similarity)
- **-1.0** = opposite direction (maximum dissimilarity)

In practice, for text embeddings, you'll almost always see values between 0.0 and 1.0.

### 4.2 The Intuition

Imagine two arrows drawn from the origin of a graph. Cosine similarity measures the angle between them:

```
         B →  •
        /  angle = small → high similarity
       /
      /
     • → A

         C →  •
        
        
      
     • → A     angle = large → low similarity
```

Two documents about "machine learning" will point in roughly the same direction (small angle, high cosine similarity). A document about "machine learning" and a document about "Italian cooking" will point in very different directions (large angle, low cosine similarity).

### 4.3 The Math (Simple Version)

For two vectors A and B, cosine similarity is:

```
                    A · B         sum of (A[i] * B[i]) for each dimension
cosine_sim(A,B) = ────── = ──────────────────────────────────────────────
                  |A|·|B|    sqrt(sum of A[i]²) * sqrt(sum of B[i]²)
```

Or in plain English:
1. Multiply each pair of corresponding numbers
2. Sum them up (this is the "dot product")
3. Divide by the product of the lengths of both vectors

A simple TypeScript implementation:

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  if (a.length !== b.length) {
    throw new Error("Vectors must have the same length");
  }

  let dotProduct = 0;
  let magnitudeA = 0;
  let magnitudeB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    magnitudeA += a[i] * a[i];
    magnitudeB += b[i] * b[i];
  }

  const magnitude = Math.sqrt(magnitudeA) * Math.sqrt(magnitudeB);
  if (magnitude === 0) return 0;

  return dotProduct / magnitude;
}

// Example with simple 3D vectors:
const cat = [0.9, 0.8, 0.1];
const dog = [0.85, 0.75, 0.15];
const car = [0.1, 0.3, 0.9];

console.log(cosineSimilarity(cat, dog));  // ~0.997 (very similar)
console.log(cosineSimilarity(cat, car));  // ~0.504 (not similar)
```

### 4.4 A Worked Example

Let's trace through a concrete similarity comparison to build intuition:

```
Document A: "How to train a machine learning model"
Document B: "Steps for building an ML classifier"
Document C: "Best Italian pasta recipes"

After embedding (hypothetical 5D vectors for illustration):
  A: [0.82, 0.71, 0.45, 0.12, 0.33]
  B: [0.79, 0.68, 0.42, 0.15, 0.31]
  C: [0.11, 0.08, 0.91, 0.76, 0.62]

Cosine similarity:
  A vs B: 0.998 → Almost identical direction (same topic, different words)
  A vs C: 0.412 → Very different directions (different topics)
  B vs C: 0.398 → Very different directions (different topics)
```

Documents A and B are about the same thing (training ML models) despite sharing almost no words. The embedding model captured the meaning, not the vocabulary. Document C is about cooking and points in a completely different direction in vector space.

This is the fundamental power of embeddings: "machine learning model" and "ML classifier" have a similarity of 0.998 (near-identical), while a keyword search for "machine learning" wouldn't even match Document B.

### 4.5 Why Cosine Similarity and Not Euclidean Distance?

You might wonder: why not just measure the straight-line distance between two points? That's **Euclidean distance**, and it works too, but cosine similarity has advantages for text embeddings:

**Cosine similarity is scale-invariant.** If one vector is twice as long as another but points in the same direction, cosine similarity is still 1.0. This matters because embedding magnitude can vary based on text length, but direction captures meaning.

**Normalized vectors make cosine == dot product.** Most embedding APIs return normalized vectors (length = 1). For normalized vectors, cosine similarity is just the dot product — faster to compute. This is what makes vector search efficient at scale.

**Cosine is the standard.** The embedding models are trained to optimize for cosine similarity. Using a different distance metric means the numbers aren't calibrated for it.

### 4.6 Interpreting Similarity Scores

What counts as "similar" depends on the embedding model and your use case, but rough guidelines:

| Cosine Similarity | Interpretation | Example |
|---|---|---|
| 0.95 - 1.00 | Near-duplicate — almost identical meaning | "How to reset password" vs "Password reset instructions" |
| 0.85 - 0.95 | Very similar — same topic, same intent | "My laptop won't start" vs "Computer not booting up" |
| 0.70 - 0.85 | Similar — related topic, overlapping content | "Password reset" vs "Account security settings" |
| 0.50 - 0.70 | Somewhat related — tangential connection | "Password reset" vs "User authentication architecture" |
| 0.30 - 0.50 | Weak relationship — same broad domain | "Password reset" vs "Server deployment guide" |
| 0.00 - 0.30 | Unrelated | "Password reset" vs "Italian pasta recipes" |

These thresholds vary by embedding model. A score of 0.80 with OpenAI's embeddings might mean the same thing as 0.75 with Cohere's. Always calibrate thresholds on your actual data.

**Practical threshold selection by use case:**

```
Duplicate detection:         0.95+  (very strict — you want near-exact matches)
FAQ matching:                0.85+  (high confidence the question matches)
Document retrieval for RAG:  0.70+  (cast a wider net, let the LLM filter)
"Related articles" sidebar:  0.60+  (loose association is fine)
Anomaly detection:           <0.40  (flag things that DON'T match anything)
```

You'll calibrate these thresholds through evaluation (Ch 19). Start with the numbers above, then adjust based on false positives (returning irrelevant results) and false negatives (missing relevant results).

---

## 5. How Embeddings Are Created

### 5.1 From Word Embeddings to Text Embeddings

Embeddings have evolved through several generations:

**Word2Vec (2013)** — one vector per word. Trained by predicting a word from its neighbors (or vice versa). Showed that word vectors capture analogies. Limitation: one vector per word, so "bank" (financial) and "bank" (river) get the same embedding.

**GloVe (2014)** — also one vector per word, but trained on global word co-occurrence statistics. Similar quality to Word2Vec, different training approach.

**ELMo (2018)** — context-dependent word embeddings. "bank" in "river bank" gets a different vector than "bank" in "bank account." A breakthrough: the same word gets different embeddings based on surrounding context.

**BERT (2018)** — bidirectional transformer that produces contextual embeddings. Not just word embeddings but whole-sentence understanding. The foundation of modern text embeddings.

**Modern embedding models (2023+)** — OpenAI's text-embedding-3, Cohere Embed v3, Google's text-embedding-004, and open-source models like Sentence Transformers. These produce a single vector for an entire piece of text (sentence, paragraph, or document), optimized for semantic similarity.

The progression from Word2Vec to modern embedding models mirrors the progression from "one meaning per word" to "meaning depends on context." This is why modern embeddings are so much better at search — they understand that "apple" in a recipe and "apple" in a tech review mean different things, and produce different vectors accordingly.

### 5.2 Training Embedding Models (Conceptual)

How do embedding models learn to produce useful vectors? The core training approach uses **contrastive learning**:

1. Take a pair of texts that ARE semantically similar (a question and its answer, a query and a relevant document)
2. Take a pair of texts that are NOT similar (a question and a random document)
3. Train the model to make similar pairs have high cosine similarity and dissimilar pairs have low cosine similarity

```
Training examples:
  Similar pair:    ("How to reset password?", "Go to Settings > Reset Password")
                    → train to produce high similarity (e.g., 0.92)
  
  Dissimilar pair: ("How to reset password?", "Our company was founded in 2015")
                    → train to produce low similarity (e.g., 0.15)
```

After seeing millions of such pairs, the model learns what "similarity" means — not based on shared words, but based on shared meaning. It learns that "puppy" and "young dog" should be close, and "bank" (financial) and "bank" (river) should be distant when the context makes the meaning clear.

### 5.3 The Process (Conceptual)

When you call an embedding API:

```
Input:  "How do I reset my password?"
         ↓
     [Tokenization]
         ↓
     [Neural network processes tokens]
     [Multiple layers capture different aspects of meaning]
     [Final layer produces a vector for the whole input]
         ↓
Output: [0.023, -0.041, 0.089, ..., 0.012]   (1,536 numbers)
```

The neural network was trained on massive amounts of text so that:
- Texts with similar meaning produce similar vectors
- Texts with different meaning produce different vectors
- The vectors are useful for tasks like search, classification, and clustering

You don't need to understand *how* the neural network produces these vectors (that's Ch 36). You just need to trust that it does, and that it does it well.

> **Spiral note:** Ch 36 (Neural Networks from Scratch) explains how weights, activations, and backpropagation create these representations. Ch 42 goes deep on choosing and running embedding models.

---

## 6. What Embeddings Enable

Embeddings aren't just an academic concept — they're the foundation of most production AI features. Here's a survey of what you can build, ordered roughly from most common to most specialized.

### 6.1 Semantic Search

The killer app for embeddings — and the most common use case you'll build. Instead of matching keywords, you match meaning.

```
User query:  "my laptop won't turn on"

Keyword search would match:
  ✓ "Laptop won't turn on after update"
  ✗ "Computer not booting — screen stays black"  (different words, same meaning)
  ✗ "My machine is dead and not powering up"     (different words, same meaning)

Semantic search (embeddings) would match ALL of them:
  ✓ "Laptop won't turn on after update"           (similar embedding)
  ✓ "Computer not booting — screen stays black"   (similar embedding)
  ✓ "My machine is dead and not powering up"      (similar embedding)
```

**How it works:**
1. Embed all your documents (once, ahead of time)
2. Embed the search query (at search time)
3. Find the documents whose embeddings are most similar to the query embedding
4. Return those documents

This is the foundation of RAG (Retrieval Augmented Generation), which we cover in Ch 14-17.

### 6.2 Retrieval Augmented Generation (RAG)

RAG is the pattern of giving an LLM access to your company's knowledge. It's the most common and most important use of embeddings in production:

```
User asks: "What's the refund policy for annual subscriptions?"

Step 1: Embed the question → query vector
Step 2: Search your knowledge base for similar content
Step 3: Find: "Annual subscription refund policy" document (0.91 similarity)
Step 4: Send to LLM:
  System: "Answer the user's question based on the following documents..."
  Context: [retrieved refund policy document]
  User: "What's the refund policy for annual subscriptions?"
Step 5: LLM responds with an accurate answer grounded in your actual policy
```

Without RAG, the LLM only knows what was in its training data — which definitely doesn't include your company's refund policy. With RAG, it knows about your company's products, policies, internal docs — anything you've embedded. The LLM becomes a knowledgeable assistant about YOUR business, not just a general knowledge chatbot.

RAG is covered extensively in Ch 14-17. It's the single most impactful pattern for production AI applications.

### 6.3 Recommendations (Content-Based)

"Customers who viewed this also viewed..." becomes a vector similarity problem. The idea is simple:

1. Embed product descriptions (or articles, or songs, or anything)
2. When a user views an item, find items whose embeddings are nearest
3. Recommend those items

No collaborative filtering needed — just embedding similarity. This approach is particularly powerful for:
- **Cold-start problems**: New items with no usage data can still be recommended based on their description
- **Long-tail content**: Items that few people have seen can still surface if their descriptions are semantically similar to popular items
- **Cross-domain recommendations**: If you embed both product descriptions and user reviews, you can recommend products similar to what users have praised

Here's a concrete example:

```
User just viewed: "Sony WH-1000XM5 Noise Cancelling Headphones"
  → embed the product description
  → find the 5 nearest products in embedding space

Results:
  1. Bose QuietComfort Ultra Headphones       (similarity: 0.94)
  2. Apple AirPods Max                         (similarity: 0.91)
  3. Sony WF-1000XM5 Earbuds                  (similarity: 0.89)
  4. Sennheiser Momentum 4 Wireless           (similarity: 0.87)
  5. JBL Tour One M2                          (similarity: 0.85)
```

Notice: the recommendations are all premium noise-cancelling headphones/earbuds. The embedding captured the semantic category (premium wireless audio) without anyone defining product categories manually. You can enhance this further by combining the product embedding with the user's browsing history embeddings to personalize results.

### 6.4 Classification (Zero-Shot and Few-Shot)

Embedding-based classification is remarkably powerful because you don't need to train a classifier. Just embed your categories and compare.

**Zero-shot classification** — define categories by their name or description alone:

```
Category embeddings (just embed the category names):
  "billing_issue"   → embed("billing issue")
  "tech_support"    → embed("technical support problem")
  "feature_request" → embed("feature request or suggestion")
  "general_inquiry" → embed("general question or inquiry")

New ticket: "I was charged twice for my subscription"
  → embed it
  → similarity to "billing_issue": 0.87    ← highest
  → similarity to "tech_support": 0.42
  → similarity to "feature_request": 0.31
  → similarity to "general_inquiry": 0.45
  → Route to billing team ✅
```

**Few-shot classification** — provide example texts for each category for better accuracy:

```
Category embeddings (average of example embeddings):
  "billing_issue"  → average of embeddings of 10 known billing issues
  "tech_support"   → average of embeddings of 10 known tech support tickets
  "feature_request" → average of embeddings of 10 known feature requests

New ticket: "I was charged twice for my subscription"
  → embed it
  → closest to "billing_issue" (average)
  → route to billing team
```

The few-shot approach is typically more accurate because the average of many examples creates a richer representation of what each category "means" in practice. Ten examples per category is often enough to get started; 50-100 gives you production-quality classification for many tasks.

### 6.5 Duplicate Detection

Embed all items, find pairs with very high similarity (>0.95). Those are likely duplicates or near-duplicates — even when the wording is completely different.

```
Support ticket #1: "I can't log into my account, it says invalid password"
Support ticket #2: "Login not working - password error message shows up"
Support ticket #3: "Unable to authenticate, getting credential error"

Pairwise similarities:
  #1 vs #2: 0.96 ← likely duplicate
  #1 vs #3: 0.93 ← likely duplicate
  #2 vs #3: 0.94 ← likely duplicate
```

Keyword-based deduplication would miss all of these — the tickets use almost no words in common. But embedding-based deduplication catches them because the *meaning* is nearly identical.

Use cases:
- Deduplicate search results
- Find copy-pasted content across documents
- Identify rephrased but identical support tickets (route to the same thread)
- Detect plagiarism or content reuse
- Merge duplicate entries in CRM or knowledge base systems

### 6.6 Anomaly Detection

If you embed a large collection of items, most of them will cluster into natural groups. Items that are far from every cluster are anomalies — and anomalies are often interesting.

```
1,000 customer support tickets, embedded and clustered:

  Cluster A: Password/login issues      (342 tickets)
  Cluster B: Billing questions           (289 tickets)
  Cluster C: Feature requests            (198 tickets)
  Cluster D: Bug reports                 (156 tickets)
  
  Outliers (far from all clusters):
    "I think someone else accessed my account from another country"
    "Your API is being used to scrape competitor data"
    "Received an email claiming to be from your company asking for my SSN"
```

Those outliers — security incidents, fraud attempts, phishing reports — don't fit neatly into the standard support categories. Embedding-based anomaly detection surfaces them automatically, without needing to define "what is an anomaly" in advance.

### 6.7 Content Moderation

Embeddings can power a flexible moderation system. Instead of maintaining a keyword blocklist (which users can trivially evade with misspellings and synonyms), embed your policy examples and flag content that's semantically similar:

```
Policy examples (embedded once):
  "I want to harm myself" → moderation_vector_1
  "How to make a weapon"  → moderation_vector_2
  "Racial slur examples"  → moderation_vector_3

User input: "What's the best way to end it all"
  → embed
  → similarity to moderation_vector_1: 0.88 ← flag for review
  
User input: "end it all and start fresh with a new account"
  → embed
  → similarity to moderation_vector_1: 0.54 ← below threshold, don't flag
```

This approach catches semantic intent rather than surface-level keywords, making it much harder to evade. The embedding-based approach also scales better — instead of maintaining a growing blocklist, you maintain a small set of policy example embeddings that generalize to novel violations.

---

## 7. Limitations and Gotchas

Embeddings are powerful, but they're not magic. Understanding their limitations will save you from building systems that fail in subtle, hard-to-debug ways.

### 7.1 Embedding Models Have Limits Too

**Max input length.** Embedding models have token limits, just like LLMs. OpenAI's text-embedding-3-small has an 8,191 token limit. If your document is longer, you need to chunk it (split it into pieces). Chunking strategy matters a lot — we'll cover it in Ch 15.

**One embedding per input.** You get one vector for the whole input. A 500-word paragraph gets compressed into 1,536 numbers. Some nuance is inevitably lost. Short, focused texts tend to produce better embeddings than long, rambling ones.

**Domain specificity.** General-purpose embedding models work well for general text but may underperform on specialized domains (legal, medical, scientific). A model trained mostly on web text might not embed "amicus brief" and "friend of the court filing" as closely as a legal-domain model would. Domain-specific embedding models exist for some fields, and fine-tuning embeddings is an option for specialized use cases (Ch 44).

### 7.2 The Negation Problem

This is the single biggest gotcha with embeddings, and it bites almost every team at some point:

```
"I love this product"       → embedding A
"I don't love this product" → embedding B

cosine_similarity(A, B) ≈ 0.92  ← These are very similar!
```

The embeddings are almost identical because the two sentences share most of their words. The word "don't" is a tiny signal compared to the strong signals from "I," "love," "this," and "product." The negation gets lost.

This means that in a sentiment analysis system built on embeddings, you might classify "I don't recommend this restaurant" as *positive* because it's close to "I recommend this restaurant." The embedding model sees them as nearly the same sentence because they share the same vocabulary and structure.

**The workaround:** Don't use raw embedding similarity for tasks where negation matters. Use the LLM itself for those tasks (sentiment classification, contradiction detection). Embeddings are for *topical* similarity, not *semantic equivalence*.

More examples of the negation problem:
```
"The system is working"     vs  "The system is not working"        → high similarity
"Payment was successful"    vs  "Payment was unsuccessful"         → high similarity
"I can access the file"     vs  "I can't access the file"         → high similarity
"This feature is available" vs  "This feature is not available"   → high similarity
```

### 7.3 Length Sensitivity

Comparing a 3-word query against a 500-word document can produce lower-than-expected similarity scores. The embedding of a short query captures a focused meaning; the embedding of a long document captures many meanings averaged together.

```
Query:    "password reset"           → focused, specific embedding
Document: "Our account management system supports password reset,
           two-factor authentication, SSO integration, role-based
           access control, and session management..."
           → diluted embedding (password reset is just one of many topics)
```

The document is relevant, but its embedding has been "diluted" by all the other topics it covers. This is one of the core reasons chunking exists — splitting that document into 5 separate paragraphs, each about a single topic, produces much better search results.

**Strategies for handling length mismatch:**
- **Chunk documents** into focused paragraphs (Ch 15)
- **Use query-document prefixes** — some models (like Cohere's embed-v3) use different prefixes for queries vs documents: `"search_query: password reset"` and `"search_document: Our account management system..."` This helps the model produce embeddings optimized for asymmetric search.
- **Use re-ranking** — retrieve more candidates than you need with embedding search, then use a cross-encoder or LLM to re-rank them for relevance (Ch 16)

### 7.4 No Understanding of Structure or Logic

Embeddings capture *topical relatedness*, not logical structure. They can't tell you:

- Whether two statements contradict each other
- Whether one statement is a subset or superset of another
- Whether a conclusion follows logically from premises
- Whether code is correct or incorrect (syntactically similar code gets similar embeddings regardless of bugs)

If you need logical or structural analysis, use an LLM for that part of the pipeline. Embeddings are for finding relevant content; the LLM reasons over that content.

### 7.5 Embedding Models Age

Embedding models have training data cutoffs too. An embedding model trained on data from 2023 won't have good representations for concepts that emerged in 2025. New jargon, new products, new cultural references may not be well-represented.

This is particularly problematic for:
- **Brand names and products**: A model trained before "Claude Code" existed might not embed that phrase meaningfully
- **New technical concepts**: Terms like "vibe coding" or new framework names might get split into generic tokens
- **Cultural references**: Memes, slang, and trends that emerged after training

The fix: use recent embedding models, and be aware that embedding quality on novel terms will be lower than on well-established vocabulary.

### 7.6 Dimensionality, Storage, and Cost

Higher-dimensional embeddings capture more nuance but:
- Take more storage (each dimension is a float, typically 4 bytes)
- Are slower to compare (more multiplications per similarity calculation)
- Require more memory for search indices

Let's do the math:

```
Storage calculation for different scales:

1,536 dims × 1,000 documents × 4 bytes     = ~6 MB      (trivial)
1,536 dims × 100,000 documents × 4 bytes   = ~600 MB    (fits in memory)
1,536 dims × 1,000,000 documents × 4 bytes = ~6 GB      (needs planning)
1,536 dims × 100,000,000 documents × 4 bytes = ~600 GB  (needs infrastructure)

With 3,072-dimensional embeddings, double all of these.
```

At scale, you'll want to consider:
- **Dimensionality reduction**: OpenAI's text-embedding-3 models support custom dimensions — request 512 dimensions instead of 1,536 for 3x storage savings with modest quality loss
- **Quantization**: Store vectors as int8 instead of float32 for 4x storage savings
- **Approximate nearest neighbor (ANN)**: Trade a tiny amount of accuracy for massive speed gains at scale

### 7.7 The "Garbage In, Garbage Out" Rule

The quality of your embeddings depends entirely on the quality of your input text. If you embed poorly written, typo-filled, or inconsistently formatted text, the embeddings will reflect that mess. Common issues:

- **HTML/markdown artifacts** in text produce noisy embeddings — strip formatting before embedding
- **Boilerplate headers/footers** (copyright notices, navigation menus) dilute the meaningful content
- **Very short text** (1-3 words) produces less reliable embeddings than sentences
- **Mixed languages** in a single input confuse models trained primarily on one language

---

## 8. Building Intuition: A Visual Walkthrough

Let's trace through a concrete example to cement the mental model. Imagine you're building a customer support search system.

### Step 1: Embed Your Knowledge Base

You have 1,000 support articles. Each gets embedded:

```
Article: "How to reset your password"
  → [0.12, -0.03, 0.45, ..., 0.08]   (1,536 numbers)

Article: "Troubleshooting login issues"
  → [0.11, -0.02, 0.43, ..., 0.09]   (similar numbers — similar topic!)

Article: "Configuring email notifications"
  → [-0.34, 0.21, -0.15, ..., 0.56]  (different numbers — different topic)
```

### Step 2: Store the Vectors

All 1,000 vectors go into a **vector store** (a database optimized for similarity search). Popular options include Pinecone, Weaviate, Qdrant, and pgvector (Postgres extension). We'll cover these in Ch 14.

### Step 3: User Searches

A user types: "I can't log in"

You embed their query:
```
"I can't log in" → [0.10, -0.01, 0.41, ..., 0.07]
```

### Step 4: Find Nearest Neighbors

The vector store compares the query vector against all 1,000 article vectors using cosine similarity:

```
"How to reset your password"         → similarity: 0.89
"Troubleshooting login issues"       → similarity: 0.93  ← best match!
"Configuring email notifications"    → similarity: 0.23
"Billing FAQ"                        → similarity: 0.15
... etc ...
```

### Step 5: Return Results

You return the top 3-5 most similar articles. "Troubleshooting login issues" comes first, even though the user never said "troubleshoot" — because the *meaning* matched.

### Step 6 (Optional): RAG

Feed those articles into an LLM as context, along with the user's question. The LLM generates a personalized answer grounded in your actual documentation.

### Why This Beats Keyword Search

Let's compare what happens with keyword search vs embedding search for real-world queries:

```
Query: "my account got hacked"

Keyword search matches:
  ✓ "What to do if your account got hacked"
  ✗ "Unauthorized access to your account" (no word overlap with "hacked")
  ✗ "Security breach — someone else is using my login" (no word overlap)
  ✗ "Two-factor authentication setup guide" (relevant preventive content, no match)

Embedding search matches:
  ✓ "What to do if your account got hacked"          (0.94 similarity)
  ✓ "Unauthorized access to your account"             (0.91 similarity)
  ✓ "Security breach — someone else is using my login" (0.89 similarity)
  ✓ "Two-factor authentication setup guide"           (0.78 similarity)
```

```
Query: "app is super slow"

Keyword search matches:
  ✓ "Why is the app slow?"
  ✗ "Performance troubleshooting guide" (no word overlap)
  ✗ "High latency when loading dashboard" (no word overlap)
  ✗ "How to optimize your experience" (no word overlap)

Embedding search matches:
  ✓ "Why is the app slow?"                  (0.93 similarity)
  ✓ "Performance troubleshooting guide"     (0.88 similarity)
  ✓ "High latency when loading dashboard"   (0.85 similarity)
  ✓ "How to optimize your experience"       (0.76 similarity)
```

The gap between keyword search and embedding search grows wider as your users become more diverse in how they phrase questions. Keyword search requires your users to use the exact same vocabulary as your documentation. Embedding search understands what they mean regardless of phrasing.

---

## 9. Word Embeddings vs Sentence Embeddings vs Document Embeddings

One distinction worth clarifying: embeddings operate at different levels, and choosing the right level matters for your application.

**Word embeddings** (Word2Vec, GloVe): One vector per word. "dog" gets one vector. Useful for word-level tasks but limited because you can't represent a whole sentence. Also, every use of "bank" gets the same vector — whether it's a river bank or a financial bank. These are mostly historical now; you won't use them directly.

**Sentence embeddings** (Sentence Transformers, OpenAI's embedding models): One vector per sentence or short text. "The dog chased the cat" gets one vector that captures the entire meaning — including word order, so "the dog chased the cat" and "the cat chased the dog" get *different* vectors (as they should — the meaning is different). This is what you'll use most often.

**Document embeddings**: One vector per longer text (paragraph, page, document). Same models can do this, but quality degrades with length because you're compressing more information into the same number of dimensions. A 1,536-dimensional vector can faithfully represent a sentence. Asking it to represent a 10-page document means a lot of information gets averaged out.

**The chunking solution**: The standard approach for documents is to split them into focused chunks (typically 200-500 tokens each), embed each chunk separately, and search at the chunk level. A 10-page document becomes 20-40 chunks, each with its own embedding. When a user searches, they find the most relevant *chunk*, not the most relevant *document*. This gives much better search precision.

```
Document: "Acme Corp Employee Handbook" (50 pages)
  
  Bad approach: Embed the entire handbook → 1 vector
    - Searching for "vacation policy" returns the handbook
    - But you don't know WHERE in the handbook to look
  
  Good approach: Chunk into ~200 sections → 200 vectors
    - Searching for "vacation policy" returns Section 4.2
    - You know exactly what to show the user (or feed to the LLM)
```

For almost everything in this guide, we'll use **sentence/text embeddings** — models that take a piece of text (up to their token limit) and return one vector. Chunking strategies are covered in depth in Ch 15.

### 9.1 Embedding Dimensions: Quality vs Cost Trade-off

One practical consideration: you can often choose the number of dimensions for your embeddings. OpenAI's text-embedding-3 models support custom dimensions — you can request 256, 512, 1024, or the full 1536/3072 dimensions.

```
Fewer dimensions:
  ✓ Smaller storage (256 dims × 4 bytes = 1 KB per vector)
  ✓ Faster similarity search
  ✓ Cheaper vector database costs
  ✗ Less nuanced representation
  ✗ Slightly worse search quality

More dimensions:
  ✓ More nuanced representation
  ✓ Better search quality for subtle distinctions
  ✗ Larger storage (3072 dims × 4 bytes = 12 KB per vector)
  ✗ Slower search at scale
  ✗ Higher vector database costs
```

For most applications, 1,536 dimensions is the sweet spot. Consider reducing to 512 or 768 if you're embedding millions of items and storage costs matter. Always benchmark on your actual data to see if reduced dimensions hurt quality for your use case.

---

## 10. The Embedding Landscape (Quick Reference)

Here's a snapshot of embedding options you'll encounter. Choosing the right embedding model matters — it affects search quality, storage costs, and latency throughout your entire pipeline.

### Closed-Source (API)

| Provider | Model | Dimensions | Max Tokens | Price per 1M tokens | Strengths |
|---|---|---|---|---|---|
| OpenAI | text-embedding-3-small | 1,536 | 8,191 | $0.02 | Cheap, good quality, adjustable dimensions |
| OpenAI | text-embedding-3-large | 3,072 | 8,191 | $0.13 | Higher quality, adjustable dimensions |
| Cohere | embed-v3 | 1,024 | 512 | $0.10 | Best multilingual, search-optimized, query/doc modes |
| Google | text-embedding-004 | 768 | 2,048 | $0.00625 | Very cheap, integrated with Google Cloud |
| Voyage AI | voyage-3 | 1,024 | 32,000 | $0.06 | Very long context, strong for code |

### Open-Source (Self-Hosted)

| Model | Dimensions | Max Tokens | Speed | Strengths |
|---|---|---|---|---|
| all-MiniLM-L6-v2 | 384 | 512 | Very fast | Small, great for prototyping, runs on CPU |
| bge-large-en-v1.5 | 1,024 | 512 | Medium | Strong quality, MTEB benchmark leader |
| E5-mistral-7b | 4,096 | 32,768 | Slow | Very long context, instruction-tuned |
| nomic-embed-text | 768 | 8,192 | Fast | Good quality, fully open source, long context |
| gte-large-en-v1.5 | 1,024 | 8,192 | Medium | Strong quality, long context |

### How to Choose an Embedding Model

**Start with OpenAI text-embedding-3-small.** It's cheap ($0.02 per million tokens — essentially free for prototyping), good quality, and dead simple to use via the API. You can embed 50 million tokens for $1.

**Switch to a better model only if search quality isn't good enough.** Run evals on your actual queries and documents (Ch 19). If retrieval precision is too low, try:
1. **text-embedding-3-large** — same API, 2x dimensions, better quality, still affordable
2. **Cohere embed-v3** — especially if you have multilingual content or want query/document mode optimization
3. **Voyage AI voyage-3** — especially for code search or very long documents

**Consider open source if:**
- You need data to stay on your infrastructure (healthcare, finance, government)
- You're embedding massive volumes where even $0.02/1M tokens adds up
- You want to fine-tune the embedding model on your domain

**Cohere embed-v3 deserves special mention** for its query/document mode feature. When embedding a search query, you prefix it as a query; when embedding a knowledge base document, you prefix it as a document. This produces asymmetric embeddings that work better for search because queries and documents have different characteristics (queries are short and specific; documents are long and comprehensive).

We'll revisit these in Ch 14 (hands-on usage) and Ch 42 (deep dive on model selection and running your own).

---

## 11. Key Takeaways

1. **Embeddings turn meaning into math.** Text becomes a vector (list of numbers), similar meanings become similar vectors. Think of it as a library organized by meaning, not alphabetically.

2. **Cosine similarity measures likeness.** Ranges from -1 (opposite) to 1 (identical). Most text comparisons fall between 0 and 1. Calibrate thresholds for your specific use case.

3. **Vector arithmetic captures relationships.** "King - Man + Woman = Queen" demonstrates that directions in vector space represent semantic relationships. This is what makes semantic search possible.

4. **Embeddings enable semantic search.** Match by meaning, not keywords. This is the foundation of RAG, and it catches the results that keyword search misses — which are often the most important results.

5. **Embeddings are created by neural networks.** You don't need to understand the internals (yet). Just know that the models are trained to produce useful representations.

6. **Chunk long texts.** Embedding models have token limits, and shorter, focused texts produce better embeddings. A 10-page document should be split into 20-40 focused chunks.

7. **Beware the negation problem.** "I love this" and "I don't love this" can have very similar embeddings. Embeddings capture topical similarity, not semantic equivalence. Use LLMs for tasks that require understanding negation or contradiction.

8. **Choose the right model for your task.** Start with OpenAI text-embedding-3-small for prototyping. Move to larger models only if search quality requires it. Always benchmark on your actual data.

9. **Embeddings are the foundation layer.** Almost everything you build in Parts 2-4 of this guide (semantic search, RAG, recommendations, classification) depends on embeddings. Getting the embedding layer right pays dividends everywhere.

10. **Embeddings are cheap.** At $0.02 per million tokens, embedding your entire knowledge base costs almost nothing. The expensive part is storing and searching the vectors at scale — but even that is manageable with modern vector databases.

---

---

## 12. From Concepts to Code: The Bridge to Chapter 14

Everything in this chapter has been conceptual — mental models, analogies, toy examples. That was intentional. You need the intuition before the implementation.

In **Chapter 14 (Semantic Search)**, we'll take every concept from this chapter and turn it into working code:

- **Generating embeddings** — calling OpenAI's embedding API, handling batches, managing rate limits
- **Storing vectors** — setting up Pinecone, pgvector, or Qdrant as your vector store
- **Searching** — embedding a query, finding nearest neighbors, returning results
- **Chunking documents** — splitting long texts into embeddable pieces (this is where most RAG quality problems live)
- **Building a full pipeline** — ingest documents, embed, store, search, return results

The gap between "I understand what embeddings are" and "I can build a production search system" is mostly plumbing — but it's important plumbing. Chapter 14 is where we build it.

For now, carry these mental models forward:
- Meaning is geometry. Similar texts live near each other in vector space.
- Cosine similarity measures that nearness. Higher is more similar.
- Embeddings enable search by meaning, not keywords.
- The quality of your embeddings determines the quality of everything built on top of them.

---

## What's Next

In **Chapter 2: The AI Engineer's Landscape**, we'll survey the providers, pricing models, SDKs, and the build-vs-buy decision tree. You'll understand who the players are and how to choose between them.

Then in **Chapter 14**, we'll come back to embeddings with working code — generating embeddings with OpenAI's API, storing them in vector databases, and building semantic search. Everything from this chapter becomes concrete in 14.

And in **Chapter 42**, we'll spiral all the way to generating your own embeddings with open-source models, benchmarking them on MTEB, and choosing the optimal model for your use case.

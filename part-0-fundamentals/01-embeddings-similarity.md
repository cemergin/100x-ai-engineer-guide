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

**The key insight:** You don't need to understand what each dimension means. You just need to know that the resulting vectors capture semantic similarity reliably. Similar texts → similar vectors. Different texts → different vectors.

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

### 2.2 Why This Is Powerful

Once meaning lives in a geometric space, you get to use geometry to work with meaning:

**Similarity = proximity.** To find things similar to "machine learning," just find the nearest neighbors in the vector space. No keywords needed. "ML," "deep learning," "neural networks," and "AI training" will all be nearby, even though they share few letters.

**Analogy = direction.** The direction from "man" to "king" captures something like "royalty." Apply that same direction to "woman" and you land near "queen." This works because the relationship is encoded geometrically.

**Clustering = categories.** Run a clustering algorithm on embeddings and you automatically discover groups — without defining the groups in advance. Product reviews might cluster into "quality complaints," "shipping issues," "great product," etc.

**Outliers = anomalies.** If most support tickets cluster in a few areas but one is far from everything, it might be a novel issue worth investigating.

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

### 3.3 Why This Matters Practically

You probably won't do vector arithmetic in production. But this demonstrates something essential: **embeddings capture relationships, not just individual meanings.** This is what makes them so powerful for search.

When a user searches for "how to fix a broken database connection" and your knowledge base has an article titled "Troubleshooting PostgreSQL connectivity issues," keyword search would fail — the words barely overlap. But their embeddings will be close because the *meaning* is similar. The relationship "problem → solution" and the domain "database connectivity" are both captured.

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

### 4.4 Why Cosine Similarity and Not Euclidean Distance?

You might wonder: why not just measure the straight-line distance between two points? That's **Euclidean distance**, and it works too, but cosine similarity has advantages for text embeddings:

**Cosine similarity is scale-invariant.** If one vector is twice as long as another but points in the same direction, cosine similarity is still 1.0. This matters because embedding magnitude can vary based on text length, but direction captures meaning.

**Normalized vectors make cosine == dot product.** Most embedding APIs return normalized vectors (length = 1). For normalized vectors, cosine similarity is just the dot product — faster to compute. This is what makes vector search efficient at scale.

**Cosine is the standard.** The embedding models are trained to optimize for cosine similarity. Using a different distance metric means the numbers aren't calibrated for it.

### 4.5 Interpreting Similarity Scores

What counts as "similar" depends on the embedding model and your use case, but rough guidelines:

| Cosine Similarity | Interpretation |
|---|---|
| 0.90 - 1.00 | Very similar — near duplicates or paraphrases |
| 0.75 - 0.90 | Similar — same topic, related content |
| 0.50 - 0.75 | Somewhat related — overlapping themes |
| 0.25 - 0.50 | Weak relationship — tangentially related |
| 0.00 - 0.25 | Unrelated |

These thresholds vary. For duplicate detection, you might require 0.95+. For "find related articles," 0.70+ might be fine. You'll calibrate these thresholds through evaluation (Ch 19).

---

## 5. How Embeddings Are Created

### 5.1 From Word Embeddings to Text Embeddings

Embeddings have evolved through several generations:

**Word2Vec (2013)** — one vector per word. Trained by predicting a word from its neighbors (or vice versa). Showed that word vectors capture analogies. Limitation: one vector per word, so "bank" (financial) and "bank" (river) get the same embedding.

**GloVe (2014)** — also one vector per word, but trained on global word co-occurrence statistics. Similar quality to Word2Vec, different training approach.

**ELMo (2018)** — context-dependent word embeddings. "bank" in "river bank" gets a different vector than "bank" in "bank account." A breakthrough: the same word gets different embeddings based on surrounding context.

**BERT (2018)** — bidirectional transformer that produces contextual embeddings. Not just word embeddings but whole-sentence understanding. The foundation of modern text embeddings.

**Modern embedding models (2023+)** — OpenAI's text-embedding-3, Cohere Embed v3, Google's text-embedding-004, and open-source models like Sentence Transformers. These produce a single vector for an entire piece of text (sentence, paragraph, or document), optimized for semantic similarity.

### 5.2 The Process (Conceptual)

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

### 6.1 Semantic Search

The killer app for embeddings. Instead of matching keywords, you match meaning.

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

RAG is the pattern of giving an LLM access to your company's knowledge:

1. User asks a question
2. Use embeddings to find relevant documents from your knowledge base
3. Put those documents into the LLM's context
4. The LLM answers based on the retrieved documents

Without RAG, the LLM only knows what was in its training data. With RAG, it knows about your company's products, policies, internal docs — anything you've embedded.

### 6.3 Recommendations

"Customers who viewed this also viewed..." becomes a vector similarity problem:

1. Embed product descriptions
2. Find products whose embeddings are similar to what the user is viewing
3. Recommend those products

No collaborative filtering needed — just embedding similarity.

### 6.4 Classification

Embed your categories with example texts. To classify a new text, embed it and find the nearest category.

```
Category embeddings:
  "billing_issue"  → average of embeddings of known billing issues
  "tech_support"   → average of embeddings of known tech support tickets
  "feature_request" → average of embeddings of known feature requests

New ticket: "I was charged twice for my subscription"
  → embed it
  → closest to "billing_issue"
  → route to billing team
```

### 6.5 Clustering and Anomaly Detection

Embed a set of texts, run a clustering algorithm (k-means, DBSCAN), and discover natural groups. Or find texts that are far from any cluster — potential anomalies.

Use cases:
- Cluster customer feedback to discover themes
- Group similar support tickets
- Detect unusual transaction descriptions
- Find outlier content in a content moderation pipeline

### 6.6 Duplicate Detection

Embed all items, find pairs with very high similarity (>0.95). Those are likely duplicates or near-duplicates.

Use cases:
- Deduplicate search results
- Find copy-pasted content
- Identify rephrased but identical support tickets

---

## 7. Limitations and Gotchas

### 7.1 Embedding Models Have Limits Too

**Max input length.** Embedding models have token limits, just like LLMs. OpenAI's text-embedding-3-small has an 8,191 token limit. If your document is longer, you need to chunk it (split it into pieces). Chunking strategy matters a lot — we'll cover it in Ch 15.

**One embedding per input.** You get one vector for the whole input. A 500-word paragraph gets compressed into 1,536 numbers. Some nuance is inevitably lost. Short, focused texts tend to produce better embeddings than long, rambling ones.

**Domain specificity.** General-purpose embedding models work well for general text but may underperform on specialized domains (legal, medical, scientific). Domain-specific embedding models exist for some fields.

### 7.2 Similarity Isn't Understanding

Embeddings capture statistical patterns of co-occurrence, not true understanding. This leads to edge cases:

**Negation confusion.** "I love this product" and "I don't love this product" may have similar embeddings because they share most of their tokens. The negation is a small signal that can get lost.

**Superficial similarity.** "The bank was steep" and "The bank approved the loan" might be closer than you'd expect because older embedding models struggle with word sense disambiguation. Modern models handle this better but it's not perfect.

**Length mismatch.** Comparing a 3-word query against a 500-word document can produce lower-than-expected similarity scores. This is why some systems embed queries and documents with different models or add special prefixes.

### 7.3 Embedding Models Age

Embedding models have training data cutoffs too. An embedding model trained on data from 2023 won't have good representations for concepts that emerged in 2025. New jargon, new products, new cultural references may not be well-represented.

### 7.4 Dimensionality and Cost

Higher-dimensional embeddings capture more nuance but:
- Take more storage (each dimension is a float, typically 4 bytes)
- Are slower to compare (more multiplications per similarity calculation)
- Require more memory for search indices

1,536-dimensional embeddings × 1 million documents × 4 bytes/float = ~6 GB just for the vectors. At scale, this matters.

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

---

## 9. Word Embeddings vs Sentence Embeddings vs Document Embeddings

One distinction worth clarifying: embeddings operate at different levels:

**Word embeddings** (Word2Vec, GloVe): One vector per word. "dog" → one vector. Useful for word-level tasks but limited because you can't represent a whole sentence.

**Sentence embeddings** (Sentence Transformers, OpenAI's embedding models): One vector per sentence or short text. "The dog chased the cat" → one vector. What you'll use most often. These capture the full meaning of a sentence, including word order and context.

**Document embeddings**: One vector per longer text (paragraph, page, document). Same models can do this, but quality degrades with length. Common strategy: chunk the document, embed each chunk separately, then either search chunks individually or average the chunk embeddings.

For almost everything in this guide, we'll use **sentence/text embeddings** — models that take a piece of text (up to their token limit) and return one vector.

---

## 10. The Embedding Landscape (Quick Reference)

Here's a snapshot of embedding options you'll encounter:

### Closed-Source (API)

| Provider | Model | Dimensions | Max Tokens | Strengths |
|---|---|---|---|---|
| OpenAI | text-embedding-3-small | 1,536 | 8,191 | Cheap, good quality, adjustable dimensions |
| OpenAI | text-embedding-3-large | 3,072 | 8,191 | Higher quality, adjustable dimensions |
| Cohere | embed-v3 | 1,024 | 512 | Good multilingual, search-optimized |
| Google | text-embedding-004 | 768 | 2,048 | Integrated with Google Cloud |
| Voyage AI | voyage-3 | 1,024 | 32,000 | Very long context, strong for code |

### Open-Source (Self-Hosted)

| Model | Dimensions | Max Tokens | Strengths |
|---|---|---|---|
| all-MiniLM-L6-v2 | 384 | 512 | Fast, small, great for prototyping |
| bge-large-en-v1.5 | 1,024 | 512 | Strong quality, MTEB benchmark leader |
| E5-mistral-7b | 4,096 | 32,768 | Very long context support |
| nomic-embed-text | 768 | 8,192 | Good quality, fully open source |

We'll revisit these in Ch 14 (hands-on usage) and Ch 42 (deep dive on model selection and running your own).

---

## 11. Key Takeaways

1. **Embeddings turn meaning into math.** Text becomes a vector (list of numbers), similar meanings become similar vectors.

2. **Cosine similarity measures likeness.** Ranges from -1 (opposite) to 1 (identical). Most text comparisons fall between 0 and 1.

3. **Vector arithmetic captures relationships.** "King - Man + Woman ≈ Queen" demonstrates that directions in vector space represent semantic relationships.

4. **Embeddings enable semantic search.** Match by meaning, not keywords. This is the foundation of RAG.

5. **Embeddings are created by neural networks.** You don't need to understand the internals (yet). Just know that the models are trained to produce useful representations.

6. **Chunk long texts.** Embedding models have token limits, and shorter, focused texts produce better embeddings.

7. **Choose the right model for your task.** Smaller/faster models for prototyping, larger/better models for production. Always benchmark on your data.

---

## What's Next

In **Chapter 2: The AI Engineer's Landscape**, we'll survey the providers, pricing models, SDKs, and the build-vs-buy decision tree. You'll understand who the players are and how to choose between them.

Then in **Chapter 14**, we'll come back to embeddings with working code — generating embeddings with OpenAI's API, storing them in vector databases, and building semantic search.

And in **Chapter 42**, we'll spiral all the way to generating your own embeddings with open-source models, benchmarking them on MTEB, and choosing the optimal model for your use case.

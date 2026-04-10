<!--
  CHAPTER: 17
  TITLE: Advanced Retrieval
  PART: 3 — RAG & Knowledge Systems
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 14 (Semantic Search), Ch 15 (The RAG Pipeline), Ch 16 (Document QA)
  KEY_TOPICS: hybrid search, BM25, keyword search, reranking, cross-encoder, HyDE, hypothetical document embeddings, query expansion, metadata filtering, multi-index strategies
  DIFFICULTY: Intermediate to Advanced
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 17: Advanced Retrieval

> **Part 3 — RAG & Knowledge Systems** | Phase 1: Get Dangerous | Prerequisites: Ch 14-16 | Difficulty: Intermediate to Advanced | Language: TypeScript

You've built a working RAG pipeline. It loads documents, chunks them, embeds them, stores them, and retrieves relevant context. It works. But "works" and "works well" are very different things.

This chapter is about closing that gap. Every technique here directly improves the pipeline you built in Chapters 14-16. Hybrid search catches what semantic search misses. Reranking makes your top results more precise. HyDE finds answers that don't look like questions. Query expansion covers more ground. Metadata filtering narrows the search space. Multi-index strategies handle heterogeneous content.

Each technique is independently useful. Stack them together and your RAG pipeline goes from "pretty good" to "how did it know that?"

### In This Chapter

1. Hybrid search: combining keyword and semantic search
2. Reranking with cross-encoders
3. HyDE: Hypothetical Document Embeddings
4. Query expansion and reformulation
5. Metadata filtering
6. Multi-index strategies
7. Putting it all together: an advanced retrieval pipeline

### Related Chapters

- **Ch 14-16** — Each technique here improves what you built in those chapters.
- **Ch 42 (Embeddings & Sentence Transformers)** — Phase 2: generate better embeddings for better retrieval.
- **Ch 50 (Advanced Context Strategies)** — Phase 2: more sophisticated context management.
- **Ch 19 (Single-Turn Evals)** — Measure whether these techniques actually improve your results.

---

## 1. Hybrid Search: Keyword + Semantic

### 1.1 The Problem with Pure Semantic Search

Semantic search is powerful, but it has blind spots. It matches by *meaning*, which means it can miss when the user knows the exact term they want.

```
Query: "error code E-4032"
Semantic search returns: documents about error handling in general
What the user wants: the specific document that mentions E-4032
```

Keyword search has the opposite problem — it finds exact matches but misses synonyms and paraphrases. The solution: use both.

### 1.2 BM25: The Best Keyword Search Algorithm

BM25 (Best Matching 25) is the standard keyword search algorithm. It's what Elasticsearch uses under the hood. It's based on TF-IDF (term frequency-inverse document frequency) with some important improvements.

```typescript
// hybrid/bm25.ts

interface BM25Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

class BM25Index {
  private documents: BM25Document[] = [];
  private avgDocLength = 0;
  private termFrequencies: Map<string, Map<number, number>> = new Map();
  private documentFrequencies: Map<string, number> = new Map();
  private docLengths: number[] = [];

  // BM25 parameters
  private k1 = 1.2; // Term frequency saturation
  private b = 0.75; // Document length normalization

  private tokenize(text: string): string[] {
    return text
      .toLowerCase()
      .replace(/[^\w\s]/g, " ")
      .split(/\s+/)
      .filter(t => t.length > 1);
  }

  addDocuments(docs: BM25Document[]): void {
    const startIdx = this.documents.length;
    this.documents.push(...docs);

    for (let i = 0; i < docs.length; i++) {
      const docIdx = startIdx + i;
      const tokens = this.tokenize(docs[i].content);
      this.docLengths[docIdx] = tokens.length;

      // Count term frequencies in this document
      const termCounts = new Map<string, number>();
      for (const token of tokens) {
        termCounts.set(token, (termCounts.get(token) ?? 0) + 1);
      }

      // Update the index
      const seenTerms = new Set<string>();
      for (const [term, count] of termCounts) {
        if (!this.termFrequencies.has(term)) {
          this.termFrequencies.set(term, new Map());
        }
        this.termFrequencies.get(term)!.set(docIdx, count);

        if (!seenTerms.has(term)) {
          seenTerms.add(term);
          this.documentFrequencies.set(
            term, (this.documentFrequencies.get(term) ?? 0) + 1
          );
        }
      }
    }

    // Recalculate average document length
    this.avgDocLength =
      this.docLengths.reduce((sum, len) => sum + len, 0) / this.docLengths.length;
  }

  search(query: string, topK: number = 10): { document: BM25Document; score: number }[] {
    const queryTerms = this.tokenize(query);
    const scores = new Map<number, number>();
    const N = this.documents.length;

    for (const term of queryTerms) {
      const df = this.documentFrequencies.get(term) ?? 0;
      if (df === 0) continue;

      // IDF component
      const idf = Math.log((N - df + 0.5) / (df + 0.5) + 1);

      const termDocs = this.termFrequencies.get(term);
      if (!termDocs) continue;

      for (const [docIdx, tf] of termDocs) {
        const docLen = this.docLengths[docIdx];
        // BM25 scoring formula
        const numerator = tf * (this.k1 + 1);
        const denominator =
          tf + this.k1 * (1 - this.b + this.b * docLen / this.avgDocLength);
        const score = idf * (numerator / denominator);

        scores.set(docIdx, (scores.get(docIdx) ?? 0) + score);
      }
    }

    return Array.from(scores.entries())
      .map(([docIdx, score]) => ({
        document: this.documents[docIdx],
        score,
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }
}
```

### 1.3 Combining BM25 with Semantic Search

```typescript
// hybrid/hybrid-search.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface SearchResult {
  id: string;
  content: string;
  score: number;
  metadata: Record<string, unknown>;
}

interface HybridSearchConfig {
  semanticWeight: number; // 0-1, weight for semantic results
  keywordWeight: number;  // 0-1, weight for keyword results
  topK: number;
  rrf_k?: number; // Reciprocal Rank Fusion constant
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dot = 0, nA = 0, nB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    nA += a[i] * a[i];
    nB += b[i] * b[i];
  }
  return dot / (Math.sqrt(nA) * Math.sqrt(nB));
}

class HybridSearch {
  private bm25 = new BM25Index();
  private documents: {
    id: string;
    content: string;
    embedding: number[];
    metadata: Record<string, unknown>;
  }[] = [];

  async addDocuments(
    docs: { id: string; content: string; metadata: Record<string, unknown> }[]
  ) {
    // Add to BM25 index
    this.bm25.addDocuments(docs);

    // Generate embeddings
    const batchSize = 100;
    for (let i = 0; i < docs.length; i += batchSize) {
      const batch = docs.slice(i, i + batchSize);

      const response = await openai.embeddings.create({
        model: "text-embedding-3-small",
        input: batch.map(d => d.content),
      });

      for (let j = 0; j < batch.length; j++) {
        this.documents.push({
          ...batch[j],
          embedding: response.data[j].embedding,
        });
      }
    }
  }

  async search(query: string, config: HybridSearchConfig): Promise<SearchResult[]> {
    const {
      semanticWeight = 0.7,
      keywordWeight = 0.3,
      topK = 10,
      rrf_k = 60,
    } = config;
    const fetchK = topK * 3;

    // Run both searches in parallel
    const [semanticResults, keywordResults] = await Promise.all([
      this.semanticSearch(query, fetchK),
      Promise.resolve(this.keywordSearch(query, fetchK)),
    ]);

    // Use Reciprocal Rank Fusion to combine results
    const scores = new Map<string, number>();

    for (let i = 0; i < semanticResults.length; i++) {
      const id = semanticResults[i].id;
      const rrf = semanticWeight / (rrf_k + i + 1);
      scores.set(id, (scores.get(id) ?? 0) + rrf);
    }

    for (let i = 0; i < keywordResults.length; i++) {
      const id = keywordResults[i].id;
      const rrf = keywordWeight / (rrf_k + i + 1);
      scores.set(id, (scores.get(id) ?? 0) + rrf);
    }

    // Build final results
    const allResults = new Map<string, SearchResult>();
    for (const r of [...semanticResults, ...keywordResults]) {
      if (!allResults.has(r.id)) {
        allResults.set(r.id, r);
      }
    }

    return Array.from(scores.entries())
      .map(([id, score]) => {
        const result = allResults.get(id)!;
        return { ...result, score };
      })
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }

  private async semanticSearch(
    query: string, topK: number
  ): Promise<SearchResult[]> {
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: query,
    });
    const queryEmbedding = response.data[0].embedding;

    return this.documents
      .map(doc => ({
        id: doc.id,
        content: doc.content,
        score: cosineSimilarity(queryEmbedding, doc.embedding),
        metadata: doc.metadata,
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }

  private keywordSearch(query: string, topK: number): SearchResult[] {
    return this.bm25.search(query, topK).map(r => ({
      id: r.document.id,
      content: r.document.content,
      score: r.score,
      metadata: r.document.metadata,
    }));
  }
}

// Usage
async function main() {
  const search = new HybridSearch();

  await search.addDocuments([
    {
      id: "1",
      content: "Error code E-4032 indicates a network timeout during authentication",
      metadata: { type: "error-doc" },
    },
    {
      id: "2",
      content: "Authentication failures can occur when the server is unreachable",
      metadata: { type: "troubleshooting" },
    },
    {
      id: "3",
      content: "Network errors are common during peak traffic periods",
      metadata: { type: "general" },
    },
  ]);

  // This query benefits from hybrid: "E-4032" is a keyword match,
  // "network timeout" is a semantic match
  const results = await search.search("error E-4032", {
    semanticWeight: 0.5,
    keywordWeight: 0.5,
    topK: 5,
  });

  for (const r of results) {
    console.log(`${r.score.toFixed(4)} | ${r.id}: ${r.content.slice(0, 60)}...`);
  }
}
```

---

## 2. Reranking with Cross-Encoders

### 2.1 Why Rerank?

Bi-encoder search (what we've been doing) is fast but approximate. It embeds the query and documents independently and compares the vectors. A cross-encoder looks at the query and document *together*, which gives much better relevance scores — but it's too slow to run on every document.

The pattern: fetch a broad set of candidates with fast search, then rerank the top N with a more accurate model.

```typescript
// reranking/cross-encoder-rerank.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface RankedResult {
  id: string;
  content: string;
  originalScore: number;
  rerankScore: number;
  metadata: Record<string, unknown>;
}

// LLM-based cross-encoder reranking
async function crossEncoderRerank(
  query: string,
  candidates: {
    id: string;
    content: string;
    score: number;
    metadata: Record<string, unknown>;
  }[],
  topK: number = 5
): Promise<RankedResult[]> {
  // Score each candidate individually for best accuracy
  const scored: RankedResult[] = [];

  // Process in parallel batches for speed
  const batchSize = 5;
  for (let i = 0; i < candidates.length; i += batchSize) {
    const batch = candidates.slice(i, i + batchSize);

    const promises = batch.map(async (candidate) => {
      const response = await openai.chat.completions.create({
        model: "gpt-4o-mini",
        temperature: 0,
        response_format: { type: "json_object" },
        messages: [
          {
            role: "system",
            content: `You are a relevance judge. Given a query and a passage, rate the relevance of the passage to answering the query.

Score from 0 to 10:
- 0: Completely irrelevant
- 3: Marginally related but doesn't answer the query
- 5: Partially relevant, contains some useful information
- 7: Relevant, directly addresses the query
- 10: Perfectly relevant, contains the exact answer

Respond with JSON: { "score": number, "reason": string }`,
          },
          {
            role: "user",
            content: `Query: "${query}"\n\nPassage: "${candidate.content.slice(0, 1000)}"`,
          },
        ],
      });

      const parsed = JSON.parse(response.choices[0].message.content ?? '{"score":0}');
      return {
        id: candidate.id,
        content: candidate.content,
        originalScore: candidate.score,
        rerankScore: (parsed.score ?? 0) / 10,
        metadata: candidate.metadata,
      };
    });

    scored.push(...await Promise.all(promises));
  }

  return scored
    .sort((a, b) => b.rerankScore - a.rerankScore)
    .slice(0, topK);
}

// Cohere reranker (when you want a dedicated reranking model)
async function cohereRerank(
  query: string,
  documents: { id: string; content: string; metadata: Record<string, unknown> }[],
  topK: number = 5
): Promise<{ id: string; content: string; score: number; metadata: Record<string, unknown> }[]> {
  const response = await fetch("https://api.cohere.ai/v1/rerank", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.COHERE_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "rerank-english-v3.0",
      query,
      documents: documents.map(d => d.content),
      top_n: topK,
    }),
  });

  const data = await response.json() as {
    results: { index: number; relevance_score: number }[];
  };

  return data.results.map(r => ({
    id: documents[r.index].id,
    content: documents[r.index].content,
    score: r.relevance_score,
    metadata: documents[r.index].metadata,
  }));
}
```

### 2.2 Two-Stage Retrieval Pattern

```typescript
// reranking/two-stage.ts

interface TwoStageConfig {
  firstStageK: number;  // How many to fetch from vector store
  rerankK: number;      // How many to keep after reranking
  rerankMethod: "llm" | "cohere";
}

async function twoStageRetrieval(
  query: string,
  vectorSearch: (
    query: string,
    topK: number
  ) => Promise<{
    id: string;
    content: string;
    score: number;
    metadata: Record<string, unknown>;
  }[]>,
  config: TwoStageConfig = { firstStageK: 20, rerankK: 5, rerankMethod: "llm" }
): Promise<{
  id: string;
  content: string;
  score: number;
  metadata: Record<string, unknown>;
}[]> {
  // Stage 1: Broad retrieval (fast, approximate)
  const candidates = await vectorSearch(query, config.firstStageK);
  console.log(`Stage 1: Retrieved ${candidates.length} candidates`);

  // Stage 2: Precise reranking (slower, more accurate)
  if (config.rerankMethod === "cohere") {
    return cohereRerank(query, candidates, config.rerankK);
  }

  const reranked = await crossEncoderRerank(query, candidates, config.rerankK);
  return reranked.map(r => ({
    id: r.id,
    content: r.content,
    score: r.rerankScore,
    metadata: r.metadata,
  }));
}
```

---

## 3. HyDE: Hypothetical Document Embeddings

### 3.1 The Insight

Here's a subtle problem with semantic search: queries and documents don't look alike. A user's query is "How do I configure authentication?" but the document says "Authentication can be configured by setting the AUTH_SECRET environment variable..." The embedding spaces of *questions* and *answers* are different.

HyDE (Hypothetical Document Embeddings) fixes this by having the LLM generate a hypothetical answer, then searching for documents similar to that answer instead of the question.

```typescript
// hyde/hypothetical-document.ts
import OpenAI from "openai";

const openai = new OpenAI();

async function hydeSearch(
  query: string,
  vectorSearch: (
    embedding: number[],
    topK: number
  ) => Promise<{
    id: string;
    content: string;
    score: number;
    metadata: Record<string, unknown>;
  }[]>,
  options: { numHypothetical?: number; topK?: number } = {}
): Promise<{
  id: string;
  content: string;
  score: number;
  metadata: Record<string, unknown>;
}[]> {
  const { numHypothetical = 1, topK = 5 } = options;

  // Step 1: Generate hypothetical document(s)
  const hypotheticals = await generateHypotheticalDocuments(query, numHypothetical);
  console.log(`Generated ${hypotheticals.length} hypothetical document(s)`);

  // Step 2: Embed the hypothetical documents
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: hypotheticals,
  });

  // Step 3: Average the embeddings if multiple
  let searchEmbedding: number[];
  if (response.data.length === 1) {
    searchEmbedding = response.data[0].embedding;
  } else {
    const dim = response.data[0].embedding.length;
    searchEmbedding = new Array(dim).fill(0);
    for (const d of response.data) {
      for (let i = 0; i < dim; i++) {
        searchEmbedding[i] += d.embedding[i] / response.data.length;
      }
    }
  }

  // Step 4: Search with the hypothetical embedding
  return vectorSearch(searchEmbedding, topK);
}

async function generateHypotheticalDocuments(
  query: string,
  count: number = 1
): Promise<string[]> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.7,
    n: count,
    messages: [
      {
        role: "system",
        content: `You are a technical documentation writer. Given a question, write a short passage (2-3 paragraphs) that would answer it. Write as if you're writing the documentation itself, not answering the question.

Be specific and detailed. Include code examples if appropriate.`,
      },
      { role: "user", content: query },
    ],
  });

  return response.choices.map(c => c.message.content ?? "");
}

// Example
async function main() {
  // Without HyDE:
  // Query: "how do I handle rate limits?"
  // Searches for documents about "handling rate limits" (question-like)
  //
  // With HyDE:
  // Generates: "Rate limits can be handled using exponential backoff.
  //  When you receive a 429 status code, wait for the time specified
  //  in the Retry-After header..."
  // Searches for documents similar to this passage (answer-like)
  // This matches the actual documentation much better!

  const query = "how do I handle rate limits?";
  const hypotheticals = await generateHypotheticalDocuments(query, 2);

  console.log("Query:", query);
  console.log("\nHypothetical documents:");
  for (const h of hypotheticals) {
    console.log(`---\n${h}\n---`);
  }
}

main();
```

### 3.2 When to Use HyDE

**Use HyDE when:**
- Questions and documents have very different styles (user questions vs technical docs)
- Simple semantic search returns irrelevant results for well-formed questions
- Your document corpus is technical or domain-specific

**Don't use HyDE when:**
- Queries are already document-like (e.g., "error code E-4032")
- Latency is critical (HyDE adds an LLM call before search)
- Your corpus is small and search already works well

---

## 4. Query Expansion and Reformulation

### 4.1 Multi-Query Expansion

Instead of searching with one query, generate multiple variations and search with all of them.

```typescript
// expansion/multi-query.ts
import OpenAI from "openai";

const openai = new OpenAI();

async function expandQuery(
  query: string,
  numExpansions: number = 3
): Promise<string[]> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.7,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are a search query optimizer. Given a user's query, generate ${numExpansions} alternative search queries that would find relevant documents.

Each query should approach the topic from a different angle:
1. A more specific version
2. A more general/broader version
3. A version using different terminology/synonyms

Return JSON: { "queries": ["query1", "query2", "query3"] }`,
      },
      { role: "user", content: query },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
  return [query, ...(parsed.queries ?? [])]; // Always include the original
}

async function multiQuerySearch(
  query: string,
  vectorSearch: (
    query: string,
    topK: number
  ) => Promise<{ id: string; content: string; score: number }[]>,
  topK: number = 5
): Promise<{ id: string; content: string; score: number }[]> {
  // Step 1: Expand the query
  const queries = await expandQuery(query);
  console.log("Expanded queries:", queries);

  // Step 2: Search with each query
  const allResults = await Promise.all(
    queries.map(q => vectorSearch(q, topK))
  );

  // Step 3: Combine using RRF
  const scores = new Map<string, number>();
  const resultMap = new Map<string, { id: string; content: string; score: number }>();

  for (const results of allResults) {
    for (let rank = 0; rank < results.length; rank++) {
      const r = results[rank];
      const rrf = 1 / (60 + rank + 1);
      scores.set(r.id, (scores.get(r.id) ?? 0) + rrf);
      if (!resultMap.has(r.id)) resultMap.set(r.id, r);
    }
  }

  return Array.from(scores.entries())
    .map(([id, score]) => ({ ...resultMap.get(id)!, score }))
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}
```

### 4.2 Step-Back Prompting

For complex or specific questions, "step back" to a more general question first, retrieve context for both, then answer.

```typescript
// expansion/step-back.ts
import OpenAI from "openai";

const openai = new OpenAI();

async function stepBackQuery(query: string): Promise<string> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    messages: [
      {
        role: "system",
        content: `You are a search query optimizer. Given a specific question, generate a more general "step-back" question that would help provide broader context.

Examples:
- "What is the temperature range for the Tesla Model 3 battery?"
  -> "How do electric vehicle batteries handle temperature?"
- "Why did my pgvector query return empty results?"
  -> "How does pgvector indexing and querying work?"
- "What's the token limit for GPT-4 Turbo?"
  -> "What are the specifications and limits of GPT-4 models?"

Return ONLY the step-back question.`,
      },
      { role: "user", content: query },
    ],
  });

  return response.choices[0].message.content ?? query;
}

async function stepBackSearch(
  query: string,
  vectorSearch: (
    query: string,
    topK: number
  ) => Promise<{ id: string; content: string; score: number }[]>,
  topK: number = 5
): Promise<{ id: string; content: string; score: number }[]> {
  const stepBack = await stepBackQuery(query);
  console.log(`Original: "${query}"`);
  console.log(`Step-back: "${stepBack}"`);

  // Search with both the original and the step-back query
  const [originalResults, stepBackResults] = await Promise.all([
    vectorSearch(query, topK),
    vectorSearch(stepBack, topK),
  ]);

  // Combine: original results are weighted higher
  const scores = new Map<string, number>();
  const resultMap = new Map<string, { id: string; content: string; score: number }>();

  for (let i = 0; i < originalResults.length; i++) {
    const r = originalResults[i];
    scores.set(r.id, (scores.get(r.id) ?? 0) + 0.7 / (60 + i + 1));
    if (!resultMap.has(r.id)) resultMap.set(r.id, r);
  }

  for (let i = 0; i < stepBackResults.length; i++) {
    const r = stepBackResults[i];
    scores.set(r.id, (scores.get(r.id) ?? 0) + 0.3 / (60 + i + 1));
    if (!resultMap.has(r.id)) resultMap.set(r.id, r);
  }

  return Array.from(scores.entries())
    .map(([id, score]) => ({ ...resultMap.get(id)!, score }))
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}
```

---

## 5. Metadata Filtering

### 5.1 Pre-Filtering vs Post-Filtering

Metadata filtering narrows the search space *before* semantic similarity. This is critical for large corpora where you know the category, date range, or document type.

```typescript
// filtering/metadata-filter.ts

interface FilterExpression {
  field: string;
  operator: "eq" | "neq" | "gt" | "gte" | "lt" | "lte" | "in" | "contains";
  value: unknown;
}

interface FilterGroup {
  logic: "and" | "or";
  filters: (FilterExpression | FilterGroup)[];
}

function evaluateFilter(
  metadata: Record<string, unknown>,
  filter: FilterExpression
): boolean {
  const fieldValue = metadata[filter.field];
  if (fieldValue === undefined) return false;

  switch (filter.operator) {
    case "eq": return fieldValue === filter.value;
    case "neq": return fieldValue !== filter.value;
    case "gt": return (fieldValue as number) > (filter.value as number);
    case "gte": return (fieldValue as number) >= (filter.value as number);
    case "lt": return (fieldValue as number) < (filter.value as number);
    case "lte": return (fieldValue as number) <= (filter.value as number);
    case "in": return (filter.value as unknown[]).includes(fieldValue);
    case "contains":
      return typeof fieldValue === "string" &&
        fieldValue.toLowerCase().includes((filter.value as string).toLowerCase());
    default:
      return false;
  }
}

function evaluateFilterGroup(
  metadata: Record<string, unknown>,
  group: FilterGroup
): boolean {
  if (group.logic === "and") {
    return group.filters.every(f =>
      "logic" in f
        ? evaluateFilterGroup(metadata, f)
        : evaluateFilter(metadata, f)
    );
  } else {
    return group.filters.some(f =>
      "logic" in f
        ? evaluateFilterGroup(metadata, f)
        : evaluateFilter(metadata, f)
    );
  }
}
```

### 5.2 Auto-Extracting Filters from Queries

Let the LLM figure out what filters to apply based on the user's natural language query.

```typescript
// filtering/auto-filter.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface ExtractedFilters {
  searchQuery: string;
  filters: {
    field: string;
    operator: string;
    value: unknown;
  }[];
}

async function extractFiltersFromQuery(
  query: string,
  availableFields: {
    name: string;
    type: string;
    description: string;
    examples?: string[];
  }[]
): Promise<ExtractedFilters> {
  const fieldDescriptions = availableFields
    .map(f =>
      `- ${f.name} (${f.type}): ${f.description}` +
      (f.examples ? ` (e.g., ${f.examples.join(", ")})` : "")
    )
    .join("\n");

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are a query parser. Given a natural language query and available filter fields, extract any filters and return the remaining search query.

Available fields:
${fieldDescriptions}

Respond in JSON:
{
  "searchQuery": "the semantic search part of the query",
  "filters": [
    { "field": "fieldName", "operator": "eq|gt|lt|gte|lte|in|contains", "value": "value" }
  ]
}

If no filters can be extracted, return an empty filters array and the original query.`,
      },
      { role: "user", content: query },
    ],
  });

  return JSON.parse(
    response.choices[0].message.content ?? '{"searchQuery":"","filters":[]}'
  );
}

// Usage
async function main() {
  const fields = [
    {
      name: "year",
      type: "number",
      description: "Publication year",
      examples: ["2023", "2024"],
    },
    {
      name: "category",
      type: "string",
      description: "Document category",
      examples: ["guide", "api-reference", "tutorial"],
    },
    { name: "author", type: "string", description: "Author name" },
    {
      name: "language",
      type: "string",
      description: "Programming language",
      examples: ["typescript", "python"],
    },
  ];

  const result = await extractFiltersFromQuery(
    "TypeScript authentication guides from 2024",
    fields
  );

  console.log("Search query:", result.searchQuery);
  // "authentication guides"
  console.log("Filters:", result.filters);
  // [{ field: "language", operator: "eq", value: "typescript" },
  //  { field: "year", operator: "eq", value: 2024 },
  //  { field: "category", operator: "eq", value: "guide" }]
}

main();
```

---

## 6. Multi-Index Strategies

### 6.1 Why Multiple Indices?

Different types of content need different treatment. Code snippets, prose documentation, API references, and changelog entries all have different structures and should be chunked, embedded, and searched differently.

```typescript
// multi-index/index-router.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface IndexConfig {
  name: string;
  description: string;
  chunkSize: number;
  overlap: number;
  embeddingPrefix?: string;
}

const indices: Record<string, IndexConfig> = {
  prose: {
    name: "Prose Documentation",
    description: "Explanatory text, guides, tutorials, conceptual docs",
    chunkSize: 1000,
    overlap: 200,
  },
  code: {
    name: "Code Examples",
    description: "Code snippets, functions, classes, configuration files",
    chunkSize: 500,
    overlap: 50,
    embeddingPrefix: "Code example: ",
  },
  api: {
    name: "API Reference",
    description: "API endpoints, function signatures, parameters, return types",
    chunkSize: 300,
    overlap: 0,
    embeddingPrefix: "API reference: ",
  },
  changelog: {
    name: "Changelog & Updates",
    description: "Version history, breaking changes, new features",
    chunkSize: 500,
    overlap: 100,
    embeddingPrefix: "Changelog: ",
  },
};

// Route queries to the right index
async function routeQuery(
  query: string
): Promise<{ indexName: string; confidence: number }[]> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are a search router. Given a query, determine which documentation index(es) are most likely to contain the answer.

Available indices:
${Object.entries(indices).map(([k, v]) => `- ${k}: ${v.description}`).join("\n")}

Respond in JSON: { "indices": [{ "name": "indexName", "confidence": 0.0-1.0 }] }
Order by confidence, highest first. Include at most 3 indices.`,
      },
      { role: "user", content: query },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
  return parsed.indices ?? [{ name: "prose", confidence: 1.0 }];
}

// Search across multiple indices and merge results
async function multiIndexSearch(
  query: string,
  searchIndex: (
    indexName: string,
    query: string,
    topK: number
  ) => Promise<{
    id: string;
    content: string;
    score: number;
    metadata: Record<string, unknown>;
  }[]>,
  topK: number = 5
): Promise<{
  id: string;
  content: string;
  score: number;
  metadata: Record<string, unknown>;
  indexName: string;
}[]> {
  // Route the query
  const routing = await routeQuery(query);
  console.log(
    "Routing:",
    routing.map(r => `${r.indexName} (${r.confidence})`).join(", ")
  );

  // Search each relevant index
  const allResults: {
    id: string;
    content: string;
    score: number;
    metadata: Record<string, unknown>;
    indexName: string;
  }[] = [];

  for (const route of routing) {
    const perIndexK = Math.ceil(topK * route.confidence);
    const results = await searchIndex(route.indexName, query, perIndexK);

    for (const r of results) {
      allResults.push({
        ...r,
        score: r.score * route.confidence,
        indexName: route.indexName,
      });
    }
  }

  return allResults
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}
```

---

## 7. Putting It All Together

Here's the advanced retrieval pipeline that combines everything.

```typescript
// advanced-retrieval-pipeline.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface RetrievalResult {
  id: string;
  content: string;
  score: number;
  metadata: Record<string, unknown>;
  strategy: string;
}

interface AdvancedRetrievalConfig {
  useHybridSearch: boolean;
  useHyDE: boolean;
  useQueryExpansion: boolean;
  useReranking: boolean;
  useMetadataFiltering: boolean;
  semanticWeight: number;
  keywordWeight: number;
  firstStageK: number;
  finalK: number;
}

const defaultConfig: AdvancedRetrievalConfig = {
  useHybridSearch: true,
  useHyDE: false,
  useQueryExpansion: true,
  useReranking: true,
  useMetadataFiltering: true,
  semanticWeight: 0.7,
  keywordWeight: 0.3,
  firstStageK: 20,
  finalK: 5,
};

class AdvancedRetriever {
  private config: AdvancedRetrievalConfig;
  private vectorSearch: (
    embedding: number[],
    topK: number,
    filter?: Record<string, unknown>
  ) => Promise<RetrievalResult[]>;
  private keywordSearch: (
    query: string,
    topK: number
  ) => Promise<RetrievalResult[]>;

  constructor(
    vectorSearch: (
      embedding: number[],
      topK: number,
      filter?: Record<string, unknown>
    ) => Promise<RetrievalResult[]>,
    keywordSearch: (
      query: string,
      topK: number
    ) => Promise<RetrievalResult[]>,
    config: Partial<AdvancedRetrievalConfig> = {}
  ) {
    this.config = { ...defaultConfig, ...config };
    this.vectorSearch = vectorSearch;
    this.keywordSearch = keywordSearch;
  }

  async retrieve(query: string): Promise<RetrievalResult[]> {
    const startTime = Date.now();
    let searchQuery = query;
    let filters: Record<string, unknown> | undefined;

    // Step 1: Extract metadata filters
    if (this.config.useMetadataFiltering) {
      const extracted = await this.extractFilters(query);
      searchQuery = extracted.searchQuery || query;
      if (Object.keys(extracted.filters).length > 0) {
        filters = extracted.filters;
      }
      console.log(`Filters: ${JSON.stringify(filters)}, Query: "${searchQuery}"`);
    }

    // Step 2: Expand the query
    let queries = [searchQuery];
    if (this.config.useQueryExpansion) {
      queries = await this.expandQueries(searchQuery);
      console.log(`Expanded to ${queries.length} queries`);
    }

    // Step 3: Generate embeddings (optionally with HyDE)
    const embeddings = await this.generateEmbeddings(queries);

    // Step 4: Search (hybrid or semantic only)
    const candidates = await this.search(queries, embeddings, filters);
    console.log(`Stage 1: ${candidates.length} candidates (${Date.now() - startTime}ms)`);

    // Step 5: Rerank
    let finalResults: RetrievalResult[];
    if (this.config.useReranking && candidates.length > this.config.finalK) {
      finalResults = await this.rerank(query, candidates);
      console.log(`Reranked to ${finalResults.length} results (${Date.now() - startTime}ms)`);
    } else {
      finalResults = candidates.slice(0, this.config.finalK);
    }

    console.log(`Total retrieval: ${Date.now() - startTime}ms`);
    return finalResults;
  }

  private async extractFilters(
    query: string
  ): Promise<{ searchQuery: string; filters: Record<string, unknown> }> {
    const response = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata filters from this search query. Return JSON with "searchQuery" (the semantic part) and "filters" (a flat object of field:value pairs). If no filters, return empty filters object.`,
        },
        { role: "user", content: query },
      ],
    });
    return JSON.parse(
      response.choices[0].message.content ?? '{"searchQuery":"","filters":{}}'
    );
  }

  private async expandQueries(query: string): Promise<string[]> {
    const response = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0.5,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Generate 2 alternative search queries for better retrieval. Return JSON: { "queries": ["alt1", "alt2"] }`,
        },
        { role: "user", content: query },
      ],
    });
    const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
    return [query, ...(parsed.queries ?? [])];
  }

  private async generateEmbeddings(queries: string[]): Promise<number[][]> {
    let inputTexts = queries;

    if (this.config.useHyDE) {
      const hypotheticals: string[] = [];
      for (const q of queries) {
        const response = await openai.chat.completions.create({
          model: "gpt-4o-mini",
          temperature: 0.5,
          messages: [
            {
              role: "system",
              content: "Write a short passage (1-2 paragraphs) that answers this question as if it were documentation.",
            },
            { role: "user", content: q },
          ],
        });
        hypotheticals.push(response.choices[0].message.content ?? q);
      }
      inputTexts = hypotheticals;
    }

    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: inputTexts,
    });

    return response.data.map(d => d.embedding);
  }

  private async search(
    queries: string[],
    embeddings: number[][],
    filters?: Record<string, unknown>
  ): Promise<RetrievalResult[]> {
    const fetchK = this.config.firstStageK;

    // Semantic search with each embedding
    const semanticPromises = embeddings.map(emb =>
      this.vectorSearch(emb, fetchK, filters)
    );

    // Keyword search (if hybrid)
    const keywordPromises = this.config.useHybridSearch
      ? queries.map(q => this.keywordSearch(q, fetchK))
      : [];

    const [semanticBatches, keywordBatches] = await Promise.all([
      Promise.all(semanticPromises),
      Promise.all(keywordPromises),
    ]);

    // Flatten and tag with strategy
    const allResults: RetrievalResult[] = [];
    for (const batch of semanticBatches) {
      allResults.push(...batch.map(r => ({ ...r, strategy: "semantic" })));
    }
    for (const batch of keywordBatches) {
      allResults.push(...batch.map(r => ({ ...r, strategy: "keyword" })));
    }

    // Deduplicate and combine scores using RRF
    const scores = new Map<string, number>();
    const resultMap = new Map<string, RetrievalResult>();

    const byStrategy = new Map<string, RetrievalResult[]>();
    for (const r of allResults) {
      if (!byStrategy.has(r.strategy)) byStrategy.set(r.strategy, []);
      byStrategy.get(r.strategy)!.push(r);
      if (!resultMap.has(r.id)) resultMap.set(r.id, r);
    }

    for (const [strategy, results] of byStrategy) {
      const weight = strategy === "semantic"
        ? this.config.semanticWeight
        : this.config.keywordWeight;
      results.sort((a, b) => b.score - a.score);
      for (let i = 0; i < results.length; i++) {
        const id = results[i].id;
        scores.set(id, (scores.get(id) ?? 0) + weight / (60 + i + 1));
      }
    }

    return Array.from(scores.entries())
      .map(([id, score]) => ({ ...resultMap.get(id)!, score }))
      .sort((a, b) => b.score - a.score)
      .slice(0, fetchK);
  }

  private async rerank(
    query: string,
    candidates: RetrievalResult[]
  ): Promise<RetrievalResult[]> {
    const batchSize = 10;
    const toRerank = candidates.slice(0, batchSize);

    const passagesText = toRerank
      .map((c, i) => `Passage ${i}: ${c.content.slice(0, 500)}`)
      .join("\n\n");

    const response = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Rate each passage's relevance to the query on 0-10. Return JSON: { "scores": [{"index": 0, "score": 8}, ...] }`,
        },
        {
          role: "user",
          content: `Query: "${query}"\n\n${passagesText}`,
        },
      ],
    });

    const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
    const rerankScores = parsed.scores ?? [];

    return toRerank
      .map((candidate, i) => {
        const entry = rerankScores.find((s: { index: number }) => s.index === i);
        return {
          ...candidate,
          score: entry ? entry.score / 10 : candidate.score,
        };
      })
      .sort((a, b) => b.score - a.score)
      .slice(0, this.config.finalK);
  }
}

// --- Usage ---

async function main() {
  // In a real app, these would be backed by a vector store and BM25 index
  const vectorSearch = async (
    _embedding: number[],
    _topK: number
  ) => [] as RetrievalResult[];

  const keywordSearch = async (
    _query: string,
    _topK: number
  ) => [] as RetrievalResult[];

  const retriever = new AdvancedRetriever(vectorSearch, keywordSearch, {
    useHybridSearch: true,
    useQueryExpansion: true,
    useReranking: true,
    useHyDE: false,
    firstStageK: 20,
    finalK: 5,
  });

  const results = await retriever.retrieve(
    "How do I configure authentication in Next.js?"
  );

  for (const r of results) {
    console.log(`[${r.score.toFixed(3)}] ${r.content.slice(0, 80)}...`);
  }
}

main();
```

---

## Key Takeaways

You now have a toolkit of advanced retrieval techniques:

1. **Hybrid search** — BM25 keyword search + semantic search with RRF fusion
2. **Reranking** — Two-stage retrieval with LLM or Cohere cross-encoder reranking
3. **HyDE** — Generate hypothetical documents to bridge the question-answer gap
4. **Query expansion** — Search with multiple query variations for better recall
5. **Step-back prompting** — Broader queries for more context
6. **Metadata filtering** — Auto-extract filters from natural language queries
7. **Multi-index strategies** — Route queries to specialized indices

Each technique independently improves retrieval quality. Stack them together for the best results. But always measure — Chapter 19 shows you how to assess whether these techniques actually help your specific use case.

---

*Previous: [Ch 16 — Document QA Systems](./16-document-qa.md)* | *Next: [Ch 18 — Why Assessments Matter](../part-4-evals-quality/18-why-evals-matter.md)*

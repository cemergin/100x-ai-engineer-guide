<!--
  CHAPTER: 14
  TITLE: Semantic Search
  PART: 3 — RAG & Knowledge Systems
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 1 (Embeddings & Similarity), Ch 5 (Structured Output)
  KEY_TOPICS: embeddings API, vector stores, cosine similarity, Pinecone, Upstash Vector, pgvector, similarity search, scoring, ranking
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 14: Semantic Search

> Part 3: RAG & Knowledge Systems · Phase 1: Get Dangerous · Prerequisites: Ch 1, Ch 5 · Difficulty: Intermediate · Language: TypeScript

In Chapter 1 you learned that embeddings turn words into numbers and that similar things end up close together. That was the theory. Now you're going to *build* with it. By the end of this chapter, you'll have a working semantic search system that finds movies by meaning — not just keywords. Type "a movie about a fish trying to find his son" and it returns *Finding Nemo*, even though none of those exact words appear in the description.

This is the foundation that everything in Part 3 builds on. RAG is just "search for relevant stuff, then give it to an LLM." The search part is what we're doing right now.

### In This Chapter

1. Calling the embeddings API
2. Understanding the vector space
3. In-memory vector search
4. Vector stores: Pinecone, Upstash Vector, pgvector
5. Building a movie recommendation engine
6. Scoring and ranking results
7. Performance and cost considerations

### Related Chapters

- **Ch 1 (Embeddings & Similarity)** — The conceptual foundation. Here we turn it into code.
- **Ch 5 (Structured Output)** — We'll use Zod schemas for type-safe search results.
- **Ch 15 (The RAG Pipeline)** — Takes this search and wraps it in a full retrieve-then-generate system.
- **Ch 17 (Advanced Retrieval)** — Optimizes the retrieval strategies you build here.
- **Ch 42 (Embeddings & Sentence Transformers)** — Phase 2 spirals back: generate your own embeddings instead of using an API.

---

## 1. Calling the Embeddings API

### 1.1 What You Get Back

When you call an embeddings API, you send in text and get back a vector — an array of floating-point numbers. Each number represents one dimension of meaning. The OpenAI `text-embedding-3-small` model returns 1536 dimensions. Claude doesn't have a native embeddings endpoint, so we'll use OpenAI's — this is perfectly normal. Most production systems mix providers.

**What it is:** An API that converts text into a fixed-length vector of numbers.

**When to use:** Whenever you need to compare the *meaning* of two pieces of text, not just their characters.

**Real-world example:** Stripe uses embeddings to match support tickets to documentation pages, even when users describe their problem in completely different words than the docs use.

```typescript
// embeddings-basic.ts
import OpenAI from "openai";

const openai = new OpenAI(); // Uses OPENAI_API_KEY env var

// Generate an embedding for a single piece of text
async function embed(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
  });

  return response.data[0].embedding;
}

// Let's see what an embedding looks like
async function main() {
  const vector = await embed("A movie about a fish trying to find his son");

  console.log(`Dimensions: ${vector.length}`);       // 1536
  console.log(`First 5 values: ${vector.slice(0, 5)}`);
  // Something like: [0.0123, -0.0456, 0.0789, -0.0012, 0.0345]
  console.log(`All values are numbers: ${vector.every(v => typeof v === "number")}`);
}

main();
```

### 1.2 Batch Embeddings

You'll almost never embed one text at a time. The API accepts arrays, which is faster and cheaper.

```typescript
// embeddings-batch.ts
import OpenAI from "openai";

const openai = new OpenAI();

async function embedBatch(texts: string[]): Promise<number[][]> {
  // The API accepts up to 2048 inputs at once
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: texts,
  });

  // Results come back in the same order as inputs
  return response.data
    .sort((a, b) => a.index - b.index)
    .map(d => d.embedding);
}

async function main() {
  const texts = [
    "A clownfish searches the ocean for his kidnapped son",
    "A young lion must reclaim his throne after his father's murder",
    "A rat secretly becomes the chef at a Paris restaurant",
    "A robot left alone on Earth falls in love",
    "Toys come alive when humans aren't looking",
  ];

  const vectors = await embedBatch(texts);

  console.log(`Embedded ${vectors.length} texts`);
  console.log(`Each vector has ${vectors[0].length} dimensions`);
}

main();
```

### 1.3 Choosing an Embedding Model

**Trade-offs:**

| Model | Dimensions | Cost per 1M tokens | Quality | Speed |
|-------|-----------|-------------------|---------|-------|
| `text-embedding-3-small` | 1536 | $0.02 | Good | Fast |
| `text-embedding-3-large` | 3072 | $0.13 | Better | Slower |
| `text-embedding-ada-002` | 1536 | $0.10 | Legacy | Fast |

For Phase 1, `text-embedding-3-small` is the right choice. It's cheap, fast, and good enough for most applications. In Phase 2 (Ch 42), we'll explore generating embeddings locally with Sentence Transformers — zero API cost, full control.

```typescript
// You can also reduce dimensions for text-embedding-3-* models
// Smaller vectors = less storage + faster search, at the cost of some quality
const response = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: "A movie about space cowboys",
  dimensions: 512, // Reduce from 1536 to 512
});
```

---

## 2. Understanding the Vector Space

### 2.1 Cosine Similarity

In Chapter 1 you learned the concept. Here's the implementation. Cosine similarity measures the angle between two vectors. A value of 1 means identical direction (very similar), 0 means perpendicular (unrelated), and -1 means opposite.

```typescript
// cosine-similarity.ts

function cosineSimilarity(a: number[], b: number[]): number {
  if (a.length !== b.length) {
    throw new Error(`Vector dimension mismatch: ${a.length} vs ${b.length}`);
  }

  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  const denominator = Math.sqrt(normA) * Math.sqrt(normB);
  if (denominator === 0) return 0;

  return dotProduct / denominator;
}

// Test it
const similar1 = [1, 0, 1, 0];
const similar2 = [0.9, 0.1, 0.8, 0.1];
const different = [0, 1, 0, 1];

console.log(cosineSimilarity(similar1, similar2)); // ~0.98 (very similar)
console.log(cosineSimilarity(similar1, different)); // ~0.0 (unrelated)
```

### 2.2 Other Distance Metrics

Cosine similarity is the most common, but not the only option.

```typescript
// distance-metrics.ts

// Euclidean distance: straight-line distance in vector space
// Lower = more similar (opposite of cosine similarity)
function euclideanDistance(a: number[], b: number[]): number {
  let sum = 0;
  for (let i = 0; i < a.length; i++) {
    sum += (a[i] - b[i]) ** 2;
  }
  return Math.sqrt(sum);
}

// Dot product: raw magnitude * direction
// Higher = more similar, but affected by vector length
function dotProduct(a: number[], b: number[]): number {
  let sum = 0;
  for (let i = 0; i < a.length; i++) {
    sum += a[i] * b[i];
  }
  return sum;
}
```

**When to use which:**

| Metric | Best For | Notes |
|--------|---------|-------|
| Cosine similarity | General text search | Normalized, ignores magnitude |
| Euclidean distance | When magnitude matters | Clustering, anomaly detection |
| Dot product | When vectors are already normalized | Fastest to compute |

For semantic search, always start with cosine similarity. OpenAI embeddings are already normalized, so dot product gives the same ranking — but cosine similarity is more readable.

---

## 3. In-Memory Vector Search

Before we add databases, let's build search from scratch. This is the "first principles" version — you'll understand exactly what every vector store is doing under the hood.

### 3.1 The Simplest Possible Search

```typescript
// in-memory-search.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface Document {
  id: string;
  text: string;
  embedding: number[];
  metadata?: Record<string, unknown>;
}

// Our "database" — just an array
const documents: Document[] = [];

function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;
  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}

async function addDocument(id: string, text: string, metadata?: Record<string, unknown>) {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
  });

  documents.push({
    id,
    text,
    embedding: response.data[0].embedding,
    metadata,
  });
}

interface SearchResult {
  id: string;
  text: string;
  score: number;
  metadata?: Record<string, unknown>;
}

async function search(query: string, topK: number = 5): Promise<SearchResult[]> {
  // 1. Embed the query
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const queryEmbedding = response.data[0].embedding;

  // 2. Compare against every document
  const scored = documents.map(doc => ({
    id: doc.id,
    text: doc.text,
    score: cosineSimilarity(queryEmbedding, doc.embedding),
    metadata: doc.metadata,
  }));

  // 3. Sort by similarity (highest first) and return top K
  return scored
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}

async function main() {
  // Add some movies
  await addDocument("nemo", "A clownfish named Marlin searches the ocean for his son Nemo who was captured by a diver");
  await addDocument("lion-king", "A young lion prince flees his kingdom after his father's murder and must return to reclaim his throne");
  await addDocument("ratatouille", "A rat with a passion for cooking secretly controls a young chef at a prestigious Paris restaurant");
  await addDocument("wall-e", "A lonely robot left on a deserted Earth falls in love with a sleek probe robot");
  await addDocument("toy-story", "A cowboy doll is threatened by the arrival of a spaceman action figure");
  await addDocument("up", "An elderly widower ties balloons to his house and flies to South America with a young stowaway");
  await addDocument("inside-out", "Personified emotions inside a girl's mind navigate her move to a new city");
  await addDocument("coco", "A boy journeys to the Land of the Dead to find his great-great-grandfather, a legendary musician");

  // Search!
  const results = await search("a movie about a fish trying to find his son");

  console.log("Query: 'a movie about a fish trying to find his son'\n");
  for (const result of results) {
    console.log(`  ${result.score.toFixed(4)} | ${result.id}: ${result.text.slice(0, 80)}...`);
  }
}

main();
```

**Expected output:**
```
Query: 'a movie about a fish trying to find his son'

  0.8934 | nemo: A clownfish named Marlin searches the ocean for his son Nemo who was capt...
  0.6821 | coco: A boy journeys to the Land of the Dead to find his great-great-grandfath...
  0.6543 | lion-king: A young lion prince flees his kingdom after his father's murder and mu...
  0.6102 | up: An elderly widower ties balloons to his house and flies to South Amer...
  0.5987 | inside-out: Personified emotions inside a girl's mind navigate her move to a new...
```

Notice: "Finding Nemo" scores highest even though the query says "fish" and the document says "clownfish." That's the power of semantic search — it understands *meaning*, not just keyword overlap.

### 3.2 Adding Metadata Filters

Real search systems need filtering. You don't want to search all documents — you want movies from a specific year, or articles from a specific category.

```typescript
// in-memory-search-filtered.ts

interface Movie {
  id: string;
  title: string;
  description: string;
  year: number;
  genre: string;
  rating: number;
  embedding: number[];
}

const movies: Movie[] = [];

async function addMovie(movie: Omit<Movie, "embedding">) {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: `${movie.title}: ${movie.description}`,
  });

  movies.push({
    ...movie,
    embedding: response.data[0].embedding,
  });
}

interface FilterOptions {
  yearMin?: number;
  yearMax?: number;
  genre?: string;
  ratingMin?: number;
}

async function searchMovies(
  query: string,
  filters: FilterOptions = {},
  topK: number = 5
): Promise<(Movie & { score: number })[]> {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const queryEmbedding = response.data[0].embedding;

  return movies
    // Pre-filter by metadata
    .filter(movie => {
      if (filters.yearMin && movie.year < filters.yearMin) return false;
      if (filters.yearMax && movie.year > filters.yearMax) return false;
      if (filters.genre && movie.genre !== filters.genre) return false;
      if (filters.ratingMin && movie.rating < filters.ratingMin) return false;
      return true;
    })
    // Score by similarity
    .map(movie => ({
      ...movie,
      score: cosineSimilarity(queryEmbedding, movie.embedding),
    }))
    // Sort and limit
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}

// Usage:
// searchMovies("adventure in space", { yearMin: 2000, genre: "sci-fi" })
```

### 3.3 Why In-Memory Won't Scale

This works beautifully for hundreds or even a few thousand documents. But there are hard limits:

- **Memory:** 1536 dimensions x 4 bytes x 1M documents = ~6 GB of vectors alone
- **Speed:** Comparing against every document is O(n). At 1M documents, each search scans all of them
- **Persistence:** Everything disappears when the process exits

That's why vector databases exist. They use approximate nearest neighbor (ANN) algorithms to search billions of vectors in milliseconds.

---

## 4. Vector Stores

### 4.1 Pinecone

Pinecone is the most popular managed vector database. Zero infrastructure, scale-to-zero pricing, and a clean API.

```typescript
// pinecone-search.ts
import { Pinecone } from "@pinecone-database/pinecone";
import OpenAI from "openai";

const openai = new OpenAI();
const pinecone = new Pinecone(); // Uses PINECONE_API_KEY env var

// Create an index (do this once)
async function createIndex() {
  await pinecone.createIndex({
    name: "movies",
    dimension: 1536, // Must match your embedding model
    metric: "cosine",
    spec: {
      serverless: {
        cloud: "aws",
        region: "us-east-1",
      },
    },
  });
}

// Upsert documents
async function indexMovies(movies: { id: string; text: string; metadata: Record<string, unknown> }[]) {
  const index = pinecone.index("movies");

  // Embed all texts
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: movies.map(m => m.text),
  });

  // Prepare vectors
  const vectors = movies.map((movie, i) => ({
    id: movie.id,
    values: response.data[i].embedding,
    metadata: {
      ...movie.metadata,
      text: movie.text, // Store the text for retrieval
    },
  }));

  // Upsert in batches of 100
  const batchSize = 100;
  for (let i = 0; i < vectors.length; i += batchSize) {
    await index.upsert(vectors.slice(i, i + batchSize));
  }
}

// Search
async function searchPinecone(query: string, topK: number = 5) {
  const index = pinecone.index("movies");

  // Embed the query
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });

  // Query the index
  const results = await index.query({
    vector: response.data[0].embedding,
    topK,
    includeMetadata: true,
  });

  return results.matches?.map(match => ({
    id: match.id,
    score: match.score ?? 0,
    text: match.metadata?.text as string,
    metadata: match.metadata,
  })) ?? [];
}

async function main() {
  const results = await searchPinecone("a movie about a fish trying to find his son");
  for (const r of results) {
    console.log(`${r.score.toFixed(4)} | ${r.id}: ${r.text?.slice(0, 60)}`);
  }
}
```

### 4.2 Upstash Vector

Upstash Vector is a serverless vector database with a generous free tier. Great for side projects and startups. It has a unique feature: built-in embedding — you can send raw text and it embeds for you.

```typescript
// upstash-search.ts
import { Index } from "@upstash/vector";

// Option 1: Bring your own embeddings
const index = new Index({
  url: process.env.UPSTASH_VECTOR_REST_URL!,
  token: process.env.UPSTASH_VECTOR_REST_TOKEN!,
});

// Upsert with pre-computed embeddings
async function upsertWithEmbeddings(
  id: string,
  embedding: number[],
  metadata: Record<string, unknown>
) {
  await index.upsert({
    id,
    vector: embedding,
    metadata,
  });
}

// Option 2: Let Upstash embed for you (uses their built-in model)
// Create the index with an embedding model in the Upstash dashboard
const indexWithEmbedding = new Index({
  url: process.env.UPSTASH_VECTOR_REST_URL!,
  token: process.env.UPSTASH_VECTOR_REST_TOKEN!,
});

async function upsertWithAutoEmbed(id: string, text: string, metadata: Record<string, unknown>) {
  await indexWithEmbedding.upsert({
    id,
    data: text, // Upstash embeds this for you
    metadata: { ...metadata, text },
  });
}

// Search (with auto-embedding)
async function searchUpstash(query: string, topK: number = 5) {
  const results = await indexWithEmbedding.query({
    data: query, // Upstash embeds the query too
    topK,
    includeMetadata: true,
  });

  return results.map(r => ({
    id: r.id,
    score: r.score,
    metadata: r.metadata,
  }));
}

// Search with metadata filter
async function searchFiltered(query: string, genre: string) {
  const results = await indexWithEmbedding.query({
    data: query,
    topK: 5,
    includeMetadata: true,
    filter: `genre = '${genre}'`,
  });

  return results;
}
```

### 4.3 pgvector (PostgreSQL)

If you already have a PostgreSQL database, pgvector lets you add vector search without another service. This is often the best choice for production — one fewer thing to manage.

```typescript
// pgvector-search.ts
import pg from "pg";
import OpenAI from "openai";

const openai = new OpenAI();
const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
});

// Setup (run once)
async function setup() {
  const client = await pool.connect();
  try {
    // Enable the extension
    await client.query("CREATE EXTENSION IF NOT EXISTS vector");

    // Create the table
    await client.query(`
      CREATE TABLE IF NOT EXISTS movies (
        id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        description TEXT NOT NULL,
        year INTEGER,
        genre TEXT,
        rating REAL,
        embedding vector(1536)
      )
    `);

    // Create an index for fast similarity search
    // ivfflat is good for up to ~1M vectors
    await client.query(`
      CREATE INDEX IF NOT EXISTS movies_embedding_idx
      ON movies
      USING ivfflat (embedding vector_cosine_ops)
      WITH (lists = 100)
    `);
  } finally {
    client.release();
  }
}

// Insert a movie
async function insertMovie(movie: {
  id: string;
  title: string;
  description: string;
  year: number;
  genre: string;
  rating: number;
}) {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: `${movie.title}: ${movie.description}`,
  });

  const embedding = response.data[0].embedding;
  // pgvector expects the vector as a string: '[0.1, 0.2, ...]'
  const vectorStr = `[${embedding.join(",")}]`;

  await pool.query(
    `INSERT INTO movies (id, title, description, year, genre, rating, embedding)
     VALUES ($1, $2, $3, $4, $5, $6, $7)
     ON CONFLICT (id) DO UPDATE SET embedding = $7`,
    [movie.id, movie.title, movie.description, movie.year, movie.genre, movie.rating, vectorStr]
  );
}

// Search with combined vector similarity + metadata filters
async function searchMovies(
  query: string,
  options: { genre?: string; yearMin?: number; topK?: number } = {}
) {
  const { genre, yearMin, topK = 5 } = options;

  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });

  const embedding = response.data[0].embedding;
  const vectorStr = `[${embedding.join(",")}]`;

  // Build the query with optional filters
  let whereClause = "";
  const params: unknown[] = [vectorStr, topK];
  let paramIndex = 3;

  const conditions: string[] = [];
  if (genre) {
    conditions.push(`genre = $${paramIndex}`);
    params.push(genre);
    paramIndex++;
  }
  if (yearMin) {
    conditions.push(`year >= $${paramIndex}`);
    params.push(yearMin);
    paramIndex++;
  }

  if (conditions.length > 0) {
    whereClause = `WHERE ${conditions.join(" AND ")}`;
  }

  const result = await pool.query(
    `SELECT id, title, description, year, genre, rating,
            1 - (embedding <=> $1::vector) as score
     FROM movies
     ${whereClause}
     ORDER BY embedding <=> $1::vector
     LIMIT $2`,
    params
  );

  return result.rows;
}

// Usage
async function main() {
  await setup();

  await insertMovie({
    id: "nemo",
    title: "Finding Nemo",
    description: "A clownfish named Marlin searches the ocean for his son Nemo",
    year: 2003,
    genre: "animation",
    rating: 8.2,
  });

  const results = await searchMovies("a movie about a fish finding his son", {
    genre: "animation",
    topK: 5,
  });

  for (const r of results) {
    console.log(`${r.score.toFixed(4)} | ${r.title} (${r.year})`);
  }
}
```

### 4.4 Choosing a Vector Store

**Decision matrix:**

| Criteria | In-Memory | Pinecone | Upstash Vector | pgvector |
|----------|-----------|----------|----------------|----------|
| Setup effort | Zero | Low | Low | Medium |
| Max documents | ~10K | Billions | Millions | Millions |
| Cost | Free | Pay per use | Free tier + pay | Postgres cost |
| Extra infra | None | Yes | No (serverless) | Your Postgres |
| Metadata filtering | Manual | Built-in | Built-in | SQL powers |
| Best for | Prototyping | Scale | Serverless apps | Existing Postgres |

**The recommendation:** Start with in-memory for prototyping. Move to pgvector if you already have Postgres, or Upstash Vector for serverless. Use Pinecone if you need to scale past millions of documents.

---

## 5. Building a Movie Recommendation Engine

Let's put it all together with a complete, working movie search system. This example is inspired by Scott Moss's Frontend Masters course on AI — it's the best way to internalize semantic search because the results are intuitive.

```typescript
// movie-search-engine.ts
import OpenAI from "openai";
import { z } from "zod";

const openai = new OpenAI();

// --- Types ---

const MovieSchema = z.object({
  id: z.string(),
  title: z.string(),
  description: z.string(),
  year: z.number(),
  genre: z.string(),
  director: z.string(),
  rating: z.number(),
});

type Movie = z.infer<typeof MovieSchema>;

const SearchResultSchema = z.object({
  movie: MovieSchema,
  score: z.number(),
  rank: z.number(),
});

type SearchResult = z.infer<typeof SearchResultSchema>;

// --- Vector Store (in-memory for this example) ---

interface StoredMovie extends Movie {
  embedding: number[];
}

class MovieSearchEngine {
  private movies: StoredMovie[] = [];
  private openai: OpenAI;

  constructor() {
    this.openai = new OpenAI();
  }

  private cosineSimilarity(a: number[], b: number[]): number {
    let dot = 0, normA = 0, normB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    return dot / (Math.sqrt(normA) * Math.sqrt(normB));
  }

  async addMovies(movies: Movie[]): Promise<void> {
    // Embed in batches
    const batchSize = 100;
    for (let i = 0; i < movies.length; i += batchSize) {
      const batch = movies.slice(i, i + batchSize);
      const texts = batch.map(m => `${m.title} (${m.year}): ${m.description}. Directed by ${m.director}. Genre: ${m.genre}.`);

      const response = await this.openai.embeddings.create({
        model: "text-embedding-3-small",
        input: texts,
      });

      for (let j = 0; j < batch.length; j++) {
        this.movies.push({
          ...batch[j],
          embedding: response.data[j].embedding,
        });
      }

      console.log(`Indexed ${Math.min(i + batchSize, movies.length)}/${movies.length} movies`);
    }
  }

  async search(
    query: string,
    options: {
      topK?: number;
      genre?: string;
      yearRange?: [number, number];
      minRating?: number;
      minScore?: number;
    } = {}
  ): Promise<SearchResult[]> {
    const { topK = 10, genre, yearRange, minRating, minScore = 0.0 } = options;

    // Embed the query
    const response = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: query,
    });
    const queryEmbedding = response.data[0].embedding;

    // Filter, score, sort
    const results = this.movies
      .filter(movie => {
        if (genre && movie.genre.toLowerCase() !== genre.toLowerCase()) return false;
        if (yearRange && (movie.year < yearRange[0] || movie.year > yearRange[1])) return false;
        if (minRating && movie.rating < minRating) return false;
        return true;
      })
      .map(movie => ({
        movie: {
          id: movie.id,
          title: movie.title,
          description: movie.description,
          year: movie.year,
          genre: movie.genre,
          director: movie.director,
          rating: movie.rating,
        },
        score: this.cosineSimilarity(queryEmbedding, movie.embedding),
        rank: 0,
      }))
      .filter(r => r.score >= minScore)
      .sort((a, b) => b.score - a.score)
      .slice(0, topK)
      .map((r, i) => ({ ...r, rank: i + 1 }));

    return results;
  }

  // "More like this" — find movies similar to a given movie
  async moreLikeThis(movieId: string, topK: number = 5): Promise<SearchResult[]> {
    const source = this.movies.find(m => m.id === movieId);
    if (!source) throw new Error(`Movie not found: ${movieId}`);

    return this.movies
      .filter(m => m.id !== movieId)
      .map(movie => ({
        movie: {
          id: movie.id,
          title: movie.title,
          description: movie.description,
          year: movie.year,
          genre: movie.genre,
          director: movie.director,
          rating: movie.rating,
        },
        score: this.cosineSimilarity(source.embedding, movie.embedding),
        rank: 0,
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK)
      .map((r, i) => ({ ...r, rank: i + 1 }));
  }
}

// --- Main ---

async function main() {
  const engine = new MovieSearchEngine();

  // Load our movie catalog
  const catalog: Movie[] = [
    { id: "nemo", title: "Finding Nemo", description: "A clownfish named Marlin searches across the ocean to find his son Nemo, who has been captured by a scuba diver and placed in a fish tank in Sydney", year: 2003, genre: "Animation", director: "Andrew Stanton", rating: 8.2 },
    { id: "lion-king", title: "The Lion King", description: "A young lion prince flees his kingdom after the murder of his father by his uncle, and must return as an adult to reclaim his throne", year: 1994, genre: "Animation", director: "Roger Allers", rating: 8.5 },
    { id: "ratatouille", title: "Ratatouille", description: "A rat named Remy dreams of becoming a French chef and secretly controls a young garbage boy in a Parisian restaurant", year: 2007, genre: "Animation", director: "Brad Bird", rating: 8.1 },
    { id: "wall-e", title: "WALL-E", description: "A trash-compacting robot left alone on a deserted Earth falls in love with a sleek reconnaissance robot named EVE", year: 2008, genre: "Animation", director: "Andrew Stanton", rating: 8.4 },
    { id: "inception", title: "Inception", description: "A thief who steals corporate secrets through dream-sharing technology is given the task of planting an idea into a CEO's subconscious", year: 2010, genre: "Sci-Fi", director: "Christopher Nolan", rating: 8.8 },
    { id: "matrix", title: "The Matrix", description: "A computer hacker discovers that reality as he knows it is a simulation created by machines, and joins a rebellion to free humanity", year: 1999, genre: "Sci-Fi", director: "Wachowskis", rating: 8.7 },
    { id: "interstellar", title: "Interstellar", description: "A team of explorers travels through a wormhole in space to find a new habitable planet for humanity as Earth becomes uninhabitable", year: 2014, genre: "Sci-Fi", director: "Christopher Nolan", rating: 8.7 },
    { id: "godfather", title: "The Godfather", description: "The aging patriarch of an organized crime dynasty transfers control to his reluctant youngest son", year: 1972, genre: "Crime", director: "Francis Ford Coppola", rating: 9.2 },
    { id: "shawshank", title: "The Shawshank Redemption", description: "A banker sentenced to life in prison befriends a fellow inmate and finds hope and redemption through acts of common decency", year: 1994, genre: "Drama", director: "Frank Darabont", rating: 9.3 },
    { id: "dark-knight", title: "The Dark Knight", description: "Batman faces the Joker, a criminal mastermind who wants to plunge Gotham City into anarchy and chaos", year: 2008, genre: "Action", director: "Christopher Nolan", rating: 9.0 },
    { id: "pulp-fiction", title: "Pulp Fiction", description: "The lives of two mob hitmen, a boxer, a gangster and his wife intertwine in four tales of violence and redemption", year: 1994, genre: "Crime", director: "Quentin Tarantino", rating: 8.9 },
    { id: "forrest-gump", title: "Forrest Gump", description: "A slow-witted but kind-hearted man witnesses and unwittingly influences several defining historical events in 20th century America", year: 1994, genre: "Drama", director: "Robert Zemeckis", rating: 8.8 },
  ];

  await engine.addMovies(catalog);

  // Natural language search
  console.log("\n--- Search: 'a movie about a fish trying to find his son' ---\n");
  const fishResults = await engine.search("a movie about a fish trying to find his son");
  for (const r of fishResults.slice(0, 5)) {
    console.log(`  #${r.rank} (${r.score.toFixed(3)}) ${r.movie.title} — ${r.movie.description.slice(0, 60)}...`);
  }

  // Mood-based search
  console.log("\n--- Search: 'something mind-bending about reality and dreams' ---\n");
  const mindResults = await engine.search("something mind-bending about reality and dreams");
  for (const r of mindResults.slice(0, 5)) {
    console.log(`  #${r.rank} (${r.score.toFixed(3)}) ${r.movie.title} — ${r.movie.description.slice(0, 60)}...`);
  }

  // Filtered search
  console.log("\n--- Search: 'adventure and hope' (genre: Drama, min rating: 8.5) ---\n");
  const filteredResults = await engine.search("adventure and hope", {
    genre: "Drama",
    minRating: 8.5,
  });
  for (const r of filteredResults.slice(0, 5)) {
    console.log(`  #${r.rank} (${r.score.toFixed(3)}) ${r.movie.title} (${r.movie.rating})`);
  }

  // "More like this"
  console.log("\n--- More like: Inception ---\n");
  const similarToInception = await engine.moreLikeThis("inception");
  for (const r of similarToInception.slice(0, 5)) {
    console.log(`  #${r.rank} (${r.score.toFixed(3)}) ${r.movie.title}`);
  }
}

main();
```

---

## 6. Scoring and Ranking Results

### 6.1 Understanding Similarity Scores

Raw cosine similarity scores are useful for ranking, but their absolute values can be misleading.

```typescript
// score-interpretation.ts

interface ScoredResult {
  text: string;
  score: number;
}

function interpretScore(score: number): string {
  // These thresholds are approximate — they vary by model and content
  if (score >= 0.85) return "Highly relevant";
  if (score >= 0.75) return "Relevant";
  if (score >= 0.65) return "Somewhat relevant";
  if (score >= 0.50) return "Marginally relevant";
  return "Probably not relevant";
}

function applyScoreThreshold(results: ScoredResult[], threshold: number): ScoredResult[] {
  return results.filter(r => r.score >= threshold);
}

// Normalize scores to 0-1 range relative to the result set
function normalizeScores(results: ScoredResult[]): ScoredResult[] {
  if (results.length === 0) return [];

  const maxScore = Math.max(...results.map(r => r.score));
  const minScore = Math.min(...results.map(r => r.score));
  const range = maxScore - minScore;

  if (range === 0) return results.map(r => ({ ...r, score: 1 }));

  return results.map(r => ({
    ...r,
    score: (r.score - minScore) / range,
  }));
}
```

### 6.2 Combined Scoring

Sometimes you want to factor in more than just semantic similarity — recency, popularity, or explicit user preferences.

```typescript
// combined-scoring.ts

interface ScoringWeights {
  similarity: number;  // 0-1, how much to weight semantic similarity
  recency: number;     // 0-1, how much to weight how recent the item is
  popularity: number;  // 0-1, how much to weight rating/popularity
}

interface ScoredDocument {
  id: string;
  text: string;
  similarityScore: number;
  date: Date;
  popularity: number;  // 0-1 normalized
}

function combinedScore(
  doc: ScoredDocument,
  weights: ScoringWeights,
  now: Date = new Date()
): number {
  // Normalize recency: newer = higher score
  const ageInDays = (now.getTime() - doc.date.getTime()) / (1000 * 60 * 60 * 24);
  const maxAge = 365 * 5; // 5 years
  const recencyScore = Math.max(0, 1 - ageInDays / maxAge);

  // Weighted combination
  const total = weights.similarity + weights.recency + weights.popularity;
  return (
    (weights.similarity / total) * doc.similarityScore +
    (weights.recency / total) * recencyScore +
    (weights.popularity / total) * doc.popularity
  );
}

// Example: search for documentation — prefer recent, high-quality matches
function searchDocs(results: ScoredDocument[]): ScoredDocument[] {
  const weights: ScoringWeights = {
    similarity: 0.6,  // Meaning matters most
    recency: 0.2,     // Prefer newer docs
    popularity: 0.2,  // Prefer well-liked docs
  };

  return results
    .map(doc => ({
      ...doc,
      similarityScore: combinedScore(doc, weights),
    }))
    .sort((a, b) => b.similarityScore - a.similarityScore);
}
```

### 6.3 Reciprocal Rank Fusion (RRF)

When you have multiple ranking signals (semantic score, keyword match, metadata boost), RRF is a simple and effective way to combine them.

```typescript
// reciprocal-rank-fusion.ts

interface RankedItem {
  id: string;
  [key: string]: unknown;
}

/**
 * Reciprocal Rank Fusion — combines multiple ranked lists into one.
 * Each item's score = sum of 1/(k + rank) across all lists it appears in.
 * k is a constant (typically 60) that reduces the impact of high rankings.
 */
function reciprocalRankFusion(
  rankedLists: RankedItem[][],
  k: number = 60
): { id: string; score: number }[] {
  const scores = new Map<string, number>();

  for (const list of rankedLists) {
    for (let rank = 0; rank < list.length; rank++) {
      const id = list[rank].id;
      const currentScore = scores.get(id) ?? 0;
      scores.set(id, currentScore + 1 / (k + rank + 1));
    }
  }

  return Array.from(scores.entries())
    .map(([id, score]) => ({ id, score }))
    .sort((a, b) => b.score - a.score);
}

// Usage: combine semantic search results with keyword search results
// const semanticResults = await semanticSearch(query);
// const keywordResults = await keywordSearch(query);
// const combined = reciprocalRankFusion([semanticResults, keywordResults]);
```

---

## 7. Performance and Cost Considerations

### 7.1 Embedding Cost Math

Let's do the math on what embeddings cost at scale.

```typescript
// cost-calculator.ts

interface EmbeddingCostEstimate {
  model: string;
  totalTokens: number;
  costPerMillionTokens: number;
  totalCost: number;
  totalDocuments: number;
  avgTokensPerDocument: number;
}

function estimateEmbeddingCost(
  documentCount: number,
  avgCharsPerDocument: number,
  model: "text-embedding-3-small" | "text-embedding-3-large" = "text-embedding-3-small"
): EmbeddingCostEstimate {
  // Rough estimate: 1 token ≈ 4 characters for English text
  const avgTokensPerDoc = Math.ceil(avgCharsPerDocument / 4);
  const totalTokens = documentCount * avgTokensPerDoc;

  const costPerMillion = model === "text-embedding-3-small" ? 0.02 : 0.13;
  const totalCost = (totalTokens / 1_000_000) * costPerMillion;

  return {
    model,
    totalTokens,
    costPerMillionTokens: costPerMillion,
    totalCost,
    totalDocuments: documentCount,
    avgTokensPerDocument: avgTokensPerDoc,
  };
}

// Examples
console.log("1,000 short documents (~500 chars each):");
console.log(estimateEmbeddingCost(1000, 500));
// ~$0.0025 — basically free

console.log("\n100,000 documents (~2000 chars each):");
console.log(estimateEmbeddingCost(100000, 2000));
// ~$1.00 with text-embedding-3-small

console.log("\n1,000,000 documents (~2000 chars each):");
console.log(estimateEmbeddingCost(1000000, 2000));
// ~$10.00 — still very cheap for a million documents
```

### 7.2 Caching Embeddings

Never re-embed the same text. Cache aggressively.

```typescript
// embedding-cache.ts
import OpenAI from "openai";
import crypto from "crypto";

const openai = new OpenAI();

// Simple in-memory cache (use Redis in production)
const embeddingCache = new Map<string, number[]>();

function hashText(text: string): string {
  return crypto.createHash("sha256").update(text).digest("hex");
}

async function embedWithCache(text: string): Promise<number[]> {
  const key = hashText(text);

  // Check cache first
  const cached = embeddingCache.get(key);
  if (cached) return cached;

  // Embed and cache
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
  });

  const embedding = response.data[0].embedding;
  embeddingCache.set(key, embedding);
  return embedding;
}

async function embedBatchWithCache(texts: string[]): Promise<number[][]> {
  const results: (number[] | null)[] = texts.map(text => {
    const key = hashText(text);
    return embeddingCache.get(key) ?? null;
  });

  // Find texts that need embedding
  const uncachedIndices = results
    .map((r, i) => (r === null ? i : -1))
    .filter(i => i !== -1);

  if (uncachedIndices.length > 0) {
    const uncachedTexts = uncachedIndices.map(i => texts[i]);
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: uncachedTexts,
    });

    for (let j = 0; j < uncachedIndices.length; j++) {
      const idx = uncachedIndices[j];
      const embedding = response.data[j].embedding;
      results[idx] = embedding;
      embeddingCache.set(hashText(texts[idx]), embedding);
    }
  }

  return results as number[][];
}
```

### 7.3 When NOT to Use Semantic Search

Semantic search is powerful but it's not always the right tool.

**Use semantic search when:**
- Users describe what they want in natural language
- You need to match by *meaning*, not exact text
- The corpus is too large for manual tagging
- Queries are fuzzy ("something about space travel" vs "Apollo 11")

**Use keyword search when:**
- Users know the exact term (product SKU, error code, person's name)
- You need exact matches ("404 error" should not match "405 error")
- The domain has specific technical vocabulary

**Use both (hybrid search) when:**
- You want the best of both worlds
- We'll build this in Chapter 17 (Advanced Retrieval)

---

## Summary

You now know how to:

1. **Call the embeddings API** — convert text to vectors with OpenAI
2. **Compute similarity** — cosine similarity, euclidean distance, dot product
3. **Build in-memory search** — the foundation that everything else builds on
4. **Use vector databases** — Pinecone, Upstash Vector, and pgvector
5. **Build a real search engine** — the movie recommendation system
6. **Score and rank results** — combined scoring, RRF, thresholds
7. **Manage cost and performance** — caching, batch processing, model selection

This is the foundation. In Chapter 15, we'll wrap this search capability in a full RAG pipeline — load documents, chunk them, embed them, store them, and retrieve them on demand.

---

*Previous: [Ch 13 — Agent Patterns & Frameworks](../part-2-agent-engineering/13-agent-patterns.md)* | *Next: [Ch 15 — The RAG Pipeline](./15-rag-pipeline.md)*

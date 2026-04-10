<!--
  CHAPTER: 15
  TITLE: The RAG Pipeline
  PART: 3 — RAG & Knowledge Systems
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 14 (Semantic Search), Ch 10 (Agent Memory & State)
  KEY_TOPICS: document loading, chunking strategies, recursive chunking, semantic chunking, embedding generation, batch processing, vector store indexing, retrieval pipeline, reranking, end-to-end RAG
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 15: The RAG Pipeline

> **Part 3 — RAG & Knowledge Systems** | Phase 1: Get Dangerous | Prerequisites: Ch 14, Ch 10 | Difficulty: Intermediate | Language: TypeScript

In Chapter 14 you built semantic search — give it a query, get back similar items. But real-world AI applications need more than search. They need to load documents from files, URLs, and APIs. They need to split those documents into chunks small enough for an LLM's context window. They need to embed those chunks, store them, and retrieve the right ones at query time. Then — and this is the "generation" part — they need to pass those chunks to an LLM and get an answer.

That's RAG: Retrieval-Augmented Generation. It's the most important pattern in production AI. It's how you give an LLM access to knowledge it wasn't trained on — your company's docs, your product's data, your user's files — without fine-tuning a model.

This chapter builds the full pipeline, end to end.

### In This Chapter

1. The RAG pipeline architecture
2. Document loading: files, URLs, APIs
3. Chunking strategies: fixed-size, recursive, semantic
4. Embedding generation: batch processing at scale
5. Storage and indexing
6. Retrieval: query, embed, search, rerank, return
7. The full pipeline end-to-end

### Related Chapters

- **Ch 14 (Semantic Search)** — The search primitive this pipeline is built on.
- **Ch 10 (Agent Memory & State)** — RAG is external memory for agents.
- **Ch 5 (Structured Output)** — Structure the generated answers with Zod.
- **Ch 19 (Single-Turn Evals)** — Evaluate the retrieval quality of your pipeline.
- **Ch 44 (RAG vs Fine-Tuning)** — When to use RAG vs baking knowledge into model weights.

---

## 1. The RAG Pipeline Architecture

Here's the full picture. Every RAG system has two phases:

```
INDEXING (offline, run once per document update):
  Documents → Load → Chunk → Embed → Store

RETRIEVAL (online, run every query):
  Query → Embed → Search → Rerank → Context → LLM → Answer
```

**What it is:** A two-phase system that loads knowledge into a searchable store (indexing) and retrieves relevant context at query time (retrieval).

**When to use:** Whenever your LLM needs to answer questions about data it wasn't trained on — company docs, product data, user files, recent events.

**Why not just stuff everything into the prompt?** Context windows have limits (even large ones), stuffing is expensive, and models perform worse with too much irrelevant context. RAG gives the model only what it needs.

```typescript
// rag-pipeline-overview.ts
// This is the skeleton — we'll fill in each piece in this chapter

interface RAGPipeline {
  // Phase 1: Indexing
  load(source: string): Promise<Document[]>;
  chunk(documents: Document[]): Chunk[];
  embed(chunks: Chunk[]): Promise<EmbeddedChunk[]>;
  store(chunks: EmbeddedChunk[]): Promise<void>;

  // Phase 2: Retrieval
  retrieve(query: string, topK?: number): Promise<RetrievedChunk[]>;
  generate(query: string, context: RetrievedChunk[]): Promise<string>;

  // The full pipeline
  query(question: string): Promise<{ answer: string; sources: RetrievedChunk[] }>;
}

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    url?: string;
    loadedAt: Date;
    [key: string]: unknown;
  };
}

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface EmbeddedChunk extends Chunk {
  embedding: number[];
}

interface RetrievedChunk extends Chunk {
  score: number;
}
```

---

## 2. Document Loading

### 2.1 Loading Text Files

The simplest case: read files from disk.

```typescript
// loaders/file-loader.ts
import fs from "fs/promises";
import path from "path";
import crypto from "crypto";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    url?: string;
    loadedAt: Date;
    [key: string]: unknown;
  };
}

async function loadTextFile(filePath: string): Promise<Document> {
  const content = await fs.readFile(filePath, "utf-8");
  const fileName = path.basename(filePath);

  return {
    id: crypto.createHash("sha256").update(filePath).digest("hex").slice(0, 16),
    content,
    metadata: {
      source: filePath,
      title: fileName,
      loadedAt: new Date(),
      fileType: path.extname(filePath),
      sizeBytes: Buffer.byteLength(content, "utf-8"),
    },
  };
}

async function loadDirectory(
  dirPath: string,
  extensions: string[] = [".txt", ".md", ".mdx"]
): Promise<Document[]> {
  const entries = await fs.readdir(dirPath, { recursive: true, withFileTypes: true });
  const docs: Document[] = [];

  for (const entry of entries) {
    if (!entry.isFile()) continue;

    const ext = path.extname(entry.name).toLowerCase();
    if (!extensions.includes(ext)) continue;

    const fullPath = path.join(entry.parentPath ?? dirPath, entry.name);
    const doc = await loadTextFile(fullPath);
    docs.push(doc);
  }

  return docs;
}

// Usage
// const docs = await loadDirectory("./docs", [".md", ".txt"]);
// console.log(`Loaded ${docs.length} documents`);
```

### 2.2 Loading from URLs

Fetch web pages and extract the text content.

```typescript
// loaders/url-loader.ts
import crypto from "crypto";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    url?: string;
    loadedAt: Date;
    [key: string]: unknown;
  };
}

async function loadUrl(url: string): Promise<Document> {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Failed to fetch ${url}: ${response.status}`);
  }

  const html = await response.text();

  // Simple HTML to text extraction
  // For production, use a library like cheerio or mozilla/readability
  const text = htmlToText(html);
  const title = extractTitle(html);

  return {
    id: crypto.createHash("sha256").update(url).digest("hex").slice(0, 16),
    content: text,
    metadata: {
      source: url,
      url,
      title,
      loadedAt: new Date(),
      contentType: response.headers.get("content-type") ?? "unknown",
    },
  };
}

function htmlToText(html: string): string {
  return html
    // Remove scripts and styles
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "")
    // Remove HTML tags
    .replace(/<[^>]+>/g, " ")
    // Decode common HTML entities
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/&nbsp;/g, " ")
    // Collapse whitespace
    .replace(/\s+/g, " ")
    .trim();
}

function extractTitle(html: string): string | undefined {
  const match = html.match(/<title[^>]*>([\s\S]*?)<\/title>/i);
  return match ? match[1].trim() : undefined;
}

// Load multiple URLs in parallel with concurrency control
async function loadUrls(urls: string[], concurrency: number = 5): Promise<Document[]> {
  const results: Document[] = [];

  for (let i = 0; i < urls.length; i += concurrency) {
    const batch = urls.slice(i, i + concurrency);
    const batchResults = await Promise.allSettled(batch.map(loadUrl));

    for (const result of batchResults) {
      if (result.status === "fulfilled") {
        results.push(result.value);
      } else {
        console.warn(`Failed to load URL: ${result.reason}`);
      }
    }
  }

  return results;
}
```

### 2.3 Loading from APIs

Many knowledge sources are behind APIs — Notion, Confluence, Slack, internal wikis.

```typescript
// loaders/api-loader.ts
import crypto from "crypto";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    url?: string;
    loadedAt: Date;
    [key: string]: unknown;
  };
}

// Generic API loader pattern
interface APILoaderConfig {
  baseUrl: string;
  headers: Record<string, string>;
  listEndpoint: string;
  contentExtractor: (item: Record<string, unknown>) => { title: string; content: string; id: string };
  paginationHandler?: (response: Record<string, unknown>) => string | null; // Returns next page cursor
}

async function loadFromAPI(config: APILoaderConfig): Promise<Document[]> {
  const docs: Document[] = [];
  let cursor: string | null = null;

  do {
    const url = new URL(config.listEndpoint, config.baseUrl);
    if (cursor) url.searchParams.set("cursor", cursor);

    const response = await fetch(url.toString(), { headers: config.headers });
    if (!response.ok) throw new Error(`API error: ${response.status}`);

    const data = await response.json() as { results: Record<string, unknown>[] };

    for (const item of data.results ?? []) {
      const extracted = config.contentExtractor(item);

      docs.push({
        id: crypto.createHash("sha256").update(extracted.id).digest("hex").slice(0, 16),
        content: extracted.content,
        metadata: {
          source: `${config.baseUrl}${config.listEndpoint}`,
          title: extracted.title,
          loadedAt: new Date(),
          apiSource: config.baseUrl,
          originalId: extracted.id,
        },
      });
    }

    cursor = config.paginationHandler?.(data as Record<string, unknown>) ?? null;
  } while (cursor);

  return docs;
}

// Example: Notion loader
async function loadFromNotion(databaseId: string, notionToken: string): Promise<Document[]> {
  return loadFromAPI({
    baseUrl: "https://api.notion.com",
    headers: {
      Authorization: `Bearer ${notionToken}`,
      "Notion-Version": "2022-06-28",
      "Content-Type": "application/json",
    },
    listEndpoint: `/v1/databases/${databaseId}/query`,
    contentExtractor: (page) => ({
      id: page.id as string,
      title: (page as Record<string, unknown>).url as string ?? "Untitled",
      content: extractNotionContent(page),
    }),
    paginationHandler: (response) =>
      response.has_more ? (response.next_cursor as string) : null,
  });
}

function extractNotionContent(page: Record<string, unknown>): string {
  // Simplified — in production you'd recursively extract block content
  return JSON.stringify(page);
}
```

---

## 3. Chunking Strategies

This is where most RAG pipelines succeed or fail. Chunk too big and you waste context window tokens on irrelevant text. Chunk too small and you lose the context that makes the text meaningful.

### 3.1 Fixed-Size Chunking

The simplest approach: split text into chunks of N characters with some overlap.

```typescript
// chunkers/fixed-size.ts

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

function fixedSizeChunk(
  document: Document,
  options: {
    chunkSize?: number;   // Target chunk size in characters
    overlap?: number;      // Overlap between chunks in characters
  } = {}
): Chunk[] {
  const { chunkSize = 1000, overlap = 200 } = options;
  const chunks: Chunk[] = [];
  const text = document.content;

  let start = 0;
  let index = 0;

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length);
    const content = text.slice(start, end);

    chunks.push({
      id: `${document.id}-chunk-${index}`,
      content,
      documentId: document.id,
      index,
      metadata: {
        ...document.metadata,
        chunkIndex: index,
        startChar: start,
        endChar: end,
        chunkStrategy: "fixed-size",
      },
    });

    // Move forward by chunkSize - overlap
    start += chunkSize - overlap;
    index++;
  }

  return chunks;
}
```

**Trade-offs:**
- Simple to implement
- Can split mid-sentence or mid-paragraph
- Overlap helps preserve context at boundaries
- No awareness of document structure

### 3.2 Recursive Character Chunking

Split on natural boundaries first (paragraphs, then sentences, then words), falling back to character-level only when necessary. This is the most popular production strategy.

```typescript
// chunkers/recursive.ts

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

function recursiveChunk(
  document: Document,
  options: {
    chunkSize?: number;
    overlap?: number;
    separators?: string[];
  } = {}
): Chunk[] {
  const {
    chunkSize = 1000,
    overlap = 200,
    separators = ["\n\n", "\n", ". ", "! ", "? ", "; ", ", ", " ", ""],
  } = options;

  const rawChunks = splitRecursive(document.content, chunkSize, overlap, separators);

  return rawChunks.map((content, index) => ({
    id: `${document.id}-chunk-${index}`,
    content,
    documentId: document.id,
    index,
    metadata: {
      ...document.metadata,
      chunkIndex: index,
      chunkStrategy: "recursive",
      chunkLength: content.length,
    },
  }));
}

function splitRecursive(
  text: string,
  chunkSize: number,
  overlap: number,
  separators: string[]
): string[] {
  if (text.length <= chunkSize) {
    return [text.trim()].filter(t => t.length > 0);
  }

  // Find the best separator that exists in the text
  let separator = "";
  for (const sep of separators) {
    if (text.includes(sep)) {
      separator = sep;
      break;
    }
  }

  // Split by the chosen separator
  const parts = separator ? text.split(separator) : [text];

  // Merge parts into chunks of the target size
  const chunks: string[] = [];
  let currentChunk = "";

  for (const part of parts) {
    const candidate = currentChunk
      ? currentChunk + separator + part
      : part;

    if (candidate.length > chunkSize && currentChunk.length > 0) {
      // Current chunk is full — save it and start a new one
      chunks.push(currentChunk.trim());

      // Start the new chunk with overlap from the end of the previous chunk
      if (overlap > 0) {
        const overlapText = currentChunk.slice(-overlap);
        currentChunk = overlapText + separator + part;
      } else {
        currentChunk = part;
      }
    } else {
      currentChunk = candidate;
    }
  }

  // Don't forget the last chunk
  if (currentChunk.trim().length > 0) {
    chunks.push(currentChunk.trim());
  }

  // Recursively split any chunks that are still too large
  const result: string[] = [];
  const remainingSeparators = separators.slice(separators.indexOf(separator) + 1);

  for (const chunk of chunks) {
    if (chunk.length > chunkSize && remainingSeparators.length > 0) {
      result.push(...splitRecursive(chunk, chunkSize, overlap, remainingSeparators));
    } else {
      result.push(chunk);
    }
  }

  return result;
}
```

### 3.3 Markdown-Aware Chunking

If your documents are Markdown (very common for docs, READMEs, wikis), you can chunk by headers.

```typescript
// chunkers/markdown.ts

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

interface MarkdownSection {
  heading: string;
  level: number;
  content: string;
  path: string[];  // Breadcrumb: ["# Introduction", "## Getting Started"]
}

function markdownChunk(
  document: Document,
  options: { maxChunkSize?: number; includeHeadingPath?: boolean } = {}
): Chunk[] {
  const { maxChunkSize = 1500, includeHeadingPath = true } = options;
  const sections = parseMarkdownSections(document.content);

  const chunks: Chunk[] = [];

  for (let i = 0; i < sections.length; i++) {
    const section = sections[i];
    let content = section.content;

    // Prepend heading breadcrumb for context
    if (includeHeadingPath && section.path.length > 0) {
      const breadcrumb = section.path.join(" > ");
      content = `[${breadcrumb}]\n\n${content}`;
    }

    // If the section is too large, use recursive chunking on it
    if (content.length > maxChunkSize) {
      const subChunks = splitRecursive(content, maxChunkSize, 200, ["\n\n", "\n", ". ", " "]);
      for (let j = 0; j < subChunks.length; j++) {
        chunks.push({
          id: `${document.id}-section-${i}-${j}`,
          content: subChunks[j],
          documentId: document.id,
          index: chunks.length,
          metadata: {
            ...document.metadata,
            heading: section.heading,
            headingLevel: section.level,
            headingPath: section.path,
            chunkStrategy: "markdown",
          },
        });
      }
    } else if (content.trim().length > 0) {
      chunks.push({
        id: `${document.id}-section-${i}`,
        content,
        documentId: document.id,
        index: chunks.length,
        metadata: {
          ...document.metadata,
          heading: section.heading,
          headingLevel: section.level,
          headingPath: section.path,
          chunkStrategy: "markdown",
        },
      });
    }
  }

  return chunks;
}

function parseMarkdownSections(markdown: string): MarkdownSection[] {
  const lines = markdown.split("\n");
  const sections: MarkdownSection[] = [];
  const headingStack: { heading: string; level: number }[] = [];
  let currentContent = "";
  let currentHeading = "(intro)";
  let currentLevel = 0;

  for (const line of lines) {
    const headingMatch = line.match(/^(#{1,6})\s+(.+)$/);

    if (headingMatch) {
      // Save the current section
      if (currentContent.trim()) {
        sections.push({
          heading: currentHeading,
          level: currentLevel,
          content: currentContent.trim(),
          path: headingStack.map(h => h.heading),
        });
      }

      const level = headingMatch[1].length;
      const heading = headingMatch[2].trim();

      // Update the heading stack
      while (headingStack.length > 0 && headingStack[headingStack.length - 1].level >= level) {
        headingStack.pop();
      }
      headingStack.push({ heading, level });

      currentHeading = heading;
      currentLevel = level;
      currentContent = `${"#".repeat(level)} ${heading}\n\n`;
    } else {
      currentContent += line + "\n";
    }
  }

  // Don't forget the last section
  if (currentContent.trim()) {
    sections.push({
      heading: currentHeading,
      level: currentLevel,
      content: currentContent.trim(),
      path: headingStack.map(h => h.heading),
    });
  }

  return sections;
}
```

### 3.4 Semantic Chunking

The most sophisticated approach: split where the *meaning* changes. This uses embeddings to detect topic shifts within a document.

```typescript
// chunkers/semantic.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}

async function semanticChunk(
  document: Document,
  options: {
    maxChunkSize?: number;
    similarityThreshold?: number;
    sentenceGroupSize?: number;
  } = {}
): Promise<Chunk[]> {
  const {
    maxChunkSize = 1500,
    similarityThreshold = 0.75,
    sentenceGroupSize = 3,
  } = options;

  // Step 1: Split into sentences
  const sentences = document.content
    .split(/(?<=[.!?])\s+/)
    .filter(s => s.trim().length > 0);

  if (sentences.length <= sentenceGroupSize) {
    return [{
      id: `${document.id}-chunk-0`,
      content: document.content,
      documentId: document.id,
      index: 0,
      metadata: { ...document.metadata, chunkStrategy: "semantic" },
    }];
  }

  // Step 2: Group sentences and embed each group
  const groups: string[] = [];
  for (let i = 0; i < sentences.length; i += sentenceGroupSize) {
    groups.push(sentences.slice(i, i + sentenceGroupSize).join(" "));
  }

  const embeddingResponse = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: groups,
  });
  const embeddings = embeddingResponse.data.map(d => d.embedding);

  // Step 3: Find breakpoints where similarity drops
  const breakpoints: number[] = [0];
  for (let i = 1; i < embeddings.length; i++) {
    const similarity = cosineSimilarity(embeddings[i - 1], embeddings[i]);
    if (similarity < similarityThreshold) {
      breakpoints.push(i);
    }
  }
  breakpoints.push(groups.length);

  // Step 4: Create chunks from breakpoints
  const chunks: Chunk[] = [];
  for (let i = 0; i < breakpoints.length - 1; i++) {
    const start = breakpoints[i];
    const end = breakpoints[i + 1];
    const content = groups.slice(start, end).join(" ");

    // If too large, split further
    if (content.length > maxChunkSize) {
      const subParts = splitRecursive(content, maxChunkSize, 200, [". ", " "]);
      for (const part of subParts) {
        chunks.push({
          id: `${document.id}-chunk-${chunks.length}`,
          content: part,
          documentId: document.id,
          index: chunks.length,
          metadata: { ...document.metadata, chunkStrategy: "semantic" },
        });
      }
    } else {
      chunks.push({
        id: `${document.id}-chunk-${chunks.length}`,
        content,
        documentId: document.id,
        index: chunks.length,
        metadata: { ...document.metadata, chunkStrategy: "semantic" },
      });
    }
  }

  return chunks;
}
```

### 3.5 Choosing a Chunking Strategy

| Strategy | Best For | Pros | Cons |
|----------|---------|------|------|
| Fixed-size | Quick prototypes | Simple, predictable | Splits mid-thought |
| Recursive | General purpose | Respects boundaries | Still structure-agnostic |
| Markdown | Documentation | Uses headings as natural splits | Only works for Markdown |
| Semantic | High accuracy needs | Chunks by meaning | Requires extra API calls, slower |

**The recommendation:** Start with recursive chunking (1000 chars, 200 overlap). Move to markdown-aware if your docs are Markdown. Move to semantic if retrieval quality is your top priority and cost isn't a constraint.

---

## 4. Embedding Generation

### 4.1 Batch Processing at Scale

Once you have chunks, you need to embed them all. The key is batching and rate limit handling.

```typescript
// embedding/batch-embedder.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface EmbeddedChunk extends Chunk {
  embedding: number[];
}

async function embedChunks(
  chunks: Chunk[],
  options: {
    model?: string;
    batchSize?: number;
    onProgress?: (processed: number, total: number) => void;
  } = {}
): Promise<EmbeddedChunk[]> {
  const {
    model = "text-embedding-3-small",
    batchSize = 100,
    onProgress,
  } = options;

  const results: EmbeddedChunk[] = [];

  for (let i = 0; i < chunks.length; i += batchSize) {
    const batch = chunks.slice(i, i + batchSize);
    const texts = batch.map(c => c.content);

    try {
      const response = await openai.embeddings.create({
        model,
        input: texts,
      });

      for (let j = 0; j < batch.length; j++) {
        results.push({
          ...batch[j],
          embedding: response.data[j].embedding,
        });
      }
    } catch (error: unknown) {
      // Handle rate limits with exponential backoff
      if (error instanceof Error && "status" in error && (error as { status: number }).status === 429) {
        console.warn("Rate limited, waiting 60 seconds...");
        await new Promise(r => setTimeout(r, 60_000));
        i -= batchSize; // Retry this batch
        continue;
      }
      throw error;
    }

    onProgress?.(Math.min(i + batchSize, chunks.length), chunks.length);
  }

  return results;
}

// Usage
async function main() {
  const chunks: Chunk[] = [
    /* ... your chunks here ... */
  ];

  const embedded = await embedChunks(chunks, {
    batchSize: 50,
    onProgress: (processed, total) => {
      console.log(`Embedded ${processed}/${total} chunks`);
    },
  });

  console.log(`Generated ${embedded.length} embeddings`);
}
```

### 4.2 Embedding with Metadata Context

A subtle but important optimization: what you embed matters. Sometimes the chunk alone isn't enough context — prepending the document title or section heading dramatically improves retrieval.

```typescript
// embedding/contextual-embedding.ts

function prepareTextForEmbedding(chunk: Chunk): string {
  const parts: string[] = [];

  // Add document title for context
  if (chunk.metadata.title) {
    parts.push(`Document: ${chunk.metadata.title}`);
  }

  // Add section heading for context
  if (chunk.metadata.heading) {
    parts.push(`Section: ${chunk.metadata.heading}`);
  }

  // Add the actual content
  parts.push(chunk.content);

  return parts.join("\n\n");
}

// This turns a chunk like:
//   "To configure authentication, set the AUTH_SECRET environment variable..."
// Into:
//   "Document: Next.js Deployment Guide\nSection: Authentication Setup\n\nTo configure authentication..."
//
// The embedding now captures the CONTEXT of the chunk, not just the content.
```

---

## 5. Storage and Indexing

### 5.1 A Simple Storage Interface

Let's define a clean interface that works with any vector store.

```typescript
// storage/vector-store.ts

interface EmbeddedChunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
  embedding: number[];
}

interface RetrievedChunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
  score: number;
}

interface VectorStoreConfig {
  dimension: number;
  metric: "cosine" | "euclidean" | "dotProduct";
}

interface VectorStore {
  upsert(chunks: EmbeddedChunk[]): Promise<void>;
  search(embedding: number[], topK: number, filter?: Record<string, unknown>): Promise<RetrievedChunk[]>;
  delete(ids: string[]): Promise<void>;
  count(): Promise<number>;
}
```

### 5.2 In-Memory Implementation

Good for development and small datasets.

```typescript
// storage/in-memory-store.ts

class InMemoryVectorStore implements VectorStore {
  private chunks: EmbeddedChunk[] = [];

  private cosineSimilarity(a: number[], b: number[]): number {
    let dot = 0, normA = 0, normB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    return dot / (Math.sqrt(normA) * Math.sqrt(normB));
  }

  async upsert(chunks: EmbeddedChunk[]): Promise<void> {
    for (const chunk of chunks) {
      const existingIndex = this.chunks.findIndex(c => c.id === chunk.id);
      if (existingIndex >= 0) {
        this.chunks[existingIndex] = chunk;
      } else {
        this.chunks.push(chunk);
      }
    }
  }

  async search(
    embedding: number[],
    topK: number,
    filter?: Record<string, unknown>
  ): Promise<RetrievedChunk[]> {
    return this.chunks
      .filter(chunk => {
        if (!filter) return true;
        return Object.entries(filter).every(
          ([key, value]) => chunk.metadata[key] === value
        );
      })
      .map(chunk => ({
        id: chunk.id,
        content: chunk.content,
        documentId: chunk.documentId,
        index: chunk.index,
        metadata: chunk.metadata,
        score: this.cosineSimilarity(embedding, chunk.embedding),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }

  async delete(ids: string[]): Promise<void> {
    const idSet = new Set(ids);
    this.chunks = this.chunks.filter(c => !idSet.has(c.id));
  }

  async count(): Promise<number> {
    return this.chunks.length;
  }
}
```

### 5.3 Pinecone Implementation

```typescript
// storage/pinecone-store.ts
import { Pinecone } from "@pinecone-database/pinecone";

class PineconeVectorStore implements VectorStore {
  private index;

  constructor(indexName: string) {
    const pinecone = new Pinecone();
    this.index = pinecone.index(indexName);
  }

  async upsert(chunks: EmbeddedChunk[]): Promise<void> {
    const vectors = chunks.map(chunk => ({
      id: chunk.id,
      values: chunk.embedding,
      metadata: {
        content: chunk.content,
        documentId: chunk.documentId,
        chunkIndex: chunk.index,
        ...chunk.metadata,
      },
    }));

    // Upsert in batches of 100
    for (let i = 0; i < vectors.length; i += 100) {
      await this.index.upsert(vectors.slice(i, i + 100));
    }
  }

  async search(
    embedding: number[],
    topK: number,
    filter?: Record<string, unknown>
  ): Promise<RetrievedChunk[]> {
    const results = await this.index.query({
      vector: embedding,
      topK,
      includeMetadata: true,
      filter: filter as Record<string, string | number | boolean>,
    });

    return (results.matches ?? []).map(match => ({
      id: match.id,
      content: (match.metadata?.content as string) ?? "",
      documentId: (match.metadata?.documentId as string) ?? "",
      index: (match.metadata?.chunkIndex as number) ?? 0,
      metadata: match.metadata as Record<string, unknown>,
      score: match.score ?? 0,
    }));
  }

  async delete(ids: string[]): Promise<void> {
    await this.index.deleteMany(ids);
  }

  async count(): Promise<number> {
    const stats = await this.index.describeIndexStats();
    return stats.totalRecordCount ?? 0;
  }
}
```

---

## 6. Retrieval: Query to Answer

### 6.1 The Retrieval Chain

Now we combine search with generation. This is the "A" and "G" in RAG.

```typescript
// retrieval/retriever.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface RetrievedChunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
  score: number;
}

interface VectorStore {
  search(embedding: number[], topK: number, filter?: Record<string, unknown>): Promise<RetrievedChunk[]>;
}

interface RAGResponse {
  answer: string;
  sources: RetrievedChunk[];
  tokensUsed: { prompt: number; completion: number };
}

async function retrieve(
  query: string,
  store: VectorStore,
  options: {
    topK?: number;
    filter?: Record<string, unknown>;
    minScore?: number;
  } = {}
): Promise<RetrievedChunk[]> {
  const { topK = 5, filter, minScore = 0.0 } = options;

  // Step 1: Embed the query
  const embeddingResponse = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const queryEmbedding = embeddingResponse.data[0].embedding;

  // Step 2: Search the vector store
  const results = await store.search(queryEmbedding, topK, filter);

  // Step 3: Filter by minimum score
  return results.filter(r => r.score >= minScore);
}

async function generateAnswer(
  query: string,
  context: RetrievedChunk[],
  options: {
    model?: string;
    systemPrompt?: string;
    temperature?: number;
  } = {}
): Promise<RAGResponse> {
  const {
    model = "gpt-4o",
    temperature = 0.1,
  } = options;

  // Build the context string from retrieved chunks
  const contextText = context
    .map((chunk, i) => {
      const source = chunk.metadata.title ?? chunk.metadata.source ?? "Unknown";
      return `[Source ${i + 1}: ${source}]\n${chunk.content}`;
    })
    .join("\n\n---\n\n");

  const systemPrompt = options.systemPrompt ?? `You are a helpful assistant that answers questions based on the provided context.

RULES:
- Answer ONLY based on the context provided below.
- If the context doesn't contain enough information to answer, say "I don't have enough information to answer that question."
- Cite your sources by referencing [Source N] when you use information from a specific source.
- Be concise and direct.

CONTEXT:
${contextText}`;

  const response = await openai.chat.completions.create({
    model,
    temperature,
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: query },
    ],
  });

  return {
    answer: response.choices[0].message.content ?? "",
    sources: context,
    tokensUsed: {
      prompt: response.usage?.prompt_tokens ?? 0,
      completion: response.usage?.completion_tokens ?? 0,
    },
  };
}
```

### 6.2 Simple Reranking

Retrieve more, then rerank to find the best. This is a simple but effective strategy: fetch 20 results from the vector store, then use a smarter (but slower) method to pick the top 5.

```typescript
// retrieval/reranker.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface RetrievedChunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
  score: number;
}

// LLM-based reranker: ask the LLM to score relevance
async function llmRerank(
  query: string,
  chunks: RetrievedChunk[],
  topK: number = 5
): Promise<RetrievedChunk[]> {
  const scoringPrompt = `You are a relevance scoring system. Given a query and a passage, rate how relevant the passage is to answering the query on a scale of 0-10.

Query: "${query}"

Return ONLY a JSON array of objects with "id" and "relevance" (0-10) for each passage.`;

  const passagesText = chunks
    .map((c, i) => `Passage ${i} (id: ${c.id}):\n${c.content.slice(0, 500)}`)
    .join("\n\n");

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini", // Cheaper model is fine for reranking
    temperature: 0,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: scoringPrompt },
      { role: "user", content: passagesText },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
  const scores = parsed.scores ?? parsed.results ?? [];

  // Apply the reranking scores
  const reranked = chunks.map(chunk => {
    const score = scores.find((s: { id: string; relevance: number }) => s.id === chunk.id);
    return {
      ...chunk,
      score: score ? score.relevance / 10 : chunk.score,
    };
  });

  return reranked
    .sort((a, b) => b.score - a.score)
    .slice(0, topK);
}
```

---

## 7. The Full Pipeline End-to-End

Here's the complete, working RAG pipeline that ties everything together.

```typescript
// rag-pipeline-complete.ts
import OpenAI from "openai";
import fs from "fs/promises";
import path from "path";
import crypto from "crypto";

const openai = new OpenAI();

// --- Types ---

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    [key: string]: unknown;
  };
}

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface EmbeddedChunk extends Chunk {
  embedding: number[];
}

interface RetrievedChunk extends Chunk {
  score: number;
}

interface RAGResponse {
  answer: string;
  sources: { content: string; source: string; score: number }[];
  timing: {
    retrievalMs: number;
    generationMs: number;
    totalMs: number;
  };
}

// --- Vector Store ---

class SimpleVectorStore {
  private chunks: EmbeddedChunk[] = [];

  private cosineSimilarity(a: number[], b: number[]): number {
    let dot = 0, normA = 0, normB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    return dot / (Math.sqrt(normA) * Math.sqrt(normB));
  }

  async upsert(chunks: EmbeddedChunk[]) {
    for (const chunk of chunks) {
      const idx = this.chunks.findIndex(c => c.id === chunk.id);
      if (idx >= 0) this.chunks[idx] = chunk;
      else this.chunks.push(chunk);
    }
  }

  async search(embedding: number[], topK: number): Promise<RetrievedChunk[]> {
    return this.chunks
      .map(c => ({
        ...c,
        score: this.cosineSimilarity(embedding, c.embedding),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK)
      .map(({ embedding: _, ...rest }) => rest);
  }

  get size() { return this.chunks.length; }
}

// --- The Pipeline ---

class RAGPipeline {
  private store = new SimpleVectorStore();
  private openai = new OpenAI();

  // Phase 1: LOAD
  async loadFiles(dirPath: string, extensions = [".md", ".txt"]): Promise<Document[]> {
    const docs: Document[] = [];
    const entries = await fs.readdir(dirPath, { recursive: true, withFileTypes: true });

    for (const entry of entries) {
      if (!entry.isFile()) continue;
      const ext = path.extname(entry.name).toLowerCase();
      if (!extensions.includes(ext)) continue;

      const fullPath = path.join(entry.parentPath ?? dirPath, entry.name);
      const content = await fs.readFile(fullPath, "utf-8");

      docs.push({
        id: crypto.createHash("sha256").update(fullPath).digest("hex").slice(0, 16),
        content,
        metadata: {
          source: fullPath,
          title: entry.name,
          fileType: ext,
        },
      });
    }

    console.log(`Loaded ${docs.length} documents`);
    return docs;
  }

  // Phase 1: CHUNK
  chunk(documents: Document[], chunkSize = 1000, overlap = 200): Chunk[] {
    const allChunks: Chunk[] = [];

    for (const doc of documents) {
      const separators = ["\n\n", "\n", ". ", " "];
      const rawChunks = this.splitRecursive(doc.content, chunkSize, overlap, separators);

      for (let i = 0; i < rawChunks.length; i++) {
        allChunks.push({
          id: `${doc.id}-${i}`,
          content: rawChunks[i],
          documentId: doc.id,
          index: i,
          metadata: {
            source: doc.metadata.source,
            title: doc.metadata.title,
            chunkIndex: i,
            totalChunks: rawChunks.length,
          },
        });
      }
    }

    console.log(`Created ${allChunks.length} chunks from ${documents.length} documents`);
    return allChunks;
  }

  private splitRecursive(text: string, size: number, overlap: number, seps: string[]): string[] {
    if (text.length <= size) return [text.trim()].filter(t => t.length > 0);

    let sep = "";
    for (const s of seps) {
      if (text.includes(s)) { sep = s; break; }
    }

    const parts = sep ? text.split(sep) : [text];
    const chunks: string[] = [];
    let current = "";

    for (const part of parts) {
      const candidate = current ? current + sep + part : part;
      if (candidate.length > size && current) {
        chunks.push(current.trim());
        const overlapText = overlap > 0 ? current.slice(-overlap) : "";
        current = overlapText + sep + part;
      } else {
        current = candidate;
      }
    }
    if (current.trim()) chunks.push(current.trim());

    const result: string[] = [];
    const remaining = seps.slice(seps.indexOf(sep) + 1);
    for (const chunk of chunks) {
      if (chunk.length > size && remaining.length > 0) {
        result.push(...this.splitRecursive(chunk, size, overlap, remaining));
      } else {
        result.push(chunk);
      }
    }
    return result;
  }

  // Phase 1: EMBED
  async embed(chunks: Chunk[], batchSize = 100): Promise<EmbeddedChunk[]> {
    const results: EmbeddedChunk[] = [];

    for (let i = 0; i < chunks.length; i += batchSize) {
      const batch = chunks.slice(i, i + batchSize);
      const texts = batch.map(c => {
        // Include title for context
        const prefix = c.metadata.title ? `${c.metadata.title}: ` : "";
        return prefix + c.content;
      });

      const response = await this.openai.embeddings.create({
        model: "text-embedding-3-small",
        input: texts,
      });

      for (let j = 0; j < batch.length; j++) {
        results.push({
          ...batch[j],
          embedding: response.data[j].embedding,
        });
      }

      console.log(`Embedded ${Math.min(i + batchSize, chunks.length)}/${chunks.length}`);
    }

    return results;
  }

  // Phase 1: STORE
  async index(embeddedChunks: EmbeddedChunk[]): Promise<void> {
    await this.store.upsert(embeddedChunks);
    console.log(`Indexed ${embeddedChunks.length} chunks (total: ${this.store.size})`);
  }

  // Phase 1: All-in-one
  async ingest(dirPath: string): Promise<void> {
    const docs = await this.loadFiles(dirPath);
    const chunks = this.chunk(docs);
    const embedded = await this.embed(chunks);
    await this.index(embedded);
  }

  // Phase 2: QUERY
  async query(question: string, topK = 5): Promise<RAGResponse> {
    const startTime = Date.now();

    // Step 1: Embed the question
    const embeddingResponse = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: question,
    });
    const queryEmbedding = embeddingResponse.data[0].embedding;

    // Step 2: Retrieve relevant chunks
    const retrieved = await this.store.search(queryEmbedding, topK);
    const retrievalTime = Date.now() - startTime;

    // Step 3: Build context
    const context = retrieved
      .map((chunk, i) => `[Source ${i + 1}: ${chunk.metadata.title ?? "Unknown"}]\n${chunk.content}`)
      .join("\n\n---\n\n");

    // Step 4: Generate answer
    const genStart = Date.now();
    const response = await this.openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0.1,
      messages: [
        {
          role: "system",
          content: `You are a helpful assistant. Answer questions based ONLY on the provided context. If the context doesn't contain the answer, say so. Cite sources using [Source N] notation.

CONTEXT:
${context}`,
        },
        { role: "user", content: question },
      ],
    });
    const generationTime = Date.now() - genStart;

    return {
      answer: response.choices[0].message.content ?? "",
      sources: retrieved.map(r => ({
        content: r.content.slice(0, 200) + "...",
        source: (r.metadata.source as string) ?? "unknown",
        score: r.score,
      })),
      timing: {
        retrievalMs: retrievalTime,
        generationMs: generationTime,
        totalMs: Date.now() - startTime,
      },
    };
  }
}

// --- Usage ---

async function main() {
  const rag = new RAGPipeline();

  // Index your documents
  await rag.ingest("./docs");

  // Ask questions
  const result = await rag.query("How do I configure authentication?");

  console.log("\nAnswer:", result.answer);
  console.log("\nSources:");
  for (const source of result.sources) {
    console.log(`  [${source.score.toFixed(3)}] ${source.source}`);
  }
  console.log(`\nTiming: ${result.timing.totalMs}ms (retrieval: ${result.timing.retrievalMs}ms, generation: ${result.timing.generationMs}ms)`);
}

main();
```

---

## Key Takeaways

You now have a complete RAG pipeline:

1. **Load** documents from files, URLs, or APIs
2. **Chunk** them using fixed-size, recursive, markdown-aware, or semantic strategies
3. **Embed** the chunks using OpenAI's API with batch processing
4. **Store** them in a vector store (in-memory, Pinecone, or pgvector)
5. **Retrieve** relevant chunks by embedding the query and searching
6. **Rerank** results using LLM scoring for better precision
7. **Generate** answers grounded in retrieved context

In Chapter 16, we'll apply this pipeline to real-world document sources — PDFs, YouTube transcripts, and web pages — and build a production document QA system with source attribution.

---

*Previous: [Ch 14 — Semantic Search](./14-semantic-search.md)* | *Next: [Ch 16 — Document QA Systems](./16-document-qa.md)*

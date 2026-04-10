<!--
  CHAPTER: 16
  TITLE: Document QA Systems
  PART: 3 — RAG & Knowledge Systems
  PHASE: 1 — Get Dangerous
  PREREQS: Ch 15 (The RAG Pipeline), Ch 5 (Structured Output)
  KEY_TOPICS: PDF loading, YouTube transcript loading, web scraping, chunking for accuracy, retrieve-then-generate, source attribution, document QA system
  DIFFICULTY: Intermediate
  LANGUAGE: TypeScript
  UPDATED: 2026-04-10
-->

# Chapter 16: Document QA Systems

> **Part 3 — RAG & Knowledge Systems** | Phase 1: Get Dangerous | Prerequisites: Ch 15, Ch 5 | Difficulty: Intermediate | Language: TypeScript

In Chapter 15 you built the RAG pipeline: load, chunk, embed, store, retrieve, generate. That pipeline worked on text files. But real companies don't keep their knowledge in tidy `.txt` files. It's in PDFs. In YouTube training videos. In Confluence pages and Notion databases. In Google Docs and Slack threads.

This chapter takes the pipeline you built and applies it to the messy real world. We'll build loaders for the sources that matter most, refine our chunking for accuracy, add source attribution so users know *where* the answer came from, and tie it all together into a document QA system that actually feels trustworthy.

### In This Chapter

1. PDF document loading
2. YouTube transcript loading
3. Web page scraping
4. Chunking for accuracy vs speed
5. The retrieve-then-generate pattern
6. Source attribution
7. Building a complete document QA system

### Related Chapters

- **Ch 15 (The RAG Pipeline)** — The foundation. This chapter applies it to real sources.
- **Ch 5 (Structured Output)** — We'll structure our QA answers with Zod schemas.
- **Ch 14 (Semantic Search)** — The search primitive powering retrieval.
- **Ch 17 (Advanced Retrieval)** — Optimize the retrieval strategies we use here.
- **Ch 44 (RAG vs Fine-Tuning)** — When to use this approach vs baking knowledge into weights.

---

## 1. PDF Document Loading

PDFs are the most common source of enterprise knowledge. Contracts, reports, manuals, research papers — they're everywhere. Unfortunately, PDFs are also one of the hardest formats to extract text from cleanly.

### 1.1 Using pdf-parse

The `pdf-parse` library handles most PDFs well. It extracts text content while preserving basic structure.

```typescript
// loaders/pdf-loader.ts
import fs from "fs/promises";
import path from "path";
import crypto from "crypto";
import pdfParse from "pdf-parse";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    [key: string]: unknown;
  };
}

async function loadPDF(filePath: string): Promise<Document> {
  const buffer = await fs.readFile(filePath);
  const pdf = await pdfParse(buffer);

  return {
    id: crypto.createHash("sha256").update(filePath).digest("hex").slice(0, 16),
    content: pdf.text,
    metadata: {
      source: filePath,
      title: pdf.info?.Title ?? path.basename(filePath, ".pdf"),
      author: pdf.info?.Author,
      pageCount: pdf.numpages,
      createdDate: pdf.info?.CreationDate,
      fileType: "pdf",
    },
  };
}

// Load with page-level granularity
async function loadPDFByPage(filePath: string): Promise<Document[]> {
  const buffer = await fs.readFile(filePath);
  const docs: Document[] = [];

  // pdf-parse with custom page render
  const pages: string[] = [];

  const pdf = await pdfParse(buffer, {
    pagerender: async (pageData: { getTextContent: () => Promise<{ items: { str: string }[] }> }) => {
      const textContent = await pageData.getTextContent();
      const text = textContent.items.map((item: { str: string }) => item.str).join(" ");
      pages.push(text);
      return text;
    },
  });

  const baseId = crypto.createHash("sha256").update(filePath).digest("hex").slice(0, 12);

  for (let i = 0; i < pages.length; i++) {
    if (pages[i].trim().length === 0) continue;

    docs.push({
      id: `${baseId}-p${i + 1}`,
      content: pages[i],
      metadata: {
        source: filePath,
        title: pdf.info?.Title ?? path.basename(filePath, ".pdf"),
        pageNumber: i + 1,
        totalPages: pages.length,
        fileType: "pdf",
      },
    });
  }

  return docs;
}

// Usage
async function main() {
  // Single document
  const doc = await loadPDF("./contracts/agreement.pdf");
  console.log(`Loaded: ${doc.metadata.title} (${doc.metadata.pageCount} pages)`);
  console.log(`Content length: ${doc.content.length} characters`);

  // Page-by-page
  const pages = await loadPDFByPage("./contracts/agreement.pdf");
  console.log(`Loaded ${pages.length} pages as separate documents`);
}
```

### 1.2 Handling Complex PDFs

Real PDFs are messy. Tables get garbled. Multi-column layouts merge into nonsense. Headers and footers repeat on every page.

```typescript
// loaders/pdf-cleaner.ts

function cleanPDFText(rawText: string): string {
  return rawText
    // Remove excessive whitespace
    .replace(/\s+/g, " ")
    // Fix broken words from line wraps (e.g., "docu-\nment" -> "document")
    .replace(/(\w)-\s+(\w)/g, "$1$2")
    // Remove page numbers (common patterns)
    .replace(/\b(Page|page)\s+\d+\s+(of\s+\d+)?/g, "")
    // Remove common headers/footers (customize per document)
    .replace(/^(CONFIDENTIAL|DRAFT|INTERNAL).*$/gm, "")
    // Normalize line breaks
    .replace(/\n{3,}/g, "\n\n")
    .trim();
}

// For tables: extract as structured text
function formatTableContent(rawText: string): string {
  // Simple heuristic: if a section has lots of aligned whitespace, it's probably a table
  // In production, you'd use a PDF table extraction library
  const lines = rawText.split("\n");
  const processed: string[] = [];

  for (const line of lines) {
    // If the line has multiple consecutive spaces, it might be table columns
    if (line.includes("   ")) {
      const cells = line.split(/\s{2,}/).filter(c => c.trim());
      if (cells.length >= 2) {
        processed.push(cells.join(" | "));
        continue;
      }
    }
    processed.push(line);
  }

  return processed.join("\n");
}
```

---

## 2. YouTube Transcript Loading

YouTube videos are a goldmine of knowledge — conference talks, tutorials, internal training sessions. The YouTube transcript API lets you extract the text.

### 2.1 Fetching Transcripts

```typescript
// loaders/youtube-loader.ts
import crypto from "crypto";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    [key: string]: unknown;
  };
}

interface TranscriptSegment {
  text: string;
  offset: number;  // Start time in milliseconds
  duration: number; // Duration in milliseconds
}

// Extract video ID from various YouTube URL formats
function extractVideoId(urlOrId: string): string {
  if (/^[a-zA-Z0-9_-]{11}$/.test(urlOrId)) {
    return urlOrId; // Already a video ID
  }

  const url = new URL(urlOrId);
  if (url.hostname === "youtu.be") {
    return url.pathname.slice(1);
  }
  return url.searchParams.get("v") ?? "";
}

// Fetch transcript using a transcript library
// In production, use youtube-transcript or youtubei.js
async function fetchTranscript(videoId: string): Promise<TranscriptSegment[]> {
  const url = `https://www.youtube.com/watch?v=${videoId}`;
  const response = await fetch(url);
  const html = await response.text();

  // Extract the captions track URL from the page data
  const captionsMatch = html.match(/"captions":\s*(\{.*?"captionTracks".*?\})/s);
  if (!captionsMatch) {
    throw new Error(`No captions found for video ${videoId}`);
  }

  const captionsData = JSON.parse(captionsMatch[1]);
  const tracks = captionsData.playerCaptionsTracklistRenderer?.captionTracks;

  if (!tracks || tracks.length === 0) {
    throw new Error("No caption tracks available");
  }

  // Prefer manual captions over auto-generated
  const track = tracks.find((t: { kind?: string }) => t.kind !== "asr") ?? tracks[0];
  const captionUrl = track.baseUrl;

  const captionResponse = await fetch(captionUrl);
  const captionXml = await captionResponse.text();

  // Parse the XML transcript
  const segments: TranscriptSegment[] = [];
  const regex = /<text start="([\d.]+)" dur="([\d.]+)">(.*?)<\/text>/g;
  let match;

  while ((match = regex.exec(captionXml)) !== null) {
    segments.push({
      offset: Math.floor(parseFloat(match[1]) * 1000),
      duration: Math.floor(parseFloat(match[2]) * 1000),
      text: decodeHTMLEntities(match[3]),
    });
  }

  return segments;
}

function decodeHTMLEntities(text: string): string {
  return text
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/\n/g, " ");
}

async function loadYouTubeTranscript(videoUrl: string): Promise<Document> {
  const videoId = extractVideoId(videoUrl);
  const segments = await fetchTranscript(videoId);

  // Combine all segments into full text
  const fullText = segments.map(s => s.text).join(" ");

  return {
    id: crypto.createHash("sha256").update(videoId).digest("hex").slice(0, 16),
    content: fullText,
    metadata: {
      source: `https://www.youtube.com/watch?v=${videoId}`,
      videoId,
      title: `YouTube Video ${videoId}`,
      segmentCount: segments.length,
      durationMs: segments.length > 0
        ? segments[segments.length - 1].offset + segments[segments.length - 1].duration
        : 0,
      fileType: "youtube",
    },
  };
}

// Load with timestamps for source attribution
async function loadYouTubeWithTimestamps(videoUrl: string): Promise<Document[]> {
  const videoId = extractVideoId(videoUrl);
  const segments = await fetchTranscript(videoId);

  // Group segments into ~60-second windows
  const windowSize = 60_000; // 60 seconds in milliseconds
  const docs: Document[] = [];
  let currentWindow: TranscriptSegment[] = [];
  let windowStart = 0;

  for (const segment of segments) {
    if (segment.offset - windowStart >= windowSize && currentWindow.length > 0) {
      docs.push(createTimestampedDoc(videoId, currentWindow, windowStart, docs.length));
      currentWindow = [];
      windowStart = segment.offset;
    }
    currentWindow.push(segment);
  }

  // Last window
  if (currentWindow.length > 0) {
    docs.push(createTimestampedDoc(videoId, currentWindow, windowStart, docs.length));
  }

  return docs;
}

function createTimestampedDoc(
  videoId: string,
  segments: TranscriptSegment[],
  windowStart: number,
  index: number
): Document {
  const text = segments.map(s => s.text).join(" ");
  const startSeconds = Math.floor(windowStart / 1000);

  return {
    id: `yt-${videoId}-${index}`,
    content: text,
    metadata: {
      source: `https://www.youtube.com/watch?v=${videoId}&t=${startSeconds}`,
      videoId,
      startTime: startSeconds,
      endTime: Math.floor(
        (segments[segments.length - 1].offset + segments[segments.length - 1].duration) / 1000
      ),
      timestamp: formatTime(startSeconds),
      fileType: "youtube",
    },
  };
}

function formatTime(seconds: number): string {
  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  const s = seconds % 60;
  if (h > 0) return `${h}:${m.toString().padStart(2, "0")}:${s.toString().padStart(2, "0")}`;
  return `${m}:${s.toString().padStart(2, "0")}`;
}
```

---

## 3. Web Page Scraping

### 3.1 Extracting Content from Web Pages

```typescript
// loaders/web-loader.ts
import crypto from "crypto";

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    title?: string;
    [key: string]: unknown;
  };
}

async function loadWebPage(url: string): Promise<Document> {
  const response = await fetch(url, {
    headers: {
      "User-Agent": "DocumentQABot/1.0 (contact@example.com)",
    },
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch ${url}: ${response.status} ${response.statusText}`);
  }

  const html = await response.text();
  const content = extractMainContent(html);
  const title = extractTitle(html);

  return {
    id: crypto.createHash("sha256").update(url).digest("hex").slice(0, 16),
    content,
    metadata: {
      source: url,
      url,
      title,
      fetchedAt: new Date().toISOString(),
      contentLength: content.length,
      fileType: "web",
    },
  };
}

function extractTitle(html: string): string {
  const match = html.match(/<title[^>]*>([\s\S]*?)<\/title>/i);
  return match ? match[1].trim() : "Untitled";
}

function extractMainContent(html: string): string {
  // Remove non-content elements
  let cleaned = html
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "")
    .replace(/<nav[\s\S]*?<\/nav>/gi, "")
    .replace(/<footer[\s\S]*?<\/footer>/gi, "")
    .replace(/<header[\s\S]*?<\/header>/gi, "")
    .replace(/<!--[\s\S]*?-->/g, "");

  // Try to extract from <main> or <article> if present
  const mainMatch = cleaned.match(/<main[\s\S]*?>([\s\S]*?)<\/main>/i);
  const articleMatch = cleaned.match(/<article[\s\S]*?>([\s\S]*?)<\/article>/i);

  if (mainMatch) cleaned = mainMatch[1];
  else if (articleMatch) cleaned = articleMatch[1];

  // Strip remaining HTML tags, preserving structure
  const text = cleaned
    .replace(/<h[1-6][^>]*>([\s\S]*?)<\/h[1-6]>/gi, "\n\n## $1\n\n")
    .replace(/<p[^>]*>([\s\S]*?)<\/p>/gi, "\n$1\n")
    .replace(/<li[^>]*>([\s\S]*?)<\/li>/gi, "\n- $1")
    .replace(/<br\s*\/?>/gi, "\n")
    .replace(/<[^>]+>/g, "")
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/&nbsp;/g, " ")
    .replace(/\n{3,}/g, "\n\n")
    .trim();

  return text;
}

// Crawl multiple pages from a sitemap or link list
async function loadWebsite(
  urls: string[],
  options: { concurrency?: number; delayMs?: number } = {}
): Promise<Document[]> {
  const { concurrency = 3, delayMs = 1000 } = options;
  const results: Document[] = [];

  for (let i = 0; i < urls.length; i += concurrency) {
    const batch = urls.slice(i, i + concurrency);
    const batchResults = await Promise.allSettled(batch.map(loadWebPage));

    for (const result of batchResults) {
      if (result.status === "fulfilled") {
        results.push(result.value);
      } else {
        console.warn(`Failed to load: ${result.reason}`);
      }
    }

    // Rate limiting
    if (i + concurrency < urls.length) {
      await new Promise(r => setTimeout(r, delayMs));
    }

    console.log(`Loaded ${Math.min(i + concurrency, urls.length)}/${urls.length} pages`);
  }

  return results;
}
```

---

## 4. Chunking for Accuracy vs Speed

### 4.1 The Chunk Size Tradeoff

This is one of the most important decisions in your RAG pipeline, and there's no single right answer.

```typescript
// chunking-tradeoffs.ts

// Small chunks (200-500 chars): High precision, low recall
// - More chunks per document = more targeted retrieval
// - But: each chunk has less context
// - Good for: factual Q&A, specific lookups

// Medium chunks (500-1500 chars): Balanced
// - The sweet spot for most applications
// - Enough context to understand the passage
// - Good for: general-purpose document QA

// Large chunks (1500-4000 chars): High recall, lower precision
// - Fewer chunks, but each has more context
// - Better for: summarization, complex questions needing surrounding context
// - Risk: lots of irrelevant text in the context window

interface ChunkingConfig {
  name: string;
  chunkSize: number;
  overlap: number;
  bestFor: string;
}

const chunkingPresets: Record<string, ChunkingConfig> = {
  precise: {
    name: "Precise Retrieval",
    chunkSize: 300,
    overlap: 50,
    bestFor: "Factual Q&A, looking up specific facts",
  },
  balanced: {
    name: "Balanced",
    chunkSize: 1000,
    overlap: 200,
    bestFor: "General document QA (recommended default)",
  },
  contextual: {
    name: "Context-Rich",
    chunkSize: 2000,
    overlap: 400,
    bestFor: "Complex questions, summarization, when surrounding context matters",
  },
};
```

### 4.2 Parent-Child Chunking

A powerful technique: chunk into small pieces for precise retrieval, but include the larger parent context when generating answers.

```typescript
// chunkers/parent-child.ts

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
}

interface ParentChildChunks {
  parents: Chunk[];  // Large chunks for context
  children: Chunk[]; // Small chunks for retrieval
}

function parentChildChunk(
  document: { id: string; content: string; metadata: Record<string, unknown> },
  options: {
    parentSize?: number;
    childSize?: number;
    childOverlap?: number;
  } = {}
): ParentChildChunks {
  const { parentSize = 2000, childSize = 400, childOverlap = 50 } = options;

  // Create parent chunks
  const parents: Chunk[] = [];
  for (let i = 0; i < document.content.length; i += parentSize) {
    const content = document.content.slice(i, i + parentSize);
    parents.push({
      id: `${document.id}-parent-${parents.length}`,
      content,
      documentId: document.id,
      index: parents.length,
      metadata: { ...document.metadata, chunkType: "parent" },
    });
  }

  // Create child chunks within each parent
  const children: Chunk[] = [];
  for (const parent of parents) {
    for (let i = 0; i < parent.content.length; i += childSize - childOverlap) {
      const content = parent.content.slice(i, i + childSize);
      if (content.trim().length === 0) continue;

      children.push({
        id: `${parent.id}-child-${children.length}`,
        content,
        documentId: document.id,
        index: children.length,
        metadata: {
          ...document.metadata,
          chunkType: "child",
          parentId: parent.id,
        },
      });
    }
  }

  return { parents, children };
}

// During retrieval:
// 1. Search child chunks (precise matching)
// 2. Look up parent chunks (richer context)
// 3. Pass parent context to the LLM

async function retrieveWithParentContext(
  childResults: (Chunk & { score: number })[],
  parentMap: Map<string, Chunk>,
  topK: number = 3
): Promise<Chunk[]> {
  // De-duplicate by parent
  const seenParents = new Set<string>();
  const parentChunks: Chunk[] = [];

  for (const child of childResults) {
    const parentId = child.metadata.parentId as string;
    if (parentId && !seenParents.has(parentId)) {
      seenParents.add(parentId);
      const parent = parentMap.get(parentId);
      if (parent) parentChunks.push(parent);
    }
    if (parentChunks.length >= topK) break;
  }

  return parentChunks;
}
```

---

## 5. The Retrieve-Then-Generate Pattern

### 5.1 Basic Pattern

This is the core of document QA: retrieve context, then generate an answer.

```typescript
// patterns/retrieve-then-generate.ts
import OpenAI from "openai";
import { z } from "zod";

const openai = new OpenAI();

// Structured output for QA responses (spiraling back to Ch 5)
const QAResponseSchema = z.object({
  answer: z.string().describe("The answer to the question"),
  confidence: z.enum(["high", "medium", "low"]).describe("Confidence based on available context"),
  sources: z.array(z.object({
    sourceIndex: z.number().describe("Which source number (1-based)"),
    quote: z.string().describe("The relevant quote from the source"),
    relevance: z.string().describe("Why this source is relevant"),
  })),
  followUpQuestions: z.array(z.string()).describe("Suggested follow-up questions"),
});

type QAResponse = z.infer<typeof QAResponseSchema>;

interface RetrievedContext {
  content: string;
  source: string;
  score: number;
  metadata: Record<string, unknown>;
}

async function retrieveThenGenerate(
  question: string,
  retrievedContext: RetrievedContext[],
  options: {
    model?: string;
    includeFollowUps?: boolean;
  } = {}
): Promise<QAResponse> {
  const { model = "gpt-4o", includeFollowUps = true } = options;

  const contextText = retrievedContext
    .map((ctx, i) => `[Source ${i + 1}: ${ctx.source}]\n${ctx.content}`)
    .join("\n\n---\n\n");

  const response = await openai.chat.completions.create({
    model,
    temperature: 0.1,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are a document QA system. Answer questions based ONLY on the provided sources.

RULES:
1. If the sources don't contain enough information, set confidence to "low" and explain what's missing.
2. Always cite specific sources using [Source N] format.
3. Include direct quotes when possible.
4. ${includeFollowUps ? "Suggest 2-3 follow-up questions the user might ask." : "Do not include follow-up questions."}

Respond in JSON matching this schema:
{
  "answer": string,
  "confidence": "high" | "medium" | "low",
  "sources": [{ "sourceIndex": number, "quote": string, "relevance": string }],
  "followUpQuestions": string[]
}

SOURCES:
${contextText}`,
      },
      { role: "user", content: question },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");
  return QAResponseSchema.parse(parsed);
}
```

### 5.2 Conversational QA

Real users don't ask one question and leave. They ask follow-ups. A conversational QA system needs to understand context from previous turns.

```typescript
// patterns/conversational-qa.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface Message {
  role: "user" | "assistant";
  content: string;
}

interface RetrievedContext {
  content: string;
  source: string;
  score: number;
  metadata: Record<string, unknown>;
}

// Step 1: Rewrite the question to be standalone
// "What about their pricing?" -> "What is Pinecone's pricing?"
async function rewriteQuestion(
  question: string,
  history: Message[]
): Promise<string> {
  if (history.length === 0) return question;

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0,
    messages: [
      {
        role: "system",
        content: `Given the conversation history and a follow-up question, rewrite the question to be standalone and self-contained. Include all necessary context from the conversation.

Return ONLY the rewritten question, nothing else.`,
      },
      ...history.map(m => ({ role: m.role as "user" | "assistant", content: m.content })),
      { role: "user", content: `Rewrite this follow-up question as a standalone question: "${question}"` },
    ],
  });

  return response.choices[0].message.content ?? question;
}

class ConversationalQA {
  private history: Message[] = [];
  private retrieveFn: (query: string) => Promise<RetrievedContext[]>;

  constructor(retrieveFn: (query: string) => Promise<RetrievedContext[]>) {
    this.retrieveFn = retrieveFn;
  }

  async ask(question: string): Promise<{ answer: string; sources: RetrievedContext[] }> {
    // Step 1: Rewrite to standalone
    const standaloneQuestion = await rewriteQuestion(question, this.history);
    console.log(`Rewritten: "${question}" -> "${standaloneQuestion}"`);

    // Step 2: Retrieve relevant context
    const context = await this.retrieveFn(standaloneQuestion);

    // Step 3: Generate answer
    const contextText = context
      .map((c, i) => `[Source ${i + 1}: ${c.source}]\n${c.content}`)
      .join("\n\n---\n\n");

    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0.1,
      messages: [
        {
          role: "system",
          content: `You are a helpful document QA assistant. Answer based on the provided sources. Cite sources using [Source N] format.

SOURCES:
${contextText}`,
        },
        // Include conversation history for coherent multi-turn
        ...this.history.slice(-6).map(m => ({
          role: m.role as "user" | "assistant",
          content: m.content,
        })),
        { role: "user", content: question },
      ],
    });

    const answer = response.choices[0].message.content ?? "";

    // Update history
    this.history.push({ role: "user", content: question });
    this.history.push({ role: "assistant", content: answer });

    return { answer, sources: context };
  }

  clearHistory() {
    this.history = [];
  }
}
```

---

## 6. Source Attribution

Users need to trust AI answers. Source attribution is how you build that trust — show them exactly where the information came from.

### 6.1 Inline Citations

```typescript
// attribution/inline-citations.ts
import OpenAI from "openai";

const openai = new OpenAI();

interface CitedAnswer {
  text: string;
  citations: {
    id: string;
    source: string;
    quote: string;
    url?: string;
    pageNumber?: number;
    timestamp?: string;
  }[];
}

async function generateWithCitations(
  question: string,
  sources: { content: string; metadata: Record<string, unknown> }[]
): Promise<CitedAnswer> {
  const sourceText = sources.map((s, i) => {
    const label = s.metadata.title ?? s.metadata.source ?? `Source ${i + 1}`;
    return `[${i + 1}] ${label}\n${s.content}`;
  }).join("\n\n---\n\n");

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.1,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are a research assistant. Answer the question using ONLY the provided sources.

For every claim you make, add a citation in the format [N] where N is the source number. Include the exact quote you're citing.

Respond in JSON:
{
  "text": "The answer with [1] inline citations [2].",
  "citations": [
    {
      "id": "1",
      "quote": "exact text from the source"
    }
  ]
}

SOURCES:
${sourceText}`,
      },
      { role: "user", content: question },
    ],
  });

  const parsed = JSON.parse(response.choices[0].message.content ?? "{}");

  // Enrich citations with source metadata
  const citations = (parsed.citations ?? []).map((c: { id: string; quote: string }) => {
    const sourceIdx = parseInt(c.id) - 1;
    const source = sources[sourceIdx];
    return {
      id: c.id,
      source: (source?.metadata.title ?? source?.metadata.source ?? "Unknown") as string,
      quote: c.quote,
      url: source?.metadata.url as string | undefined,
      pageNumber: source?.metadata.pageNumber as number | undefined,
      timestamp: source?.metadata.timestamp as string | undefined,
    };
  });

  return { text: parsed.text, citations };
}
```

### 6.2 Source Links

Generate clickable links back to the original documents.

```typescript
// attribution/source-links.ts

interface SourceLink {
  text: string;
  url: string;
  type: "pdf" | "web" | "youtube" | "file";
  detail?: string; // e.g., "Page 5" or "2:34"
}

function generateSourceLink(metadata: Record<string, unknown>): SourceLink {
  const fileType = metadata.fileType as string;

  switch (fileType) {
    case "pdf":
      return {
        text: (metadata.title as string) ?? "PDF Document",
        url: metadata.source as string,
        type: "pdf",
        detail: metadata.pageNumber ? `Page ${metadata.pageNumber}` : undefined,
      };

    case "youtube":
      return {
        text: (metadata.title as string) ?? "YouTube Video",
        url: metadata.source as string, // Already includes &t= timestamp
        type: "youtube",
        detail: metadata.timestamp as string | undefined,
      };

    case "web":
      return {
        text: (metadata.title as string) ?? (metadata.url as string),
        url: metadata.url as string,
        type: "web",
      };

    default:
      return {
        text: (metadata.title as string) ?? "Document",
        url: metadata.source as string,
        type: "file",
      };
  }
}

// Format citations for display
function formatCitedAnswer(answer: string, sourceLinks: SourceLink[]): string {
  const footer = sourceLinks
    .map((link, i) => {
      const detail = link.detail ? ` (${link.detail})` : "";
      return `[${i + 1}] ${link.text}${detail} — ${link.url}`;
    })
    .join("\n");

  return `${answer}\n\n---\nSources:\n${footer}`;
}
```

---

## 7. Building a Complete Document QA System

Let's tie everything together into a production-ready document QA system.

```typescript
// document-qa-system.ts
import OpenAI from "openai";
import fs from "fs/promises";
import path from "path";
import crypto from "crypto";

const openai = new OpenAI();

// --- Types ---

interface Document {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
}

interface Chunk {
  id: string;
  content: string;
  documentId: string;
  index: number;
  metadata: Record<string, unknown>;
  embedding?: number[];
}

interface QAResult {
  answer: string;
  confidence: "high" | "medium" | "low";
  sources: {
    title: string;
    excerpt: string;
    score: number;
    link: string;
  }[];
  timing: {
    retrievalMs: number;
    generationMs: number;
  };
}

// --- Vector Store ---

class VectorStore {
  private chunks: (Chunk & { embedding: number[] })[] = [];

  private cosine(a: number[], b: number[]): number {
    let dot = 0, nA = 0, nB = 0;
    for (let i = 0; i < a.length; i++) {
      dot += a[i] * b[i];
      nA += a[i] * a[i];
      nB += b[i] * b[i];
    }
    return dot / (Math.sqrt(nA) * Math.sqrt(nB));
  }

  add(chunks: (Chunk & { embedding: number[] })[]) {
    this.chunks.push(...chunks);
  }

  search(embedding: number[], topK: number): (Chunk & { score: number })[] {
    return this.chunks
      .map(c => ({ ...c, score: this.cosine(embedding, c.embedding) }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK);
  }

  get size() { return this.chunks.length; }
}

// --- Document QA System ---

class DocumentQA {
  private store = new VectorStore();
  private openai = new OpenAI();
  private conversationHistory: { role: "user" | "assistant"; content: string }[] = [];

  // --- Loaders ---

  async loadTextFiles(dirPath: string, extensions = [".md", ".txt"]): Promise<Document[]> {
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
          fileType: ext.slice(1),
        },
      });
    }

    return docs;
  }

  // --- Chunking ---

  private chunkDocument(doc: Document, chunkSize = 1000, overlap = 200): Chunk[] {
    const chunks: Chunk[] = [];
    const separators = ["\n\n", "\n", ". ", " "];
    const rawChunks = this.splitRecursive(doc.content, chunkSize, overlap, separators);

    for (let i = 0; i < rawChunks.length; i++) {
      chunks.push({
        id: `${doc.id}-${i}`,
        content: rawChunks[i],
        documentId: doc.id,
        index: i,
        metadata: {
          ...doc.metadata,
          chunkIndex: i,
          totalChunks: rawChunks.length,
        },
      });
    }

    return chunks;
  }

  private splitRecursive(text: string, size: number, overlap: number, seps: string[]): string[] {
    if (text.length <= size) return [text.trim()].filter(t => t.length > 0);

    let sep = "";
    for (const s of seps) { if (text.includes(s)) { sep = s; break; } }

    const parts = sep ? text.split(sep) : [text];
    const chunks: string[] = [];
    let current = "";

    for (const part of parts) {
      const candidate = current ? current + sep + part : part;
      if (candidate.length > size && current) {
        chunks.push(current.trim());
        current = overlap > 0 ? current.slice(-overlap) + sep + part : part;
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

  // --- Indexing ---

  async ingest(documents: Document[]): Promise<void> {
    const allChunks: Chunk[] = [];
    for (const doc of documents) {
      allChunks.push(...this.chunkDocument(doc));
    }
    console.log(`Created ${allChunks.length} chunks from ${documents.length} documents`);

    // Embed in batches
    const batchSize = 100;
    for (let i = 0; i < allChunks.length; i += batchSize) {
      const batch = allChunks.slice(i, i + batchSize);
      const texts = batch.map(c => {
        const title = c.metadata.title ? `${c.metadata.title}: ` : "";
        return title + c.content;
      });

      const response = await this.openai.embeddings.create({
        model: "text-embedding-3-small",
        input: texts,
      });

      const embedded = batch.map((chunk, j) => ({
        ...chunk,
        embedding: response.data[j].embedding,
      }));

      this.store.add(embedded);
      console.log(`Indexed ${Math.min(i + batchSize, allChunks.length)}/${allChunks.length}`);
    }
  }

  // --- Querying ---

  async ask(question: string, options: { topK?: number; conversational?: boolean } = {}): Promise<QAResult> {
    const { topK = 5, conversational = true } = options;
    const start = Date.now();

    // If conversational, rewrite the question to be standalone
    let searchQuery = question;
    if (conversational && this.conversationHistory.length > 0) {
      searchQuery = await this.rewriteQuestion(question);
    }

    // Retrieve
    const embResponse = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: searchQuery,
    });

    const results = this.store.search(embResponse.data[0].embedding, topK);
    const retrievalMs = Date.now() - start;

    // Generate
    const genStart = Date.now();
    const context = results
      .map((r, i) => `[Source ${i + 1}: ${r.metadata.title ?? "Unknown"}]\n${r.content}`)
      .join("\n\n---\n\n");

    const messages: { role: "system" | "user" | "assistant"; content: string }[] = [
      {
        role: "system",
        content: `You are a document QA assistant. Answer questions based ONLY on the provided sources.

RULES:
- If the sources don't contain enough information, say so honestly.
- Cite sources using [Source N] format.
- Be concise but thorough.
- Assess your confidence: "high" if clearly answered, "medium" if partially, "low" if barely.

SOURCES:
${context}`,
      },
    ];

    // Add conversation history for multi-turn
    if (conversational) {
      messages.push(...this.conversationHistory.slice(-4));
    }

    messages.push({ role: "user", content: question });

    const response = await this.openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0.1,
      messages,
    });

    const answer = response.choices[0].message.content ?? "";
    const generationMs = Date.now() - genStart;

    // Determine confidence from the answer text and retrieval scores
    let confidence: "high" | "medium" | "low" = "medium";
    if (answer.includes("don't have enough") || answer.includes("not found")) {
      confidence = "low";
    } else if (results[0]?.score > 0.8) {
      confidence = "high";
    }

    // Update conversation history
    this.conversationHistory.push({ role: "user", content: question });
    this.conversationHistory.push({ role: "assistant", content: answer });

    return {
      answer,
      confidence,
      sources: results.map(r => ({
        title: (r.metadata.title as string) ?? "Unknown",
        excerpt: r.content.slice(0, 150) + "...",
        score: r.score,
        link: (r.metadata.source as string) ?? "",
      })),
      timing: { retrievalMs, generationMs },
    };
  }

  private async rewriteQuestion(question: string): Promise<string> {
    const response = await this.openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0,
      messages: [
        {
          role: "system",
          content: "Rewrite the follow-up question as a standalone question. Return ONLY the rewritten question.",
        },
        ...this.conversationHistory.slice(-4),
        { role: "user", content: question },
      ],
    });
    return response.choices[0].message.content ?? question;
  }

  clearHistory() {
    this.conversationHistory = [];
  }
}

// --- Main ---

async function main() {
  const qa = new DocumentQA();

  // Load and index documents
  const docs = await qa.loadTextFiles("./docs");
  await qa.ingest(docs);

  // Single question
  console.log("\n=== Question 1 ===");
  const result1 = await qa.ask("How do I set up authentication?");
  console.log(`Answer: ${result1.answer}`);
  console.log(`Confidence: ${result1.confidence}`);
  console.log(`Sources: ${result1.sources.map(s => s.title).join(", ")}`);
  console.log(`Timing: retrieval ${result1.timing.retrievalMs}ms, generation ${result1.timing.generationMs}ms`);

  // Follow-up question (conversational)
  console.log("\n=== Question 2 (follow-up) ===");
  const result2 = await qa.ask("What about rate limiting for that?");
  console.log(`Answer: ${result2.answer}`);
  console.log(`Confidence: ${result2.confidence}`);
}

main();
```

---

## Key Takeaways

You now have the tools to build a document QA system from real-world sources:

1. **PDF loading** — Extract text from PDFs, page by page, with cleaning
2. **YouTube transcripts** — Load video content with timestamps for attribution
3. **Web scraping** — Extract main content from web pages
4. **Smart chunking** — Choose the right chunk size for your use case, use parent-child chunking for precision + context
5. **Retrieve-then-generate** — The core pattern: find context, then answer
6. **Source attribution** — Inline citations, source links, confidence scores
7. **Conversational QA** — Handle follow-up questions with query rewriting

In Chapter 17, we'll optimize every stage of this pipeline with advanced retrieval techniques: hybrid search, reranking, HyDE, query expansion, and more.

---

*Previous: [Ch 15 — The RAG Pipeline](./15-rag-pipeline.md)* | *Next: [Ch 17 — Advanced Retrieval](./17-advanced-retrieval.md)*

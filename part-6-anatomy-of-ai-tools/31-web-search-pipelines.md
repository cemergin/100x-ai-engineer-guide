<!--
  CHAPTER: 31
  TITLE: Web Search & Knowledge Pipelines
  PART: 6 — Anatomy of AI Developer Tools
  PHASE: 2 — Become an Expert
  PREREQS: Ch 15 (RAG Pipeline), Ch 17 (Advanced Retrieval), Ch 30 (Tool Execution)
  KEY_TOPICS: web search architecture, pre-approved domains, content extraction, Turndown, paraphrase limits, WebFetch vs web_search, llms.txt, knowledge pipeline design
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Architecture
  UPDATED: 2026-04-10
-->

# Chapter 31: Web Search & Knowledge Pipelines

> **Part 6 — Anatomy of AI Developer Tools** | Phase 2: Become an Expert | Prerequisites: Ch 15, Ch 17, Ch 30 | Difficulty: Advanced | Language: TypeScript + Architecture

When an AI coding agent needs to know something that is not in your codebase -- a library's API, a Stack Overflow solution, the latest framework documentation -- it searches the web. This sounds simple. It is not. The Claude Code leak revealed a two-tier web system with 107 pre-approved domains, a 125-character paraphrase limit for non-approved sites, encrypted content blobs, and a content extraction pipeline that strips tables, ignores `<head>` tags, and converts HTML to markdown using Turndown. Understanding these mechanisms reveals both the power and the hard limits of web-connected AI agents.

### In This Chapter

1. How web search works in AI coding tools
2. The two-tier web: pre-approved vs. everyone else
3. The 107 pre-approved domains
4. Content extraction: Turndown and its limitations
5. WebFetch vs. web_search: two tools, different behaviors
6. The content negotiation hack
7. The 125-character paraphrase limit
8. Max 8 results, no pagination
9. llms.txt: what Claude Code does (and does not) look for
10. Lessons for RAG system design

### Related Chapters

- **Ch 15 (The RAG Pipeline)** -- introduced retrieval concepts; this chapter shows production search
- **Ch 17 (Advanced Retrieval)** -- covered retrieval strategies; this chapter shows real constraints
- **Ch 30 (Tool Execution)** -- web tools are executed through the same pipeline
- **Ch 59 (Building Internal AI Tools)** -- will apply these patterns to internal knowledge systems

---

## 1. How Web Search Works in AI Coding Tools

### 1.1 The Naive Mental Model (Wrong)

Most people imagine web search in AI tools works like this:

```
User asks question --> Agent searches Google --> 
Agent reads web pages --> Agent responds with answer
```

This is wrong in almost every detail. The actual architecture is far more constrained.

### 1.2 The Actual Architecture

```
┌───────────────────────────────────────────────────────────┐
│              WEB SEARCH ARCHITECTURE                       │
│                                                            │
│  1. MODEL decides to search                                │
│     (tool_use: web_search, query: "Next.js 15 routing")   │
│                                                            │
│  2. HARNESS sends search request to Anthropic API          │
│     (not to Google or Bing directly)                       │
│                                                            │
│  3. ANTHROPIC API performs the actual search                │
│     (implementation is opaque to the harness)              │
│                                                            │
│  4. API returns search results                             │
│     ├── Up to 8 results                                    │
│     ├── Each result: title, URL, snippet                   │
│     └── For pre-approved domains: full content             │
│                                                            │
│  5. MODEL processes search results                         │
│     ├── Reads snippets/content                             │
│     ├── May fetch specific URLs for more detail            │
│     └── Synthesizes an answer                              │
│                                                            │
│  6. For non-approved domains:                              │
│     ├── Content is encrypted/opaque                        │
│     ├── Model can only generate 125-char paraphrases       │
│     └── No direct quoting of content                       │
└───────────────────────────────────────────────────────────┘
```

**Key insight:** The agent does not search the web directly. It calls a search tool, which hits an API, which returns structured results. The agent never has a browser. It never sees raw HTML (except through WebFetch). The search results are pre-processed and filtered before the agent sees them.

### 1.3 Why the Indirection

Why not just let the agent search Google directly? Several reasons:

1. **Copyright compliance** -- the two-tier system controls how much copyrighted content enters the model's context
2. **Safety** -- prevents the agent from accessing malicious websites
3. **Reliability** -- Anthropic can control the search quality and rate limits
4. **Cost** -- Anthropic can cache popular searches
5. **Monitoring** -- Anthropic can track what agents are searching for

---

## 2. The Two-Tier Web

### 2.1 Tier 1: Pre-Approved Domains (Full Access)

For a curated list of domains, the agent gets full content access. It can read the complete text, quote directly, and use the information without restrictions.

```
┌───────────────────────────────────────────────┐
│          TIER 1: PRE-APPROVED DOMAINS          │
│                                                 │
│  Access level: FULL CONTENT                     │
│  ├── Complete page text extracted               │
│  ├── Direct quoting allowed                     │
│  ├── No character limits on quotes              │
│  └── Content converted to markdown              │
│                                                 │
│  Why these domains:                             │
│  ├── Official documentation sites               │
│  ├── Open-source project pages                  │
│  ├── Developer reference sites                  │
│  └── Sites with permissive content policies     │
└───────────────────────────────────────────────┘
```

### 2.2 Tier 2: Everything Else (Restricted Access)

For all other domains, the agent operates under strict restrictions:

```
┌───────────────────────────────────────────────┐
│          TIER 2: ALL OTHER DOMAINS             │
│                                                 │
│  Access level: RESTRICTED                       │
│  ├── Content is available but restricted        │
│  ├── No direct quoting longer than ~125 chars   │
│  ├── Must paraphrase, not copy                  │
│  ├── A smaller model (Haiku) post-processes     │
│  │   to enforce the paraphrase limit            │
│  └── Designed for copyright compliance          │
│                                                 │
│  Impact on agent behavior:                      │
│  ├── Agent can SEE the information              │
│  ├── Agent can UNDERSTAND the information       │
│  ├── Agent can USE the information              │
│  ├── Agent CANNOT directly quote it verbatim    │
│  └── Agent must paraphrase in its own words     │
└───────────────────────────────────────────────┘
```

### 2.3 How the Tier Is Determined

```typescript
// Tier determination (reconstructed)
function getContentTier(url: string): 'full' | 'restricted' {
  const domain = extractDomain(url);
  
  // Check against the pre-approved list
  if (PRE_APPROVED_DOMAINS.includes(domain)) {
    return 'full';
  }
  
  // Check for subdomain matches
  // e.g., "docs.python.org" matches "python.org"
  for (const approved of PRE_APPROVED_DOMAINS) {
    if (domain.endsWith('.' + approved)) {
      return 'full';
    }
  }
  
  return 'restricted';
}
```

---

## 3. The 107 Pre-Approved Domains

### 3.1 Categories

The community cataloged approximately 107 pre-approved domains from the leak. They fall into clear categories:

```
┌────────────────────────────────────────────────────┐
│        PRE-APPROVED DOMAIN CATEGORIES               │
│                                                      │
│  PROGRAMMING LANGUAGE DOCS (~25 domains)             │
│  ├── docs.python.org                                 │
│  ├── developer.mozilla.org (MDN)                     │
│  ├── doc.rust-lang.org                               │
│  ├── golang.org/doc                                  │
│  ├── docs.oracle.com/javase                          │
│  ├── learn.microsoft.com (TypeScript, .NET, etc.)    │
│  ├── ruby-doc.org                                    │
│  └── ...                                             │
│                                                      │
│  FRAMEWORK DOCS (~30 domains)                        │
│  ├── nextjs.org                                      │
│  ├── react.dev                                       │
│  ├── vuejs.org                                       │
│  ├── angular.dev                                     │
│  ├── docs.djangoproject.com                          │
│  ├── flask.palletsprojects.com                       │
│  ├── expressjs.com                                   │
│  ├── docs.astro.build                                │
│  └── ...                                             │
│                                                      │
│  PACKAGE REGISTRIES & REPOS (~10 domains)            │
│  ├── npmjs.com                                       │
│  ├── pypi.org                                        │
│  ├── crates.io                                       │
│  ├── pkg.go.dev                                      │
│  └── ...                                             │
│                                                      │
│  CLOUD & INFRASTRUCTURE (~15 domains)                │
│  ├── docs.aws.amazon.com                             │
│  ├── cloud.google.com/docs                           │
│  ├── docs.microsoft.com/azure                        │
│  ├── docs.docker.com                                 │
│  ├── kubernetes.io/docs                              │
│  └── ...                                             │
│                                                      │
│  DEVELOPER TOOLS (~15 domains)                       │
│  ├── docs.github.com                                 │
│  ├── git-scm.com/docs                                │
│  ├── docs.anthropic.com                              │
│  ├── platform.openai.com/docs                        │
│  ├── vercel.com/docs                                 │
│  └── ...                                             │
│                                                      │
│  REFERENCE & STANDARDS (~12 domains)                 │
│  ├── en.wikipedia.org                                │
│  ├── w3.org                                          │
│  ├── ietf.org                                        │
│  ├── json-schema.org                                 │
│  └── ...                                             │
└────────────────────────────────────────────────────┘
```

### 3.2 Notable Absences

Some notable domains that are NOT on the pre-approved list:

```
NOT PRE-APPROVED (restricted access):
├── stackoverflow.com     -- Surprising given its utility
├── medium.com            -- Blog platform, mixed content
├── dev.to               -- Developer blog platform
├── reddit.com           -- Discussion forums
├── twitter.com / x.com  -- Social media
├── news.ycombinator.com -- Hacker News
└── Most personal blogs
```

**Why Stack Overflow is restricted:** Likely a licensing/copyright decision. Stack Overflow content is licensed under CC BY-SA, which requires attribution for reproduction. The paraphrase limit ensures the agent does not reproduce SO answers verbatim without attribution.

### 3.3 The Implication for Developers

If you are writing documentation for a library, being on the pre-approved list means Claude Code can fully access and quote your docs. If you are not on the list, your documentation is only available in paraphrased form.

This creates an incentive: **high-quality, official documentation on pre-approved domains is more accessible to AI agents than blog posts on personal sites.**

---

## 4. Content Extraction: Turndown and Its Limitations

### 4.1 The Turndown Pipeline

When the agent fetches a web page, the HTML is converted to markdown using Turndown:

```typescript
// Content extraction pipeline (reconstructed)
async function extractContent(
  url: string,
  html: string
): Promise<string> {
  // Step 1: Parse the HTML
  const dom = parseHTML(html);
  
  // Step 2: Extract the body only
  // <head> is completely ignored
  const body = dom.querySelector('body');
  if (!body) return '';
  
  // Step 3: Remove unwanted elements
  removeElements(body, [
    'script',
    'style',
    'nav',
    'header',
    'footer',
    'aside',
    'iframe',
    'noscript',
  ]);
  
  // Step 4: Convert to markdown using Turndown
  const turndown = new TurndownService({
    headingStyle: 'atx',
    codeBlockStyle: 'fenced',
  });
  
  // Turndown configuration: NO TABLE SUPPORT
  // Tables are stripped or converted to text
  
  const markdown = turndown.turndown(body.innerHTML);
  
  // Step 5: Clean up the markdown
  const cleaned = cleanMarkdown(markdown);
  
  // Step 6: Truncate to size limit
  return truncate(cleaned, MAX_CONTENT_SIZE);
}
```

### 4.2 What Gets Lost

The Turndown conversion is lossy. Here is what survives and what is lost:

```
┌──────────────────────────────────────────────────────┐
│            CONTENT EXTRACTION: WHAT SURVIVES           │
│                                                        │
│  SURVIVES WELL:                                        │
│  ├── Headings (h1-h6)                                  │
│  ├── Paragraphs                                        │
│  ├── Code blocks (pre/code tags)                       │
│  ├── Lists (ordered and unordered)                     │
│  ├── Links (href text preserved)                       │
│  ├── Bold and italic text                              │
│  └── Blockquotes                                       │
│                                                        │
│  LOST OR DEGRADED:                                     │
│  ├── Tables (converted to plain text, structure lost)  │
│  ├── <head> content (title, meta, structured data)     │
│  ├── Images (alt text may survive, image data lost)    │
│  ├── Interactive elements (forms, buttons)             │
│  ├── CSS styling (visual hierarchy lost)               │
│  ├── JavaScript-rendered content (never executed)      │
│  ├── Videos and embeds                                 │
│  ├── Diagrams and charts                               │
│  └── Sidebar navigation and related links              │
└──────────────────────────────────────────────────────┘
```

### 4.3 The Table Problem

Tables are a significant blind spot. Many documentation sites use tables for:
- API reference (method signatures, parameters, return types)
- Configuration options (setting name, type, default, description)
- Comparison matrices (feature A vs. feature B)
- Pricing information

When Turndown encounters a table, it typically converts it to a flat text representation that loses the column/row structure:

```
Original HTML table:
┌────────────┬──────────┬─────────────────┐
│ Parameter  │ Type     │ Description     │
├────────────┼──────────┼─────────────────┤
│ model      │ string   │ The model ID    │
│ max_tokens │ number   │ Max output len  │
│ messages   │ array    │ Input messages  │
└────────────┴──────────┴─────────────────┘

After Turndown (typical):
Parameter Type Description
model string The model ID
max_tokens number Max output len
messages array Input messages

The structure is lost. The agent must infer that
"string" goes with "model" from position alone.
```

**Implication for documentation authors:** If you want AI agents to understand your docs well, use code blocks or lists instead of tables for structured data. Or provide a markdown version of your docs (see the llms.txt discussion in section 9).

### 4.4 The <head> Problem

The entire `<head>` element is ignored during extraction. This means:

```
LOST from <head>:
├── <title> -- the page title
├── <meta name="description"> -- page summary
├── <meta name="keywords"> -- topic keywords
├── Structured data (JSON-LD, schema.org)
├── OpenGraph tags (og:title, og:description)
├── Canonical URL
└── Author and date information
```

This is a significant loss. The `<head>` often contains the most concise summary of what the page is about. Losing it means the agent must infer the page's topic from the body content alone.

---

## 5. WebFetch vs. web_search: Two Tools, Different Behaviors

### 5.1 web_search: Query-Based Discovery

```typescript
// web_search tool (reconstructed)
const WEB_SEARCH_TOOL = {
  name: 'web_search',
  description: 'Search the web for information',
  input_schema: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'The search query'
      }
    },
    required: ['query']
  },
  
  // Internal implementation
  async execute(input: { query: string }): Promise<SearchResult[]> {
    // Calls Anthropic's search API (not Google directly)
    const results = await anthropicSearch(input.query);
    
    // Returns up to 8 results
    // Each result has: title, url, snippet
    // For pre-approved domains: may include full content
    return results.slice(0, 8);
  }
};
```

**Behavior:**
- Input: a search query (like a Google search)
- Output: up to 8 results with titles, URLs, and snippets
- Pre-approved domains: may include full page content
- Other domains: snippet only, content is restricted
- No pagination (you get 8 results and that is it)

### 5.2 WebFetch: URL-Based Retrieval

```typescript
// WebFetch tool (reconstructed)
const WEB_FETCH_TOOL = {
  name: 'WebFetch',
  description: 'Fetch content from a specific URL',
  input_schema: {
    type: 'object',
    properties: {
      url: {
        type: 'string',
        description: 'The URL to fetch'
      }
    },
    required: ['url']
  },
  
  async execute(input: { url: string }): Promise<string> {
    // Fetch the URL
    const response = await fetch(input.url, {
      headers: {
        // The content negotiation hack
        'Accept': 'text/markdown, text/plain, text/html',
        'User-Agent': 'Claude-Code/1.0',
      }
    });
    
    const contentType = response.headers.get('content-type');
    const body = await response.text();
    
    // If the server returns markdown, use it directly
    if (contentType?.includes('text/markdown')) {
      return truncate(body, MAX_CONTENT_SIZE);
    }
    
    // Otherwise, extract content from HTML
    return extractContent(input.url, body);
  }
};
```

**Behavior:**
- Input: a specific URL
- Output: the page content, extracted to markdown
- Uses the Turndown pipeline for HTML
- Subject to the same tier restrictions for quoting
- Can use content negotiation to request markdown

### 5.3 When to Use Which

```
┌──────────────────────────────────────────────────┐
│         web_search vs. WebFetch                   │
│                                                    │
│  web_search:                                       │
│  ├── "What is the API for X?"                      │
│  ├── "How do I fix error Y?"                       │
│  ├── "What are the options for Z?"                 │
│  └── Discovery: you don't know which page has      │
│      the answer                                    │
│                                                    │
│  WebFetch:                                         │
│  ├── "Read https://docs.example.com/api"           │
│  ├── "Check the README at this GitHub URL"         │
│  ├── "Get the content from this documentation      │
│  │   page I found"                                 │
│  └── Targeted: you know exactly which page         │
│      to read                                       │
│                                                    │
│  Common pattern:                                   │
│  1. web_search to find the right page              │
│  2. WebFetch to read the full content              │
└──────────────────────────────────────────────────┘
```

---

## 6. The Content Negotiation Hack

### 6.1 How It Works

When Claude Code's WebFetch tool makes a request, it sends an Accept header that prefers markdown:

```
Accept: text/markdown, text/plain, text/html
```

If the server understands this header and has a markdown version of the content, it can serve markdown directly -- bypassing the Turndown conversion entirely.

### 6.2 Why This Matters

Markdown from the server is dramatically better than HTML-to-Turndown markdown:

```
Server-provided markdown:
  - Tables are proper markdown tables (preserved structure)
  - Code blocks have correct language annotations
  - Links are clean
  - No noise from navigation, ads, footers
  - Structured exactly as the author intended

Turndown-extracted markdown:
  - Tables are flat text (lost structure)
  - Code blocks may have wrong language
  - Links may include navigation noise
  - Contains artifacts from HTML conversion
  - Structure depends on HTML quality
```

### 6.3 For Documentation Authors

If you maintain documentation that AI agents will read, consider serving markdown when the Accept header requests it:

```typescript
// Express middleware for serving markdown to AI agents
function markdownMiddleware(
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) {
  const acceptsMarkdown = req.headers.accept
    ?.includes('text/markdown');
  
  if (acceptsMarkdown && req.path.startsWith('/docs/')) {
    // Serve the markdown source instead of rendered HTML
    const mdPath = getMarkdownPath(req.path);
    
    if (fs.existsSync(mdPath)) {
      res.setHeader('Content-Type', 'text/markdown');
      return res.sendFile(mdPath);
    }
  }
  
  next();
}
```

---

## 7. The 125-Character Paraphrase Limit

### 7.1 How It Works

For non-pre-approved domains, the agent can access the content but cannot quote it verbatim. The enforcement mechanism:

```typescript
// Paraphrase enforcement (reconstructed)
// This runs AFTER the model generates a response
// that references restricted content

async function enforceParaphraseLimit(
  response: string,
  restrictedSources: RestrictedSource[]
): Promise<string> {
  // Check if the response contains long verbatim quotes
  // from restricted sources
  
  for (const source of restrictedSources) {
    // Find any substring > 125 chars that matches source content
    const verbatimQuotes = findVerbatimQuotes(
      response,
      source.content,
      125  // character limit
    );
    
    if (verbatimQuotes.length > 0) {
      // Use Haiku to paraphrase the quotes
      const paraphrased = await paraphraseWithHaiku(
        response,
        verbatimQuotes
      );
      return paraphrased;
    }
  }
  
  return response;
}

async function paraphraseWithHaiku(
  response: string,
  quotes: VerbatimQuote[]
): Promise<string> {
  // Send to Haiku for paraphrasing
  const result = await anthropic.messages.create({
    model: 'claude-haiku-4-20250514',
    messages: [{
      role: 'user',
      content: `Rewrite the following text to paraphrase any 
verbatim quotes longer than 125 characters. Keep the meaning 
but use different words.

${response}

Quotes to paraphrase:
${quotes.map(q => q.text).join('\n---\n')}`
    }],
    max_tokens: 2048,
  });
  
  return extractText(result);
}
```

### 7.2 What 125 Characters Looks Like

For perspective, 125 characters is approximately:

```
"This is an example of what 125 characters looks like in
 practice. It is not very much text at all. About two se"
 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 That's approximately 125 characters.
```

This is roughly 1-2 sentences. The agent can quote a short phrase but not a paragraph.

### 7.3 Impact on Agent Behavior

The paraphrase limit means the agent must synthesize rather than copy:

```
Question: "How do I configure webpack with Next.js?"

From pre-approved domain (nextjs.org):
  "According to the Next.js docs: 'To customize webpack
   config, you can define a custom webpack function in
   your next.config.js file. The function receives the
   existing webpack config as an argument...'"
   [Direct, accurate quote]

From restricted domain (random blog):
  "According to a blog post, you can customize webpack
   in Next.js by creating a custom configuration function
   in your next.config.js that receives the existing
   config and allows you to modify it."
   [Paraphrased, may lose precision]
```

The practical effect: agents give better answers from pre-approved domains because they can quote exact syntax, configuration examples, and API signatures.

---

## 8. Max 8 Results, No Pagination

### 8.1 The Constraint

The web_search tool returns a maximum of 8 results per query. There is no pagination -- you cannot ask for results 9-16.

```typescript
// You get what you get
const searchResults = await web_search({
  query: "React useEffect cleanup pattern"
});
// searchResults.length <= 8
// No: web_search({ query: "...", page: 2 })
```

### 8.2 Why This Matters

8 results is significantly fewer than a typical Google search page (10 results) and there is no way to go deeper. This means:

1. **Query quality matters enormously.** A vague query wastes your 8 slots on irrelevant results.
2. **The agent must be strategic.** It often searches multiple times with refined queries rather than paginating.
3. **Niche topics suffer.** If the answer is on result #12 of Google, the agent will not find it.

### 8.3 Search Strategies That Work

```typescript
// BAD: Vague query, wastes 8 result slots
web_search({ query: "how to fix bug" });

// GOOD: Specific query, likely to hit on first 8
web_search({ query: "React useState stale closure fix" });

// GOOD: Multiple targeted searches
web_search({ query: "Next.js 15 app router parallel routes" });
web_search({ query: "Next.js parallel routes example code" });

// GOOD: Include version numbers and specific terms
web_search({ query: "drizzle-orm 0.30 json column postgres" });
```

---

## 9. llms.txt: What Claude Code Does (and Does Not) Look For

### 9.1 What llms.txt Is

The llms.txt initiative proposes that websites publish a `/llms.txt` file (similar to robots.txt) that provides an AI-friendly summary of the site's content:

```
# llms.txt example (from the llms.txt spec)
# This file tells AI agents about the site

> This is the official documentation for the Acme Widget API.
> It provides REST endpoints for widget management.

## Docs
- [Authentication](https://docs.acme.com/auth): How to authenticate
- [Widgets API](https://docs.acme.com/api/widgets): CRUD operations
- [Webhooks](https://docs.acme.com/webhooks): Event notifications

## Optional
- [Changelog](https://docs.acme.com/changelog): Version history
```

### 9.2 Claude Code Does NOT Look for llms.txt

Despite what many guides and blog posts claim, the leaked source revealed that Claude Code does not automatically check for `/llms.txt` when searching or fetching web content:

```
Common claim: "Add llms.txt and AI agents will find it!"
Reality:       Claude Code never requests /llms.txt

The WebFetch tool fetches the URL you give it.
The web_search tool searches for your query.
Neither tool automatically checks /llms.txt.
```

### 9.3 When llms.txt IS Useful

llms.txt is still valuable, just not in the way people think:

```
NOT automatic: Claude Code does not check for llms.txt

BUT useful when:
1. A human tells the agent to check llms.txt
   ("Fetch https://docs.acme.com/llms.txt first")
   
2. A CLAUDE.md file references it
   ("For Acme API docs, start with their llms.txt:
    https://docs.acme.com/llms.txt")
   
3. A skill includes it in its context
   
4. Search results happen to link to it
```

### 9.4 What Actually Helps AI Agents Read Your Docs

Instead of (or in addition to) llms.txt, do this:

```
┌──────────────────────────────────────────────────────┐
│     MAKING YOUR DOCS AI-AGENT-FRIENDLY                │
│                                                        │
│  1. SERVE MARKDOWN (content negotiation)               │
│     When Accept: text/markdown, serve .md files        │
│     Bypass Turndown entirely                           │
│                                                        │
│  2. USE CODE BLOCKS, NOT TABLES                        │
│     Tables are lost in Turndown                        │
│     Code blocks survive perfectly                      │
│                                                        │
│  3. PUT KEY INFO IN TEXT, NOT <head>                   │
│     <head> is stripped. Put titles, descriptions,      │
│     and metadata in the visible body text              │
│                                                        │
│  4. STRUCTURE WITH HEADINGS                            │
│     h1-h6 survive extraction perfectly                 │
│     Use them for logical organization                  │
│                                                        │
│  5. INCLUDE COMPLETE CODE EXAMPLES                     │
│     Code blocks in <pre><code> survive well            │
│     Include enough context to be useful standalone     │
│                                                        │
│  6. AVOID JAVASCRIPT-RENDERED CONTENT                  │
│     WebFetch does not execute JS                       │
│     SSR or static HTML is essential                    │
│                                                        │
│  7. KEEP URLS CLEAN AND DESCRIPTIVE                    │
│     /docs/api/widgets/create is better than            │
│     /docs?id=1234&section=api                          │
│     Descriptive URLs help search ranking               │
└──────────────────────────────────────────────────────┘
```

---

## 10. Lessons for RAG System Design

### 10.1 What Claude Code's Search Teaches Us About RAG

Claude Code's web search is a production RAG system. Its constraints reveal hard-earned lessons:

**Lesson 1: Content quality matters more than quantity.**
```
Claude Code: 8 results max, no pagination.
Your RAG:    Retrieved chunks should be FEW and HIGH QUALITY.
             Don't dump 50 chunks into context.
             5 excellent chunks beat 50 mediocre ones.
```

**Lesson 2: Content extraction is lossy -- design for it.**
```
Claude Code: Turndown loses tables, <head>, and structure.
Your RAG:    Your document parsing is lossy too.
             PDF parsing loses layout. HTML parsing loses style.
             Test your extraction pipeline and see what
             survives. Design your source docs for extraction.
```

**Lesson 3: Tiered access is a practical pattern.**
```
Claude Code: Pre-approved domains get full access, others
             are restricted.
Your RAG:    Not all sources are equal. Tier them:
             - Internal docs: full access, high trust
             - Verified sources: full access, medium trust
             - External sources: restricted, low trust
```

**Lesson 4: Paraphrasing is a copyright-safe retrieval pattern.**
```
Claude Code: 125-char limit forces paraphrasing.
Your RAG:    If you retrieve copyrighted content, consider
             forcing the model to synthesize rather than
             copy. This protects you legally.
```

**Lesson 5: Search query quality determines result quality.**
```
Claude Code: 8 results, no pagination. Query must be precise.
Your RAG:    Invest in query preprocessing. Expand queries,
             use HyDE (Ch 17), decompose complex questions.
             Better queries = better retrieval = better answers.
```

### 10.2 Building a Knowledge Pipeline

Combining lessons from Claude Code's search with RAG best practices from Ch 15-17:

```typescript
// A knowledge pipeline inspired by Claude Code's architecture
interface KnowledgePipeline {
  // Tiered sources (like the two-tier web)
  sources: {
    tier1: InternalSource[];    // Full access (your docs)
    tier2: TrustedSource[];     // Full access (partner docs)
    tier3: ExternalSource[];    // Restricted (everything else)
  };
  
  // Content extraction (like Turndown)
  extractors: {
    html: TurndownExtractor;
    pdf: PDFExtractor;
    markdown: PassthroughExtractor;  // Best case
  };
  
  // Search with result limits (like max 8)
  search(query: string, options: SearchOptions): SearchResult[];
  
  // Content access control (like tier restrictions)
  getContent(
    result: SearchResult
  ): { content: string; quotePolicy: 'full' | 'paraphrase' };
}

// Implementation
class ProductionKnowledgePipeline implements KnowledgePipeline {
  async search(
    query: string,
    options: SearchOptions = {}
  ): Promise<SearchResult[]> {
    const maxResults = options.maxResults || 8; // Like Claude Code
    
    // Search across all tiers
    const results = await Promise.all([
      this.searchTier1(query),
      this.searchTier2(query),
      this.searchTier3(query),
    ]);
    
    // Merge and rank
    const merged = this.mergeAndRank(
      results.flat(),
      maxResults
    );
    
    // Tier 1 results are boosted (prefer internal docs)
    return merged.sort((a, b) =>
      (b.relevance + b.tierBoost) - (a.relevance + a.tierBoost)
    );
  }
  
  getContent(result: SearchResult): ContentAccess {
    switch (result.tier) {
      case 1:
        return {
          content: result.fullContent,
          quotePolicy: 'full'
        };
      case 2:
        return {
          content: result.fullContent,
          quotePolicy: 'full'
        };
      case 3:
        return {
          content: result.extractedContent,
          quotePolicy: 'paraphrase'
        };
    }
  }
}
```

### 10.3 The Web Search Checklist for Your Tools

If you are building a tool that needs web search:

```
┌──────────────────────────────────────────────────────┐
│        WEB SEARCH IMPLEMENTATION CHECKLIST            │
│                                                        │
│  CONTENT EXTRACTION:                                   │
│  [ ] HTML to markdown conversion (Turndown or equiv)   │
│  [ ] Handle tables explicitly (don't lose structure)   │
│  [ ] Extract from <body> (strip nav, header, footer)   │
│  [ ] Support content negotiation (Accept: text/md)     │
│  [ ] Truncate results to fit context budget            │
│                                                        │
│  RESULT QUALITY:                                       │
│  [ ] Limit results (8-10 max, quality over quantity)   │
│  [ ] Rank by relevance to the query                    │
│  [ ] Prefer official/authoritative sources             │
│  [ ] Handle JavaScript-rendered sites (or acknowledge  │
│      the limitation)                                   │
│                                                        │
│  LEGAL/COMPLIANCE:                                     │
│  [ ] Respect robots.txt                                │
│  [ ] Consider copyright for long quotes                │
│  [ ] Implement paraphrase limits for risky content     │
│  [ ] Log source URLs for attribution                   │
│                                                        │
│  COST:                                                 │
│  [ ] Extracted content goes into context (costs tokens) │
│  [ ] Truncate aggressively (most pages are 90% noise)  │
│  [ ] Cache search results (same query = same results)  │
│  [ ] Limit searches per session                        │
└──────────────────────────────────────────────────────┘
```

---

## 11. Key Takeaways

1. **Web search is a two-tier system.** 107 pre-approved domains get full content access. Everything else is restricted to ~125-character quotes, enforced by Haiku post-processing.

2. **Content extraction uses Turndown.** HTML is converted to markdown. Tables are lost. `<head>` is ignored. JavaScript-rendered content is invisible.

3. **WebFetch and web_search are different tools.** web_search discovers pages (up to 8 results). WebFetch retrieves a specific URL. Common pattern: search first, then fetch.

4. **The content negotiation hack:** WebFetch sends `Accept: text/markdown`. If your server responds with markdown, it bypasses Turndown and gets perfectly structured content.

5. **Max 8 results with no pagination.** Query quality is everything. Specific, targeted queries beat vague ones.

6. **Claude Code does NOT check llms.txt.** Despite popular claims. It is useful when explicitly referenced, but not automatically discovered.

7. **For your own docs:** serve markdown, use code blocks not tables, put key info in the body not head, and avoid JavaScript-rendered content.

8. **For your own RAG systems:** tier your sources, limit result count, invest in query quality, and handle extraction loss gracefully.

---

*Previous: [Chapter 30 -- Tool Execution & Permissions](./30-tool-execution.md)* · *Next: [Chapter 32 -- Multi-Agent Coordination](./32-multi-agent-coordination.md)* explores how multiple agent instances work together -- the tick loop, background agents, and delegation patterns.

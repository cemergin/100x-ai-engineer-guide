<!--
  CHAPTER: 60
  TITLE: Skills Marketplaces & Knowledge Sharing
  PART: 12 — AI Platform Engineering
  PHASE: 2 — Become an Expert
  PREREQS: Ch 25 (skills/plugins/automation), Ch 33 (skill architecture internals), Ch 59 (internal AI tools)
  KEY_TOPICS: skills marketplace, Dojo pattern, Sensei recommender, Git-backed skills, skill lifecycle, quality control, knowledge compounding, skill discovery, company knowledge, MCP + RAG
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 60: Skills Marketplaces & Knowledge Sharing

> **Part 12 — AI Platform Engineering** | Phase 2: Become an Expert | Prerequisites: Ch 25, Ch 33, Ch 59 | Difficulty: Advanced | Language: TypeScript + Python

In Chapter 25, you learned to write skills: markdown files with instructions that customize how an AI agent behaves. In Chapter 33, you studied how Claude Code packages, distributes, and protects skills internally. In Chapter 59, you built the internal AI platform. Now you build the system that lets every skill, prompt, and workflow that anyone creates become available to everyone else.

This is the Dojo chapter. Dojo is Ramp's skills marketplace: 350+ skills, Git-backed, versioned, code-reviewed, searchable, and recommended by an AI system called Sensei. The result is a flywheel where every person's breakthrough becomes everyone's baseline. An engineer writes a skill for generating database migrations. That skill gets reviewed, published to Dojo, and within a day, 50 other engineers are using it. A salesperson writes a skill for analyzing competitive deals. That skill gets surfaced by Sensei to every other salesperson who opens a similar deal.

Knowledge compounding is the most important concept in this chapter. AI skills are not like traditional documentation that people read once and forget. They are *executable knowledge* -- instructions that produce results every time they are invoked. When you share a skill, you are not sharing information. You are sharing *capability*. And capabilities compound.

### In This Chapter
- The Dojo pattern: how Ramp built a skills marketplace
- Git-backed, versioned, code-reviewed skills
- The Sensei: AI-guided skill recommendations
- Skill lifecycle: create, test, share, iterate, deprecate
- Quality control: reviews, eval-based quality gates
- Discovery UX: search, browse, personalized recommendations
- Knowledge compounding: why shared skills are different from shared docs
- Building a skills marketplace (practical architecture)
- The /ask-nelo pattern: company knowledge via MCP + RAG

### Related Chapters
- **Ch 25 (Skills, Plugins & Automation)** -- writing individual skills; this chapter distributes them
- **Ch 33 (Skills, Plugins & Distribution)** -- how Claude Code handles skill distribution internally; this chapter builds your own system
- **Ch 4 (Prompt Engineering)** -- skills are executable prompts; marketplace curates the best ones
- **Ch 15 (The RAG Pipeline)** -- company knowledge skills use RAG; marketplace makes them discoverable
- **Ch 59 (Building Internal AI Tools)** -- the platform these skills run on
- **Ch 61 (Self-Reinforcing Systems)** -- the marketplace feedback loop feeds self-improvement
- **Ch 62 (AI Adoption & Enablement)** -- marketplace drives adoption by raising the floor

---

## 1. Why Skills Marketplaces

### 1.1 The Knowledge Compounding Problem

Every organization has the same problem: expertise is siloed. The engineer who figured out the perfect prompt for generating database migrations keeps it in their personal notes. The salesperson who built an amazing competitive analysis workflow uses it alone. The PM who discovered how to get the AI to write great PRDs tells three people in Slack.

Traditional knowledge sharing (wikis, docs, Slack messages) suffers from write-once-read-never syndrome. People share knowledge as text, and text is passive. You have to find it, read it, understand it, and then manually apply it. The dropout rate at each step is enormous.

**Skills are different because they are executable.** A skill is not "here is how to write a good database migration." A skill is "paste this into the AI tool and it will write the database migration for you." The user does not need to understand the underlying knowledge. They just need to invoke the skill.

This changes the math of knowledge sharing fundamentally:

```
TRADITIONAL KNOWLEDGE SHARING
  Knowledge created -> documented -> discovered -> read -> understood -> applied
  Dropout per step:     30%          50%         40%      60%        50%
  Net utilization: ~2.5% of created knowledge gets applied by others

SKILL-BASED KNOWLEDGE SHARING
  Skill created -> published -> discovered -> invoked
  Dropout per step:    5%        30%        10%
  Net utilization: ~60% of created skills get used by others
```

These numbers are approximate, but the directional insight is real. When Ramp launched Dojo with 350+ skills, adoption was not a communication problem. It was a *discovery* problem -- which is why they built Sensei.

### 1.2 The Dojo Pattern

Ramp's Dojo is a skills marketplace with these properties:

**Git-backed.** Every skill is a file in a Git repository. Changes go through pull requests with code review. The entire history of every skill is preserved.

**Versioned.** Skills have versions. When a skill is updated, users on the old version are not broken. They can choose when to upgrade.

**Code-reviewed.** New skills and updates go through the same review process as code. Reviewers check for quality, safety, and overlap with existing skills.

**Searchable.** Full-text search across skill names, descriptions, and content. Browse by category. Filter by team, author, or use case.

**Recommended.** Sensei, an AI recommendation system, proactively suggests skills based on your role, the tools you use, and what you are currently doing.

**Measured.** Usage analytics show which skills are popular, which are abandoned, and which correlate with productivity improvements.

### 1.3 The Flywheel

```
+---------------------------------------------+
|                                              |
|   Person A discovers a great AI workflow     |
|              |                               |
|              v                               |
|   Person A packages it as a skill            |
|              |                               |
|              v                               |
|   Skill is reviewed and published to Dojo    |
|              |                               |
|              v                               |
|   Sensei recommends it to Person B, C, D...  |
|              |                               |
|              v                               |
|   Persons B, C, D use it and give feedback   |
|              |                               |
|              v                               |
|   Person A (or E) improves the skill         |
|              |                               |
|              v                               |
|   Improved skill benefits everyone           |
|              |                               |
|              +------------------------------>|
|                                              |
|   "One person's breakthrough becomes         |
|    everyone's baseline"                      |
|                                              |
+---------------------------------------------+
```

The flywheel accelerates over time. More skills in the marketplace means more people find useful skills. More people using skills means more feedback. More feedback means better skills. Better skills attract more users. At Ramp, this flywheel took about six weeks to reach escape velocity: the rate of new skills being published exceeded the rate of skills being deprecated.

---

## 2. Skill Architecture for a Marketplace

### 2.1 The Skill Schema

Every skill in the marketplace follows a consistent schema. This is an extension of the skill format from Chapter 25, with marketplace metadata added.

```typescript
// skill-schema.ts -- The marketplace skill format

interface MarketplaceSkill {
  // Identity
  id: string; // Unique identifier, e.g., "db-migration-generator"
  name: string; // Human-readable name
  version: string; // Semver: "1.2.3"
  author: {
    userId: string;
    name: string;
    team: string;
  };

  // Discovery
  description: string; // One-line description
  longDescription: string; // Detailed description with examples
  category: SkillCategory;
  tags: string[];
  targetRoles: string[]; // "engineer", "pm", "sales", "all"
  targetTools: string[]; // Which integrations this skill uses

  // Content
  content: string; // The actual skill markdown/instructions
  examples: {
    input: string; // Example user prompt
    output: string; // Expected behavior
  }[];

  // Quality
  status: "draft" | "review" | "published" | "deprecated";
  reviewers: string[];
  evalScore?: number; // Automated quality score (0-100)
  usageCount: number;
  rating: number; // Average user rating (1-5)
  ratingCount: number;

  // Metadata
  createdAt: Date;
  updatedAt: Date;
  dependencies: string[]; // Other skills this one depends on
  changelog: {
    version: string;
    date: Date;
    changes: string;
  }[];
}

type SkillCategory =
  | "engineering"
  | "data-analysis"
  | "sales"
  | "product"
  | "design"
  | "operations"
  | "security"
  | "general";
```

### 2.2 Skill Content Format

```markdown
<!-- Example skill file: skills/engineering/db-migration-generator.md -->
---
id: db-migration-generator
name: Database Migration Generator
version: 2.1.0
author: jane.doe
category: engineering
tags: [database, migrations, postgresql, prisma]
targetRoles: [engineer]
targetTools: [github, snowflake]
description: Generate safe, reversible database migrations from schema change descriptions
---

# Database Migration Generator

You are an expert database engineer specializing in safe, reversible
PostgreSQL migrations. When the user describes a schema change, generate
a complete migration using our team's patterns.

## Rules
1. Always generate both UP and DOWN migrations
2. Use IF NOT EXISTS / IF EXISTS for idempotency
3. Never drop columns directly -- rename to _deprecated_ first
4. Add comments explaining each change
5. For large tables (>1M rows), use batched operations
6. Always include a data validation query

## Our Stack
- PostgreSQL 15
- Prisma ORM for the application
- Raw SQL migrations for schema changes
- flyway-style versioning: V{timestamp}__{description}.sql

## Examples

### Adding a column
User: "Add an email_verified boolean to the users table"
Output: Migration with ALTER TABLE, default value, index if appropriate

### Renaming a column
User: "Rename user_name to username in the accounts table"
Output: Migration that adds new column, copies data, deprecates old column
```

### 2.3 The Skill Repository Structure

```
skills-marketplace/
  skills/
    engineering/
      db-migration-generator.md
      code-review-checklist.md
      api-endpoint-generator.md
      test-writer.md
      ...
    sales/
      competitive-analysis.md
      deal-summary.md
      email-drafting.md
      ...
    product/
      prd-writer.md
      user-story-generator.md
      metric-definition.md
      ...
    data-analysis/
      sql-query-helper.md
      dashboard-builder.md
      ...
    general/
      meeting-summarizer.md
      email-writer.md
      ...
  evals/
    db-migration-generator.eval.ts
    code-review-checklist.eval.ts
    ...
  catalog.json          <- Auto-generated index of all skills
  CONTRIBUTING.md       <- How to create and submit skills
  .github/
    workflows/
      skill-validation.yml   <- CI: lint, eval, catalog update
      skill-review.yml       <- Auto-assign reviewers
```

---

## 3. Building the Marketplace Backend

### 3.1 The Skill Registry

```typescript
// registry.ts -- Skill registry and search

interface SkillIndex {
  skills: MarketplaceSkill[];
  categories: Map<SkillCategory, number>;
  tags: Map<string, number>;
  lastUpdated: Date;
}

interface SkillFilters {
  category?: SkillCategory;
  role?: string;
  minRating?: number;
  status?: string;
  author?: string;
  tags?: string[];
}

class SkillRegistry {
  private index: SkillIndex;
  private searchEmbeddings: Map<string, number[]>; // skill ID -> embedding
  private embeddingService: EmbeddingService;

  constructor(embeddingService: EmbeddingService) {
    this.embeddingService = embeddingService;
    this.index = {
      skills: [],
      categories: new Map(),
      tags: new Map(),
      lastUpdated: new Date(),
    };
    this.searchEmbeddings = new Map();
  }

  async indexSkill(skill: MarketplaceSkill): Promise<void> {
    // Add to index
    const existing = this.index.skills.findIndex((s) => s.id === skill.id);
    if (existing >= 0) {
      this.index.skills[existing] = skill;
    } else {
      this.index.skills.push(skill);
    }

    // Update category and tag counts
    this.rebuildFacets();

    // Generate search embedding
    const searchText = [
      skill.name,
      skill.description,
      skill.longDescription,
      skill.tags.join(" "),
    ].join(" ");
    const embedding = await this.embeddingService.embed(searchText);
    this.searchEmbeddings.set(skill.id, embedding);
  }

  // Text search
  search(query: string, filters?: SkillFilters): MarketplaceSkill[] {
    const queryLower = query.toLowerCase();

    return this.index.skills
      .filter((skill) => {
        if (filters?.category && skill.category !== filters.category) return false;
        if (filters?.role && !skill.targetRoles.includes(filters.role)) return false;
        if (filters?.minRating && skill.rating < filters.minRating) return false;
        if (filters?.status && skill.status !== filters.status) return false;

        if (!query) return true; // Empty query = return all matching filters

        return (
          skill.name.toLowerCase().includes(queryLower) ||
          skill.description.toLowerCase().includes(queryLower) ||
          skill.tags.some((t) => t.toLowerCase().includes(queryLower))
        );
      })
      .sort((a, b) => {
        const scoreA = this.relevanceScore(a, queryLower);
        const scoreB = this.relevanceScore(b, queryLower);
        return scoreB - scoreA;
      });
  }

  // Semantic search using embeddings
  async semanticSearch(
    query: string,
    filters?: SkillFilters,
    limit: number = 10
  ): Promise<MarketplaceSkill[]> {
    const queryEmbedding = await this.embeddingService.embed(query);

    const scored = this.index.skills
      .filter((skill) => {
        if (filters?.category && skill.category !== filters.category) return false;
        if (filters?.role && !skill.targetRoles.includes(filters.role)) return false;
        if (skill.status !== "published") return false;
        return true;
      })
      .map((skill) => ({
        skill,
        score: cosineSimilarity(
          queryEmbedding,
          this.searchEmbeddings.get(skill.id) || []
        ),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);

    return scored.map((s) => s.skill);
  }

  getPopular(category?: SkillCategory, limit: number = 10): MarketplaceSkill[] {
    return this.index.skills
      .filter(
        (s) =>
          s.status === "published" &&
          (!category || s.category === category)
      )
      .sort((a, b) => b.usageCount - a.usageCount)
      .slice(0, limit);
  }

  getRecent(limit: number = 10): MarketplaceSkill[] {
    return this.index.skills
      .filter((s) => s.status === "published")
      .sort((a, b) => b.updatedAt.getTime() - a.updatedAt.getTime())
      .slice(0, limit);
  }

  private relevanceScore(skill: MarketplaceSkill, query: string): number {
    let score = 0;
    if (skill.name.toLowerCase().includes(query)) score += 10;
    if (skill.description.toLowerCase().includes(query)) score += 5;
    if (skill.tags.some((t) => t.toLowerCase() === query)) score += 8;
    score += skill.rating * 2;
    score += Math.log(skill.usageCount + 1);
    return score;
  }

  private rebuildFacets(): void {
    this.index.categories.clear();
    this.index.tags.clear();

    for (const skill of this.index.skills) {
      if (skill.status !== "published") continue;
      const catCount = this.index.categories.get(skill.category) || 0;
      this.index.categories.set(skill.category, catCount + 1);

      for (const tag of skill.tags) {
        const tagCount = this.index.tags.get(tag) || 0;
        this.index.tags.set(tag, tagCount + 1);
      }
    }
  }
}

function cosineSimilarity(a: number[], b: number[]): number {
  if (a.length !== b.length || a.length === 0) return 0;
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
```

### 3.2 The Git-Backed Storage Layer

```typescript
// git-store.ts -- Skills stored in Git for versioning and review
import { readFile, readdir } from "fs/promises";
import { join } from "path";
import matter from "gray-matter";

class GitSkillStore {
  private repoPath: string;
  private branch: string;

  constructor(repoPath: string, branch: string = "main") {
    this.repoPath = repoPath;
    this.branch = branch;
  }

  async loadAllSkills(): Promise<MarketplaceSkill[]> {
    const skillsDir = join(this.repoPath, "skills");
    const categories = await readdir(skillsDir);
    const skills: MarketplaceSkill[] = [];

    for (const category of categories) {
      const categoryDir = join(skillsDir, category);
      const files = await readdir(categoryDir);

      for (const file of files) {
        if (!file.endsWith(".md")) continue;

        const content = await readFile(join(categoryDir, file), "utf-8");
        const parsed = matter(content);

        skills.push({
          id: parsed.data.id,
          name: parsed.data.name,
          version: parsed.data.version,
          author: parsed.data.author,
          description: parsed.data.description,
          longDescription: this.extractLongDescription(parsed.content),
          category: category as SkillCategory,
          tags: parsed.data.tags || [],
          targetRoles: parsed.data.targetRoles || ["all"],
          targetTools: parsed.data.targetTools || [],
          content: parsed.content,
          examples: this.extractExamples(parsed.content),
          status: "published",
          reviewers: [],
          usageCount: 0,
          rating: 0,
          ratingCount: 0,
          createdAt: new Date(),
          updatedAt: new Date(),
          dependencies: parsed.data.dependencies || [],
          changelog: parsed.data.changelog || [],
        });
      }
    }

    return skills;
  }

  private extractLongDescription(content: string): string {
    const lines = content.split("\n");
    const descLines: string[] = [];
    let foundTitle = false;

    for (const line of lines) {
      if (line.startsWith("# ")) {
        foundTitle = true;
        continue;
      }
      if (foundTitle && line.trim() === "") {
        if (descLines.length > 0) break;
        continue;
      }
      if (foundTitle && !line.startsWith("#")) {
        descLines.push(line);
      }
    }

    return descLines.join(" ").trim();
  }

  private extractExamples(content: string): { input: string; output: string }[] {
    const examples: { input: string; output: string }[] = [];
    const exampleRegex = /### (.+)\nUser: "(.+)"\nOutput: (.+)/g;
    let match;

    while ((match = exampleRegex.exec(content)) !== null) {
      examples.push({
        input: match[2],
        output: match[3],
      });
    }

    return examples;
  }
}
```

---

## 4. The Sensei: AI-Guided Skill Recommendations

### 4.1 What Sensei Does

Sensei is the AI system that answers the question: "Given who you are and what you are doing right now, which skills would help you most?"

It combines three signals:

1. **User profile.** Your role, team, tools you have access to, skills you have used before.
2. **Activity context.** What are you currently doing in the platform? Writing code? Analyzing data? Drafting an email?
3. **Skill metadata.** Category, tags, usage statistics, ratings, recency.

### 4.2 Building Sensei

```typescript
// sensei.ts -- AI-driven skill recommendation engine

interface UserProfile {
  userId: string;
  role: string;
  teams: string[];
  tools: string[];
  skillUsageHistory: {
    skillId: string;
    usageCount: number;
    lastUsed: Date;
    rating?: number;
  }[];
}

interface RecommendationContext {
  currentActivity?: string;
  recentQueries?: string[];
  currentTools?: string[];
}

class Sensei {
  private registry: SkillRegistry;
  private embeddingService: EmbeddingService;

  constructor(registry: SkillRegistry, embeddingService: EmbeddingService) {
    this.registry = registry;
    this.embeddingService = embeddingService;
  }

  async recommend(
    profile: UserProfile,
    context: RecommendationContext,
    limit: number = 5
  ): Promise<{
    skill: MarketplaceSkill;
    reason: string;
    relevanceScore: number;
  }[]> {
    // Get candidate skills
    const candidates = this.registry.search("", {
      role: profile.role,
      status: "published",
    });

    // Score each candidate
    const scored = await Promise.all(
      candidates.map(async (skill) => {
        const score = await this.scoreSkill(skill, profile, context);
        return { skill, ...score };
      })
    );

    // Filter out skills the user already uses heavily
    const heavilyUsed = new Set(
      profile.skillUsageHistory
        .filter((h) => h.usageCount > 10)
        .map((h) => h.skillId)
    );

    return scored
      .filter((s) => !heavilyUsed.has(s.skill.id))
      .sort((a, b) => b.relevanceScore - a.relevanceScore)
      .slice(0, limit);
  }

  private async scoreSkill(
    skill: MarketplaceSkill,
    profile: UserProfile,
    context: RecommendationContext
  ): Promise<{ relevanceScore: number; reason: string }> {
    let score = 0;
    const reasons: string[] = [];

    // Role match
    if (
      skill.targetRoles.includes(profile.role) ||
      skill.targetRoles.includes("all")
    ) {
      score += 20;
      reasons.push(`Designed for ${profile.role}s`);
    }

    // Team match
    if (skill.author.team && profile.teams.includes(skill.author.team)) {
      score += 10;
      reasons.push("Created by your team");
    }

    // Tool overlap
    const toolOverlap = skill.targetTools.filter((t) =>
      profile.tools.includes(t)
    ).length;
    if (toolOverlap > 0) {
      score += toolOverlap * 15;
      reasons.push(`Uses ${toolOverlap} of your connected tools`);
    }

    // Popularity signal
    score += Math.min(skill.usageCount / 10, 20);
    if (skill.usageCount > 50) {
      reasons.push(`Used by ${skill.usageCount}+ people`);
    }

    // Rating signal
    if (skill.ratingCount > 5) {
      score += skill.rating * 5;
      if (skill.rating > 4) {
        reasons.push(`Highly rated (${skill.rating.toFixed(1)}/5)`);
      }
    }

    // Activity context match (semantic similarity)
    if (context.currentActivity) {
      const activityEmbedding = await this.embeddingService.embed(
        context.currentActivity
      );
      const skillEmbedding = await this.embeddingService.embed(
        `${skill.name} ${skill.description}`
      );
      const similarity = cosineSimilarity(activityEmbedding, skillEmbedding);
      if (similarity > 0.6) {
        score += similarity * 30;
        reasons.push("Relevant to what you're doing right now");
      }
    }

    // Recency bonus
    const daysSinceUpdate =
      (Date.now() - skill.updatedAt.getTime()) / (1000 * 60 * 60 * 24);
    if (daysSinceUpdate < 7) {
      score += 10;
      reasons.push("Recently updated");
    }

    return {
      relevanceScore: score,
      reason: reasons.slice(0, 3).join(". "),
    };
  }

  // Proactive recommendations: called when user context changes
  async onContextChange(
    profile: UserProfile,
    newContext: RecommendationContext
  ): Promise<string | null> {
    const recommendations = await this.recommend(profile, newContext, 1);

    if (recommendations.length > 0 && recommendations[0].relevanceScore > 50) {
      const rec = recommendations[0];
      return (
        `Tip: The "${rec.skill.name}" skill might help here. ` +
        `${rec.reason}. Say "use ${rec.skill.id}" to try it.`
      );
    }

    return null;
  }
}
```

### 4.3 Sensei in the UI

Sensei integrates into the platform UI in three ways:

**Proactive suggestions.** As the user works, Sensei watches the context and surfaces relevant skills as non-intrusive suggestions. "Tip: The 'SQL Query Optimizer' skill might help with this Snowflake query."

**Search enhancement.** When users search the marketplace, Sensei re-ranks results based on personal relevance, not just text match.

**Onboarding recommendations.** For new users, Sensei generates a personalized "starter pack" of skills based on role and team. A new engineer on the payments team gets different recommendations than a new PM on the growth team.

---

## 5. Skill Lifecycle

### 5.1 The Full Lifecycle

```
CREATE --> TEST --> REVIEW --> PUBLISH --> ITERATE --> DEPRECATE

+---------+   +----------+   +----------+   +----------+
| CREATE  |-->|   TEST   |-->|  REVIEW  |-->| PUBLISH  |
|         |   |          |   |          |   |          |
| Author  |   | Evals    |   | Peers    |   | Available|
| writes  |   | run in   |   | review   |   | to all   |
| skill   |   | CI       |   | the PR   |   | users    |
+---------+   +----------+   +----------+   +----+-----+
                                                  |
                                            +-----v------+
                                            |  ITERATE   |
                                            |            |
                                            | User       |
                                            | feedback   |
                                            | drives     |
                                            | updates    |
                                            +-----+------+
                                                  |
                                            +-----v------+
                                            | DEPRECATE  |
                                            |            |
                                            | Replaced   |
                                            | by better  |
                                            | skill or   |
                                            | no longer  |
                                            | relevant   |
                                            +------------+
```

### 5.2 Automated Quality Gates (Eval-Based)

Every skill gets automated evals in CI. The eval checks that the skill produces good results for its example inputs.

```typescript
// skill-eval.ts -- Automated skill quality evaluation
import Anthropic from "@anthropic-ai/sdk";

interface SkillEvalResult {
  skillId: string;
  passed: boolean;
  overallScore: number;
  examples: {
    input: string;
    expectedBehavior: string;
    actualOutput: string;
    score: number;
    feedback: string;
  }[];
}

async function evaluateSkill(skill: MarketplaceSkill): Promise<SkillEvalResult> {
  const client = new Anthropic();
  const examples: SkillEvalResult["examples"] = [];

  for (const example of skill.examples) {
    // Run the skill with the example input
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      system: skill.content,
      messages: [{ role: "user", content: example.input }],
    });

    const actualOutput = response.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");

    // Judge the output quality
    const judge = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1024,
      system: `You are evaluating an AI skill's output quality.
Score from 0-100 based on:
- Correctness: Does it follow the skill's instructions?
- Completeness: Does it cover all required elements?
- Quality: Is the output well-structured and useful?
- Safety: Does it avoid harmful patterns?

Respond as JSON: { "score": number, "feedback": "string" }`,
      messages: [
        {
          role: "user",
          content: [
            `Skill: ${skill.name}`,
            `Skill instructions (excerpt): ${skill.content.slice(0, 500)}`,
            `Example input: ${example.input}`,
            `Expected behavior: ${example.output}`,
            `Actual output: ${actualOutput}`,
          ].join("\n"),
        },
      ],
    });

    const judgeText = judge.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");

    try {
      const jsonMatch = judgeText.match(/\{[\s\S]*\}/);
      const parsed = JSON.parse(jsonMatch![0]);
      examples.push({
        input: example.input,
        expectedBehavior: example.output,
        actualOutput: actualOutput.slice(0, 500),
        score: parsed.score,
        feedback: parsed.feedback,
      });
    } catch {
      examples.push({
        input: example.input,
        expectedBehavior: example.output,
        actualOutput: actualOutput.slice(0, 500),
        score: 0,
        feedback: "Failed to parse judge response",
      });
    }
  }

  const overallScore =
    examples.length > 0
      ? examples.reduce((sum, e) => sum + e.score, 0) / examples.length
      : 0;

  return {
    skillId: skill.id,
    passed: overallScore >= 70,
    overallScore,
    examples,
  };
}
```

### 5.3 CI Pipeline for Skills

```yaml
# .github/workflows/skill-validation.yml
name: Skill Validation

on:
  pull_request:
    paths:
      - 'skills/**/*.md'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: npm install

      - name: Lint skill frontmatter
        run: npm run lint:skills

      - name: Check for duplicate skill IDs
        run: npm run check:duplicates

      - name: Run skill evals
        run: npm run eval:skills -- --changed-only
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Update catalog
        run: npm run catalog:build

      - name: Post eval results to PR
        uses: actions/github-script@v7
        with:
          script: |
            const results = require('./eval-results.json');
            const body = results.map(r =>
              `| ${r.skillId} | ${r.passed ? 'PASS' : 'FAIL'} | ${r.overallScore.toFixed(0)}/100 |`
            ).join('\n');
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Skill Eval Results\n\n| Skill | Status | Score |\n|-------|--------|-------|\n${body}`
            });
```

---

## 6. The /ask-nelo Pattern: Company Knowledge as a Skill

### 6.1 What It Is

One of the most powerful skill patterns: a company knowledge base exposed through the AI tool. Users type `/ask-nelo how does our authentication system work?` and get an answer grounded in actual internal documentation, code, and architecture decisions.

This combines MCP (for connecting to data sources) with RAG (for retrieving relevant documents) with the skills framework (for packaging it as a reusable, shareable capability).

### 6.2 Architecture

```
User: "/ask-nelo how does our auth system work?"
     |
     v
+------------------+
|  Skill Layer     |  <- The /ask-nelo skill provides system prompt + instructions
+--------+---------+
         |
         v
+------------------+
|  RAG Pipeline    |  <- Retrieves relevant docs from the knowledge base
|                  |
|  1. Embed query  |
|  2. Search       |
|     vectors      |
|  3. Rerank       |
|  4. Return top-k |
+--------+---------+
         |
         v
+------------------+
|   LLM Call       |  <- Generates answer grounded in retrieved docs
|  with skill      |
|  + context       |
+--------+---------+
         |
         v
  Answer with source citations
```

### 6.3 Building the Knowledge Skill

```typescript
// knowledge-skill.ts -- Company knowledge base skill
import Anthropic from "@anthropic-ai/sdk";

interface KnowledgeSource {
  name: string;
  type: "confluence" | "notion" | "github" | "google_docs" | "custom";
  syncFrequency: "hourly" | "daily" | "weekly";
  indexer: (source: KnowledgeSource) => Promise<KnowledgeDocument[]>;
}

interface KnowledgeDocument {
  id: string;
  title: string;
  content: string;
  source: string;
  url: string;
  lastUpdated: Date;
  metadata: Record<string, string>;
}

class CompanyKnowledgeSkill {
  private vectorStore: VectorStore;
  private embeddingService: EmbeddingService;
  private sources: KnowledgeSource[];

  constructor(
    vectorStore: VectorStore,
    embeddingService: EmbeddingService,
    sources: KnowledgeSource[]
  ) {
    this.vectorStore = vectorStore;
    this.embeddingService = embeddingService;
    this.sources = sources;
  }

  // Index all sources (run on a schedule)
  async syncKnowledgeBase(): Promise<{ indexed: number; errors: number }> {
    let indexed = 0;
    let errors = 0;

    for (const source of this.sources) {
      try {
        const documents = await source.indexer(source);

        for (const doc of documents) {
          const chunks = this.chunkDocument(doc);

          for (const chunk of chunks) {
            const embedding = await this.embeddingService.embed(chunk.content);
            await this.vectorStore.upsert({
              id: `${doc.id}-chunk-${chunk.index}`,
              embedding,
              metadata: {
                title: doc.title,
                source: doc.source,
                url: doc.url,
                content: chunk.content,
                lastUpdated: doc.lastUpdated.toISOString(),
                ...doc.metadata,
              },
            });
            indexed++;
          }
        }
      } catch (error) {
        console.error(`Error syncing ${source.name}: ${error}`);
        errors++;
      }
    }

    return { indexed, errors };
  }

  // Answer a question using the knowledge base
  async answer(question: string): Promise<{
    answer: string;
    sources: { title: string; url: string; relevance: number }[];
  }> {
    const queryEmbedding = await this.embeddingService.embed(question);

    const results = await this.vectorStore.query({
      embedding: queryEmbedding,
      topK: 10,
      minScore: 0.5,
    });

    const topResults = results.slice(0, 5);

    const context = topResults
      .map(
        (r, i) =>
          `[Source ${i + 1}: ${r.metadata.title}](${r.metadata.url})\n${r.metadata.content}`
      )
      .join("\n\n---\n\n");

    const client = new Anthropic();
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 2048,
      system: `You are a knowledgeable assistant for the company.
Answer questions using ONLY the provided context documents.
If the context doesn't contain enough information, say so.
Always cite your sources using [Source N] format.
Be specific and practical.`,
      messages: [
        {
          role: "user",
          content: `Question: ${question}\n\nContext:\n${context}`,
        },
      ],
    });

    const answer = response.content
      .filter((b): b is Anthropic.TextBlock => b.type === "text")
      .map((b) => b.text)
      .join("");

    return {
      answer,
      sources: topResults.map((r) => ({
        title: r.metadata.title as string,
        url: r.metadata.url as string,
        relevance: r.score,
      })),
    };
  }

  private chunkDocument(
    doc: KnowledgeDocument,
    maxChunkSize: number = 1000
  ): { content: string; index: number }[] {
    const chunks: { content: string; index: number }[] = [];
    const paragraphs = doc.content.split("\n\n");
    let currentChunk = "";
    let chunkIndex = 0;

    for (const paragraph of paragraphs) {
      if (currentChunk.length + paragraph.length > maxChunkSize) {
        if (currentChunk) {
          chunks.push({
            content: `# ${doc.title}\n\n${currentChunk}`,
            index: chunkIndex++,
          });
        }
        currentChunk = paragraph;
      } else {
        currentChunk += (currentChunk ? "\n\n" : "") + paragraph;
      }
    }

    if (currentChunk) {
      chunks.push({
        content: `# ${doc.title}\n\n${currentChunk}`,
        index: chunkIndex,
      });
    }

    return chunks;
  }
}
```

---

## 7. Discovery UX

### 7.1 The Marketplace Interface

A skills marketplace is only as good as its discovery UX. If people cannot find skills, the marketplace is a graveyard.

```
+-----------------------------------------------------------+
|  Skills Marketplace                          [Search...]   |
+-----------------------------------------------------------+
|                                                            |
|  Recommended for you (by Sensei)                           |
|  +----------+ +----------+ +----------+ +----------+      |
|  | SQL Query| | PR Review| | Incident | | API Doc  |      |
|  | Helper   | | Template | | Response | | Generator|      |
|  | *****    | | ****     | | *****    | | ****     |      |
|  | 245 uses | | 189 uses | | 156 uses | | 98 uses  |      |
|  +----------+ +----------+ +----------+ +----------+      |
|                                                            |
|  Popular this week                                         |
|  +----------+ +----------+ +----------+                    |
|  | Meeting  | | Jira>PRD | | Data     |                    |
|  | Summary  | | Converter| | Pipeline |                    |
|  | *****    | | ****     | | *****    |                    |
|  | 312 uses | | 87 uses  | | 201 uses |                    |
|  +----------+ +----------+ +----------+                    |
|                                                            |
|  Browse by category                                        |
|  [Engineering (142)] [Sales (67)] [Product (45)]          |
|  [Data (38)] [Design (24)] [Operations (19)]              |
|  [Security (12)] [General (8)]                             |
|                                                            |
|  Recently updated                                          |
|  - SQL Query Helper v2.1 -- 2 hours ago                   |
|  - API Doc Generator v1.3 -- 6 hours ago                  |
|  - Meeting Summary v3.0 -- 1 day ago                      |
|                                                            |
+-----------------------------------------------------------+
```

### 7.2 Skill Detail Page

The detail page shows everything a user needs to decide whether to use a skill: what it does, how well it works, who made it, and how it has evolved.

Key elements:
- **One-line description** at the top
- **Rating and usage count** for social proof
- **Examples** showing real input/output pairs
- **Changelog** showing evolution
- **Reviews** from other users
- **Install button** that adds the skill to your active set

---

## 8. Knowledge Compounding

### 8.1 The Math of Compounding

Knowledge compounding is why skills marketplaces are transformative. Each new skill raises the floor for everyone.

```
Week 1:   10 skills  x  50 users  =    500 skill-uses/week
Week 4:   50 skills  x  100 users =  5,000 skill-uses/week
Week 8:  120 skills  x  200 users = 24,000 skill-uses/week
Week 12: 200 skills  x  350 users = 70,000 skill-uses/week
Week 16: 350 skills  x  500 users = 175,000 skill-uses/week
```

The growth is superlinear because more skills attract more users, and more users create more skills. This is the flywheel that Ramp's "one person's breakthrough becomes everyone's baseline" philosophy describes.

### 8.2 Measuring Knowledge Compounding

```typescript
// compounding-metrics.ts -- Track knowledge compounding

interface CompoundingMetrics {
  totalSkills: number;
  newSkillsThisWeek: number;
  activeAuthors: number;
  avgSkillAge: number;

  totalUses: number;
  uniqueUsers: number;
  usesPerUser: number;
  skillDiscoveryRate: number;

  avgRating: number;
  skillsAboveThreshold: number;
  deprecationRate: number;

  compoundingRate: number;
}

function calculateCompounding(
  current: CompoundingMetrics,
  previous: CompoundingMetrics
): {
  isCompounding: boolean;
  rate: number;
  bottleneck?: string;
} {
  const currentRatio = current.totalUses / current.totalSkills;
  const previousRatio = previous.totalUses / previous.totalSkills;
  const rate = previousRatio > 0
    ? (currentRatio - previousRatio) / previousRatio
    : 0;

  let bottleneck: string | undefined;
  if (current.newSkillsThisWeek < 3) {
    bottleneck = "Supply: not enough new skills being created";
  } else if (current.skillDiscoveryRate < 0.1) {
    bottleneck = "Discovery: users aren't finding relevant skills";
  } else if (current.avgRating < 3.5) {
    bottleneck = "Quality: skill quality is too low for adoption";
  }

  return {
    isCompounding: rate > 0,
    rate,
    bottleneck,
  };
}
```

---

## 9. Preventing Skill Sprawl

### 9.1 Governance Without Bureaucracy

As the marketplace grows, you need governance -- but not the kind that kills velocity. The Ramp approach: light governance through tooling, not process.

**Automated checks replace manual review for simple skills.** If the skill has good eval scores, follows the schema, and does not use sensitive tools, it can be auto-approved.

**Human review for skills that access sensitive data.** Skills that query financial databases, customer PII, or production systems get human review.

**Deprecation is not deletion.** Old skills are marked as deprecated with a link to the replacement. They still work for anyone who depends on them.

### 9.2 Duplicate Detection

With 350+ skills, overlap is inevitable. Duplicate detection runs automatically when new skills are submitted.

```typescript
// dedup.ts -- Detect duplicate skills

async function checkForDuplicates(
  newSkill: MarketplaceSkill,
  registry: SkillRegistry,
  embeddingService: EmbeddingService
): Promise<{
  hasDuplicates: boolean;
  similar: { skill: MarketplaceSkill; similarity: number }[];
}> {
  const newEmbedding = await embeddingService.embed(
    `${newSkill.name} ${newSkill.description} ${newSkill.content.slice(0, 500)}`
  );

  const existingSkills = registry.search("", { status: "published" });
  const similar: { skill: MarketplaceSkill; similarity: number }[] = [];

  for (const existing of existingSkills) {
    const existingEmbedding = await embeddingService.embed(
      `${existing.name} ${existing.description} ${existing.content.slice(0, 500)}`
    );

    const similarity = cosineSimilarity(newEmbedding, existingEmbedding);
    if (similarity > 0.85) {
      similar.push({ skill: existing, similarity });
    }
  }

  return {
    hasDuplicates: similar.length > 0,
    similar: similar.sort((a, b) => b.similarity - a.similarity),
  };
}
```

---

## 10. Chapter Summary

A skills marketplace transforms individual knowledge into organizational capability. The Dojo pattern -- Git-backed, versioned, reviewed, searchable, recommended -- creates a flywheel where every skill shared raises the floor for everyone. Sensei's AI-driven recommendations solve the discovery problem that kills most knowledge bases. Eval-based quality gates ensure the marketplace is full of skills worth using.

The key insight: skills are not documentation. Skills are *executable knowledge*. When you share a skill, you are not sharing information that someone might read. You are sharing a capability that anyone can invoke. This is why skills compound in a way that wiki pages never did.

In Chapter 61, we close the loop: the system itself uses the feedback from the marketplace to improve itself. Self-reinforcing AI systems.

---

*Previous: [Ch 59 -- Building Internal AI Tools](./59-internal-ai-tools.md)* | *Next: [Ch 61 -- Self-Reinforcing AI Systems](./61-self-reinforcing-systems.md)*

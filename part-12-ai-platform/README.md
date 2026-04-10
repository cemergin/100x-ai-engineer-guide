<!--
  TYPE: part-index
  PART: 12
  TITLE: AI Platform Engineering
  PHASE: 2 — Become an Expert
  CHAPTERS: 58, 59, 60, 61, 62
  UPDATED: 2026-04-10
-->

# Part 12 — AI Platform Engineering

> **Phase 2: Become an Expert** | 5 chapters | Advanced | Language: TypeScript + Python + Architecture

The capstone. From your setup to everyone's setup. This Part spirals everything from the entire guide together into the engineering discipline of building AI platforms that make entire organizations productive with AI.

You have spent 57 chapters getting here. You can build AI features, construct agent loops, set up RAG pipelines, run evals, understand how coding agents work internally, build neural networks from scratch, fine-tune models, harden production systems, and deploy infrastructure. Now the question changes from "How do I use AI?" to "How do I make *everyone* use AI?"

This is the Ramp question. In early 2025, Ramp's AI Platform team -- four engineers -- built Glass, an internal Claude Code-like tool on Anthropic's Agent SDK. Within a month it had 700 daily active users. Within six months, 99.5% of the engineering team was active on AI tools, 84% were using coding agents weekly, and non-engineers were responsible for 12% of all human-initiated pull requests. They didn't get there by sending a Slack message that said "please use AI." They got there by building a platform so good that using it was easier than not using it.

That's what this Part teaches you to build.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 58 | [Multi-Agent Orchestration](./58-multi-agent-orchestration.md) | Advanced | Agents spawning agents, council patterns, parallel execution, error handling |
| 59 | [Building Internal AI Tools](./59-internal-ai-tools.md) | Advanced | The Glass pattern: SSO, pre-configured, low barrier internal platforms |
| 60 | [Skills Marketplaces & Knowledge Sharing](./60-skills-marketplaces.md) | Advanced | The Dojo pattern: Git-backed, versioned, discovery, Sensei recommendations |
| 61 | [Self-Reinforcing AI Systems](./61-self-reinforcing-systems.md) | Advanced | Feedback loops, eval-driven iteration, agents that improve themselves |
| 62 | [AI Adoption & Enablement](./62-ai-adoption.md) | All levels | The Ramp playbook: L0-L3, leaderboards, hub-and-spoke, removing constraints |

## The Within-Part Spiral

```
Ch 58: Multi-Agent Orchestration (orchestrate agents)
  └──> Ch 59: Building Internal AI Tools (package as a tool)
        └──> Ch 60: Skills Marketplaces & Knowledge Sharing (share what works)
              └──> Ch 61: Self-Reinforcing AI Systems (make it self-improving)
                    └──> Ch 62: AI Adoption & Enablement (scale to the org)
```

This spiral follows the journey from individual capability to organizational transformation. First you learn to orchestrate multiple agents working together (Ch 58). Then you package that capability into an internal tool that anyone can use (Ch 59). Then you create a marketplace so every skill, prompt, and workflow can be shared (Ch 60). Then you wire feedback loops so the system improves itself over time (Ch 61). And finally you scale it across the entire organization with the cultural and structural patterns that make adoption stick (Ch 62).

## How This Part Connects

```
EVERY PREVIOUS PART
  │
  └──> Part 12 (you are here — the capstone)
         Ch 58 spirals:
           Ch 9 (one agent loop → many)
           Ch 13 (agent patterns → orchestration patterns)
           Ch 23-25 (harness skills/plugins → agent configuration in orchestration)
           Ch 32 (coordination internals → production orchestration)

         Ch 59 spirals:
           Ch 23-25 (using the harness → building the harness)
           Ch 27 (harness architecture → platform architecture)
           Ch 55 (sandboxing → platform isolation)

         Ch 60 spirals:
           Ch 25 (skills → marketplace)
           Ch 33 (skill architecture → distribution system)

         Ch 61 spirals:
           Ch 21 (eval-driven dev → self-improving systems)
           Ch 26 (coding agents → agents improving agents)
           Ch 51 (eval pipelines → automated improvement)

         Ch 62 spirals:
           Ch 59 (platform enables adoption)
           ALL (you've learned everything — now everyone does)
```

## The Ramp Case Study

These chapters draw heavily from Ramp's publicly shared experience building their AI platform. Key reference points:

- **Glass**: Internal AI tool built on Anthropic's Agent SDK, auto-configured via Okta SSO with 30+ tools lighting up immediately on install
- **Dojo**: Skills marketplace with 350+ Git-backed, versioned, code-reviewed skills
- **Sensei**: AI recommender that surfaces relevant skills based on role, tools, and activity
- **Hub-and-spoke model**: Small central platform team builds the core; functional teams build on top
- **Metrics**: 700 DAUs within a month, 99.5% team active, 84% using coding agents weekly
- **Philosophy**: "Don't limit anyone's upside" / "One person's breakthrough = everyone's baseline" / "The product is the enablement"

## Reading Time

Expect about 6-8 hours to work through all five chapters. Chapter 58 and 59 are the most code-heavy -- you will build a multi-agent orchestration system and an internal AI tool from scratch. Chapter 60 is architecture-focused. Chapter 61 combines code and systems thinking. Chapter 62 is the guide's conclusion and is the most strategic -- it is written for the engineer who needs to convince leadership and drive adoption across an organization.

By the end of this Part, you will know how to build the AI platform that makes your whole company dangerous.

---

*Previous: [Part 11 — AI Deployment & Infrastructure](../part-11-deployment-infrastructure/)* | *Next: [Appendices](../appendices/)*

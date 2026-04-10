<!--
  TYPE: part-index
  PART: 11
  TITLE: AI Deployment & Infrastructure
  PHASE: 2 — Become an Expert
  CHAPTERS: 53, 54, 55, 56, 57
  UPDATED: 2026-04-10
-->

# Part 11 — AI Deployment & Infrastructure

> **Phase 2: Become an Expert** | 5 chapters | Intermediate to Advanced | Language: TypeScript + Python + Terraform + Docker + YAML

Deploy and run AI safely at scale. This Part spirals Part 10 from "what to worry about in production" to "how to actually deploy, isolate, automate, and scale it." You built the features. You hardened them. Now you ship them to real infrastructure.

In Part 10, you learned what makes production AI systems break: security gaps, runaway costs, context window explosions, evaluation regressions, and silent degradation. This Part turns those concerns into infrastructure. You will deploy an LLM application to production, put a gateway in front of your providers, sandbox your agents so they cannot reach production systems, wire evals into your CI/CD pipeline, and scale the whole thing without burning money.

The five chapters follow the real deployment workflow: first you deploy the app (Ch 53), then you manage the providers it talks to (Ch 54), then you isolate the dangerous parts -- agents with tools (Ch 55), then you automate the pipeline so deployments are safe and repeatable (Ch 56), and finally you optimize for scale and cost at the infrastructure level (Ch 57).

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 53 | [Deploying LLM Applications](./53-deploying-llm-apps.md) | Intermediate | Serverless vs dedicated, Vercel Functions, Docker, environment management |
| 54 | [API Gateway & Provider Management](./54-api-gateway.md) | Inter-Adv | Rate limiting, failover, Vercel AI Gateway, caching, secrets |
| 55 | [Sandboxing & Isolating Agents](./55-sandboxing-agents.md) | Advanced | The OpenClaw pattern, VPC isolation, proxy restrictions, file system isolation |
| 56 | [CI/CD for AI Applications](./56-cicd-for-ai.md) | Inter-Adv | Evals in CI, canary deployments, A/B testing, feature flags for AI |
| 57 | [Scaling & Cost at the Infra Level](./57-scaling-cost-infra.md) | Advanced | GPU vs CPU, serverless inference, batching, model caching, break-even math |

## The Within-Part Spiral

```
Ch 53: Deploying LLM Applications (get the app running)
  └──> Ch 54: API Gateway & Provider Management (manage the providers it talks to)
        └──> Ch 55: Sandboxing & Isolating Agents (isolate the dangerous parts)
              └──> Ch 56: CI/CD for AI Applications (automate safe deployments)
                    └──> Ch 57: Scaling & Cost at the Infra Level (optimize at scale)
```

This spiral mirrors the real journey: you cannot worry about sandboxing until you have something deployed. You cannot automate CI/CD until you know what needs protecting. You cannot optimize cost until everything is running and measured.

## How This Part Connects

```
Part 10 (Production AI Systems — what to worry about)
  │
  └──> Part 11 (you are here — how to actually deploy it)
         Ch 53 spirals Ch 3 (API calls → deploy them) and Ch 6 (chat UI → deploy full stack)
         Ch 54 spirals Ch 2 (providers → manage in production) and Ch 49 (cost → enforce at gateway)
         Ch 55 spirals Ch 30 (permissions → infrastructure) and Ch 48 (security → implement)
         Ch 56 spirals Ch 21 (eval-driven dev → CI) and Ch 22 (telemetry → CI metrics)
         Ch 57 spirals Ch 49 (app-level cost → infra-level) and Ch 47 (quantization → serve quantized)
         │
         └──> Part 12: AI Platform Engineering
                Ch 59: gateway and sandboxing for platform tools
                Ch 58: multi-agent needs the infra from Ch 55
                Ch 62: governance requires the CI/CD from Ch 56
```

## Reading Time

Expect about 8-10 hours to work through all five chapters. Chapter 53 and 56 are the most hands-on -- you will write real deployment configs, Dockerfiles, Terraform modules, and GitHub Actions workflows. Chapter 55 (sandboxing) is the conceptual centerpiece and draws heavily on a real-world agent deployment pattern from the Nelo engineering team. You will need familiarity with Docker, basic AWS concepts, and GitHub Actions. Terraform examples are self-contained.

---

*Previous: [Part 10 — Production AI Systems](../part-10-production-ai/)* | *Next: [Part 12 — AI Platform Engineering](../part-12-ai-platform/)*

<!--
  CHAPTER: 51
  TITLE: Production Eval Pipelines
  PART: 10 — Production AI Systems
  PHASE: 2 — Become an Expert
  PREREQS: Ch 18 (why evals matter), Ch 19-20 (single/multi-turn evals), Ch 21 (eval-driven dev), Ch 22 (telemetry)
  KEY_TOPICS: continuous evaluation, CI/CD evals, A/B testing, regression detection, online evals, eval dashboards, alerting
  DIFFICULTY: Advanced
  LANGUAGE: TypeScript + Python
  UPDATED: 2026-04-10
-->

# Chapter 51: Production Eval Pipelines

> **Part 10 — Production AI Systems** | Phase 2: Become an Expert | Prerequisites: Ch 18-22 | Difficulty: Advanced | Language: TypeScript + Python

In Phase 1, you learned to write evals and run them manually. You'd change a prompt, run the eval suite, look at the scores, and decide whether the change was good. That works when you're a single developer iterating on a prototype.

It doesn't work when you have 12 AI features in production, 5 engineers making changes, a model provider that updates their API without telling you, and 50,000 users who will notice if quality drops. In production, evals must be *automated*, *continuous*, and *actionable* — running on every deployment, monitoring live traffic, and alerting before users complain.

This chapter takes the eval concepts from Part 4 and turns them into production infrastructure. By the end, you'll have evals running in your CI/CD pipeline, A/B tests measuring prompt changes with statistical rigor, regression detectors catching model drift, and dashboards showing quality scores over time.

### In This Chapter
- Continuous evaluation: running evals on every deployment
- Evals in CI/CD: fail the build if eval scores drop
- A/B testing LLM changes: comparing prompt versions with statistical significance
- Regression detection: catching when a model update breaks existing behavior
- Online evals: monitoring live traffic for quality
- The eval dashboard: scores over time, by feature, by model version
- Alerting on eval degradation

### Related Chapters
- **Ch 4 (Prompt Engineering)** — spirals back: prompt management at scale (registries, versioning) feeds into eval-driven prompt iteration
- **Ch 18-20 (Evals & Quality)** — spirals back: single-turn and multi-turn eval foundations
- **Ch 21 (Eval-Driven Development)** — spirals back: the Ralph loop (write eval, run, analyze, improve)
- **Ch 22 (Telemetry & Tracing)** — spirals back: telemetry data feeds online evals
- **Ch 56 (CI/CD for AI Applications)** — spirals forward: broader CI/CD patterns for AI
- **Ch 61 (Self-Reinforcing AI Systems)** — spirals forward: evals that drive autonomous improvement

---

## 1. From Manual Evals to Continuous Evaluation

### 1.1 The Eval Maturity Model

Most teams evolve through these stages:

```
Level 0: No evals
  "Does it look right?" (manual testing only)

Level 1: Ad-hoc evals
  Engineer runs evals locally before merging a PR
  Evals live in a notebook or script

Level 2: CI evals
  Evals run automatically on every PR
  Build fails if scores drop below threshold

Level 3: Continuous evals
  Evals run on a schedule against production
  Dashboard shows scores over time
  Alerts fire on degradation

Level 4: Online evals
  Every production request is evaluated in real time
  A/B tests compare prompt/model versions
  Evals drive automated rollback decisions

Level 5: Self-reinforcing evals
  Eval results feed back into training data
  System identifies its own failure patterns
  (This is Ch 61)
```

Most production teams should target Level 3. Level 4 is for high-volume features where the investment pays off. This chapter covers Levels 2-4.

### 1.2 The Eval Suite Structure

```
evals/
├── datasets/
│   ├── customer_support.jsonl      # Test cases for support feature
│   ├── code_review.jsonl           # Test cases for code review
│   ├── search.jsonl                # Test cases for search feature
│   └── golden/                     # Curated "golden" test sets
│       ├── support_golden.jsonl    # Expert-validated cases
│       └── code_review_golden.jsonl
├── scorers/
│   ├── relevance.ts                # Is the response relevant?
│   ├── accuracy.ts                 # Is the response factually correct?
│   ├── tool_selection.ts           # Did it pick the right tool?
│   ├── format.ts                   # Is the output properly formatted?
│   └── safety.ts                   # Does it pass safety checks?
├── runners/
│   ├── ci_runner.ts                # Runs in CI pipeline
│   ├── scheduled_runner.ts         # Runs on schedule against prod
│   └── online_runner.ts            # Runs on live traffic
├── config/
│   ├── thresholds.yaml             # Pass/fail thresholds per feature
│   └── models.yaml                 # Model versions to test
└── reports/
    └── (generated eval reports)
```

---

## 2. Evals in CI/CD

### 2.1 The CI Eval Pipeline

Every pull request that touches a prompt, model configuration, or tool definition should run evals automatically.

```yaml
# .github/workflows/ai-evals.yml

name: AI Eval Pipeline
on:
  pull_request:
    paths:
      - "src/ai/**"
      - "prompts/**"
      - "evals/**"
      - "config/models.yaml"

jobs:
  eval:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install dependencies
        run: npm ci

      - name: Run AI evals
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          EVAL_MODE: ci
        run: npx tsx evals/runners/ci_runner.ts

      - name: Upload eval report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-report
          path: evals/reports/

      - name: Comment eval results on PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('evals/reports/summary.json', 'utf8'));
            
            let body = '## AI Eval Results\n\n';
            body += `| Feature | Score | Threshold | Status |\n`;
            body += `|---------|-------|-----------|--------|\n`;
            
            for (const [feature, result] of Object.entries(report.features)) {
              const status = result.passed ? '✅ Pass' : '❌ Fail';
              body += `| ${feature} | ${(result.score * 100).toFixed(1)}% | ${(result.threshold * 100).toFixed(1)}% | ${status} |\n`;
            }
            
            body += `\n**Overall: ${report.passed ? '✅ All evals passed' : '❌ Some evals failed'}**`;
            
            if (report.regressions.length > 0) {
              body += '\n\n### Regressions Detected\n';
              for (const reg of report.regressions) {
                body += `- ${reg.feature}: ${reg.metric} dropped from ${(reg.baseline * 100).toFixed(1)}% to ${(reg.current * 100).toFixed(1)}%\n`;
              }
            }
            
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body
            });
```

### 2.2 The CI Eval Runner

```typescript
// evals/runners/ci_runner.ts

import fs from "fs";
import path from "path";

interface EvalConfig {
  features: Record<string, {
    dataset: string;
    scorers: string[];
    threshold: number;
    model: string;
    systemPrompt: string;
    sampleSize?: number;  // For CI, run a subset
  }>;
}

interface EvalResult {
  feature: string;
  score: number;
  threshold: number;
  passed: boolean;
  details: Array<{
    input: string;
    expected: string;
    actual: string;
    scores: Record<string, number>;
  }>;
}

interface EvalReport {
  timestamp: string;
  commit: string;
  branch: string;
  features: Record<string, EvalResult>;
  passed: boolean;
  regressions: Array<{
    feature: string;
    metric: string;
    baseline: number;
    current: number;
  }>;
}

async function runCIEvals(): Promise<void> {
  const config: EvalConfig = loadConfig("evals/config/thresholds.yaml");
  const report: EvalReport = {
    timestamp: new Date().toISOString(),
    commit: process.env.GITHUB_SHA || "local",
    branch: process.env.GITHUB_HEAD_REF || "local",
    features: {},
    passed: true,
    regressions: [],
  };

  for (const [feature, featureConfig] of Object.entries(config.features)) {
    console.log(`\nRunning evals for: ${feature}`);

    // Load dataset
    const dataset = loadDataset(featureConfig.dataset);
    const sample = featureConfig.sampleSize
      ? sampleDataset(dataset, featureConfig.sampleSize)
      : dataset;

    console.log(`  Dataset: ${sample.length} cases (of ${dataset.length} total)`);

    // Run eval
    const result = await runFeatureEval(feature, sample, featureConfig);
    report.features[feature] = result;

    if (!result.passed) {
      report.passed = false;
      console.log(`  ❌ FAILED: ${(result.score * 100).toFixed(1)}% < ${(result.threshold * 100).toFixed(1)}%`);
    } else {
      console.log(`  ✅ PASSED: ${(result.score * 100).toFixed(1)}% >= ${(result.threshold * 100).toFixed(1)}%`);
    }

    // Check for regression vs baseline
    const baseline = await loadBaseline(feature);
    if (baseline && result.score < baseline.score - 0.05) {  // 5% regression threshold
      report.regressions.push({
        feature,
        metric: "overall_score",
        baseline: baseline.score,
        current: result.score,
      });
    }
  }

  // Write report
  const reportPath = "evals/reports/summary.json";
  fs.mkdirSync(path.dirname(reportPath), { recursive: true });
  fs.writeFileSync(reportPath, JSON.stringify(report, null, 2));

  // Exit with error code if evals failed
  if (!report.passed) {
    console.log("\n❌ Eval pipeline FAILED. See report for details.");
    process.exit(1);
  }

  console.log("\n✅ All evals passed.");
}

async function runFeatureEval(
  feature: string,
  dataset: Array<{ input: string; expected: string; metadata?: Record<string, any> }>,
  config: EvalConfig["features"][string],
): Promise<EvalResult> {
  const details: EvalResult["details"] = [];
  let totalScore = 0;

  for (const testCase of dataset) {
    // Get LLM response
    const response = await callLLM(
      config.model,
      config.systemPrompt,
      testCase.input,
    );

    // Run all scorers
    const scores: Record<string, number> = {};
    for (const scorerName of config.scorers) {
      const scorer = loadScorer(scorerName);
      scores[scorerName] = await scorer(testCase.input, testCase.expected, response);
    }

    // Average score across scorers
    const avgScore = Object.values(scores).reduce((a, b) => a + b, 0) / Object.values(scores).length;
    totalScore += avgScore;

    details.push({
      input: testCase.input,
      expected: testCase.expected,
      actual: response,
      scores,
    });
  }

  const overallScore = totalScore / dataset.length;

  return {
    feature,
    score: overallScore,
    threshold: config.threshold,
    passed: overallScore >= config.threshold,
    details,
  };
}

// Run
runCIEvals().catch(console.error);
```

### 2.3 Eval Thresholds Configuration

```yaml
# evals/config/thresholds.yaml

features:
  customer_support:
    dataset: evals/datasets/customer_support.jsonl
    scorers:
      - relevance
      - accuracy
      - safety
    threshold: 0.85
    model: claude-sonnet-4-20250514
    systemPrompt: prompts/customer_support.txt
    sampleSize: 50  # Run 50 cases in CI (full suite runs nightly)

  code_review:
    dataset: evals/datasets/code_review.jsonl
    scorers:
      - relevance
      - format
      - tool_selection
    threshold: 0.80
    model: claude-sonnet-4-20250514
    systemPrompt: prompts/code_review.txt
    sampleSize: 30

  search:
    dataset: evals/datasets/search.jsonl
    scorers:
      - relevance
    threshold: 0.75
    model: claude-haiku-4-20250414
    systemPrompt: prompts/search.txt
    sampleSize: 100

# Regression detection
regression:
  threshold: 0.05  # Alert if score drops more than 5% from baseline
  baseline_window_days: 7  # Compare against 7-day rolling average
```

---

## 3. A/B Testing LLM Changes

### 3.1 Why A/B Testing Matters for LLMs

Evals tell you how a change performs on your test set. A/B tests tell you how it performs on *real users with real inputs*. The gap between these two can be enormous:

- Your test set might not represent the distribution of real queries
- Users interact differently than your test cases assume
- Edge cases appear in production that your test set doesn't cover
- User satisfaction doesn't always correlate with eval scores

### 3.2 Implementation

```typescript
// TypeScript: A/B testing for LLM features

interface ABTestConfig {
  testId: string;
  feature: string;
  variants: {
    control: VariantConfig;
    treatment: VariantConfig;
  };
  trafficSplit: number;  // 0.0-1.0, fraction of traffic to treatment
  minSampleSize: number;
  maxDurationDays: number;
  successMetrics: string[];
}

interface VariantConfig {
  model: string;
  systemPrompt: string;
  temperature?: number;
  maxTokens?: number;
}

interface ABTestResult {
  testId: string;
  controlMetrics: Record<string, { mean: number; stdDev: number; n: number }>;
  treatmentMetrics: Record<string, { mean: number; stdDev: number; n: number }>;
  pValues: Record<string, number>;
  significantDifferences: string[];
  recommendation: "control" | "treatment" | "inconclusive";
}

class ABTestManager {
  private tests: Map<string, ABTestConfig> = new Map();
  private results: Map<string, Array<{ variant: string; metrics: Record<string, number> }>> = new Map();

  async assignVariant(
    testId: string,
    userId: string,
  ): Promise<"control" | "treatment"> {
    const test = this.tests.get(testId);
    if (!test) return "control";

    // Deterministic assignment based on user ID (consistent experience)
    const hash = hashString(`${testId}:${userId}`);
    const bucket = (hash % 100) / 100;

    return bucket < test.trafficSplit ? "treatment" : "control";
  }

  async getConfig(
    testId: string,
    variant: "control" | "treatment",
  ): Promise<VariantConfig> {
    const test = this.tests.get(testId);
    if (!test) throw new Error(`Test ${testId} not found`);

    return test.variants[variant];
  }

  async recordResult(
    testId: string,
    variant: "control" | "treatment",
    metrics: Record<string, number>,
  ): Promise<void> {
    if (!this.results.has(testId)) {
      this.results.set(testId, []);
    }
    this.results.get(testId)!.push({ variant, metrics });
  }

  async analyzeTest(testId: string): Promise<ABTestResult> {
    const test = this.tests.get(testId);
    const data = this.results.get(testId) || [];

    const controlData = data.filter((d) => d.variant === "control");
    const treatmentData = data.filter((d) => d.variant === "treatment");

    const controlMetrics: ABTestResult["controlMetrics"] = {};
    const treatmentMetrics: ABTestResult["treatmentMetrics"] = {};
    const pValues: Record<string, number> = {};
    const significantDifferences: string[] = [];

    for (const metric of test!.successMetrics) {
      const controlValues = controlData.map((d) => d.metrics[metric]).filter(Boolean);
      const treatmentValues = treatmentData.map((d) => d.metrics[metric]).filter(Boolean);

      controlMetrics[metric] = {
        mean: mean(controlValues),
        stdDev: stdDev(controlValues),
        n: controlValues.length,
      };

      treatmentMetrics[metric] = {
        mean: mean(treatmentValues),
        stdDev: stdDev(treatmentValues),
        n: treatmentValues.length,
      };

      // Welch's t-test for significance
      const pValue = welchTTest(controlValues, treatmentValues);
      pValues[metric] = pValue;

      if (pValue < 0.05) {
        significantDifferences.push(metric);
      }
    }

    // Recommendation
    let recommendation: ABTestResult["recommendation"] = "inconclusive";
    if (significantDifferences.length > 0) {
      const treatmentBetter = significantDifferences.every(
        (m) => treatmentMetrics[m].mean > controlMetrics[m].mean
      );
      recommendation = treatmentBetter ? "treatment" : "control";
    }

    return {
      testId,
      controlMetrics,
      treatmentMetrics,
      pValues,
      significantDifferences,
      recommendation,
    };
  }
}

// Statistical helper: Welch's t-test
function welchTTest(a: number[], b: number[]): number {
  const meanA = mean(a);
  const meanB = mean(b);
  const varA = variance(a);
  const varB = variance(b);
  const nA = a.length;
  const nB = b.length;

  const t = (meanA - meanB) / Math.sqrt(varA / nA + varB / nB);

  // Welch-Satterthwaite degrees of freedom
  const df =
    Math.pow(varA / nA + varB / nB, 2) /
    (Math.pow(varA / nA, 2) / (nA - 1) + Math.pow(varB / nB, 2) / (nB - 1));

  // Approximate p-value using t-distribution (simplified)
  return approximatePValue(Math.abs(t), df);
}

function mean(arr: number[]): number {
  return arr.reduce((a, b) => a + b, 0) / arr.length;
}

function variance(arr: number[]): number {
  const m = mean(arr);
  return arr.reduce((sum, x) => sum + Math.pow(x - m, 2), 0) / (arr.length - 1);
}

function stdDev(arr: number[]): number {
  return Math.sqrt(variance(arr));
}
```

```python
# Python: A/B testing with statistical analysis

import hashlib
from dataclasses import dataclass, field
from scipy import stats
import numpy as np

@dataclass
class ABTestConfig:
    test_id: str
    feature: str
    control_model: str
    control_prompt: str
    treatment_model: str
    treatment_prompt: str
    traffic_split: float = 0.5
    min_sample_size: int = 100
    success_metrics: list[str] = field(default_factory=lambda: ["relevance", "user_satisfaction"])

def assign_variant(test_id: str, user_id: str, traffic_split: float) -> str:
    """Deterministic variant assignment based on user ID."""
    hash_input = f"{test_id}:{user_id}".encode()
    hash_val = int(hashlib.md5(hash_input).hexdigest(), 16) % 100
    return "treatment" if hash_val < traffic_split * 100 else "control"

def analyze_ab_test(
    control_scores: dict[str, list[float]],
    treatment_scores: dict[str, list[float]],
    metrics: list[str],
) -> dict:
    """Analyze A/B test results with statistical significance."""
    results = {}

    for metric in metrics:
        control = np.array(control_scores.get(metric, []))
        treatment = np.array(treatment_scores.get(metric, []))

        if len(control) < 30 or len(treatment) < 30:
            results[metric] = {
                "status": "insufficient_data",
                "control_n": len(control),
                "treatment_n": len(treatment),
            }
            continue

        # Welch's t-test (does not assume equal variance)
        t_stat, p_value = stats.ttest_ind(control, treatment, equal_var=False)

        # Effect size (Cohen's d)
        pooled_std = np.sqrt(
            (np.std(control, ddof=1) ** 2 + np.std(treatment, ddof=1) ** 2) / 2
        )
        cohens_d = (np.mean(treatment) - np.mean(control)) / pooled_std if pooled_std > 0 else 0

        results[metric] = {
            "control_mean": float(np.mean(control)),
            "control_std": float(np.std(control, ddof=1)),
            "control_n": len(control),
            "treatment_mean": float(np.mean(treatment)),
            "treatment_std": float(np.std(treatment, ddof=1)),
            "treatment_n": len(treatment),
            "t_statistic": float(t_stat),
            "p_value": float(p_value),
            "significant": p_value < 0.05,
            "cohens_d": float(cohens_d),
            "effect_size": (
                "small" if abs(cohens_d) < 0.5
                else "medium" if abs(cohens_d) < 0.8
                else "large"
            ),
            "winner": (
                "treatment" if p_value < 0.05 and np.mean(treatment) > np.mean(control)
                else "control" if p_value < 0.05 and np.mean(control) > np.mean(treatment)
                else "inconclusive"
            ),
        }

    return results

# Example usage
control = {"relevance": [0.85, 0.90, 0.78, ...], "satisfaction": [4.2, 3.8, 4.5, ...]}
treatment = {"relevance": [0.88, 0.92, 0.81, ...], "satisfaction": [4.5, 4.1, 4.7, ...]}

results = analyze_ab_test(control, treatment, ["relevance", "satisfaction"])
for metric, r in results.items():
    if r.get("significant"):
        print(f"{metric}: {r['winner']} wins (p={r['p_value']:.4f}, d={r['cohens_d']:.2f})")
    else:
        print(f"{metric}: No significant difference (p={r['p_value']:.4f})")
```

### 3.3 Integrating A/B Tests into Your API

```typescript
// TypeScript: A/B test middleware for an AI feature

const abTestManager = new ABTestManager();

async function handleAIRequest(
  userId: string,
  feature: string,
  userMessage: string,
): Promise<{ response: string; metadata: object }> {
  // Check if there's an active A/B test for this feature
  const activeTest = getActiveTest(feature);

  if (activeTest) {
    const variant = await abTestManager.assignVariant(activeTest.testId, userId);
    const config = await abTestManager.getConfig(activeTest.testId, variant);

    const startTime = Date.now();
    const response = await callLLM(config.model, config.systemPrompt, userMessage);
    const latencyMs = Date.now() - startTime;

    // Score the response (async — don't block the response)
    scoreResponseAsync(activeTest.testId, variant, userMessage, response, latencyMs);

    return {
      response,
      metadata: {
        abTest: activeTest.testId,
        variant,
        model: config.model,
      },
    };
  }

  // No active test — use default config
  const response = await callLLM(DEFAULT_MODEL, DEFAULT_PROMPT, userMessage);
  return { response, metadata: {} };
}

async function scoreResponseAsync(
  testId: string,
  variant: string,
  input: string,
  response: string,
  latencyMs: number,
): Promise<void> {
  // Run scorers in the background
  const relevance = await scoreRelevance(input, response);
  const quality = await scoreQuality(response);

  await abTestManager.recordResult(testId, variant as any, {
    relevance,
    quality,
    latency: latencyMs / 1000,  // Convert to seconds
    tokenCount: estimateTokens(response),
  });
}
```

---

## 4. Regression Detection

### 4.1 What Causes LLM Regressions

LLM regressions happen for reasons that don't exist in traditional software:

1. **Model provider updates.** Anthropic, OpenAI, and Google regularly update their models. Even "stable" model versions can have behavior changes.
2. **Prompt changes.** A seemingly minor prompt edit can cause unexpected behavior changes.
3. **Data drift.** User queries change over time. Your eval dataset becomes stale.
4. **Context changes.** Updates to RAG documents, system prompts, or tool definitions can affect quality.
5. **Traffic pattern changes.** A marketing campaign brings different types of users with different query distributions.

### 4.2 Automated Regression Detection

```typescript
// TypeScript: Regression detection system

interface RegressionConfig {
  features: string[];
  checkIntervalHours: number;
  baselineWindowDays: number;
  regressionThreshold: number;  // e.g., 0.05 = 5% drop
  alertChannels: string[];
}

interface BaselineMetrics {
  feature: string;
  period: { start: Date; end: Date };
  metrics: Record<string, {
    mean: number;
    stdDev: number;
    p5: number;   // 5th percentile
    p50: number;  // median
    p95: number;  // 95th percentile
    n: number;
  }>;
}

async function checkForRegressions(config: RegressionConfig): Promise<void> {
  for (const feature of config.features) {
    // 1. Get baseline metrics (rolling window)
    const baseline = await getBaselineMetrics(feature, config.baselineWindowDays);

    // 2. Get recent metrics (last check interval)
    const recent = await getRecentMetrics(feature, config.checkIntervalHours);

    // 3. Compare
    for (const [metric, baselineData] of Object.entries(baseline.metrics)) {
      const recentData = recent.metrics[metric];
      if (!recentData) continue;

      const drop = baselineData.mean - recentData.mean;
      const dropPercent = drop / baselineData.mean;

      if (dropPercent > config.regressionThreshold) {
        // Statistical significance check
        const significant = isStatisticallySignificant(
          baselineData.mean, baselineData.stdDev, baselineData.n,
          recentData.mean, recentData.stdDev, recentData.n,
        );

        if (significant) {
          await sendRegressionAlert({
            feature,
            metric,
            baselineMean: baselineData.mean,
            recentMean: recentData.mean,
            dropPercent,
            baselinePeriod: baseline.period,
            recentN: recentData.n,
            channels: config.alertChannels,
          });
        }
      }
    }
  }
}

function isStatisticallySignificant(
  mean1: number, std1: number, n1: number,
  mean2: number, std2: number, n2: number,
  alpha: number = 0.05,
): boolean {
  // Welch's t-test
  const se = Math.sqrt((std1 ** 2 / n1) + (std2 ** 2 / n2));
  const t = Math.abs(mean1 - mean2) / se;

  // Simplified: for large N, t > 1.96 is significant at alpha=0.05
  return t > 1.96;
}
```

### 4.3 Scheduled Regression Checks

```python
# Python: Scheduled regression detection (runs as a cron job)

import asyncio
from datetime import datetime, timedelta

async def scheduled_regression_check():
    """Run every 6 hours to check for quality regressions."""
    features = ["customer_support", "code_review", "search", "summarization"]

    for feature in features:
        # Get baseline (7-day rolling average)
        baseline = await get_eval_scores(
            feature=feature,
            start=datetime.now() - timedelta(days=7),
            end=datetime.now() - timedelta(hours=6),
        )

        # Get recent (last 6 hours)
        recent = await get_eval_scores(
            feature=feature,
            start=datetime.now() - timedelta(hours=6),
            end=datetime.now(),
        )

        if len(recent) < 10:
            continue  # Not enough data

        baseline_mean = sum(baseline) / len(baseline)
        recent_mean = sum(recent) / len(recent)
        drop = (baseline_mean - recent_mean) / baseline_mean

        if drop > 0.05:  # 5% regression threshold
            await send_alert(
                channel="ai-quality-alerts",
                title=f"Quality regression: {feature}",
                message=(
                    f"Score dropped from {baseline_mean:.3f} to {recent_mean:.3f} "
                    f"({drop:.1%} drop) in the last 6 hours.\n"
                    f"Baseline: {len(baseline)} samples over 7 days\n"
                    f"Recent: {len(recent)} samples in last 6 hours"
                ),
                severity="warning" if drop < 0.10 else "critical",
            )

        # Log metrics for the dashboard
        await log_metric(f"eval.{feature}.score", recent_mean)
        await log_metric(f"eval.{feature}.baseline", baseline_mean)
        await log_metric(f"eval.{feature}.regression", drop)

# Schedule: run every 6 hours
# In production, use your scheduler (cron, AWS EventBridge, etc.)
```

---

## 5. Online Evals

### 5.1 Evaluating Live Traffic

Online evals score a sample of production requests in real time. Unlike CI evals (which use test datasets), online evals measure actual user interactions.

```typescript
// TypeScript: Online eval pipeline

interface OnlineEvalConfig {
  sampleRate: number;  // 0.0-1.0, fraction of requests to evaluate
  scorers: string[];
  asyncScoring: boolean;  // Don't block the response
}

const ONLINE_EVAL_CONFIG: Record<string, OnlineEvalConfig> = {
  customer_support: {
    sampleRate: 0.10,  // Evaluate 10% of requests
    scorers: ["relevance", "safety", "hallucination"],
    asyncScoring: true,
  },
  code_review: {
    sampleRate: 0.05,  // Evaluate 5% of requests
    scorers: ["relevance", "accuracy"],
    asyncScoring: true,
  },
};

async function onlineEvalMiddleware(
  feature: string,
  input: string,
  response: string,
  context: { userId: string; sessionId: string; model: string },
): Promise<void> {
  const config = ONLINE_EVAL_CONFIG[feature];
  if (!config) return;

  // Sample
  if (Math.random() > config.sampleRate) return;

  if (config.asyncScoring) {
    // Score asynchronously — don't add latency to user response
    setImmediate(async () => {
      try {
        await scoreAndRecord(feature, input, response, context, config.scorers);
      } catch (error) {
        console.error("Online eval error:", error);
      }
    });
  } else {
    await scoreAndRecord(feature, input, response, context, config.scorers);
  }
}

async function scoreAndRecord(
  feature: string,
  input: string,
  response: string,
  context: { userId: string; sessionId: string; model: string },
  scorerNames: string[],
): Promise<void> {
  const scores: Record<string, number> = {};

  for (const scorerName of scorerNames) {
    const scorer = getScorer(scorerName);
    scores[scorerName] = await scorer(input, response);
  }

  // Record to your metrics system
  for (const [metric, score] of Object.entries(scores)) {
    emitMetric(`eval.online.${feature}.${metric}`, score, {
      model: context.model,
      userId: context.userId,
    });
  }

  // Flag low-quality responses for human review
  const avgScore = Object.values(scores).reduce((a, b) => a + b, 0) / Object.values(scores).length;
  if (avgScore < 0.5) {
    await flagForReview({
      feature,
      input,
      response,
      scores,
      context,
      reason: `Low average score: ${avgScore.toFixed(2)}`,
    });
  }
}
```

### 5.2 LLM-as-Judge for Online Evals

For complex quality dimensions (helpfulness, coherence, completeness), use an LLM as a judge:

```python
# Python: LLM-as-judge online eval

import anthropic

client = anthropic.Anthropic()

async def judge_response_quality(
    user_input: str,
    ai_response: str,
    feature: str,
) -> dict:
    """Use a separate LLM call to judge response quality."""
    judge_prompt = f"""Evaluate this AI response on a scale of 1-5 for each dimension.

User's question: {user_input}

AI's response: {ai_response}

Rate each dimension (1=terrible, 5=excellent):
1. Relevance: Does the response address the user's actual question?
2. Accuracy: Are the factual claims correct (as far as you can tell)?
3. Helpfulness: Would this response actually help the user?
4. Completeness: Does it cover all aspects of the question?
5. Safety: Is the response appropriate and free of harmful content?

Respond with JSON:
{{
  "relevance": 1-5,
  "accuracy": 1-5,
  "helpfulness": 1-5,
  "completeness": 1-5,
  "safety": 1-5,
  "overall": 1-5,
  "issues": ["list any specific issues found"]
}}"""

    response = client.messages.create(
        model="claude-haiku-4-20250414",  # Cheap model for judging
        max_tokens=300,
        system="You are a quality evaluator. Be critical but fair. Respond only with JSON.",
        messages=[{"role": "user", "content": judge_prompt}],
    )

    import json
    scores = json.loads(response.content[0].text)

    # Normalize to 0-1 scale
    return {
        "relevance": scores["relevance"] / 5,
        "accuracy": scores["accuracy"] / 5,
        "helpfulness": scores["helpfulness"] / 5,
        "completeness": scores["completeness"] / 5,
        "safety": scores["safety"] / 5,
        "overall": scores["overall"] / 5,
        "issues": scores.get("issues", []),
    }
```

### 5.3 User Feedback Integration

The ultimate eval is user satisfaction. Collect explicit and implicit feedback:

```typescript
// TypeScript: User feedback collection

interface FeedbackSignals {
  explicit: {
    thumbsUp: boolean | null;  // Direct rating
    rating: number | null;  // 1-5 scale
    comment: string | null;
  };
  implicit: {
    followUpQuestion: boolean;  // Did user ask a follow-up? (maybe unsatisfied)
    sessionContinued: boolean;  // Did user stay or leave?
    taskCompleted: boolean;  // Did user complete their goal?
    responseEdited: boolean;  // Did user modify the AI's output?
    copiedResponse: boolean;  // Did user copy the response? (good sign)
  };
}

function calculateImplicitSatisfaction(signals: FeedbackSignals["implicit"]): number {
  let score = 0.5;  // Neutral starting point

  if (signals.copiedResponse) score += 0.2;
  if (signals.taskCompleted) score += 0.2;
  if (signals.sessionContinued) score += 0.05;
  if (signals.followUpQuestion) score -= 0.1;
  if (signals.responseEdited) score -= 0.15;

  return Math.max(0, Math.min(1, score));
}
```

---

## 6. The Eval Dashboard

### 6.1 What to Show

```
┌───────────────────────────────────────────────────────────────┐
│                    AI Quality Dashboard                        │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ Overall Quality Score: 87.3% (7-day avg)  ↑ 1.2% from last week │
│                                                               │
├──────────────┬────────────┬────────────┬──────────────────────┤
│ Feature      │ Score (7d) │ Trend      │ Status               │
├──────────────┼────────────┼────────────┼──────────────────────┤
│ Support      │ 89.1%      │ → stable   │ ✅ Healthy            │
│ Code Review  │ 84.7%      │ ↓ -2.3%    │ ⚠️ Watch              │
│ Search       │ 91.2%      │ ↑ +1.8%    │ ✅ Healthy            │
│ Summarize    │ 82.1%      │ ↓ -4.1%    │ ❌ Regression         │
├──────────────┴────────────┴────────────┴──────────────────────┤
│                                                               │
│ Active A/B Tests                                              │
│ ─────────────                                                 │
│ support-prompt-v3: Treatment +3.2% (p=0.03, n=1,247)  ✅ Sig │
│ code-review-haiku: Treatment -1.1% (p=0.42, n=389)    → More │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ Recent Regressions                                            │
│ ─────────────────                                             │
│ Apr 8: Summarize dropped 4.1% after model update              │
│        ↳ Root cause: Claude Sonnet 4 behavior change          │
│        ↳ Fix: Updated system prompt, score recovered          │
│                                                               │
│ Mar 29: Code Review dropped 6.2% after prompt change          │
│         ↳ Root cause: New prompt confused tool selection       │
│         ↳ Fix: Rolled back prompt, investigating              │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ Score Distribution (Last 24h)                                 │
│                                                               │
│   5│ ████████████████████████████████████████ 42%             │
│   4│ ██████████████████████████ 28%                           │
│   3│ ████████████████ 17%                                     │
│   2│ ████████ 9%                                              │
│   1│ ████ 4%                                                  │
│    └────────────────────────────────────────                  │
│                                                               │
│ Items Flagged for Human Review: 23 (last 24h)                │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 6.2 Building the Dashboard Data Layer

```typescript
// TypeScript: Dashboard data aggregation

interface DashboardData {
  overallScore: { current: number; previous: number; trend: number };
  features: Array<{
    name: string;
    score7d: number;
    trend7d: number;
    status: "healthy" | "watch" | "regression";
    scoreHistory: Array<{ date: string; score: number }>;
  }>;
  activeTests: Array<{
    id: string;
    effect: number;
    pValue: number;
    sampleSize: number;
    significant: boolean;
  }>;
  recentRegressions: Array<{
    date: string;
    feature: string;
    drop: number;
    rootCause: string;
    resolution: string;
  }>;
  flaggedForReview: number;
}

async function buildDashboardData(): Promise<DashboardData> {
  const features = ["customer_support", "code_review", "search", "summarization"];

  const featureData = await Promise.all(
    features.map(async (feature) => {
      const scores7d = await getEvalScores(feature, 7);
      const scores14d = await getEvalScores(feature, 14);

      const current = mean(scores7d);
      const previous = mean(scores14d.slice(0, scores14d.length / 2));
      const trend = current - previous;

      return {
        name: feature,
        score7d: current,
        trend7d: trend,
        status: (
          trend < -0.05 ? "regression" :
          trend < -0.02 ? "watch" :
          "healthy"
        ) as "healthy" | "watch" | "regression",
        scoreHistory: await getDailyScores(feature, 30),
      };
    })
  );

  const overallCurrent = mean(featureData.map((f) => f.score7d));
  const overallPrevious = mean(
    await Promise.all(features.map((f) => getEvalScores(f, 14).then((s) => mean(s.slice(0, s.length / 2)))))
  );

  return {
    overallScore: {
      current: overallCurrent,
      previous: overallPrevious,
      trend: overallCurrent - overallPrevious,
    },
    features: featureData,
    activeTests: await getActiveABTests(),
    recentRegressions: await getRecentRegressions(30),
    flaggedForReview: await getFlaggedCount(1),
  };
}
```

---

## 7. Alerting on Eval Degradation

### 7.1 Alert Configuration

```typescript
// TypeScript: Eval alerting system

interface AlertRule {
  name: string;
  feature: string;
  metric: string;
  condition: "below_threshold" | "regression" | "anomaly";
  threshold?: number;
  regressionPercent?: number;
  severity: "info" | "warning" | "critical";
  channels: string[];  // Slack channels, PagerDuty, email
  cooldownMinutes: number;  // Don't re-alert within this window
}

const ALERT_RULES: AlertRule[] = [
  {
    name: "support_quality_critical",
    feature: "customer_support",
    metric: "overall_score",
    condition: "below_threshold",
    threshold: 0.70,
    severity: "critical",
    channels: ["#ai-incidents", "pagerduty"],
    cooldownMinutes: 30,
  },
  {
    name: "support_quality_warning",
    feature: "customer_support",
    metric: "overall_score",
    condition: "regression",
    regressionPercent: 5,
    severity: "warning",
    channels: ["#ai-quality"],
    cooldownMinutes: 120,
  },
  {
    name: "safety_score_any_feature",
    feature: "*",  // All features
    metric: "safety",
    condition: "below_threshold",
    threshold: 0.90,
    severity: "critical",
    channels: ["#ai-incidents", "pagerduty", "#security"],
    cooldownMinutes: 15,
  },
  {
    name: "hallucination_spike",
    feature: "*",
    metric: "hallucination_rate",
    condition: "anomaly",
    severity: "warning",
    channels: ["#ai-quality"],
    cooldownMinutes: 60,
  },
];

async function evaluateAlertRules(
  feature: string,
  metrics: Record<string, number>,
): Promise<void> {
  for (const rule of ALERT_RULES) {
    if (rule.feature !== "*" && rule.feature !== feature) continue;

    const metricValue = metrics[rule.metric];
    if (metricValue === undefined) continue;

    let triggered = false;
    let message = "";

    switch (rule.condition) {
      case "below_threshold":
        if (metricValue < rule.threshold!) {
          triggered = true;
          message = `${feature}/${rule.metric} is ${(metricValue * 100).toFixed(1)}% ` +
            `(below threshold of ${(rule.threshold! * 100).toFixed(1)}%)`;
        }
        break;

      case "regression":
        const baseline = await getBaselineScore(feature, rule.metric);
        const dropPercent = ((baseline - metricValue) / baseline) * 100;
        if (dropPercent > rule.regressionPercent!) {
          triggered = true;
          message = `${feature}/${rule.metric} dropped ${dropPercent.toFixed(1)}% ` +
            `from baseline (${(baseline * 100).toFixed(1)}% → ${(metricValue * 100).toFixed(1)}%)`;
        }
        break;

      case "anomaly":
        const anomaly = await detectAnomaly(feature, rule.metric, metricValue);
        if (anomaly.isAnomaly) {
          triggered = true;
          message = anomaly.message;
        }
        break;
    }

    if (triggered && !(await isInCooldown(rule.name, rule.cooldownMinutes))) {
      await sendAlert(rule.severity, rule.name, message, rule.channels);
      await setCooldown(rule.name, rule.cooldownMinutes);
    }
  }
}
```

---

## 8. Key Takeaways

Production eval pipelines transform evals from a manual quality check into automated infrastructure that continuously validates your AI features.

**The complete eval infrastructure:**

1. **CI evals** — run on every PR that touches AI code. Fail the build if scores drop. Takes ~5-15 minutes.

2. **Scheduled evals** — run your full eval suite nightly against production. Detect gradual drift and model changes.

3. **Online evals** — score a sample of live traffic in real time. Catch issues that test datasets miss.

4. **A/B tests** — compare prompt/model changes with statistical rigor. Don't ship changes based on vibes.

5. **Regression detection** — automated comparison against rolling baselines. Alert before users notice.

6. **Dashboard** — quality scores over time, by feature, by model. The single source of truth for AI quality.

7. **Alerting** — graduated alerts (info, warning, critical) with cooldowns. Critical safety issues page on-call.

The investment in eval infrastructure pays for itself the first time it catches a regression before your users do. And it keeps paying every time it prevents a bad prompt change from shipping.

> **Spiral note:** This chapter brings the eval concepts from Ch 18-21 to production maturity. Ch 56 (CI/CD for AI) will cover the broader deployment pipeline that evals plug into. Ch 61 (Self-Reinforcing AI Systems) will take this further — using eval results to automatically improve the system.

> **Next chapter:** When evals detect a problem, you need to find and fix it. Ch 52 covers AI observability and incident response — the monitoring, debugging, and incident management patterns specific to LLM-powered systems.

---

*Previous: [Advanced Context Strategies](./50-advanced-context.md)* | *Next: [AI Observability & Incidents](./52-observability-incidents.md)*

<!--
  CHAPTER: 44
  TITLE: RAG vs Fine-Tuning
  PART: 9 — Fine-Tuning & Training
  PHASE: 2 — Become an Expert
  PREREQS: Ch 15 (RAG Pipeline), Ch 21 (Eval-Driven Dev), Ch 43 (Model Selection)
  KEY_TOPICS: RAG, fine-tuning, knowledge injection, cost comparison, latency, decision framework, hybrid approaches, when to fine-tune
  DIFFICULTY: Inter→Adv
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 44: RAG vs Fine-Tuning

> **Part 9 — Fine-Tuning & Training** | Phase 2: Become an Expert | Prerequisites: Ch 15, Ch 21, Ch 43 | Difficulty: Inter-Adv | Language: Python

You have two ways to give an LLM knowledge it does not have out of the box. In Chapter 15, you built RAG pipelines — retrieve relevant documents at query time and stuff them into the context. In Chapter 43, you learned about model architectures and sizes. Now you face the fundamental question: should you retrieve knowledge at query time (RAG) or bake it into the model's weights (fine-tuning)?

This is not a theoretical question. It has real consequences for your costs, latency, accuracy, and maintenance burden. Teams that choose wrong waste months building infrastructure they do not need or, worse, ship a product that does not work well enough because they chose the approach that was easier rather than the one that fits the problem.

This chapter gives you a clear framework for deciding. By the end, you will know when RAG wins, when fine-tuning wins, when you need both, and how to measure whether your choice was correct.

### In This Chapter
- The fundamental question: retrieve at query time or train into weights?
- When RAG wins: changing data, source attribution, large knowledge bases
- When fine-tuning wins: style changes, domain language, specific task behavior
- When to use both: the hybrid approach
- Cost, quality, and latency trade-offs with real numbers
- The decision framework as code
- Common mistakes and how to avoid them

### Related Chapters
- **Ch 15 (RAG Pipeline)** — you built the retrieval approach; now you evaluate it
- **Ch 21 (Eval-Driven Dev)** — use evals to determine which approach wins
- **Ch 43 (Model Selection)** — pick the right base model for either approach
- **Ch 45 (Fine-Tuning with LoRA)** — if you decide to fine-tune, here is how

---

## 1. The Fundamental Question

### 1.1 Two Approaches to Knowledge

```
RAG (Retrieval-Augmented Generation):
  ┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
  │  Query   │───→│  Search  │───→│  Context  │───→│  Model   │───→ Answer
  └──────────┘    │  vector  │    │  (query + │    │  (base   │
                  │  store   │    │  documents)│    │  model)  │
                  └──────────┘    └───────────┘    └──────────┘
  
  Knowledge lives OUTSIDE the model, in a searchable document store.
  At query time, you find relevant docs and include them in the prompt.

Fine-Tuning:
  ┌──────────┐    ┌───────────────────┐
  │  Query   │───→│  Fine-tuned model │───→ Answer
  └──────────┘    │  (knowledge baked │
                  │   into weights)   │
                  └───────────────────┘
  
  Knowledge lives INSIDE the model's weights.
  The model "knows" your domain because it was trained on your data.
```

### 1.2 What Each Approach Actually Does

```python
# Let's make this concrete with an example

# Scenario: You're building a customer support bot for your SaaS product.
# The bot needs to answer questions about your product's features, pricing,
# and troubleshooting steps.

# --- RAG APPROACH ---
# You embed your documentation, pricing page, and support articles.
# At query time, you retrieve relevant docs and include them in the prompt.

rag_example = {
    "query": "How do I export my data?",
    "retrieved_docs": [
        "To export data, go to Settings > Data > Export. Select CSV or JSON format...",
        "Data exports are available on Pro and Enterprise plans...",
    ],
    "prompt": """Based on the following documentation, answer the user's question.

Documentation:
{docs}

Question: How do I export my data?""",
    "answer": "Based on the documentation, you can export your data by going to Settings > Data > Export...",
}

# --- FINE-TUNING APPROACH ---
# You fine-tune a model on thousands of support conversations.
# The model learns your product's terminology, tone, and common answers.

finetuning_example = {
    "training_data": [
        {"prompt": "How do I export my data?", 
         "response": "To export data, go to Settings > Data > Export. Select CSV or JSON format..."},
        {"prompt": "What plans include data export?",
         "response": "Data export is available on Pro and Enterprise plans..."},
        # ... thousands more examples
    ],
    "query": "How do I export my data?",
    "answer": "To export your data, navigate to Settings > Data > Export...",
    # No retrieval needed — the model "knows" this
}

print("RAG: Knowledge retrieved at query time from external documents")
print("Fine-Tuning: Knowledge learned during training, stored in weights")
print()
print("Same question, different mechanisms, potentially different answers.")
```

---

## 2. When RAG Wins

### 2.1 The RAG Advantages

RAG is the right choice more often than most people think. Here are the scenarios where it clearly wins:

```python
rag_wins = {
    "Frequently changing data": {
        "example": "Product docs updated weekly, pricing changes quarterly",
        "why_rag": "Update the document store in minutes. Fine-tuning takes hours/days.",
        "why_not_ft": "You'd need to re-fine-tune every time data changes.",
    },
    "Need source attribution": {
        "example": "Legal compliance requiring citing specific documents",
        "why_rag": "You know exactly which documents were used. Include citations.",
        "why_not_ft": "Fine-tuned models can't tell you WHERE they learned something.",
    },
    "Large knowledge base": {
        "example": "10,000+ support articles, entire product documentation",
        "why_rag": "Embed and index everything. Retrieve only what's relevant.",
        "why_not_ft": "Can't fit 10K articles into a fine-tuning dataset effectively.",
    },
    "Factual accuracy is critical": {
        "example": "Medical information, financial data, legal advice",
        "why_rag": "Ground responses in source documents. Reduce hallucination.",
        "why_not_ft": "Fine-tuned models still hallucinate — just more confidently.",
    },
    "Multiple domains / multi-tenant": {
        "example": "Each customer has their own knowledge base",
        "why_rag": "Each customer gets their own document store. One model serves all.",
        "why_not_ft": "You'd need a separate fine-tuned model per customer.",
    },
    "Budget constraints": {
        "example": "Startup with limited compute budget",
        "why_rag": "Embedding docs is cheap. No GPU training needed.",
        "why_not_ft": "Fine-tuning requires GPU hours and expertise.",
    },
}

print("When RAG Wins:")
print("=" * 70)
for scenario, details in rag_wins.items():
    print(f"\n  {scenario}")
    print(f"    Example:  {details['example']}")
    print(f"    Why RAG:  {details['why_rag']}")
    print(f"    Why not FT: {details['why_not_ft']}")
```

### 2.2 RAG Quality Patterns

```python
# The quality of RAG depends on retrieval quality
# Here's how to diagnose RAG problems

rag_failure_modes = {
    "Retrieval misses relevant docs": {
        "symptom": "Model says 'I don't have information about that' for questions you know are covered",
        "fix": "Improve embeddings, add query expansion, try hybrid search (Ch 17)",
    },
    "Retrieved docs are relevant but model ignores them": {
        "symptom": "Model hallucinates despite having correct docs in context",
        "fix": "Improve prompt template, put key info at start/end (not middle)",
    },
    "Too many retrieved docs overwhelm the context": {
        "symptom": "Model's answers are vague or confused",
        "fix": "Reduce top-k, improve chunking, add reranking",
    },
    "Docs are relevant but answer requires synthesis": {
        "symptom": "Answer needs to combine info from multiple docs — model fails",
        "fix": "Consider fine-tuning for synthesis tasks, or use chain-of-thought",
    },
}

print("RAG Failure Modes and Fixes:")
print("-" * 60)
for mode, details in rag_failure_modes.items():
    print(f"\n  Problem: {mode}")
    print(f"  Symptom: {details['symptom']}")
    print(f"  Fix:     {details['fix']}")
```

---

## 3. When Fine-Tuning Wins

### 3.1 The Fine-Tuning Advantages

Fine-tuning wins when you need to change the model's behavior, not just its knowledge.

```python
finetuning_wins = {
    "Style and tone changes": {
        "example": "Your brand has a specific voice — casual, technical, formal",
        "why_ft": "Fine-tune on examples of your desired tone. Model adopts it naturally.",
        "why_not_rag": "RAG gives the model info, but can't reliably change how it talks.",
    },
    "Domain-specific language": {
        "example": "Medical terminology, legal jargon, proprietary acronyms",
        "why_ft": "Model learns the vocabulary and how to use it correctly.",
        "why_not_rag": "RAG retrieves docs but model may still misuse domain terms.",
    },
    "Consistent output format": {
        "example": "Always respond in a specific JSON structure, specific report format",
        "why_ft": "Fine-tune on examples of the exact format. Near-perfect consistency.",
        "why_not_rag": "RAG doesn't help with formatting — that's a behavioral change.",
    },
    "Smaller model for specific task": {
        "example": "Need GPT-4 quality but at GPT-3.5 cost for one specific task",
        "why_ft": "Fine-tune a small model on high-quality examples for that specific task.",
        "why_not_rag": "RAG can't make a small model smarter — it just gives it more context.",
    },
    "Latency-sensitive applications": {
        "example": "Real-time code completion, inline suggestions",
        "why_ft": "No retrieval step needed. Single model call, lower latency.",
        "why_not_rag": "RAG adds retrieval latency (embedding + search + larger context).",
    },
    "Reducing prompt length": {
        "example": "Currently using 2000-token system prompts with examples and rules",
        "why_ft": "Bake those examples and rules into weights. Shorter prompts = cheaper.",
        "why_not_rag": "RAG adds tokens (retrieved docs). Fine-tuning removes them.",
    },
}

print("When Fine-Tuning Wins:")
print("=" * 70)
for scenario, details in finetuning_wins.items():
    print(f"\n  {scenario}")
    print(f"    Example:  {details['example']}")
    print(f"    Why FT:   {details['why_ft']}")
    print(f"    Why not RAG: {details['why_not_rag']}")
```

### 3.2 Fine-Tuning Quality Patterns

```python
# Fine-tuning is powerful but has failure modes too

ft_failure_modes = {
    "Too little training data": {
        "symptom": "Model overfits — repeats training examples verbatim",
        "minimum": "At least 100-500 high-quality examples, ideally 1000+",
        "fix": "Augment with synthetic data (Ch 46), or use RAG instead",
    },
    "Low-quality training data": {
        "symptom": "Model learns bad habits from noisy/incorrect examples",
        "fix": "Quality over quantity — clean your data ruthlessly (Ch 46)",
    },
    "Catastrophic forgetting": {
        "symptom": "Model gets good at your task but forgets how to do general things",
        "fix": "Use LoRA (Ch 45) — only modifies a small fraction of weights",
    },
    "Knowledge becomes stale": {
        "symptom": "Fine-tuned model gives outdated answers",
        "fix": "Add RAG for factual knowledge, use fine-tuning only for behavior",
    },
}

print("Fine-Tuning Failure Modes:")
print("-" * 60)
for mode, details in ft_failure_modes.items():
    print(f"\n  Problem: {mode}")
    print(f"  Symptom: {details['symptom']}")
    print(f"  Fix:     {details['fix']}")
```

---

## 4. When to Use Both: The Hybrid Approach

### 4.1 Fine-Tune the Behavior, RAG the Knowledge

The most powerful approach is often to combine both: fine-tune the model for behavior (style, format, domain language) and use RAG for factual knowledge.

```python
# The hybrid architecture

hybrid_architecture = """
┌──────────┐    ┌──────────┐    ┌───────────────┐    ┌──────────────────┐
│  Query   │───→│  Search  │───→│   Context     │───→│  FINE-TUNED      │───→ Answer
└──────────┘    │  vector  │    │  (query +     │    │  model           │
                │  store   │    │   documents)  │    │  (knows HOW to   │
                └──────────┘    └───────────────┘    │   answer in your │
                                                     │   style/format)  │
                                                     └──────────────────┘

Fine-tuning handles: style, tone, format, domain vocabulary, task behavior
RAG handles: factual knowledge, current information, source attribution
"""
print(hybrid_architecture)

# Example: medical support chatbot
hybrid_example = {
    "fine_tuned_for": [
        "Medical terminology usage",
        "Empathetic, professional tone",
        "Always including safety disclaimers",
        "Structured response format (Assessment, Recommendation, References)",
    ],
    "rag_for": [
        "Current treatment guidelines (updated quarterly)",
        "Drug interaction databases",
        "Patient-specific medical records",
        "Latest research papers",
    ],
}

print("Example: Medical Support Chatbot")
print("-" * 50)
print("\nFine-tuned for (behavior):")
for item in hybrid_example["fine_tuned_for"]:
    print(f"  - {item}")
print("\nRAG for (knowledge):")
for item in hybrid_example["rag_for"]:
    print(f"  - {item}")
```

### 4.2 Implementation Pattern

```python
# Pseudo-code for a hybrid RAG + fine-tuned model system

from dataclasses import dataclass
from typing import Optional


@dataclass
class HybridConfig:
    """Configuration for a hybrid RAG + fine-tuned system."""
    # Fine-tuned model settings
    model_name: str = "your-org/your-fine-tuned-model"
    max_tokens: int = 500
    temperature: float = 0.3
    
    # RAG settings
    embedding_model: str = "BAAI/bge-base-en-v1.5"
    top_k: int = 5
    min_relevance_score: float = 0.7
    
    # System prompt (shorter because behavior is in the weights)
    system_prompt: str = "Answer the user's question using the provided context."


class HybridAssistant:
    """
    Combines a fine-tuned model (for behavior) with RAG (for knowledge).
    
    The fine-tuned model knows HOW to answer:
    - In the right style and tone
    - Using domain-specific vocabulary
    - In the expected output format
    
    RAG provides WHAT to answer:
    - Current, factual information
    - Cited sources
    - Context-specific details
    """
    
    def __init__(self, config: HybridConfig):
        self.config = config
        # In practice, load your fine-tuned model and embedding model here
        # self.model = load_fine_tuned_model(config.model_name)
        # self.retriever = load_retriever(config.embedding_model)
        print(f"Hybrid assistant initialized")
        print(f"  Model: {config.model_name}")
        print(f"  Embeddings: {config.embedding_model}")
    
    def answer(self, query: str) -> dict:
        """Answer a query using the hybrid approach."""
        
        # Step 1: Retrieve relevant documents
        # retrieved_docs = self.retriever.search(query, top_k=self.config.top_k)
        
        # Step 2: Filter by relevance
        # relevant_docs = [d for d in retrieved_docs if d.score >= self.config.min_relevance_score]
        
        # Step 3: Build context (shorter prompt because model is fine-tuned)
        # context = "\n".join([d.text for d in relevant_docs])
        
        # Step 4: Generate with fine-tuned model
        # response = self.model.generate(
        #     system=self.config.system_prompt,
        #     context=context,
        #     query=query,
        #     max_tokens=self.config.max_tokens,
        #     temperature=self.config.temperature,
        # )
        
        # Step 5: Include sources
        # return {
        #     "answer": response,
        #     "sources": [d.metadata for d in relevant_docs],
        #     "confidence": max(d.score for d in relevant_docs) if relevant_docs else 0,
        # }
        
        return {"status": "pseudo-code — implement with your model and retriever"}


config = HybridConfig(
    model_name="your-org/support-bot-v2-lora",
    system_prompt="Answer using the provided context. Be concise and cite sources.",
)
assistant = HybridAssistant(config)
print("\nHybrid system ready!")
print("Fine-tuned model handles: style, tone, format")
print("RAG handles: factual content, citations")
```

---

## 5. Cost, Quality, and Latency Trade-Offs

### 5.1 The Numbers

```python
# Real cost comparison for a typical scenario

# Scenario: Customer support bot handling 10,000 queries/day
queries_per_day = 10_000
queries_per_month = queries_per_day * 30

# Average tokens per query
avg_input_tokens = 500   # Query + system prompt + (optional) retrieved docs
avg_output_tokens = 200  # Response

# ==========================================
# Option 1: Cloud API with RAG
# ==========================================
print("=" * 60)
print("Option 1: Cloud API (GPT-4o-mini) + RAG")
print("=" * 60)

# RAG adds ~1000 tokens of retrieved context
rag_input_tokens = avg_input_tokens + 1000  # Retrieved docs

api_input_cost = 0.15 / 1_000_000  # GPT-4o-mini input
api_output_cost = 0.60 / 1_000_000  # GPT-4o-mini output

monthly_api_cost = queries_per_month * (
    rag_input_tokens * api_input_cost + 
    avg_output_tokens * api_output_cost
)

# Vector database cost
vector_db_cost = 25  # Pinecone starter plan or similar
embedding_cost = 5   # Re-embedding new docs monthly

total_rag_api = monthly_api_cost + vector_db_cost + embedding_cost

print(f"  API inference cost: ${monthly_api_cost:.2f}/month")
print(f"  Vector DB cost:     ${vector_db_cost:.2f}/month")
print(f"  Embedding cost:     ${embedding_cost:.2f}/month")
print(f"  Total:              ${total_rag_api:.2f}/month")
print(f"  Latency:            ~800ms (embedding + search + generation)")
print(f"  Setup time:         ~1 week")

# ==========================================
# Option 2: Fine-tuned cloud model
# ==========================================
print(f"\n{'=' * 60}")
print("Option 2: Fine-tuned GPT-4o-mini")
print("=" * 60)

# Fine-tuned model: shorter prompts (no RAG context), but higher per-token cost
ft_input_tokens = avg_input_tokens  # No retrieved docs needed
ft_api_input_cost = 0.30 / 1_000_000   # Fine-tuned models cost ~2x
ft_api_output_cost = 1.20 / 1_000_000

monthly_ft_api = queries_per_month * (
    ft_input_tokens * ft_api_input_cost + 
    avg_output_tokens * ft_api_output_cost
)

training_cost = 25  # One-time, amortized monthly
total_ft_api = monthly_ft_api + training_cost

print(f"  API inference cost: ${monthly_ft_api:.2f}/month")
print(f"  Training cost:      ${training_cost:.2f}/month (amortized)")
print(f"  Total:              ${total_ft_api:.2f}/month")
print(f"  Latency:            ~400ms (no retrieval step)")
print(f"  Setup time:         ~2-3 weeks (data + training + eval)")

# ==========================================
# Option 3: Self-hosted fine-tuned model
# ==========================================
print(f"\n{'=' * 60}")
print("Option 3: Self-hosted fine-tuned 7B model")
print("=" * 60)

gpu_cost_per_hour = 0.50  # T4 instance
hours_per_month = 24 * 30  # Always-on for this volume

monthly_gpu = gpu_cost_per_hour * hours_per_month
training_compute = 10  # Monthly re-training cost

total_self_hosted = monthly_gpu + training_compute

print(f"  GPU cost:           ${monthly_gpu:.2f}/month (always-on T4)")
print(f"  Training cost:      ${training_compute:.2f}/month")
print(f"  Total:              ${total_self_hosted:.2f}/month")
print(f"  Latency:            ~200ms (no network hop to API)")
print(f"  Setup time:         ~3-4 weeks (infra + training + deployment)")

# ==========================================
# Option 4: Hybrid (self-hosted FT + local RAG)
# ==========================================
print(f"\n{'=' * 60}")
print("Option 4: Hybrid (self-hosted FT model + local RAG)")
print("=" * 60)

total_hybrid = monthly_gpu + training_compute + 10  # +$10 for vector DB

print(f"  GPU cost:           ${monthly_gpu:.2f}/month")
print(f"  Training + DB:      $20.00/month")
print(f"  Total:              ${total_hybrid:.2f}/month")
print(f"  Latency:            ~300ms (local retrieval + local generation)")
print(f"  Setup time:         ~4-5 weeks")

# ==========================================
# Summary
# ==========================================
print(f"\n{'=' * 60}")
print("SUMMARY (10K queries/day)")
print("=" * 60)
print(f"  Cloud API + RAG:       ${total_rag_api:>8.2f}/month  |  ~800ms  |  1 week setup")
print(f"  Cloud Fine-tuned:      ${total_ft_api:>8.2f}/month  |  ~400ms  |  2-3 weeks setup")
print(f"  Self-hosted FT:        ${total_self_hosted:>8.2f}/month  |  ~200ms  |  3-4 weeks setup")
print(f"  Hybrid (self-hosted):  ${total_hybrid:>8.2f}/month  |  ~300ms  |  4-5 weeks setup")
```

### 5.2 Quality Comparison

```python
# Quality comparison across different scenarios

quality_comparison = {
    "Factual accuracy": {
        "RAG": "High — grounded in source documents",
        "Fine-tuning": "Moderate — can hallucinate confidently",
        "Hybrid": "Highest — grounded + well-formatted",
        "winner": "RAG or Hybrid",
    },
    "Response style/tone": {
        "RAG": "Base model's style (generic)",
        "Fine-tuning": "Your trained style (specific)",
        "Hybrid": "Your trained style (specific)",
        "winner": "Fine-tuning or Hybrid",
    },
    "Handling novel questions": {
        "RAG": "Good — if relevant docs exist",
        "Fine-tuning": "Moderate — may not generalize",
        "Hybrid": "Best — retrieves + adapts",
        "winner": "Hybrid",
    },
    "Output format consistency": {
        "RAG": "Inconsistent (depends on prompt)",
        "Fine-tuning": "Very consistent (learned)",
        "Hybrid": "Very consistent",
        "winner": "Fine-tuning or Hybrid",
    },
    "Keeping up with changes": {
        "RAG": "Easy — update document store",
        "Fine-tuning": "Hard — retrain required",
        "Hybrid": "Easy for knowledge, hard for behavior",
        "winner": "RAG",
    },
}

print("Quality Comparison:")
print("=" * 70)
for dimension, comparison in quality_comparison.items():
    print(f"\n  {dimension}:")
    print(f"    RAG:         {comparison['RAG']}")
    print(f"    Fine-tuning: {comparison['Fine-tuning']}")
    print(f"    Hybrid:      {comparison['Hybrid']}")
    print(f"    Winner:      {comparison['winner']}")
```

---

## 6. The Decision Framework

### 6.1 The Flowchart in Code

```python
def should_rag_or_finetune(
    data_changes_frequently: bool,
    need_source_attribution: bool,
    knowledge_base_size: str,       # "small" (<100 docs), "medium", "large" (>10K docs)
    need_style_change: bool,
    need_consistent_format: bool,
    domain_specific_language: bool,
    latency_sensitive: bool,
    budget: str,                    # "low", "medium", "high"
    team_ml_expertise: bool,
) -> str:
    """
    Decision framework for RAG vs Fine-Tuning.
    Returns recommendation with reasoning.
    """
    
    reasons_rag = []
    reasons_ft = []
    
    if data_changes_frequently:
        reasons_rag.append("Data changes frequently — RAG updates instantly")
    
    if need_source_attribution:
        reasons_rag.append("Need source attribution — RAG provides document references")
    
    if knowledge_base_size == "large":
        reasons_rag.append("Large knowledge base — RAG scales to millions of docs")
    
    if need_style_change:
        reasons_ft.append("Need style/tone change — fine-tuning bakes it in")
    
    if need_consistent_format:
        reasons_ft.append("Need consistent output format — fine-tuning learns format")
    
    if domain_specific_language:
        reasons_ft.append("Domain-specific language — fine-tuning learns vocabulary")
    
    if latency_sensitive:
        reasons_ft.append("Latency-sensitive — fine-tuning avoids retrieval overhead")
    
    if budget == "low":
        reasons_rag.append("Low budget — RAG requires no training compute")
    
    if not team_ml_expertise:
        reasons_rag.append("Limited ML expertise — RAG is simpler to implement")
    
    # Score
    rag_score = len(reasons_rag)
    ft_score = len(reasons_ft)
    
    if rag_score > ft_score + 1:
        recommendation = "RAG"
    elif ft_score > rag_score + 1:
        recommendation = "FINE-TUNING"
    else:
        recommendation = "HYBRID (both)"
    
    return {
        "recommendation": recommendation,
        "rag_score": rag_score,
        "ft_score": ft_score,
        "reasons_for_rag": reasons_rag,
        "reasons_for_ft": reasons_ft,
    }


# Test scenarios
scenarios = [
    {
        "name": "Customer support bot",
        "params": dict(
            data_changes_frequently=True,
            need_source_attribution=True,
            knowledge_base_size="large",
            need_style_change=True,
            need_consistent_format=True,
            domain_specific_language=False,
            latency_sensitive=False,
            budget="medium",
            team_ml_expertise=True,
        )
    },
    {
        "name": "Code review assistant",
        "params": dict(
            data_changes_frequently=False,
            need_source_attribution=False,
            knowledge_base_size="small",
            need_style_change=True,
            need_consistent_format=True,
            domain_specific_language=True,
            latency_sensitive=True,
            budget="high",
            team_ml_expertise=True,
        )
    },
    {
        "name": "Legal document search",
        "params": dict(
            data_changes_frequently=True,
            need_source_attribution=True,
            knowledge_base_size="large",
            need_style_change=False,
            need_consistent_format=False,
            domain_specific_language=True,
            latency_sensitive=False,
            budget="low",
            team_ml_expertise=False,
        )
    },
]

print("Decision Framework Results:")
print("=" * 60)
for scenario in scenarios:
    result = should_rag_or_finetune(**scenario["params"])
    print(f"\n  Scenario: {scenario['name']}")
    print(f"  Recommendation: {result['recommendation']} "
          f"(RAG: {result['rag_score']}, FT: {result['ft_score']})")
    if result["reasons_for_rag"]:
        print(f"  RAG reasons:")
        for r in result["reasons_for_rag"]:
            print(f"    + {r}")
    if result["reasons_for_ft"]:
        print(f"  FT reasons:")
        for r in result["reasons_for_ft"]:
            print(f"    + {r}")
```

### 6.2 The Quick Reference

```
DEFINITELY RAG:
  - Data changes weekly or more
  - Need to cite specific sources
  - Knowledge base > 10K documents
  - Low budget / no ML expertise
  - Multiple tenants with different knowledge

DEFINITELY FINE-TUNING:
  - Need specific style/tone/personality
  - Need consistent output format
  - Domain-specific jargon that base model doesn't know
  - Want to shrink a big model's ability into a small model
  - Latency is critical and you can't afford retrieval overhead

PROBABLY HYBRID:
  - Need both factual accuracy AND style consistency
  - Building a domain-specific assistant
  - Complex tasks requiring both knowledge and behavior
  - You have the engineering resources for both
```

---

## 7. Common Mistakes

### 7.1 Mistakes People Make

```python
common_mistakes = {
    "Fine-tuning to teach facts": {
        "what_they_do": "Fine-tune on a Q&A dataset of company facts",
        "why_it_fails": "Model memorizes training examples but hallucinates on new questions",
        "what_to_do": "Use RAG for facts. Fine-tune for behavior only.",
    },
    "RAG for style transfer": {
        "what_they_do": "Retrieve examples of desired writing style and include in prompt",
        "why_it_fails": "Style examples eat context window space. Model follows style inconsistently.",
        "what_to_do": "Fine-tune for style. Even 100 examples of your desired style works.",
    },
    "Skipping evaluation": {
        "what_they_do": "Pick RAG or fine-tuning based on blog posts, not data",
        "why_it_fails": "No way to know if your choice is correct without measuring",
        "what_to_do": "Build an eval set first (Ch 21). Test both approaches. Let data decide.",
    },
    "Fine-tuning a huge model": {
        "what_they_do": "Fine-tune Llama 70B when 7B would suffice",
        "why_it_fails": "10x cost, 10x slower, marginal quality improvement for the specific task",
        "what_to_do": "Start with the smallest model. Scale up only if eval says you need to.",
    },
    "Ignoring the hybrid option": {
        "what_they_do": "Treat it as RAG vs fine-tuning (either/or)",
        "why_it_fails": "Many tasks need both factual grounding AND behavioral adaptation",
        "what_to_do": "Consider hybrid first for complex tasks.",
    },
}

print("Common Mistakes:")
print("=" * 60)
for mistake, details in common_mistakes.items():
    print(f"\n  Mistake: {mistake}")
    print(f"  What they do:    {details['what_they_do']}")
    print(f"  Why it fails:    {details['why_it_fails']}")
    print(f"  What to do:      {details['what_to_do']}")
```

### 7.2 The Eval-First Approach

```python
# The right process: eval-driven decision making

evaluation_process = """
Step 1: Build your evaluation set FIRST
  - 50-100 question/answer pairs for your specific task
  - Include easy cases, hard cases, and edge cases
  - Have humans grade expected answers

Step 2: Baseline — test with base model + good prompting
  - Before any RAG or fine-tuning, how well does a good prompt work?
  - This is your baseline. Everything else must beat it.

Step 3: Test RAG
  - Build a simple RAG pipeline
  - Run your eval set
  - Measure: accuracy, relevance, latency, cost

Step 4: Test Fine-Tuning (if you have training data)
  - Fine-tune on your domain data
  - Run your eval set
  - Measure: accuracy, style match, latency, cost

Step 5: Test Hybrid (if both show promise)
  - Combine fine-tuned model with RAG
  - Run your eval set
  - Measure everything

Step 6: Decide based on numbers
  - Which approach wins on YOUR eval set?
  - Factor in cost, latency, maintenance burden
  - Choose. Ship. Iterate.
"""
print(evaluation_process)
```

---

## 8. Key Takeaways

1. **RAG for knowledge, fine-tuning for behavior.** This is the single most important principle. RAG teaches the model what to say. Fine-tuning teaches it how to say it.

2. **Start with RAG.** It is faster to implement, cheaper to run, easier to update, and good enough for most use cases. Only add fine-tuning when you have specific behavioral requirements that RAG cannot meet.

3. **Hybrid is often the answer.** For complex applications (domain-specific assistants, specialized support bots), combine fine-tuning for behavior with RAG for knowledge.

4. **Eval before deciding.** Build your evaluation set first. Test multiple approaches. Let the numbers decide. Never choose based on intuition or blog posts.

5. **Fine-tuning does not fix hallucination.** A fine-tuned model still hallucinates — it just does it more confidently and in your brand voice. If factual accuracy matters, you need RAG.

6. **The cost calculation is nuanced.** API + RAG wins at low volume. Self-hosted fine-tuned wins at high volume. The break-even depends on your specific numbers.

---

## What's Next

You have decided that fine-tuning is the right approach (or part of the right approach) for your task. In **Chapter 45: Fine-Tuning with LoRA**, you will do it — hands-on, with real code, fine-tuning a model on custom data using parameter-efficient techniques that run on a single GPU.

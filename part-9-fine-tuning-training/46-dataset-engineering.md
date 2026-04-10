<!--
  CHAPTER: 46
  TITLE: Dataset Engineering
  PART: 9 — Fine-Tuning & Training
  PHASE: 2 — Become an Expert
  PREREQS: Ch 18 (Why Evals Matter), Ch 45 (Fine-Tuning with LoRA)
  KEY_TOPICS: dataset engineering, training data, data quality, instruction format, conversational format, synthetic data, data augmentation, annotation, Hugging Face datasets, Argilla, overfitting, data leakage
  DIFFICULTY: Advanced
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 46: Dataset Engineering

> **Part 9 — Fine-Tuning & Training** | Phase 2: Become an Expert | Prerequisites: Ch 18, Ch 45 | Difficulty: Advanced | Language: Python

In Chapter 45, you fine-tuned a model with LoRA. It worked because the training data was clean, well-formatted, and appropriate for the task. In practice, getting the data right is the hardest part of fine-tuning — and the part that determines whether your model is good or useless.

The AI industry has a saying: "garbage in, garbage out." For fine-tuning, this is even more true than usual. A model fine-tuned on 500 high-quality examples will outperform one fine-tuned on 5,000 noisy examples. Data quality is more important than data quantity, more important than model size, and more important than hyperparameter tuning. It is the single highest-leverage variable in the entire fine-tuning process.

This chapter covers the full lifecycle of training data: formats, collection strategies, quality filtering, synthetic data generation, augmentation, annotation tools, and common pitfalls. By the end, you will have a systematic process for building datasets that make fine-tuning work.

### In This Chapter
- Why data quality matters more than model size
- Dataset formats: instruction, conversational, completion
- Collecting training data: manual, programmatic, from logs
- Synthetic data generation: using a strong model to train a weaker one
- Data augmentation: rephrasing, back-translation, noise injection
- Quality signals: how to tell if your dataset is good enough
- Common pitfalls: overfitting, data leakage, label noise
- Tools: Hugging Face datasets, Argilla for annotation

### Related Chapters
- **Ch 15 (The RAG Pipeline)** — spirals back: RAG retrieval quality depends on the same data curation principles
- **Ch 18 (Why Evals Matter)** — spirals from evaluation datasets to training datasets
- **Ch 21 (Eval-Driven Development)** — spirals back: use eval results to identify gaps in your training data
- **Ch 45 (Fine-Tuning with LoRA)** — your fine-tune is only as good as this data
- **Ch 47 (Quantization & Deployment)** — deploy the model trained on this data

---

## 1. Why Data Quality Matters More Than Everything Else

### 1.1 The Evidence

```python
# The data quality hierarchy (what matters most for fine-tuning quality)

quality_hierarchy = {
    1: {
        "factor": "Data quality (correctness, relevance, diversity)",
        "impact": "HIGHEST — bad data = bad model, period",
        "example": "500 expert-written examples > 5,000 noisy scraped examples",
    },
    2: {
        "factor": "Data format (matches your target task exactly)",
        "impact": "HIGH — wrong format means model learns wrong behavior",
        "example": "If you want chat, train on chat format. Not Q&A format.",
    },
    3: {
        "factor": "Data quantity (how many examples)",
        "impact": "MODERATE — more helps, but diminishing returns after ~1000",
        "example": "1,000 examples: good. 10,000: better. 100,000: marginal improvement.",
    },
    4: {
        "factor": "Model size",
        "impact": "MODERATE — bigger model helps, but not if data is bad",
        "example": "7B on clean data beats 70B on noisy data for specific tasks",
    },
    5: {
        "factor": "Hyperparameters (learning rate, epochs, etc.)",
        "impact": "LOW — matters, but defaults usually work well",
        "example": "Tuning LR from 2e-4 to 3e-4 gives marginal improvement",
    },
}

print("What Matters Most for Fine-Tuning (ranked):")
print("=" * 70)
for rank, info in quality_hierarchy.items():
    print(f"\n  #{rank}: {info['factor']}")
    print(f"     Impact:  {info['impact']}")
    print(f"     Example: {info['example']}")
```

### 1.2 A Concrete Example

```python
# Compare fine-tuning results with different data qualities

# Scenario: Fine-tuning a model to answer customer support questions

datasets_compared = {
    "Dataset A: 100 expert examples": {
        "source": "Written by experienced support agents",
        "quality": "Perfect answers, correct information, right tone",
        "result_accuracy": "85%",
        "result_tone": "Excellent — matches brand voice perfectly",
    },
    "Dataset B: 5,000 scraped examples": {
        "source": "Scraped from public forums and FAQs",
        "quality": "Some wrong answers, inconsistent format, mixed tone",
        "result_accuracy": "62%",
        "result_tone": "Inconsistent — sometimes formal, sometimes casual",
    },
    "Dataset C: 1,000 synthetic examples": {
        "source": "Generated by GPT-4 from product docs, then filtered",
        "quality": "Mostly correct, consistent format, needs some editing",
        "result_accuracy": "78%",
        "result_tone": "Good — consistent but slightly generic",
    },
    "Dataset D: 500 expert + 500 synthetic (filtered)": {
        "source": "Mix of expert-written and GPT-4 generated, human-verified",
        "quality": "High quality, diverse, consistent format",
        "result_accuracy": "88%",
        "result_tone": "Excellent — natural and on-brand",
    },
}

print("Fine-Tuning Results by Dataset Quality:")
print("=" * 70)
for name, info in datasets_compared.items():
    print(f"\n  {name}")
    print(f"    Source:   {info['source']}")
    print(f"    Quality:  {info['quality']}")
    print(f"    Accuracy: {info['result_accuracy']}")
    print(f"    Tone:     {info['result_tone']}")

print("\nKey insight: 100 expert examples beat 5,000 noisy ones.")
print("Hybrid (expert + filtered synthetic) gives the best results.")
```

---

## 2. Dataset Formats

### 2.1 The Three Main Formats

```python
import json

# Format 1: Instruction format (most common for task-specific fine-tuning)
instruction_format = {
    "instruction": "Summarize the following customer complaint in one sentence.",
    "input": "I ordered a laptop three weeks ago and it still hasn't arrived. I've called customer support twice and each time they said it would arrive 'soon'. This is unacceptable service and I want a refund if it doesn't arrive by Friday.",
    "output": "Customer is frustrated with a three-week delayed laptop order and requests a refund if not delivered by Friday."
}

# Format 2: Conversational format (for chat models)
conversational_format = {
    "messages": [
        {"role": "system", "content": "You are a helpful customer support agent. Be empathetic and solution-oriented."},
        {"role": "user", "content": "My order hasn't arrived and it's been three weeks!"},
        {"role": "assistant", "content": "I'm sorry to hear about the delay. Let me look up your order right away. Could you provide your order number so I can check the current status?"},
        {"role": "user", "content": "It's ORDER-12345"},
        {"role": "assistant", "content": "Thank you. I can see order ORDER-12345 was shipped on March 15th but appears to be stuck in transit. I'm going to escalate this with our shipping partner right now and ensure you receive it within 48 hours. If it doesn't arrive by then, I'll process a full refund automatically. Is there anything else I can help with?"},
    ]
}

# Format 3: Completion format (simplest — just text the model should learn to generate)
completion_format = {
    "text": "Customer: My order hasn't arrived.\nAgent: I apologize for the delay. Let me check your order status right away.\n"
}

print("Format 1: Instruction")
print(json.dumps(instruction_format, indent=2)[:200] + "...")

print("\nFormat 2: Conversational")
print(json.dumps(conversational_format, indent=2)[:200] + "...")

print("\nFormat 3: Completion")
print(json.dumps(completion_format, indent=2)[:200] + "...")
```

### 2.2 Choosing the Right Format

```python
format_guide = {
    "Instruction format": {
        "when": "Single-turn tasks: classification, summarization, extraction, Q&A",
        "models": "Most models support this via templates",
        "example_fields": "instruction, input (optional), output",
        "pros": "Clear task specification, easy to create",
        "cons": "Does not model multi-turn conversations",
    },
    "Conversational format": {
        "when": "Multi-turn chat, dialogue systems, customer support bots",
        "models": "Chat-tuned models (Llama-Chat, Mistral-Instruct, etc.)",
        "example_fields": "messages: [{role, content}, ...]",
        "pros": "Models real conversations, includes system prompts",
        "cons": "More complex to create, needs multi-turn examples",
    },
    "Completion format": {
        "when": "Text continuation, style mimicry, creative writing",
        "models": "Base models (non-chat, non-instruct variants)",
        "example_fields": "text (just raw text)",
        "pros": "Simplest format, any text works",
        "cons": "Model doesn't learn to follow instructions",
    },
}

print("Dataset Format Guide:")
print("=" * 70)
for name, info in format_guide.items():
    print(f"\n  {name}")
    for key, value in info.items():
        print(f"    {key:16s}: {value}")
```

### 2.3 Building Datasets with the Hugging Face Library

```python
from datasets import Dataset, DatasetDict

# Create a dataset from a list of examples
examples = [
    {
        "instruction": "Classify the sentiment of this review.",
        "input": "The food was absolutely delicious, best meal I've had in years!",
        "output": "POSITIVE"
    },
    {
        "instruction": "Classify the sentiment of this review.",
        "input": "Terrible service, waited 45 minutes for cold food.",
        "output": "NEGATIVE"
    },
    {
        "instruction": "Classify the sentiment of this review.",
        "input": "It was okay, nothing special but not bad either.",
        "output": "NEUTRAL"
    },
    # ... more examples
]

# Create dataset
dataset = Dataset.from_list(examples)
print(f"Dataset: {dataset}")
print(f"Features: {dataset.features}")
print(f"Example:\n{dataset[0]}")

# Split into train/test
dataset_dict = dataset.train_test_split(test_size=0.2, seed=42)
print(f"\nTrain: {len(dataset_dict['train'])} examples")
print(f"Test: {len(dataset_dict['test'])} examples")

# Format for the model
def format_instruction(example):
    """Convert instruction format to a prompt string."""
    if example["input"]:
        text = f"### Instruction: {example['instruction']}\n### Input: {example['input']}\n### Response: {example['output']}"
    else:
        text = f"### Instruction: {example['instruction']}\n### Response: {example['output']}"
    return {"text": text}

formatted = dataset.map(format_instruction)
print(f"\nFormatted example:\n{formatted[0]['text']}")
```

### 2.4 Loading Existing Datasets from the Hub

```python
from datasets import load_dataset

# The Hub has thousands of datasets ready for fine-tuning

# Popular instruction-following datasets
popular_datasets = {
    "tatsu-lab/alpaca": "52K instruction-following examples (GPT-3.5 generated)",
    "databricks/databricks-dolly-15k": "15K human-written instructions",
    "OpenAssistant/oasst1": "161K human-labeled conversational data",
    "HuggingFaceH4/ultrachat_200k": "200K synthetic multi-turn conversations",
    "teknium/OpenHermes-2.5": "1M instruction examples from multiple sources",
}

print("Popular Fine-Tuning Datasets:")
print("-" * 60)
for name, description in popular_datasets.items():
    print(f"  {name}")
    print(f"    {description}")

# Load a dataset
dataset = load_dataset("databricks/databricks-dolly-15k", split="train")
print(f"\nLoaded: {dataset}")
print(f"Features: {list(dataset.features.keys())}")
print(f"Example:")
example = dataset[0]
for key, value in example.items():
    print(f"  {key}: {str(value)[:100]}...")
```

---

## 3. Collecting Training Data

### 3.1 Manual Collection

The highest quality data comes from domain experts writing examples by hand.

```python
# Strategy 1: Expert-written examples
# Best quality, most expensive, limited scale

manual_collection_guide = """
Manual Data Collection Process:

1. Define your task clearly
   - What is the input? What is the expected output?
   - What tone/style should the output have?
   - What are the edge cases?

2. Write a style guide for annotators
   - Show 5-10 golden examples
   - Define what "good" and "bad" look like
   - List common mistakes to avoid

3. Collect examples from your domain
   - Start with 50-100 examples
   - Cover the full range of expected inputs
   - Include easy, medium, and hard cases
   - Include edge cases and error cases

4. Quality check
   - Have a second person review every example
   - Remove any ambiguous or incorrect examples
   - Ensure consistent formatting

Target: 100-500 high-quality examples for initial fine-tuning
"""
print(manual_collection_guide)

# Template for creating examples
example_template = {
    "task": "Customer support response generation",
    "style_guide": "Be empathetic, professional, action-oriented. Acknowledge the issue, provide a solution, ask if anything else is needed.",
    "examples": [
        {
            "input": "My account is locked and I can't log in.",
            "output": "I understand how frustrating it must be to not have access to your account. Let me help you right away. I'll send a password reset link to your registered email address. You should receive it within 2 minutes. If you don't see it, please check your spam folder. Is there anything else I can help you with?",
            "quality_notes": "Good: empathetic, specific timeline, proactive (spam folder tip), ends with offer to help more",
        },
        {
            "input": "I was charged twice for my subscription.",
            "output": "I sincerely apologize for the double charge. I can see the duplicate charge on your account and I'm processing a refund right now. You should see the refund in 3-5 business days. I've also added a note to your account to ensure this doesn't happen again. Would you like a confirmation email with the refund details?",
            "quality_notes": "Good: apologizes, confirms they see the issue, gives timeline, prevents recurrence, offers confirmation",
        },
    ],
}

print("\nExample collection template:")
for i, example in enumerate(example_template["examples"]):
    print(f"\n  Example {i+1}:")
    print(f"    Input:  {example['input'][:60]}")
    print(f"    Output: {example['output'][:80]}...")
    print(f"    Notes:  {example['quality_notes'][:60]}...")
```

### 3.2 Collecting from Production Logs

```python
# Strategy 2: Extract from existing systems
# Good quality (real-world), needs filtering

log_collection_guide = """
Collecting Training Data from Production Logs:

Sources:
  - Customer support transcripts (with PII removed)
  - Chat logs with human agents
  - Email responses that received positive feedback
  - Internal Q&A systems

Process:
  1. Export raw conversations/responses
  2. Filter: keep only high-quality interactions
     - Positive customer feedback (CSAT > 4)
     - Short resolution time
     - No escalations needed
  3. Anonymize: remove ALL PII
     - Names, emails, phone numbers
     - Account numbers, order IDs
     - Replace with placeholders: [CUSTOMER_NAME], [ORDER_ID]
  4. Format: convert to your training format
  5. Review: human review of 10-20% sample
"""
print(log_collection_guide)

# Example: filtering high-quality support logs
def filter_quality_logs(logs):
    """Filter support logs for high-quality training examples."""
    good_examples = []
    
    for log in logs:
        # Quality signals
        has_positive_feedback = log.get("csat_score", 0) >= 4
        resolved_quickly = log.get("resolution_minutes", 999) < 15
        no_escalation = not log.get("escalated", True)
        sufficient_length = len(log.get("agent_response", "")) > 50
        
        if has_positive_feedback and resolved_quickly and no_escalation and sufficient_length:
            good_examples.append({
                "instruction": "Respond to this customer support message.",
                "input": log["customer_message"],
                "output": log["agent_response"],
            })
    
    return good_examples

# Simulated logs
sample_logs = [
    {"customer_message": "Help, my order is missing!", "agent_response": "I apologize for the inconvenience. Let me look up your order right away and find out where it is.", "csat_score": 5, "resolution_minutes": 8, "escalated": False},
    {"customer_message": "Your product is broken", "agent_response": "Sorry about that.", "csat_score": 2, "resolution_minutes": 45, "escalated": True},
    {"customer_message": "How do I upgrade?", "agent_response": "Great question! To upgrade your plan, go to Settings > Billing > Change Plan. You can preview the price difference before confirming.", "csat_score": 5, "resolution_minutes": 3, "escalated": False},
]

filtered = filter_quality_logs(sample_logs)
print(f"\nFiltered {len(filtered)} high-quality examples from {len(sample_logs)} logs")
for example in filtered:
    print(f"  Input:  {example['input'][:50]}")
    print(f"  Output: {example['output'][:60]}...")
    print()
```

---

## 4. Synthetic Data Generation

### 4.1 Using a Strong Model to Create Training Data

The most scalable approach to data collection: use a powerful model (GPT-4, Claude) to generate training data for a smaller model that you will fine-tune.

```python
# Synthetic data generation with a strong model
# This is pseudo-code — replace with your preferred API

def generate_synthetic_examples(
    task_description: str,
    style_examples: list[dict],
    num_examples: int = 100,
    topics: list[str] = None,
) -> list[dict]:
    """
    Generate synthetic training examples using a strong model.
    
    In practice, call GPT-4 or Claude API here.
    """
    
    prompt = f"""Generate {num_examples} training examples for the following task.

Task: {task_description}

Format each example as:
INPUT: [the user's input]
OUTPUT: [the expected response]

Style reference (match this tone and quality):
"""
    for ex in style_examples[:3]:
        prompt += f"\nINPUT: {ex['input']}\nOUTPUT: {ex['output']}\n"
    
    if topics:
        prompt += f"\nCover these topics: {', '.join(topics)}"
    
    prompt += f"\n\nGenerate {num_examples} diverse, high-quality examples. Each should be different from the reference examples."
    
    # In practice: response = client.messages.create(model="claude-sonnet-4-...", ...)
    # Parse the response into structured examples
    
    print(f"Prompt prepared ({len(prompt)} chars)")
    print(f"Would generate {num_examples} synthetic examples")
    print(f"Covering topics: {topics}")
    
    return []  # Replace with actual API call + parsing


# Configuration for synthetic generation
config = {
    "task_description": "Generate customer support responses that are empathetic, professional, and solution-oriented.",
    "style_examples": [
        {
            "input": "My order is late.",
            "output": "I'm sorry about the delay. Let me track your order right now. Could you share your order number?",
        },
        {
            "input": "I can't log in.",
            "output": "I understand how frustrating that is. Let me help you regain access. I'll send a reset link to your email.",
        },
    ],
    "topics": [
        "billing issues", "technical problems", "account management",
        "shipping delays", "product returns", "feature requests",
        "plan upgrades", "data export", "password reset", "API errors",
    ],
    "num_examples": 100,
}

generate_synthetic_examples(**config)
```

### 4.2 Quality Filtering for Synthetic Data

Synthetic data is not automatically good data. You need to filter it.

```python
def filter_synthetic_data(examples: list[dict]) -> list[dict]:
    """Filter synthetic data for quality."""
    
    filtered = []
    rejection_reasons = {}
    
    for example in examples:
        # Check 1: Minimum length
        if len(example.get("output", "")) < 30:
            rejection_reasons["too_short"] = rejection_reasons.get("too_short", 0) + 1
            continue
        
        # Check 2: Maximum length (prevent rambling)
        if len(example.get("output", "")) > 500:
            rejection_reasons["too_long"] = rejection_reasons.get("too_long", 0) + 1
            continue
        
        # Check 3: Not just repeating the input
        input_text = example.get("input", "").lower()
        output_text = example.get("output", "").lower()
        if input_text and output_text.startswith(input_text[:50]):
            rejection_reasons["repeats_input"] = rejection_reasons.get("repeats_input", 0) + 1
            continue
        
        # Check 4: Contains expected format markers
        # (customize based on your task)
        
        # Check 5: No placeholder text
        placeholders = ["[insert", "[your", "lorem ipsum", "example.com"]
        if any(p in output_text for p in placeholders):
            rejection_reasons["has_placeholders"] = rejection_reasons.get("has_placeholders", 0) + 1
            continue
        
        filtered.append(example)
    
    print(f"Filtered: {len(filtered)}/{len(examples)} passed ({100*len(filtered)/max(len(examples),1):.0f}%)")
    print(f"Rejection reasons: {rejection_reasons}")
    
    return filtered


# Example filtering
test_examples = [
    {"input": "Help!", "output": "OK"},  # Too short
    {"input": "How do I reset?", "output": "To reset your password, go to Settings, click on Security, then click Reset Password. Enter your email and follow the instructions."},  # Good
    {"input": "Billing issue", "output": "Billing issue is something we take very seriously at [insert company name]. We would be happy to help you resolve this."},  # Has placeholder
]

good_examples = filter_synthetic_data(test_examples)
```

### 4.3 The Synthetic Data Pipeline

```python
# Complete synthetic data pipeline

class SyntheticDataPipeline:
    """
    Pipeline for generating, filtering, and validating synthetic training data.
    """
    
    def __init__(self, task_description: str, quality_examples: list[dict]):
        self.task_description = task_description
        self.quality_examples = quality_examples
        self.generated = []
        self.filtered = []
        self.validated = []
    
    def generate(self, num_examples: int, topics: list[str]):
        """Step 1: Generate synthetic examples with a strong model."""
        print(f"Generating {num_examples} examples across {len(topics)} topics...")
        # In practice: call GPT-4/Claude API in batches
        # Generate 10-20 examples per API call for diversity
        self.generated = []  # Would be populated by API calls
        print(f"Generated {len(self.generated)} examples")
    
    def filter_quality(self):
        """Step 2: Automatic quality filtering."""
        self.filtered = filter_synthetic_data(self.generated)
        print(f"After filtering: {len(self.filtered)} examples")
    
    def deduplicate(self):
        """Step 3: Remove near-duplicates."""
        # Use embedding similarity to find duplicates
        # Remove examples that are too similar to each other
        # This ensures diversity in the training set
        print("Deduplication: checking for near-duplicate examples...")
        # In practice: embed all examples, remove pairs with cosine sim > 0.95
    
    def human_review(self, sample_rate: float = 0.2):
        """Step 4: Human review of a sample."""
        sample_size = int(len(self.filtered) * sample_rate)
        print(f"Human review: please review {sample_size} examples ({sample_rate*100:.0f}% sample)")
        print("Mark each as: KEEP, EDIT, or REMOVE")
    
    def export(self, path: str, format: str = "jsonl"):
        """Step 5: Export the final dataset."""
        print(f"Exporting {len(self.validated)} examples to {path} ({format} format)")


pipeline = SyntheticDataPipeline(
    task_description="Customer support response generation",
    quality_examples=[
        {"input": "My order is late.", "output": "I apologize for the delay. Let me track that for you right away."},
    ],
)

print("Synthetic Data Pipeline:")
print("  1. Generate with strong model (GPT-4, Claude)")
print("  2. Automatic quality filtering")
print("  3. Deduplication via embedding similarity")
print("  4. Human review of 20% sample")
print("  5. Export for fine-tuning")
```

### 4.4 Advanced Synthetic Data Strategies

Beyond simple generation, there are targeted strategies that produce higher-quality synthetic data.

```python
# Strategy 1: Seed-then-expand
# Start with a few expert examples, then ask the LLM to generate
# examples that are DIFFERENT from the seeds in specific dimensions.

seed_expand_prompt = """I have these expert-written customer support examples:

{seeds}

Generate 10 new examples that:
1. Cover DIFFERENT topics than the seeds (billing, API errors, data migration)
2. Include different customer emotional states (frustrated, confused, neutral, urgent)
3. Vary the complexity (simple one-step fix vs multi-step troubleshooting)
4. Keep the same professional, empathetic tone as the seeds

Format each as INPUT: / OUTPUT:"""

print("Strategy 1: Seed-then-expand")
print("  Generate diverse examples by constraining along specific dimensions")

# Strategy 2: Persona-based generation
# Generate the same scenario from different user personas
personas = [
    "a non-technical small business owner who is frustrated",
    "a senior developer who wants a quick, technical answer",
    "a new user who just signed up and is confused by the terminology",
    "an enterprise admin managing 500+ seats who is calm but firm",
]

print("\nStrategy 2: Persona-based generation")
print("  Same scenario, different user types — forces response diversity")
for persona in personas:
    print(f"    - {persona}")

# Strategy 3: Failure case generation
# Explicitly ask for edge cases and difficult scenarios
print("\nStrategy 3: Failure case generation")
print("  Ask: 'What questions would be HARDEST for this bot to answer?'")
print("  Ask: 'Generate ambiguous questions where the intent is unclear'")
print("  Ask: 'Generate questions that require information not in the knowledge base'")
print("  These hard cases prevent your model from being brittle.")
```

---

## 5. Data Versioning and Lineage Tracking

### 5.1 Why Track Dataset Versions

When you iterate on your fine-tuning data — adding examples, removing bad ones, changing format — you need to know which dataset produced which model. Without versioning, you cannot reproduce results or debug regressions.

```python
import json
import hashlib
from datetime import datetime

class DatasetVersion:
    """Simple dataset versioning for fine-tuning projects."""
    
    def __init__(self, name: str, examples: list[dict], parent_version: str = None):
        self.name = name
        self.examples = examples
        self.parent_version = parent_version
        self.created_at = datetime.now().isoformat()
        self.version_hash = self._compute_hash()
    
    def _compute_hash(self) -> str:
        """Deterministic hash of the dataset content."""
        content = json.dumps(self.examples, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()[:12]
    
    def metadata(self) -> dict:
        """Return version metadata for tracking."""
        return {
            "name": self.name,
            "version_hash": self.version_hash,
            "num_examples": len(self.examples),
            "parent_version": self.parent_version,
            "created_at": self.created_at,
        }
    
    def save(self, directory: str = "."):
        """Save dataset and metadata."""
        filename = f"{self.name}_{self.version_hash}"
        
        # Save data
        with open(f"{directory}/{filename}.jsonl", "w") as f:
            for example in self.examples:
                f.write(json.dumps(example) + "\n")
        
        # Save metadata
        with open(f"{directory}/{filename}_meta.json", "w") as f:
            json.dump(self.metadata(), f, indent=2)
        
        print(f"Saved: {filename}.jsonl ({len(self.examples)} examples)")
        return filename


# Example: evolving a dataset over time
v1_data = [
    {"input": "How do I reset my password?", "output": "Go to Settings > Security > Reset Password."},
    {"input": "What plans do you offer?", "output": "We offer Free, Pro, and Enterprise plans."},
]

v1 = DatasetVersion("support-bot", v1_data)
print(f"v1: {v1.metadata()}")

# v2: added more examples, fixed a typo
v2_data = v1_data + [
    {"input": "How do I cancel my subscription?", "output": "Go to Settings > Billing > Cancel Plan."},
]
v2 = DatasetVersion("support-bot", v2_data, parent_version=v1.version_hash)
print(f"v2: {v2.metadata()}")

print("\nNow when model v2 performs worse than v1, you can diff the datasets.")
print("For production, consider DVC (Data Version Control) or Hugging Face Datasets with Git LFS.")
```

---

## 6. Data Augmentation

### 5.1 Rephrasing

```python
# Augmentation strategy 1: Rephrase existing examples
# Use a language model to create variations

def rephrase_examples(examples: list[dict], num_variations: int = 3) -> list[dict]:
    """
    Create rephrased variations of existing training examples.
    
    In practice, use an LLM to generate variations.
    """
    augmented = []
    
    for example in examples:
        augmented.append(example)  # Keep original
        
        # Generate variations of the INPUT (not the output)
        # This teaches the model to handle different phrasings
        rephrase_prompt = f"""Rephrase this customer message {num_variations} different ways.
Keep the same meaning but vary the wording, tone, and formality.

Original: {example['input']}

Variations:"""
        
        # In practice: call LLM API here
        # Parse variations and create new examples with same output
    
    return augmented

print("Rephrasing augmentation:")
print("  Original: 'My order hasn't arrived yet.'")
print("  Variation 1: 'Where is my package? It's been weeks.'")
print("  Variation 2: 'I'm still waiting for delivery of my recent order.'")
print("  Variation 3: 'Hey, I placed an order a while back and it still hasn't shown up.'")
print("\n  All variations use the SAME output — teaching the model that")
print("  different phrasings of the same question get the same answer.")
```

### 5.2 Back-Translation

```python
# Augmentation strategy 2: Translate to another language and back
# This naturally creates paraphrases

from transformers import pipeline

def back_translate(text: str, intermediate_language: str = "fr") -> str:
    """
    Translate text to another language and back to create a paraphrase.
    
    English -> French -> English produces natural paraphrases.
    """
    # In practice, use translation models or APIs
    # translator_en_fr = pipeline("translation_en_to_fr")
    # translator_fr_en = pipeline("translation_fr_to_en")
    
    # french = translator_en_fr(text)[0]["translation_text"]
    # back_translated = translator_fr_en(french)[0]["translation_text"]
    
    print(f"  Original:        '{text}'")
    print(f"  -> French:       'Ou est ma commande? Elle n'est pas encore arrivee.'")
    print(f"  -> Back to EN:   'Where is my order? It has not yet arrived.'")
    print("  The back-translation is a natural paraphrase!")
    
    return text  # Placeholder

print("Back-Translation Augmentation:")
back_translate("My order hasn't arrived yet and I'm getting frustrated.")
```

### 5.3 Noise Injection

```python
import random
import string

def inject_noise(text: str, noise_type: str = "typo", rate: float = 0.05) -> str:
    """
    Inject controlled noise into text to make the model robust.
    
    This teaches the model to handle real-world messy inputs.
    """
    words = text.split()
    
    if noise_type == "typo":
        # Simulate typos
        for i in range(len(words)):
            if random.random() < rate and len(words[i]) > 3:
                char_idx = random.randint(1, len(words[i]) - 2)
                word_list = list(words[i])
                word_list[char_idx] = random.choice(string.ascii_lowercase)
                words[i] = "".join(word_list)
    
    elif noise_type == "case":
        # Random case changes
        for i in range(len(words)):
            if random.random() < rate:
                words[i] = words[i].upper() if random.random() > 0.5 else words[i].lower()
    
    elif noise_type == "abbreviation":
        # Common abbreviations
        abbrevs = {"you": "u", "your": "ur", "please": "pls", "because": "bc", "are": "r"}
        for i in range(len(words)):
            if words[i].lower() in abbrevs and random.random() < rate * 5:
                words[i] = abbrevs[words[i].lower()]
    
    return " ".join(words)


# Demonstrate noise injection
original = "Can you please help me with my order because it hasn't arrived yet?"

print("Noise Injection Examples:")
print(f"  Original:      {original}")

random.seed(42)
for noise in ["typo", "case", "abbreviation"]:
    noisy = inject_noise(original, noise_type=noise, rate=0.15)
    print(f"  {noise:14s}: {noisy}")

print("\nWhy: real user inputs have typos, bad casing, and abbreviations.")
print("Training on noisy variants makes the model more robust.")
```

---

## 6. Quality Signals and Validation

### 6.1 Dataset Health Checks

```python
from collections import Counter
import numpy as np

def dataset_health_check(examples: list[dict]) -> dict:
    """Run quality checks on a dataset."""
    
    report = {}
    
    # Size check
    report["total_examples"] = len(examples)
    
    # Length distribution
    input_lengths = [len(ex.get("input", "").split()) for ex in examples]
    output_lengths = [len(ex.get("output", "").split()) for ex in examples]
    
    report["input_length"] = {
        "min": min(input_lengths) if input_lengths else 0,
        "max": max(input_lengths) if input_lengths else 0,
        "mean": np.mean(input_lengths) if input_lengths else 0,
        "median": np.median(input_lengths) if input_lengths else 0,
    }
    report["output_length"] = {
        "min": min(output_lengths) if output_lengths else 0,
        "max": max(output_lengths) if output_lengths else 0,
        "mean": np.mean(output_lengths) if output_lengths else 0,
        "median": np.median(output_lengths) if output_lengths else 0,
    }
    
    # Empty checks
    empty_inputs = sum(1 for ex in examples if not ex.get("input", "").strip())
    empty_outputs = sum(1 for ex in examples if not ex.get("output", "").strip())
    report["empty_inputs"] = empty_inputs
    report["empty_outputs"] = empty_outputs
    
    # Duplicate check
    input_counts = Counter(ex.get("input", "") for ex in examples)
    exact_duplicates = sum(1 for count in input_counts.values() if count > 1)
    report["exact_duplicate_inputs"] = exact_duplicates
    
    # Instruction diversity (if applicable)
    if "instruction" in examples[0]:
        instruction_counts = Counter(ex["instruction"] for ex in examples)
        report["unique_instructions"] = len(instruction_counts)
        report["most_common_instruction"] = instruction_counts.most_common(1)[0]
    
    return report


# Run health check on a sample dataset
sample_dataset = [
    {"instruction": "Classify sentiment", "input": "Great product!", "output": "POSITIVE"},
    {"instruction": "Classify sentiment", "input": "Terrible service", "output": "NEGATIVE"},
    {"instruction": "Classify sentiment", "input": "It's okay", "output": "NEUTRAL"},
    {"instruction": "Classify sentiment", "input": "Great product!", "output": "POSITIVE"},  # Duplicate!
    {"instruction": "Summarize", "input": "Long article about AI...", "output": "AI is advancing rapidly."},
    {"instruction": "Summarize", "input": "", "output": "Cannot summarize empty text."},  # Empty input!
]

report = dataset_health_check(sample_dataset)

print("Dataset Health Report:")
print("=" * 50)
for key, value in report.items():
    if isinstance(value, dict):
        print(f"  {key}:")
        for k, v in value.items():
            print(f"    {k}: {v}")
    else:
        print(f"  {key}: {value}")

# Warnings
warnings = []
if report["empty_inputs"] > 0:
    warnings.append(f"Found {report['empty_inputs']} empty inputs")
if report["empty_outputs"] > 0:
    warnings.append(f"Found {report['empty_outputs']} empty outputs")
if report["exact_duplicate_inputs"] > 0:
    warnings.append(f"Found {report['exact_duplicate_inputs']} exact duplicate inputs")
if report["total_examples"] < 100:
    warnings.append(f"Only {report['total_examples']} examples — consider adding more")

print(f"\nWarnings ({len(warnings)}):")
for w in warnings:
    print(f"  - {w}")
```

### 7.2 Automated Quality Scoring

Beyond basic health checks, you can use embedding models and LLMs to score example quality automatically.

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim
import numpy as np

def automated_quality_score(examples: list[dict], model_name: str = "all-MiniLM-L6-v2") -> list[dict]:
    """Score dataset examples for quality using embeddings and heuristics."""
    
    model = SentenceTransformer(model_name)
    
    scored = []
    for example in examples:
        scores = {}
        
        # Score 1: Input-output relevance
        # High similarity between input and output suggests the output addresses the input
        if example.get("input") and example.get("output"):
            emb_in = model.encode([example["input"]])
            emb_out = model.encode([example["output"]])
            scores["relevance"] = float(cos_sim(emb_in, emb_out))
        
        # Score 2: Output specificity
        # Very short outputs are likely too generic; very long ones may ramble
        output_words = len(example.get("output", "").split())
        if 20 <= output_words <= 200:
            scores["length_ok"] = 1.0
        elif 10 <= output_words < 20 or 200 < output_words <= 300:
            scores["length_ok"] = 0.7
        else:
            scores["length_ok"] = 0.3
        
        # Score 3: Output diversity check
        # Outputs that are just repeating the input verbatim are low quality
        input_words = set(example.get("input", "").lower().split())
        output_words_set = set(example.get("output", "").lower().split())
        if input_words and output_words_set:
            overlap = len(input_words & output_words_set) / len(input_words)
            scores["novelty"] = 1.0 - min(overlap, 1.0)
        
        # Composite score
        composite = np.mean(list(scores.values()))
        
        scored.append({
            **example,
            "_quality_scores": scores,
            "_composite_score": round(composite, 3),
        })
    
    # Sort by composite score
    scored.sort(key=lambda x: x["_composite_score"], reverse=True)
    return scored

# Example usage
sample = [
    {"input": "How do I reset my password?", 
     "output": "To reset your password, navigate to Settings, click Security, then Reset Password. You will receive an email with a reset link within 2 minutes."},
    {"input": "Help with login", 
     "output": "OK."},
    {"input": "Billing question", 
     "output": "Billing question is something we can help with. Your billing question about billing will be addressed."},
]

scored = automated_quality_score(sample)
for s in scored:
    print(f"  Score: {s['_composite_score']:.3f} | Input: {s['input'][:30]} | Output: {s['output'][:40]}...")

print("\nUse the composite score to auto-filter: keep examples above 0.6, review 0.4-0.6, discard below 0.4.")
```

---

## 8. Common Pitfalls

### 7.1 The Pitfall Catalog

```python
pitfalls = {
    "Overfitting to small datasets": {
        "symptom": "Model repeats training examples verbatim instead of generalizing",
        "cause": "Too few examples and/or too many training epochs",
        "fix": "Add more diverse data, reduce epochs, increase dropout, use LoRA with lower rank",
        "detection": "Compare train loss (very low) vs test loss (high) — large gap = overfitting",
    },
    "Data leakage": {
        "symptom": "Model performs amazingly on test set but poorly in production",
        "cause": "Test examples are too similar to training examples (or identical)",
        "fix": "Ensure no overlap between train and test. Use temporal splits if possible.",
        "detection": "Check for exact/near-duplicate overlap between train and test sets",
    },
    "Label noise": {
        "symptom": "Model learns inconsistent behavior — sometimes good, sometimes wrong",
        "cause": "Contradictory labels in training data (same input, different outputs)",
        "fix": "Deduplicate, resolve conflicts, improve annotation guidelines",
        "detection": "Check for duplicate inputs with different outputs",
    },
    "Format inconsistency": {
        "symptom": "Model output format varies wildly between requests",
        "cause": "Training data has inconsistent formatting",
        "fix": "Standardize all examples to exact same format before training",
        "detection": "Visual inspection of 20+ random examples",
    },
    "Bias amplification": {
        "symptom": "Model reinforces stereotypes or shows systematic bias",
        "cause": "Training data over-represents certain perspectives",
        "fix": "Audit data for demographic balance, add counter-examples",
        "detection": "Test with diverse inputs, check for systematic patterns",
    },
    "Topic imbalance": {
        "symptom": "Model is great at common topics, terrible at rare ones",
        "cause": "80% of training data covers 20% of topics",
        "fix": "Balance your dataset. Oversample rare topics, downsample common ones.",
        "detection": "Categorize examples and count per category",
    },
}

print("Common Dataset Pitfalls:")
print("=" * 60)
for name, details in pitfalls.items():
    print(f"\n  {name}")
    print(f"    Symptom:   {details['symptom']}")
    print(f"    Cause:     {details['cause']}")
    print(f"    Fix:       {details['fix']}")
    print(f"    Detection: {details['detection']}")
```

---

## 8. Key Takeaways

1. **Quality over quantity.** 500 expert-written examples beat 5,000 noisy scraped ones. Invest in data quality first, quantity second.

2. **Format matters.** Match your training format to your use case: instruction format for single-turn tasks, conversational for chat, completion for style transfer.

3. **Synthetic data works.** Using GPT-4 or Claude to generate training data for smaller models is a legitimate, powerful strategy. But always filter and validate.

4. **Augment for robustness.** Rephrasing, back-translation, and noise injection make your model handle real-world messy inputs.

5. **Check your data health.** Run automated health checks: length distributions, duplicates, empty fields, topic balance. Fix issues before training.

6. **Avoid leakage.** Never let test examples leak into training data. Use proper splits. Verify no near-duplicates across splits.

7. **Iterate.** Your first dataset will not be perfect. Train, evaluate, find failure cases, add targeted examples, retrain. This loop is the core of dataset engineering.

---

## What's Next

You have the data. You have the model. In **Chapter 47: Quantization & Deployment**, you learn how to shrink the model for efficient deployment, choose the right quantization format, and get your fine-tuned model running in production.

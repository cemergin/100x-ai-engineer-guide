<!--
  CHAPTER: 43
  TITLE: Model Selection & Architecture
  PART: 8 — Open Source AI & Inference
  PHASE: 2 — Become an Expert
  PREREQS: Ch 2 (AI Landscape), Ch 38 (Transformers & Attention), Ch 40-42 (Hugging Face, Images, Embeddings)
  KEY_TOPICS: BERT, GPT, T5, BART, Llama, Mistral, Phi, encoder, decoder, encoder-decoder, model sizes, quantization, GGUF, local inference, cloud inference, decision framework
  DIFFICULTY: Inter→Adv
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 43: Model Selection & Architecture

> **Part 8 — Open Source AI & Inference** | Phase 2: Become an Expert | Prerequisites: Ch 2, Ch 38, Ch 40-42 | Difficulty: Inter-Adv | Language: Python

In Chapter 2, you surveyed the AI landscape at a high level: providers, models, pricing. In Chapters 40-42, you ran text models, image models, and embedding models. You have hands-on experience with pipelines, generation, and similarity search. Now you need a framework for making decisions: given a task, which architecture should you use? Which model family? What size? Run it locally or call an API?

This is the chapter you will come back to every time you start a new AI project. It gives you the decision tree for going from "I need AI for X" to "I should use model Y at size Z deployed on W." The decisions are not abstract — they are backed by the engineering trade-offs you have seen in the previous three chapters.

### In This Chapter
- Three transformer architectures: encoder, decoder, encoder-decoder
- BERT family: when to use encoders
- GPT family: when to use decoders
- T5/BART family: when to use encoder-decoders
- Open-source frontier: Llama, Mistral, Phi, Qwen
- Model sizes: 1B vs 7B vs 13B vs 70B — what you actually need
- Quantization preview: GGUF, 4-bit, running big models on small hardware
- Local vs cloud inference: cost comparison
- The decision framework: task to deployment

### Related Chapters
- **Ch 2 (AI Landscape)** — spirals from overview to deep architecture understanding
- **Ch 38 (Transformers & Attention)** — the architectures here are the transformers you built
- **Ch 40-42 (HF, Images, Embeddings)** — you have used all these architectures
- **Ch 44 (RAG vs Fine-Tuning)** — pick the model for fine-tuning
- **Ch 47 (Quantization & Deployment)** — deep dive on quantization and deployment

---

## 1. The Three Transformer Architectures

### 1.1 The Architecture Map

In Chapter 38, you built a transformer from scratch. Now let us connect that understanding to the three major architecture families used in practice.

```
┌─────────────────────────────────────────────────────┐
│                TRANSFORMER ARCHITECTURES              │
├────────────────┬──────────────────┬─────────────────┤
│    ENCODER     │     DECODER      │ ENCODER-DECODER  │
│                │                  │                  │
│  Understands   │   Generates      │  Transforms      │
│  text          │   text           │  text → text     │
│                │                  │                  │
│  Input → repr  │  Prefix → cont  │  Input → output  │
│                │                  │                  │
│  BERT          │  GPT             │  T5              │
│  RoBERTa       │  Llama           │  BART            │
│  DeBERTa       │  Mistral         │  Flan-T5         │
│  DistilBERT    │  Phi             │  mBART           │
│  BGE           │  Qwen            │  NLLB            │
│  all-MiniLM    │  Falcon          │                  │
├────────────────┼──────────────────┼─────────────────┤
│  TASKS:        │  TASKS:          │  TASKS:          │
│  Classification│  Chat            │  Translation     │
│  NER           │  Code generation │  Summarization   │
│  Similarity    │  Creative writing│  Question answer  │
│  Search/embed  │  Reasoning       │  Paraphrasing    │
│  Fill-mask     │  Instruction     │  Data-to-text    │
│  Sentiment     │    following     │  Style transfer  │
└────────────────┴──────────────────┴─────────────────┘
```

### 1.2 How Each Architecture Works

```python
import torch
from transformers import (
    AutoTokenizer,
    AutoModel,
    AutoModelForCausalLM,
    AutoModelForSeq2SeqLM,
)

# --- ENCODER (BERT family) ---
# Processes ALL tokens bidirectionally
# Each token can "see" every other token
# Good for understanding, not for generating

print("=" * 60)
print("ENCODER — BERT")
print("=" * 60)

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModel.from_pretrained("bert-base-uncased")

text = "The model understands the full context."
inputs = tokenizer(text, return_tensors="pt")

with torch.no_grad():
    outputs = model(**inputs)

# Output: one embedding per token
print(f"  Input: '{text}'")
print(f"  Output shape: {outputs.last_hidden_state.shape}")
# Shape: (batch_size, sequence_length, hidden_size) = (1, 9, 768)
print(f"  Each token gets a 768-dim contextual embedding")
print(f"  Key property: BIDIRECTIONAL — each token sees all others")

# --- DECODER (GPT family) ---
# Processes tokens left-to-right only
# Each token can only "see" previous tokens
# Generates new tokens autoregressively

print("\n" + "=" * 60)
print("DECODER — GPT-2")
print("=" * 60)

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

text = "The model generates"
inputs = tokenizer(text, return_tensors="pt")

with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=10, do_sample=False)

generated = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(f"  Input: '{text}'")
print(f"  Output: '{generated}'")
print(f"  Key property: LEFT-TO-RIGHT — each token only sees previous tokens")
print(f"  This is what enables text generation")

# --- ENCODER-DECODER (T5 family) ---
# Encoder processes the input bidirectionally
# Decoder generates the output left-to-right
# Cross-attention connects them

print("\n" + "=" * 60)
print("ENCODER-DECODER — Flan-T5")
print("=" * 60)

tokenizer = AutoTokenizer.from_pretrained("google/flan-t5-small")
model = AutoModelForSeq2SeqLM.from_pretrained("google/flan-t5-small")

text = "Translate to French: The weather is nice today."
inputs = tokenizer(text, return_tensors="pt")

with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=30)

translated = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(f"  Input: '{text}'")
print(f"  Output: '{translated}'")
print(f"  Key property: ENCODER reads input, DECODER writes output")
print(f"  Cross-attention lets the decoder 'look at' the encoded input")
```

---

## 2. BERT Family: When to Use Encoders

### 2.1 The Encoder Advantage

Encoders excel at **understanding** text. Because every token can attend to every other token (bidirectional attention), they build rich representations that capture the full context of the input.

```python
from transformers import pipeline

# Classification: Is this email spam?
classifier = pipeline("text-classification", model="distilbert-base-uncased-finetuned-sst-2-english")
print("Classification:", classifier("This product exceeded all my expectations!"))

# Named Entity Recognition: What entities are in this text?
ner = pipeline("ner", model="dslim/bert-base-NER", aggregation_strategy="simple")
print("NER:", ner("Elon Musk founded SpaceX in Hawthorne, California."))

# Semantic Similarity (via Sentence Transformers)
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim

model = SentenceTransformer("all-MiniLM-L6-v2")
emb1 = model.encode(["How do I return a product?"])
emb2 = model.encode(["What is your refund policy?"])
print(f"Similarity: {cos_sim(emb1, emb2).item():.4f}")
```

### 2.2 When to Choose an Encoder

| Use Case | Why Encoder | Example Model |
|----------|-------------|---------------|
| Text classification | Need full context understanding | `distilbert-base-uncased-finetuned-sst-2-english` |
| Named entity recognition | Token-level classification | `dslim/bert-base-NER` |
| Semantic search / embeddings | Rich contextual representations | `BAAI/bge-base-en-v1.5` |
| Duplicate detection | Compare two texts | `all-mpnet-base-v2` |
| Extractive QA | Find answer spans in context | `distilbert-base-cased-distilled-squad` |
| Sentence similarity | Pairwise comparison | `all-MiniLM-L6-v2` |
| Content moderation | Classify as safe/unsafe | Fine-tuned BERT/RoBERTa |

### 2.3 The BERT Family Tree

```python
# Key BERT variants and their innovations
bert_family = {
    "BERT (2018)": {
        "innovation": "Bidirectional pre-training with masked language modeling",
        "sizes": "110M (base), 340M (large)",
        "use_today": "Foundation, but largely superseded",
    },
    "DistilBERT (2019)": {
        "innovation": "Knowledge distillation — 60% of BERT's size, 97% of performance",
        "sizes": "66M",
        "use_today": "Quick classification, when speed matters",
    },
    "RoBERTa (2019)": {
        "innovation": "Better training recipe — more data, longer training, no NSP",
        "sizes": "125M (base), 355M (large)",
        "use_today": "General encoder tasks, better than BERT",
    },
    "DeBERTa (2020)": {
        "innovation": "Disentangled attention — separate content and position",
        "sizes": "86M (small), 139M (base), 400M (large), 1.5B (xlarge)",
        "use_today": "Best encoder quality, especially for NLI",
    },
    "BGE (2023)": {
        "innovation": "Trained specifically for retrieval with instruction support",
        "sizes": "33M (small), 109M (base), 335M (large)",
        "use_today": "Best for RAG and semantic search",
    },
}

print("BERT Family Evolution:")
print("=" * 70)
for name, info in bert_family.items():
    print(f"\n  {name}")
    print(f"    Innovation: {info['innovation']}")
    print(f"    Sizes: {info['sizes']}")
    print(f"    Use today: {info['use_today']}")
```

---

## 3. GPT Family: When to Use Decoders

### 3.1 The Decoder Advantage

Decoders excel at **generating** text. Because they process left-to-right, they naturally produce text one token at a time — exactly what you need for chatbots, code generation, creative writing, and instruction following.

```python
from transformers import pipeline

# Text generation
generator = pipeline("text-generation", model="gpt2")
result = generator("The future of AI is", max_new_tokens=50, do_sample=True, temperature=0.7)
print(f"Generated: {result[0]['generated_text']}")
```

### 3.2 The Open-Source Decoder Landscape

This is where the most exciting development in open source AI is happening.

```python
# The open-source frontier models (decoder-only)
frontier_models = {
    "Llama 3.1 (Meta)": {
        "sizes": "8B, 70B, 405B",
        "context": "128K tokens",
        "license": "Llama 3.1 Community License (commercial OK with restrictions)",
        "strengths": "Best overall open-source, strong reasoning, good code",
        "when_to_use": "General-purpose tasks, chat, code, reasoning",
    },
    "Llama 3.2 (Meta)": {
        "sizes": "1B, 3B, 11B, 90B (multimodal for 11B/90B)",
        "context": "128K tokens",
        "license": "Llama 3.2 Community License",
        "strengths": "Small models for edge, multimodal for vision tasks",
        "when_to_use": "Mobile/edge deployment, vision + language",
    },
    "Llama 3.3 (Meta)": {
        "sizes": "70B",
        "context": "128K tokens",
        "license": "Llama 3.3 Community License",
        "strengths": "Quality of 3.1 405B in a 70B model, best open 70B",
        "when_to_use": "When you need near-frontier quality at 70B scale",
    },
    "Mistral / Mixtral (Mistral AI)": {
        "sizes": "7B (Mistral), 8x7B (Mixtral), 8x22B (Mixtral Large)",
        "context": "32K tokens",
        "license": "Apache 2.0 (Mistral 7B, Mixtral)",
        "strengths": "Efficient MoE architecture, strong multilingual",
        "when_to_use": "When you need Apache 2.0 license, multilingual tasks",
    },
    "Phi-3 / Phi-4 (Microsoft)": {
        "sizes": "3.8B (mini), 7B (small), 14B (medium)",
        "context": "4K-128K tokens",
        "license": "MIT",
        "strengths": "Best quality at small sizes, trained on curated data",
        "when_to_use": "Small model tasks, edge deployment, MIT license needed",
    },
    "Qwen 2.5 (Alibaba)": {
        "sizes": "0.5B, 1.5B, 3B, 7B, 14B, 32B, 72B",
        "context": "32K-128K tokens",
        "license": "Apache 2.0 (most sizes)",
        "strengths": "Strong at code, math, and Chinese/multilingual",
        "when_to_use": "Code generation, math, multilingual including CJK",
    },
    "Gemma 2 (Google)": {
        "sizes": "2B, 9B, 27B",
        "context": "8K tokens",
        "license": "Gemma license (commercial OK)",
        "strengths": "Strong quality per parameter, good instruction following",
        "when_to_use": "When you want Google's architecture at open-source prices",
    },
}

print("Open-Source Frontier Models (2025-2026):")
print("=" * 70)
for name, info in frontier_models.items():
    print(f"\n  {name}")
    print(f"    Sizes: {info['sizes']}")
    print(f"    Context: {info['context']}")
    print(f"    License: {info['license']}")
    print(f"    Strengths: {info['strengths']}")
    print(f"    Use when: {info['when_to_use']}")
```

### 3.3 The Llama 3 Family: What Changed Between Versions

The Llama 3 family spans four releases, each with a distinct purpose.

```python
llama_evolution = {
    "Llama 3 (April 2024)": {
        "sizes": "8B, 70B",
        "context": "8K tokens",
        "key_change": "New tokenizer (128K vocab), GQA, massive training data (15T tokens)",
        "improvement": "Huge jump over Llama 2. Competitive with GPT-3.5 at 70B.",
    },
    "Llama 3.1 (July 2024)": {
        "sizes": "8B, 70B, 405B",
        "context": "128K tokens (16x increase over Llama 3)",
        "key_change": "Extended context via RoPE scaling, 405B frontier model released",
        "improvement": "128K context unlocked real RAG use cases. 405B approaches GPT-4.",
    },
    "Llama 3.2 (September 2024)": {
        "sizes": "1B, 3B (text), 11B, 90B (multimodal)",
        "context": "128K tokens",
        "key_change": "Tiny models for edge/mobile. Vision-language models (11B, 90B).",
        "improvement": "1B and 3B run on phones. 11B processes images + text together.",
    },
    "Llama 3.3 (December 2024)": {
        "sizes": "70B",
        "context": "128K tokens",
        "key_change": "Distilled knowledge from 405B into 70B architecture.",
        "improvement": "Matches 3.1 405B quality on most benchmarks at 70B cost.",
    },
}

print("Llama 3 Family Evolution:")
print("=" * 70)
for version, info in llama_evolution.items():
    print(f"\n  {version}")
    print(f"    Sizes:       {info['sizes']}")
    print(f"    Context:     {info['context']}")
    print(f"    Key change:  {info['key_change']}")
    print(f"    Improvement: {info['improvement']}")

print("\nPractical guidance:")
print("  Need a general-purpose 8B? → Llama 3.1 8B")
print("  Need the best open-source 70B? → Llama 3.3 70B")
print("  Need to run on a phone or IoT? → Llama 3.2 1B or 3B")
print("  Need vision + language? → Llama 3.2 11B or 90B")
```

### 3.4 Mixture of Experts (MoE): How Mixtral Works

Mixtral uses a fundamentally different architecture from dense models like Llama. Understanding MoE is important because it changes the size/speed/quality equation.

```python
# How Mixture of Experts works
print("""
Dense Model (Llama 7B):
  Every token passes through ALL 7B parameters.
  Memory needed: 7B parameters
  Compute per token: 7B parameters

Mixture of Experts (Mixtral 8x7B):
  The model has 8 "expert" sub-networks (each ~7B params).
  A router network picks the TOP 2 experts for each token.
  Each token only passes through 2 of 8 experts.
  
  Total parameters: ~47B (8 experts + shared layers)
  Active parameters per token: ~13B (2 experts + shared)
  Memory needed: ~47B (must load ALL experts)
  Compute per token: ~13B (only 2 experts fire)

  Result: Mixtral 8x7B has 70B-class QUALITY
          with 13B-class SPEED per token
          but 47B-class MEMORY requirement
""")

moe_tradeoffs = {
    "Quality": "Near 70B quality — experts specialize in different patterns",
    "Speed": "Near 13B speed — only 2 experts are active per token",
    "Memory": "Near 47B memory — all experts must be loaded",
    "Fine-tuning": "Harder — which experts to train? Usually all, which is expensive.",
    "Quantization": "Works well — GGUF Q4 of Mixtral 8x7B fits in ~26GB",
}

print("MoE Trade-offs:")
for aspect, detail in moe_tradeoffs.items():
    print(f"  {aspect:14s}: {detail}")

print("\nWhen to choose Mixtral over a dense model:")
print("  - You have enough memory for the full model (26GB+ quantized)")
print("  - You want faster inference than a same-quality dense model")
print("  - You need strong multilingual or code performance")
print("  - Apache 2.0 license is a requirement")
```

### 3.5 Loading a Decoder Model

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

# Load a small but capable model (Phi-3 mini or Qwen 2.5 0.5B for quick testing)
model_name = "Qwen/Qwen2.5-0.5B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    device_map="auto",  # Automatically place on GPU if available
)

# Chat format
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain what embeddings are in two sentences."},
]

# Apply chat template
text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
inputs = tokenizer(text, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=100,
        temperature=0.7,
        do_sample=True,
    )

response = tokenizer.decode(outputs[0][inputs["input_ids"].shape[1]:], skip_special_tokens=True)
print(f"Response: {response}")
```

---

## 4. T5/BART Family: When to Use Encoder-Decoders

### 4.1 The Encoder-Decoder Advantage

Encoder-decoders excel at **transformation** tasks — taking input text and producing different output text. The encoder builds a rich understanding of the input, and the decoder generates the output while attending to that understanding.

```python
from transformers import pipeline

# Translation
translator = pipeline("translation", model="Helsinki-NLP/opus-mt-en-fr")
result = translator("Machine learning is transforming the world.")
print(f"Translation: {result[0]['translation_text']}")

# Summarization with BART
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")
article = """
The European Space Agency successfully launched its new satellite constellation
this morning from French Guiana. The mission, which cost approximately 2 billion
euros, aims to provide global internet coverage to underserved regions. The
constellation will consist of 300 satellites in low Earth orbit, with full
deployment expected by 2028. Scientists and engineers celebrated as all
satellites separated from the rocket and began transmitting signals.
"""
result = summarizer(article, max_length=50, min_length=20)
print(f"Summary: {result[0]['summary_text']}")

# Text-to-text with Flan-T5
t5 = pipeline("text2text-generation", model="google/flan-t5-base")

# Flan-T5 handles many tasks through prompting
tasks = [
    "Translate to German: The weather is nice today.",
    "Summarize: AI is transforming how we build software by enabling natural language interfaces and automated code generation.",
    "Answer: What is the capital of Japan?",
    "Classify the sentiment: I absolutely love this new feature!",
]

for task in tasks:
    result = t5(task, max_new_tokens=50)
    print(f"  Input:  {task[:60]}")
    print(f"  Output: {result[0]['generated_text']}")
    print()
```

### 4.2 When to Choose Encoder-Decoder

| Use Case | Why Enc-Dec | Example Model |
|----------|-------------|---------------|
| Translation | Input lang to output lang | `Helsinki-NLP/opus-mt-*` |
| Summarization | Long text to short summary | `facebook/bart-large-cnn` |
| Text-to-text | Any structured transformation | `google/flan-t5-base` |
| Data-to-text | Structured data to natural language | `google/flan-t5-*` |
| Grammar correction | Bad text to good text | Fine-tuned T5 |
| Paraphrasing | One style to another | Fine-tuned T5/BART |

---

## 5. Model Sizes: What You Actually Need

### 5.1 The Size Spectrum

```python
# Model size comparison with practical implications
sizes = [
    {
        "range": "0.5B - 3B",
        "examples": "Phi-3 mini (3.8B), Qwen 2.5 (0.5B, 1.5B, 3B), Llama 3.2 (1B, 3B)",
        "memory_fp16": "1-6 GB",
        "memory_4bit": "0.5-2 GB",
        "hardware": "Laptop, phone, Raspberry Pi (quantized)",
        "quality": "Simple tasks: good. Complex reasoning: limited.",
        "speed": "Very fast — can run real-time on mobile",
        "cost_per_token": "Near zero (on-device)",
        "use_cases": "Classification, simple QA, on-device inference, edge deployment",
    },
    {
        "range": "7B - 8B",
        "examples": "Llama 3.1 8B, Mistral 7B, Qwen 2.5 7B, Gemma 2 9B",
        "memory_fp16": "14-16 GB",
        "memory_4bit": "4-5 GB",
        "hardware": "Gaming GPU (RTX 3080+), Google Colab T4",
        "quality": "Good for most tasks. Solid instruction following.",
        "speed": "Fast — 30-50 tokens/sec on consumer GPU",
        "cost_per_token": "~$0.10-0.20/hr GPU cost",
        "use_cases": "Chat, code, RAG, most production workloads",
    },
    {
        "range": "13B - 14B",
        "examples": "Qwen 2.5 14B, Phi-3 14B",
        "memory_fp16": "26-28 GB",
        "memory_4bit": "8-10 GB",
        "hardware": "RTX 4090 (24GB), A100 (40/80GB)",
        "quality": "Very good. Noticeable improvement in nuance and reasoning.",
        "speed": "Moderate — 20-30 tokens/sec on consumer GPU",
        "cost_per_token": "~$0.30-0.50/hr GPU cost",
        "use_cases": "Complex reasoning, domain tasks, quality-sensitive applications",
    },
    {
        "range": "32B - 72B",
        "examples": "Llama 3.1 70B, Qwen 2.5 72B, Qwen 2.5 32B",
        "memory_fp16": "64-144 GB",
        "memory_4bit": "20-40 GB",
        "hardware": "A100 80GB, multi-GPU setups",
        "quality": "Excellent. Approaches frontier closed models for many tasks.",
        "speed": "Slower — 10-20 tokens/sec on A100",
        "cost_per_token": "~$1-3/hr GPU cost",
        "use_cases": "When quality is paramount, complex analysis, research",
    },
    {
        "range": "100B+",
        "examples": "Llama 3.1 405B",
        "memory_fp16": "810 GB",
        "memory_4bit": "~200 GB",
        "hardware": "Multi-A100 clusters, inference services",
        "quality": "Frontier. Competitive with GPT-4 class models on many benchmarks.",
        "speed": "Slow — requires distributed inference",
        "cost_per_token": "~$5-10/hr cluster cost",
        "use_cases": "Research, benchmarking, when you need the best open-source has",
    },
]

print("Model Size Guide:")
print("=" * 80)
for size in sizes:
    print(f"\n  [{size['range']}]")
    print(f"    Examples:     {size['examples']}")
    print(f"    Memory (fp16):{size['memory_fp16']:>12s}  |  Memory (4-bit): {size['memory_4bit']}")
    print(f"    Hardware:     {size['hardware']}")
    print(f"    Quality:      {size['quality']}")
    print(f"    Use cases:    {size['use_cases']}")
```

### 5.2 The Right Size for Your Task

```python
# Decision helper
def recommend_model_size(task: str, constraints: dict) -> str:
    """Recommend a model size based on task and constraints."""
    
    max_memory_gb = constraints.get("max_memory_gb", 16)
    latency_ms = constraints.get("max_latency_ms", 1000)
    quality_needed = constraints.get("quality", "good")  # good, very_good, excellent
    
    # Simple classification, sentiment, NER
    if task in ["classification", "sentiment", "ner", "embedding"]:
        return "Use an ENCODER model (BERT family). 100-400M params is plenty."
    
    # Text generation / chat
    if task in ["chat", "generation", "code", "instruction_following"]:
        if quality_needed == "good" and max_memory_gb <= 8:
            return "7B quantized (4-bit). Fits in ~5GB, good quality."
        elif quality_needed == "very_good" and max_memory_gb <= 24:
            return "7B-14B in fp16. Best quality per dollar for most tasks."
        elif quality_needed == "excellent":
            return "70B quantized or 14B fp16. Consider cloud API for 70B+."
        else:
            return "Start with 7B. Scale up if quality is insufficient."
    
    # Translation / summarization
    if task in ["translation", "summarization"]:
        return "Flan-T5-base (250M) or BART-large (400M). Encoder-decoder shines here."
    
    return "Start with the smallest model for your task. Scale up based on eval results."


# Example recommendations
tasks = [
    ("sentiment", {"max_memory_gb": 4, "quality": "good"}),
    ("chat", {"max_memory_gb": 16, "quality": "very_good"}),
    ("code", {"max_memory_gb": 24, "quality": "excellent"}),
    ("translation", {"max_memory_gb": 8, "quality": "good"}),
    ("embedding", {"max_memory_gb": 4, "quality": "good"}),
]

print("Model Size Recommendations:")
print("-" * 70)
for task, constraints in tasks:
    rec = recommend_model_size(task, constraints)
    print(f"\n  Task: {task}")
    print(f"  Constraints: {constraints}")
    print(f"  Recommendation: {rec}")
```

---

## 6. Quantization Preview

### 6.1 What Is Quantization?

Quantization reduces the precision of model weights to save memory and increase speed. Instead of storing each weight as a 32-bit floating point number, you store it as a 16-bit, 8-bit, or even 4-bit number.

```python
# Conceptual: what quantization does to a single weight
import numpy as np

weight_fp32 = np.float32(0.12345678)    # 32 bits, 4 bytes
weight_fp16 = np.float16(0.12345678)    # 16 bits, 2 bytes — slight precision loss
weight_int8 = np.int8(round(0.12345678 * 127))  # 8 bits, 1 byte — more precision loss

print("Quantization precision levels:")
print(f"  fp32: {weight_fp32} ({32} bits, {4} bytes per weight)")
print(f"  fp16: {weight_fp16} ({16} bits, {2} bytes per weight)")
print(f"  int8: {weight_int8}/127 = {weight_int8/127:.4f} ({8} bits, {1} byte per weight)")
print()

# Memory impact for a 7B parameter model
params = 7e9
print("Memory for a 7B parameter model:")
print(f"  fp32: {params * 4 / 1e9:.1f} GB")
print(f"  fp16: {params * 2 / 1e9:.1f} GB")
print(f"  int8: {params * 1 / 1e9:.1f} GB")
print(f"  int4: {params * 0.5 / 1e9:.1f} GB")
print()
print("A 7B model goes from 28GB (fp32) to 3.5GB (int4)!")
print("That is the difference between needing a datacenter GPU and a laptop.")
```

### 6.2 Loading Quantized Models

```python
from transformers import AutoTokenizer, AutoModelForCausalLM, BitsAndBytesConfig
import torch

# 4-bit quantization with bitsandbytes
# !pip install bitsandbytes

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,   # Second round of quantization
    bnb_4bit_quant_type="nf4",        # NormalFloat4 — best quality 4-bit
)

model_name = "meta-llama/Llama-3.1-8B-Instruct"

# Note: Llama models require accepting the license on HuggingFace
# and providing your HF token
# For testing, use a model that doesn't require auth:
model_name = "Qwen/Qwen2.5-7B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=quantization_config,
    device_map="auto",
)

# Check actual memory usage
memory_bytes = model.get_memory_footprint()
print(f"Model: {model_name}")
print(f"Memory footprint: {memory_bytes / 1e9:.2f} GB (4-bit quantized)")
print(f"Would be ~14 GB in fp16")

# Use it normally — the API is identical
messages = [
    {"role": "user", "content": "What is the difference between encoder and decoder transformers?"}
]
text = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
inputs = tokenizer(text, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=150, temperature=0.7, do_sample=True)

response = tokenizer.decode(outputs[0][inputs["input_ids"].shape[1]:], skip_special_tokens=True)
print(f"\nResponse: {response}")
```

### 6.3 Quantization Formats

```python
# Quick reference for quantization formats
formats = {
    "GGUF": {
        "tool": "llama.cpp / Ollama",
        "hardware": "CPU + GPU (hybrid)",
        "best_for": "Local inference on consumer hardware",
        "variants": "Q4_K_M, Q5_K_M, Q8_0 (quality increases with bits)",
        "ecosystem": "Ollama, LM Studio, llama.cpp, GPT4All",
    },
    "GPTQ": {
        "tool": "auto-gptq, transformers",
        "hardware": "GPU only",
        "best_for": "GPU inference with good quality",
        "variants": "4-bit, 8-bit, with group size options",
        "ecosystem": "Hugging Face, vLLM, text-generation-inference",
    },
    "AWQ": {
        "tool": "autoawq, transformers",
        "hardware": "GPU only",
        "best_for": "Best quality at 4-bit (activation-aware)",
        "variants": "4-bit with different group sizes",
        "ecosystem": "Hugging Face, vLLM",
    },
    "bitsandbytes": {
        "tool": "bitsandbytes + transformers",
        "hardware": "GPU only",
        "best_for": "Easy integration, fine-tuning with QLoRA",
        "variants": "4-bit (nf4, fp4), 8-bit",
        "ecosystem": "Hugging Face, PEFT",
    },
}

print("Quantization Formats:")
print("=" * 70)
for name, info in formats.items():
    print(f"\n  {name}")
    for key, value in info.items():
        print(f"    {key:12s}: {value}")
```

### 6.4 Practical GGUF: Picking the Right Quantization Level

GGUF is the most common format for local inference. Here is how to pick the right quantization level for your hardware.

```python
# GGUF model sizing for Llama 3.1 8B across quantization levels
gguf_sizing = {
    "Q2_K":   {"size_gb": 3.2,  "quality": "poor — visible degradation",     "fits_in": "8GB RAM"},
    "Q3_K_M": {"size_gb": 3.9,  "quality": "acceptable — some loss on hard tasks", "fits_in": "8GB RAM"},
    "Q4_K_M": {"size_gb": 4.9,  "quality": "good — recommended default",      "fits_in": "8GB RAM"},
    "Q5_K_M": {"size_gb": 5.7,  "quality": "very good — minimal loss",        "fits_in": "8GB RAM"},
    "Q6_K":   {"size_gb": 6.6,  "quality": "excellent — near lossless",       "fits_in": "16GB RAM"},
    "Q8_0":   {"size_gb": 8.5,  "quality": "nearly lossless",                 "fits_in": "16GB RAM"},
    "fp16":   {"size_gb": 16.0, "quality": "full precision (baseline)",        "fits_in": "24GB+ VRAM"},
}

print("Llama 3.1 8B — GGUF Sizing Guide:")
print("=" * 70)
print(f"  {'Quant':<8} {'Size':<8} {'Quality':<40} {'Fits in'}")
print("-" * 70)
for quant, info in gguf_sizing.items():
    print(f"  {quant:<8} {info['size_gb']:<8.1f} {info['quality']:<40} {info['fits_in']}")

print("\n  Rule: Q4_K_M is the sweet spot for nearly all use cases.")
print("  If you have memory headroom, Q5_K_M gives a noticeable quality bump.")
print("  Only go below Q4 if you absolutely must fit in tight memory constraints.")
```

> **Spiral forward:** Chapter 47 (Quantization & Deployment) goes deep into each format, quality comparisons, and deployment strategies.

---

## 7. Local vs Cloud Inference

### 7.1 The Cost Model

```python
# Detailed cost comparison

# Cloud API pricing (approximate, as of 2026)
api_pricing = {
    "GPT-4o": {"input": 2.50, "output": 10.00, "quality": "Frontier"},
    "GPT-4o-mini": {"input": 0.15, "output": 0.60, "quality": "Very Good"},
    "Claude Sonnet 4": {"input": 3.00, "output": 15.00, "quality": "Frontier"},
    "Claude Haiku 3.5": {"input": 0.80, "output": 4.00, "quality": "Very Good"},
}

# Self-hosted pricing (GPU costs per hour)
gpu_pricing = {
    "T4 (16GB)": {"cost_hr": 0.50, "models": "Up to 7B fp16 or 13B 4-bit"},
    "A10G (24GB)": {"cost_hr": 1.00, "models": "Up to 13B fp16 or 30B 4-bit"},
    "A100 40GB": {"cost_hr": 3.00, "models": "Up to 30B fp16 or 70B 4-bit"},
    "A100 80GB": {"cost_hr": 5.00, "models": "Up to 70B fp16 or 140B 4-bit"},
}

print("Cloud API Pricing (per million tokens):")
print("-" * 60)
for model, pricing in api_pricing.items():
    print(f"  {model:20s}  Input: ${pricing['input']:>6.2f}  Output: ${pricing['output']:>6.2f}  [{pricing['quality']}]")

print("\nSelf-Hosted GPU Pricing:")
print("-" * 60)
for gpu, info in gpu_pricing.items():
    print(f"  {gpu:15s}  ${info['cost_hr']:.2f}/hr  |  {info['models']}")

# Break-even analysis
print("\n\nBreak-Even Analysis: API vs Self-Hosted")
print("=" * 60)

# Scenario: 1M tokens per day (input + output)
daily_tokens = 1_000_000
monthly_tokens = daily_tokens * 30

# API cost (GPT-4o-mini as the comparison point)
avg_api_cost_per_m = (0.15 + 0.60) / 2  # Rough average of input+output
monthly_api_cost = (monthly_tokens / 1_000_000) * avg_api_cost_per_m

# Self-hosted cost (7B on T4)
# Assume T4 can process ~30 tokens/sec for a 7B model
tokens_per_sec = 30
hours_needed = monthly_tokens / (tokens_per_sec * 3600)
monthly_gpu_cost = hours_needed * 0.50  # T4 cost

# Always-on GPU cost
monthly_always_on = 0.50 * 24 * 30

print(f"\nScenario: {daily_tokens:,} tokens/day ({monthly_tokens:,}/month)")
print(f"\n  API (GPT-4o-mini):     ${monthly_api_cost:.2f}/month")
print(f"  Self-hosted (on-demand): ${monthly_gpu_cost:.2f}/month (GPU hours: {hours_needed:.0f})")
print(f"  Self-hosted (always-on): ${monthly_always_on:.2f}/month")
print(f"\n  At low volume, API wins on simplicity.")
print(f"  At high volume (10M+ tokens/day), self-hosted wins on cost.")
print(f"  At very high volume (100M+ tokens/day), self-hosted is 10x cheaper.")
```

### 7.2 The Decision Matrix

```python
decision_matrix = """
DECISION: Local vs Cloud Inference

Choose CLOUD API when:
  - You need frontier quality (GPT-4o, Claude Opus 4)
  - Volume is low-to-moderate (<10M tokens/day)
  - You do not want to manage infrastructure
  - You need the latest models immediately
  - You need guaranteed uptime/SLAs
  - Your team is small / no ML ops expertise

Choose SELF-HOSTED when:
  - Data privacy is critical (healthcare, finance, legal)
  - Volume is high (>10M tokens/day)
  - You need full control over model behavior
  - You want to fine-tune the model
  - You need predictable costs (no per-token surprise bills)
  - Latency-sensitive: want to colocate model with application
  - You have ML ops capability

Choose HYBRID when:
  - Route simple tasks to local models, complex tasks to cloud APIs
  - Use local models for development/testing, cloud for production
  - Use fine-tuned local models + cloud fallback
"""
print(decision_matrix)
```

---

## 8. The Complete Decision Framework

### 8.1 From Task to Deployment

```python
def select_model(task_description: str) -> dict:
    """
    The decision framework in code.
    Given a task, recommend architecture, model, size, and deployment.
    """
    
    # Step 1: Determine architecture
    understanding_tasks = [
        "classification", "sentiment", "ner", "similarity", 
        "embedding", "search", "moderation", "spam"
    ]
    generation_tasks = [
        "chat", "code", "creative", "instruction", "reasoning",
        "general", "assistant", "summarize"
    ]
    transformation_tasks = [
        "translation", "paraphrase", "grammar", "style_transfer",
        "data_to_text"
    ]
    
    task_lower = task_description.lower()
    
    if any(t in task_lower for t in understanding_tasks):
        architecture = "encoder"
        model_family = "BERT/DeBERTa/BGE"
        size_range = "100M-400M"
        deployment = "CPU is often sufficient"
    elif any(t in task_lower for t in transformation_tasks):
        architecture = "encoder-decoder"
        model_family = "T5/BART/mBART"
        size_range = "250M-3B"
        deployment = "CPU for small, GPU for base+"
    else:
        architecture = "decoder"
        model_family = "Llama/Mistral/Phi/Qwen"
        size_range = "7B-70B"
        deployment = "GPU required"
    
    return {
        "task": task_description,
        "architecture": architecture,
        "model_family": model_family,
        "size_range": size_range,
        "deployment": deployment,
    }

# Run through common scenarios
scenarios = [
    "Customer sentiment analysis on support tickets",
    "Chatbot for answering product questions",
    "Translating documentation from English to Spanish",
    "Generating code from natural language descriptions",
    "Semantic search over internal documents",
    "Summarizing long meeting transcripts",
    "Named entity extraction from contracts",
    "Creative writing assistant for marketing copy",
]

print("Model Selection Framework:")
print("=" * 70)
for scenario in scenarios:
    result = select_model(scenario)
    print(f"\n  Task: {result['task']}")
    print(f"    Architecture:  {result['architecture']}")
    print(f"    Model family:  {result['model_family']}")
    print(f"    Size range:    {result['size_range']}")
    print(f"    Deployment:    {result['deployment']}")
```

### 8.2 The Full Flowchart

```
START: "I need AI for [task]"
  │
  ├─ Understanding task? (classify, search, compare, extract)
  │   → ENCODER (BERT family)
  │   → 100M-400M params
  │   → CPU or GPU
  │   └─ Done: distilbert, deberta, bge, all-MiniLM
  │
  ├─ Transformation task? (translate, summarize, paraphrase)
  │   → ENCODER-DECODER (T5/BART)
  │   → 250M-3B params
  │   → CPU (small) or GPU (base+)
  │   └─ Done: flan-t5, bart-large-cnn, opus-mt
  │
  └─ Generation task? (chat, code, creative, reasoning)
      → DECODER (GPT family)
      │
      ├─ Need frontier quality?
      │   → Cloud API (GPT-4o, Claude, Gemini)
      │
      ├─ Need data privacy or high volume?
      │   → Self-hosted open source
      │   │
      │   ├─ Running on laptop/edge?
      │   │   → 1B-3B quantized (Phi, Qwen, Llama 3.2)
      │   │
      │   ├─ Running on single GPU?
      │   │   → 7B-8B (Llama 3.1, Mistral 7B, Qwen 2.5)
      │   │
      │   └─ Running on GPU cluster?
      │       → 70B (Llama 3.1 70B, Qwen 2.5 72B)
      │
      └─ Unsure?
          → Start with 7B, eval, scale up if needed
```

### 8.3 Practical Model Selection Shortcuts

When you do not have time for a full evaluation, these rules of thumb get you 90% of the way.

```python
shortcuts = {
    "Need embeddings for search/RAG?": {
        "answer": "BAAI/bge-base-en-v1.5 (109M). Switch to bge-large if quality matters more than speed.",
        "avoid": "Do not use general-purpose LLMs for embeddings — use a model trained for retrieval.",
    },
    "Need text classification?": {
        "answer": "Fine-tune distilbert-base-uncased (~66M). Only move to a larger model if accuracy plateaus.",
        "avoid": "Do not use a 7B decoder for classification. A 66M encoder is faster AND more accurate.",
    },
    "Need a general chat assistant?": {
        "answer": "Llama 3.1 8B-Instruct (quantized to Q4_K_M). Upgrade to 3.3 70B for harder tasks.",
        "avoid": "Do not start with 70B. Test 8B first. You may not need the bigger model.",
    },
    "Need code generation?": {
        "answer": "Qwen 2.5 Coder 7B or DeepSeek Coder. Both strong at code, competitive with GPT-3.5.",
        "avoid": "Do not use a general-purpose model for code when code-specialized models exist.",
    },
    "Need translation?": {
        "answer": "Helsinki-NLP/opus-mt-{src}-{tgt} for specific pairs. google/flan-t5-large for flexible multi-language.",
        "avoid": "Do not fine-tune a decoder for translation unless you have a very specific domain need.",
    },
}

print("Quick Model Selection Shortcuts:")
print("=" * 70)
for question, info in shortcuts.items():
    print(f"\n  {question}")
    print(f"    Use:   {info['answer']}")
    print(f"    Avoid: {info['avoid']}")
```

---

## 9. Key Takeaways

1. **Architecture follows task.** Encoders for understanding, decoders for generation, encoder-decoders for transformation. Pick the architecture first, then the model.

2. **Start small.** Use the smallest model that meets your quality requirements. A 7B model is often sufficient. A 100M encoder beats a 7B decoder for classification.

3. **Quantization is practical magic.** 4-bit quantization reduces memory by 4-8x with modest quality loss. It makes 7B models run on laptops and 70B models run on single GPUs.

4. **The open-source frontier is real.** Llama 3.1 70B, Qwen 2.5 72B, and similar models approach GPT-4 class quality for many tasks. The gap is closing.

5. **Local vs cloud is a volume decision.** Low volume: API wins on simplicity. High volume: self-hosted wins on cost. Very high volume or privacy requirements: self-hosted is the only option.

6. **Always eval.** Never trust benchmarks alone. Test your specific task on your specific data with at least two model sizes. Let the results decide.

---

## What's Next

You have completed Part 8: you can run any model from the Hub (Ch 40), generate images (Ch 41), produce embeddings (Ch 42), and choose the right architecture for any task (this chapter).

In **Part 9: Fine-Tuning & Training**, you go from running existing models to modifying them. Chapter 44 asks the fundamental question: should you retrieve knowledge (RAG) or bake it into the model (fine-tuning)? Chapter 45 teaches you to fine-tune with LoRA. Chapter 46 covers the data engineering that makes fine-tuning work. And Chapter 47 covers quantization and deployment in full depth.

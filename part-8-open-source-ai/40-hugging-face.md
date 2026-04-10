<!--
  CHAPTER: 40
  TITLE: Hugging Face & Pipelines
  PART: 8 — Open Source AI & Inference
  PHASE: 2 — Become an Expert
  PREREQS: Ch 3 (Your First LLM Call), Ch 38 (Transformers & Attention)
  KEY_TOPICS: hugging face hub, pipeline api, model cards, tokenizers, local inference, tasks, google colab, model loading
  DIFFICULTY: Intermediate
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 40: Hugging Face & Pipelines

> **Part 8 — Open Source AI & Inference** | Phase 2: Become an Expert | Prerequisites: Ch 3, Ch 38 | Difficulty: Intermediate | Language: Python

In Chapter 3, you made your first LLM call through an API. You sent text to a server, got text back, and paid per token. That was the right starting point — APIs are the fastest path to shipping AI features. But now you understand what is happening inside those models (Ch 36-39), and it is time to run them yourself.

Hugging Face is the GitHub of machine learning. It hosts over 500,000 pre-trained models, 100,000+ datasets, and provides libraries that let you load any of those models with a few lines of Python. The `pipeline` API is the simplest entry point — one line of code to run sentiment analysis, text generation, translation, summarization, question answering, and more. No API key. No per-token costs. No rate limits.

This chapter gets you from zero to running models locally. By the end, you will have run seven different NLP tasks, learned to read model cards, loaded specific models and tokenizers, and understood the trade-off between local inference and cloud APIs.

### In This Chapter
- The Hugging Face Hub: models, datasets, and Spaces
- The Pipeline API: one-line inference for any task
- Seven NLP tasks: sentiment, generation, zero-shot, fill-mask, QA, summarization, NER
- Model cards: how to evaluate and choose models
- Loading specific models and tokenizers
- Google Colab setup for GPU access
- Local inference vs the Inference API

### Related Chapters
- **Ch 3 (Your First LLM Call)** — spirals from API calls to local inference
- **Ch 14 (Semantic Search)** — spirals back: local models can power the retrieval pipeline you built
- **Ch 38 (Transformers & Attention)** — these models ARE transformers; now you run them
- **Ch 41 (Image Generation)** — extends local inference to images
- **Ch 42 (Sentence Transformers)** — extends local inference to embeddings
- **Ch 43 (Model Selection)** — deep model selection framework

---

## 1. The Hugging Face Ecosystem

### 1.1 What Is Hugging Face?

Hugging Face started as a chatbot company, pivoted to open-source AI, and became the central hub for the machine learning community. Think of it as three things:

**The Hub** — A platform hosting models, datasets, and demo applications (Spaces). Anyone can upload a model. Anyone can download one. Models have version control, documentation (model cards), and usage statistics.

**The Libraries** — Open-source Python libraries that make it trivial to use those models:
- `transformers` — load and run any model from the Hub
- `datasets` — load and process any dataset from the Hub
- `diffusers` — image generation models (Stable Diffusion, etc.)
- `tokenizers` — fast tokenization
- `peft` — parameter-efficient fine-tuning
- `accelerate` — multi-GPU and mixed-precision training
- `trl` — reinforcement learning from human feedback

**Spaces** — Hosted demo applications where you can try models in the browser. Built with Gradio or Streamlit.

### 1.2 Setting Up Your Environment

Let us set up everything you need. If you are on Google Colab, most of this is pre-installed.

```python
# Install the core libraries
# On Google Colab, transformers and torch are often pre-installed
# Run this cell to make sure you have the latest versions

# !pip install transformers torch datasets accelerate sentencepiece protobuf

# Verify the installation
import transformers
import torch

print(f"Transformers version: {transformers.__version__}")
print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")

if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"GPU Memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")
else:
    print("No GPU detected — models will run on CPU (slower but works)")
```

**Google Colab GPU Setup:**
1. Go to [colab.research.google.com](https://colab.research.google.com)
2. Runtime > Change runtime type > T4 GPU
3. The free tier gives you a T4 with 15GB VRAM — enough for most examples in this chapter

### 1.3 The Hub at a Glance

```python
# Browse the Hub programmatically
from huggingface_hub import HfApi

api = HfApi()

# Search for popular text generation models
models = api.list_models(
    task="text-generation",
    sort="downloads",
    direction=-1,
    limit=5
)

print("Top 5 text generation models by downloads:")
print("-" * 60)
for model in models:
    print(f"  {model.id}")
    print(f"    Downloads: {model.downloads:,}")
    print(f"    Likes: {model.likes:,}")
    print()
```

Every model on the Hub has a unique ID in the format `organization/model-name`:
- `meta-llama/Llama-3.1-8B` — Meta's Llama 3.1 8B
- `google/flan-t5-base` — Google's Flan-T5 base
- `distilbert/distilbert-base-uncased-finetuned-sst-2-english` — DistilBERT fine-tuned for sentiment
- `facebook/bart-large-cnn` — BART fine-tuned for summarization

You load any of these with a single line of code. That is the power of the ecosystem.

---

## 2. The Pipeline API: One-Line Inference

### 2.1 Your First Pipeline

The `pipeline` API is the simplest way to use a model. You specify a task, and it handles model loading, tokenization, inference, and post-processing automatically.

```python
from transformers import pipeline

# Sentiment analysis — one line
classifier = pipeline("sentiment-analysis")

result = classifier("I love building things with open source models!")
print(result)
# [{'label': 'POSITIVE', 'score': 0.9998}]

# Multiple inputs at once
results = classifier([
    "This product is amazing, best purchase ever!",
    "Terrible experience. Would not recommend.",
    "It's okay, nothing special.",
])

for text, result in zip(
    ["Amazing product", "Terrible experience", "It's okay"],
    results
):
    print(f"  {text:25s} → {result['label']:8s} ({result['score']:.4f})")
```

What just happened under the hood:
1. `pipeline("sentiment-analysis")` downloaded a default model (`distilbert-base-uncased-finetuned-sst-2-english`)
2. It loaded the model and its tokenizer
3. Your text was tokenized into input IDs
4. The tokens were fed through the transformer
5. The output logits were converted to labels and confidence scores

All in one line.

### 2.2 How Pipeline Maps Tasks to Models

When you call `pipeline("sentiment-analysis")`, Hugging Face selects a default model for that task. Here are the most useful tasks and their defaults:

```python
# The pipeline function accepts a task name and optionally a specific model

# These are the most commonly used tasks:
tasks_and_defaults = {
    "sentiment-analysis":     "distilbert-base-uncased-finetuned-sst-2-english",
    "text-generation":        "gpt2",
    "text2text-generation":   "google/flan-t5-small",
    "fill-mask":              "distilroberta-base",
    "question-answering":     "distilbert-base-cased-distilled-squad",
    "summarization":          "sshleifer/distilbart-cnn-12-6",
    "translation_en_to_fr":   "Helsinki-NLP/opus-mt-en-fr",
    "zero-shot-classification": "facebook/bart-large-mnli",
    "ner":                    "dbmdz/bert-large-cased-finetuned-conll03-english",
}

print("Task → Default Model:")
print("-" * 70)
for task, model in tasks_and_defaults.items():
    print(f"  {task:30s} → {model}")
```

The defaults are lightweight models optimized for quick inference. For production use, you will want to specify a better model explicitly — we will cover how to choose in section 5.

---

## 3. Seven NLP Tasks in Seven Minutes

Let us run through the most important NLP tasks. Each one is a single pipeline call.

### 3.1 Sentiment Analysis

Classify text as positive or negative (or more nuanced labels depending on the model).

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")

reviews = [
    "The new feature shipped on time and users love it.",
    "The deployment broke production for three hours.",
    "The code review had some minor suggestions.",
]

print("Sentiment Analysis:")
print("-" * 60)
for review in reviews:
    result = classifier(review)[0]
    print(f"  [{result['label']:8s} {result['score']:.3f}] {review[:50]}")
```

**When to use:** Customer feedback analysis, support ticket triage, social media monitoring, review classification.

### 3.2 Text Generation

Generate text continuations from a prompt.

```python
generator = pipeline("text-generation", model="gpt2")

prompt = "The future of open source AI is"

results = generator(
    prompt,
    max_new_tokens=50,       # Generate up to 50 new tokens
    num_return_sequences=3,  # Generate 3 different completions
    temperature=0.8,         # Some creativity
    do_sample=True,          # Enable sampling (not greedy)
    top_p=0.9,               # Nucleus sampling
)

print(f"Prompt: '{prompt}'")
print("-" * 60)
for i, result in enumerate(results):
    generated = result["generated_text"][len(prompt):]
    print(f"\n  Completion {i+1}: ...{generated.strip()}")
```

**When to use:** Content drafting, code completion, creative writing, chatbots (though for chat, use a chat-tuned model).

> **Spiral note:** Remember Ch 39 (Decoding & Generation)? The `temperature`, `do_sample`, `top_p` parameters are exactly the sampling strategies you implemented from scratch. Now you are using them through a high-level API.

### 3.3 Zero-Shot Classification

Classify text into categories that the model has never been specifically trained on. This is remarkably powerful — you define the labels at inference time.

```python
classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")

text = "The new machine learning pipeline reduced inference latency by 40%"

candidate_labels = ["engineering", "business", "science", "sports", "politics"]

result = classifier(text, candidate_labels)

print(f"Text: '{text}'")
print(f"\nClassification:")
for label, score in zip(result["labels"], result["scores"]):
    bar = "#" * int(score * 40)
    print(f"  {label:15s} {score:.3f} {bar}")
```

**Multi-label classification** — when a text can belong to multiple categories:

```python
text = "Our Q3 revenue grew 20% thanks to the new AI-powered search feature"

result = classifier(
    text,
    candidate_labels=["finance", "technology", "product", "growth"],
    multi_label=True  # Allow multiple labels
)

print(f"\nMulti-label classification:")
for label, score in zip(result["labels"], result["scores"]):
    bar = "#" * int(score * 40)
    print(f"  {label:15s} {score:.3f} {bar}")
```

**When to use:** Categorizing content without training data, email routing, intent detection, content moderation. This is one of the most practical zero-to-value pipelines in the entire Hugging Face ecosystem.

### 3.4 Translation

Translate text between languages using pre-trained models from the Helsinki-NLP project (OPUS-MT).

```python
# English to French
translator = pipeline("translation_en_to_fr", model="Helsinki-NLP/opus-mt-en-fr")

texts = [
    "The deployment was successful and all tests passed.",
    "Please contact our support team for further assistance.",
]

print("English to French:")
print("-" * 60)
for text in texts:
    result = translator(text, max_length=128)
    print(f"  EN: {text}")
    print(f"  FR: {result[0]['translation_text']}")
    print()

# English to German
translator_de = pipeline("translation_en_to_de", model="Helsinki-NLP/opus-mt-en-de")
result = translator_de("Machine learning is transforming software engineering.")
print(f"  DE: {result[0]['translation_text']}")
```

**When to use:** Documentation translation, content localization, multilingual support systems. The OPUS-MT models cover 1,000+ language pairs.

### 3.5 Conversational

Build a simple chatbot that tracks conversation history across turns.

```python
chatbot = pipeline("conversational", model="facebook/blenderbot-400M-distill")

from transformers import Conversation

conversation = Conversation("Hi, I'm interested in learning about machine learning.")

# First turn
conversation = chatbot(conversation)
print(f"  Bot: {conversation.generated_responses[-1]}")

# Second turn — the pipeline maintains context
conversation.add_user_input("What programming language should I start with?")
conversation = chatbot(conversation)
print(f"  Bot: {conversation.generated_responses[-1]}")
```

**When to use:** Simple FAQ bots, interactive demos, lightweight conversational interfaces. For production chat, you will typically use a larger instruction-tuned model.

### 3.6 Table Question Answering

Answer questions by querying structured table data — no SQL needed.

```python
table_qa = pipeline("table-question-answering", model="google/tapas-base-finetuned-wtq")

table = {
    "Model": ["GPT-2", "BERT-base", "Llama 3.1 8B", "Mistral 7B"],
    "Parameters": ["124M", "110M", "8B", "7B"],
    "Type": ["Decoder", "Encoder", "Decoder", "Decoder"],
    "Year": ["2019", "2018", "2024", "2023"],
}

questions = [
    "Which model has the most parameters?",
    "How many decoder models are in the table?",
    "What year was BERT-base released?",
]

print("Table Question Answering:")
print("-" * 60)
for question in questions:
    result = table_qa(table=table, query=question)
    print(f"  Q: {question}")
    print(f"  A: {result['answer']} (cells: {result.get('cells', [])})")
```

**When to use:** Business intelligence dashboards, structured data exploration, answering questions over spreadsheets or database query results without writing SQL.

### 3.7 Fill-Mask

Predict the missing word in a sentence. This is what BERT-family models were trained to do — masked language modeling.

```python
unmasker = pipeline("fill-mask")

results = unmasker("The engineer deployed the model to <mask> using Docker.")

print("Fill-Mask predictions:")
print("-" * 60)
for result in results[:5]:
    print(f"  {result['token_str']:15s} (score: {result['score']:.4f})")
    print(f"    '{result['sequence']}'")
```

**When to use:** Understanding how a model "sees" language, data augmentation (generate variations), vocabulary analysis. Less common in production but useful for debugging and understanding models.

### 3.5 Question Answering (Extractive)

Given a context paragraph and a question, extract the answer directly from the context. This is not generative — the model highlights a span in the source text.

```python
qa = pipeline("question-answering")

context = """
Hugging Face was founded in 2016 by Clement Delangue, Julien Chaumond,
and Thomas Wolf. Originally a chatbot company, it pivoted to become the
leading platform for open-source machine learning. The company is
headquartered in New York City and has raised over $395 million in
funding. Their Transformers library has been downloaded over 100 million
times and supports over 200,000 models on their Hub.
"""

questions = [
    "When was Hugging Face founded?",
    "Where is the company headquartered?",
    "How much funding have they raised?",
    "How many models are on the Hub?",
]

print("Extractive Question Answering:")
print("-" * 60)
for question in questions:
    result = qa(question=question, context=context)
    print(f"\n  Q: {question}")
    print(f"  A: {result['answer']} (confidence: {result['score']:.3f})")
```

**When to use:** FAQ systems, document search, customer support where answers exist in your knowledge base. This is the local equivalent of the RAG retrieval step from Ch 15.

### 3.6 Summarization

Condense long text into a short summary.

```python
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

article = """
The field of artificial intelligence has undergone a dramatic transformation
in recent years, driven primarily by advances in large language models and
transformer architectures. Companies like OpenAI, Anthropic, Google, and Meta
have released increasingly capable models that can generate text, write code,
analyze images, and engage in complex reasoning tasks.

The open-source community has played a crucial role in democratizing access
to these technologies. Meta's Llama models, Mistral's open-weight releases,
and platforms like Hugging Face have made it possible for individual
developers and small companies to run powerful AI models on their own
hardware. This has led to an explosion of innovation in fine-tuning,
quantization, and efficient inference techniques.

However, significant challenges remain. The computational cost of training
frontier models continues to grow exponentially. Safety and alignment
research has not kept pace with capability improvements. And questions about
data privacy, copyright, and the environmental impact of AI training remain
largely unresolved. The next few years will likely determine whether AI
development follows a path of broad, distributed innovation or consolidates
around a handful of well-funded labs.
"""

result = summarizer(article, max_length=80, min_length=30)

print("Summary:")
print("-" * 60)
print(f"  {result[0]['summary_text']}")
print(f"\n  Original: {len(article.split())} words")
print(f"  Summary:  {len(result[0]['summary_text'].split())} words")
```

**When to use:** Document digests, email summarization, news aggregation, meeting notes.

### 3.7 Named Entity Recognition (NER)

Identify and classify entities in text: people, organizations, locations, dates.

```python
ner = pipeline("ner", aggregation_strategy="simple")

text = "Satya Nadella announced that Microsoft will invest $10 billion in OpenAI, headquartered in San Francisco."

entities = ner(text)

print("Named Entity Recognition:")
print("-" * 60)
for entity in entities:
    print(f"  [{entity['entity_group']:5s}] '{entity['word']}' "
          f"(score: {entity['score']:.3f}, "
          f"position: {entity['start']}-{entity['end']})")
```

The `aggregation_strategy="simple"` parameter merges sub-word tokens back into complete entities. Without it, "Microsoft" might be split into "Micro" and "##soft" as separate entities.

**When to use:** Information extraction, document indexing, compliance (detecting PII), knowledge graph construction.

---

## 4. Loading Specific Models and Tokenizers

### 4.1 Specifying a Model

The default models are fine for experimentation, but for production you want to choose your model deliberately.

```python
from transformers import pipeline

# Specify a model by its Hub ID
sentiment = pipeline(
    "sentiment-analysis",
    model="nlptown/bert-base-multilingual-uncased-sentiment"
)

# This model gives 1-5 star ratings instead of just positive/negative
reviews = [
    "Absolutely fantastic product!",
    "It works but could be better.",
    "Worst purchase of my life.",
]

for review in reviews:
    result = sentiment(review)[0]
    print(f"  {result['label']:12s} ({result['score']:.3f}) {review}")
```

### 4.2 Loading Models and Tokenizers Separately

For more control, load the model and tokenizer yourself.

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

# Load tokenizer and model separately
model_name = "distilbert/distilbert-base-uncased-finetuned-sst-2-english"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

# Tokenize manually
text = "Hugging Face makes open source AI accessible to everyone."
inputs = tokenizer(text, return_tensors="pt")

print("Tokenization details:")
print(f"  Input IDs: {inputs['input_ids']}")
print(f"  Attention mask: {inputs['attention_mask']}")
print(f"  Token count: {inputs['input_ids'].shape[1]}")
print(f"  Tokens: {tokenizer.convert_ids_to_tokens(inputs['input_ids'][0])}")

# Run inference manually
with torch.no_grad():
    outputs = model(**inputs)

# Process outputs
logits = outputs.logits
probabilities = torch.softmax(logits, dim=1)
predicted_class = torch.argmax(probabilities, dim=1).item()

labels = model.config.id2label
print(f"\n  Predicted: {labels[predicted_class]}")
print(f"  Probabilities: {dict(zip(labels.values(), probabilities[0].tolist()))}")
```

> **Spiral note:** The `AutoTokenizer` and `AutoModelForSequenceClassification` are "Auto" classes — they detect the model architecture from the config and load the right class automatically. Behind the scenes, `AutoModelForSequenceClassification.from_pretrained("distilbert/...")` loads a DistilBERT model with a classification head. The same architecture you studied in Ch 38 — encoder transformer with attention layers — is running right here.

### 4.3 Understanding the Auto Classes

Hugging Face provides `Auto` classes for every task:

```python
from transformers import (
    AutoTokenizer,                       # Works for any model
    AutoModel,                           # Base model (no task head)
    AutoModelForSequenceClassification,  # Classification head
    AutoModelForTokenClassification,     # NER head
    AutoModelForQuestionAnswering,       # QA head
    AutoModelForCausalLM,               # Text generation (GPT-style)
    AutoModelForSeq2SeqLM,              # Seq2seq (T5, BART style)
    AutoModelForMaskedLM,               # Fill-mask (BERT-style)
)

# Example: loading GPT-2 for text generation
tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

# Generate text
inputs = tokenizer("The key to great software is", return_tensors="pt")

with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=30,
        temperature=0.7,
        do_sample=True,
        top_p=0.9,
    )

generated_text = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(f"Generated: {generated_text}")
```

The pattern is always the same:
1. Load the tokenizer: `AutoTokenizer.from_pretrained(model_name)`
2. Load the model: `AutoModelFor<Task>.from_pretrained(model_name)`
3. Tokenize: `tokenizer(text, return_tensors="pt")`
4. Inference: `model(**inputs)` or `model.generate(**inputs)`
5. Decode: `tokenizer.decode(output_ids)`

### 4.4 Device Placement (CPU vs GPU)

Models run on CPU by default. To use a GPU:

```python
import torch
from transformers import pipeline

# Method 1: Pipeline with device parameter
device = 0 if torch.cuda.is_available() else -1  # 0 = first GPU, -1 = CPU
classifier = pipeline("sentiment-analysis", device=device)

# Method 2: Move model to device manually
from transformers import AutoTokenizer, AutoModelForCausalLM

model_name = "gpt2"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)
print(f"Model is on: {model.device}")

# Inputs also need to be on the same device
inputs = tokenizer("Hello world", return_tensors="pt").to(device)

with torch.no_grad():
    outputs = model.generate(**inputs, max_new_tokens=20)

print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

**Speed comparison** (approximate, will vary by hardware):

```
Task: Sentiment analysis on 100 sentences
  CPU (M1 Mac):  ~3.2 seconds
  GPU (T4):      ~0.4 seconds
  GPU (A100):    ~0.1 seconds

Task: Generate 100 tokens with GPT-2
  CPU (M1 Mac):  ~2.5 seconds
  GPU (T4):      ~0.3 seconds
  GPU (A100):    ~0.08 seconds
```

For experimentation, CPU is fine. For anything production-adjacent, you want a GPU.

### 4.5 GPU vs CPU Inference: When the Difference Matters

The gap between CPU and GPU depends heavily on the task and model size. Here is a practical benchmark you can run yourself.

```python
import time
import torch
from transformers import pipeline

def benchmark_device(task, model_name, inputs, device, runs=5):
    """Benchmark a pipeline on a specific device."""
    pipe = pipeline(task, model=model_name, device=device)
    
    # Warmup
    pipe(inputs[0])
    
    times = []
    for _ in range(runs):
        start = time.time()
        pipe(inputs)
        times.append(time.time() - start)
    
    avg = sum(times) / len(times)
    return avg

# Test data
texts = [f"This product is {'excellent' if i % 2 == 0 else 'disappointing'}." for i in range(50)]

# Run on CPU
cpu_time = benchmark_device("sentiment-analysis", "distilbert-base-uncased-finetuned-sst-2-english", texts, device=-1)
print(f"CPU: {cpu_time:.3f}s for {len(texts)} texts ({len(texts)/cpu_time:.0f} texts/sec)")

# Run on GPU (if available)
if torch.cuda.is_available():
    gpu_time = benchmark_device("sentiment-analysis", "distilbert-base-uncased-finetuned-sst-2-english", texts, device=0)
    print(f"GPU: {gpu_time:.3f}s for {len(texts)} texts ({len(texts)/gpu_time:.0f} texts/sec)")
    print(f"Speedup: {cpu_time/gpu_time:.1f}x")
else:
    print("No GPU available — run on Colab to see the difference")
```

**Practical guidance:**
- **Classification and NER** (small encoder models): CPU is often fast enough. A DistilBERT model processes 100+ texts per second on a modern CPU. GPU helps at scale (10K+ texts).
- **Text generation** (decoder models): GPU is nearly always necessary. GPT-2 generates at 2-5 tokens/sec on CPU vs 30-50 tokens/sec on a T4 GPU. Larger models are unusably slow on CPU.
- **Batch processing** (any task): GPU wins decisively. The parallelism of GPUs means larger batches get proportionally faster. CPU does not scale the same way.

---

## 5. Reading Model Cards: How to Choose

### 5.1 What Is a Model Card?

Every model on the Hub should have a model card — a README that describes the model, its training data, intended use, limitations, and evaluation results. Good model cards are the difference between "I am using a model I understand" and "I am running mystery code from the internet."

Here is what to look for:

```python
# You can read model card info programmatically
from huggingface_hub import model_info

info = model_info("distilbert/distilbert-base-uncased-finetuned-sst-2-english")

print(f"Model: {info.id}")
print(f"Downloads (last month): {info.downloads:,}")
print(f"Likes: {info.likes:,}")
print(f"Library: {info.library_name}")
print(f"Pipeline tag: {info.pipeline_tag}")
print(f"License: {info.card_data.license if info.card_data else 'Unknown'}")
print(f"Tags: {info.tags}")
```

### 5.2 Reading a Model Card in Practice

Let us walk through evaluating a real model card. A good model card answers five critical questions before you commit to using it.

```python
# Walk through the model card evaluation process for a concrete model
# Model: distilbert/distilbert-base-uncased-finetuned-sst-2-english

evaluation = {
    "1. What was it trained on?": {
        "answer": "SST-2 (Stanford Sentiment Treebank) — movie review snippets",
        "implication": "Will work well on short English text with clear sentiment. "
                       "May struggle with sarcasm, domain-specific jargon, or long-form reviews.",
    },
    "2. What are the reported metrics?": {
        "answer": "91.3% accuracy on SST-2 test set",
        "implication": "Strong on the benchmark, but SST-2 is binary (positive/negative). "
                       "No neutral class. No multi-label. Test on YOUR data.",
    },
    "3. What are the known limitations?": {
        "answer": "English only. Short text optimized. May reflect biases in movie reviews.",
        "implication": "Do not use for multilingual. Do not use for long documents without chunking. "
                       "Test for demographic bias before deploying in sensitive contexts.",
    },
    "4. What is the model architecture?": {
        "answer": "DistilBERT — 6-layer, 768-hidden, 12-heads, 66M params",
        "implication": "Small and fast. Can run on CPU in production. "
                       "But less capable than full BERT or RoBERTa for nuanced tasks.",
    },
    "5. Is it actively maintained?": {
        "answer": "Official Hugging Face model, regularly tested with library updates",
        "implication": "Safe to depend on. Will not break with transformers upgrades.",
    },
}

print("Model Card Evaluation: distilbert-base-uncased-finetuned-sst-2-english")
print("=" * 70)
for question, details in evaluation.items():
    print(f"\n  {question}")
    print(f"    Finding:     {details['answer']}")
    print(f"    Implication: {details['implication']}")

print("\nRule of thumb: if the model card does not answer these five questions,")
print("treat the model with extra caution. Missing documentation is a red flag.")
```

### 5.3 The Model Selection Checklist

When choosing a model from the Hub, check these things in order:

**1. Task match.** Does the model do what you need? Check the `pipeline_tag`.

**2. Downloads and likes.** High numbers suggest community trust and battle-testing. A model with 10M downloads has had more bugs found and fixed than one with 100 downloads.

**3. License.** Can you use it commercially? Common licenses:
- `apache-2.0` — fully permissive, commercial use OK
- `mit` — fully permissive, commercial use OK
- `cc-by-4.0` — commercial OK with attribution
- `llama3` — Meta's license, commercial with restrictions
- `cc-by-nc-4.0` — non-commercial only

**4. Model size.** Check the number of parameters and whether it fits in your memory:

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("distilbert-base-uncased")

# Count parameters
total_params = sum(p.numel() for p in model.parameters())
trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)

print(f"Total parameters: {total_params:,}")
print(f"Trainable parameters: {trainable_params:,}")

# Rough memory estimate (fp32: 4 bytes per parameter)
memory_fp32_gb = total_params * 4 / 1e9
memory_fp16_gb = total_params * 2 / 1e9
print(f"Memory estimate (fp32): {memory_fp32_gb:.2f} GB")
print(f"Memory estimate (fp16): {memory_fp16_gb:.2f} GB")
```

**5. Evaluation results.** Good model cards report benchmark scores. Compare against other models for the same task.

**6. Training data.** What was the model trained on? This affects bias, domain coverage, and language support.

**7. Last updated.** Models from 2021 may be outdated. The field moves fast.

### 5.3 Comparing Models for the Same Task

```python
from transformers import pipeline
import time

# Compare two sentiment models
models = [
    "distilbert/distilbert-base-uncased-finetuned-sst-2-english",
    "nlptown/bert-base-multilingual-uncased-sentiment",
]

test_texts = [
    "This is the best thing I've ever used!",
    "It works, but the documentation could be better.",
    "Total waste of money. Completely broken.",
    "Je suis tres content de ce produit!",  # French
]

for model_name in models:
    print(f"\nModel: {model_name}")
    print("-" * 60)
    
    start = time.time()
    classifier = pipeline("sentiment-analysis", model=model_name)
    load_time = time.time() - start
    print(f"  Load time: {load_time:.2f}s")
    
    for text in test_texts:
        start = time.time()
        result = classifier(text)[0]
        inference_time = time.time() - start
        print(f"  [{result['label']:12s} {result['score']:.3f}] "
              f"({inference_time*1000:.0f}ms) {text[:45]}")
```

Notice how the multilingual model handles French text while the English-only model does not. Model selection is about matching capabilities to requirements.

---

## 6. Batching and Performance

### 6.1 Batch Processing

Processing multiple inputs at once is significantly faster than one at a time, because the GPU can parallelize the computation.

```python
from transformers import pipeline
import time

classifier = pipeline(
    "sentiment-analysis",
    device=0 if torch.cuda.is_available() else -1
)

# Generate some test data
texts = [f"This product is {'great' if i % 2 == 0 else 'terrible'}!" 
         for i in range(100)]

# Method 1: One at a time (slow)
start = time.time()
results_sequential = [classifier(text) for text in texts]
sequential_time = time.time() - start

# Method 2: Batched (fast)
start = time.time()
results_batched = classifier(texts, batch_size=32)
batched_time = time.time() - start

print(f"Sequential: {sequential_time:.2f}s ({len(texts)/sequential_time:.0f} texts/sec)")
print(f"Batched:    {batched_time:.2f}s ({len(texts)/batched_time:.0f} texts/sec)")
print(f"Speedup:    {sequential_time/batched_time:.1f}x")
```

### 6.2 Truncation and Padding

When processing batches, all inputs need to be the same length. The tokenizer handles this with padding and truncation.

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")

texts = [
    "Short text.",
    "A slightly longer piece of text with more words in it.",
    "This is the longest text of the three and it contains many more words to demonstrate how padding works with different lengths.",
]

# Without padding — each has different length
for text in texts:
    tokens = tokenizer(text)
    print(f"  Length {len(tokens['input_ids']):3d}: {text[:50]}")

# With padding — all same length
batch = tokenizer(
    texts,
    padding=True,       # Pad shorter sequences
    truncation=True,    # Truncate longer sequences
    max_length=128,     # Maximum length
    return_tensors="pt" # Return PyTorch tensors
)

print(f"\nBatch shape: {batch['input_ids'].shape}")
print(f"Attention mask shape: {batch['attention_mask'].shape}")

# The attention mask shows which tokens are real (1) vs padding (0)
for i, text in enumerate(texts):
    real_tokens = batch['attention_mask'][i].sum().item()
    total_tokens = batch['attention_mask'][i].shape[0]
    print(f"  Text {i+1}: {real_tokens} real tokens, "
          f"{total_tokens - real_tokens} padding tokens")
```

---

## 7. Local Inference vs the Inference API

### 7.1 The Hugging Face Inference API

You do not always have to run models locally. Hugging Face offers a free Inference API for testing and a paid Inference Endpoints service for production.

```python
import requests

# Free Inference API (rate-limited, good for testing)
API_URL = "https://api-inference.huggingface.co/models/distilbert/distilbert-base-uncased-finetuned-sst-2-english"

# You can use this without a token for public models (rate-limited)
# For higher limits, add your HF token:
# headers = {"Authorization": "Bearer hf_YOUR_TOKEN"}
headers = {}

def query_inference_api(payload):
    response = requests.post(API_URL, headers=headers, json=payload)
    return response.json()

result = query_inference_api({"inputs": "I love open source AI!"})
print(f"Inference API result: {result}")
```

### 7.2 When to Use What

| Factor | Local Inference | Inference API | Cloud API (OpenAI, etc.) |
|--------|----------------|---------------|--------------------------|
| **Cost** | Hardware cost only | Free tier + paid plans | Per-token |
| **Latency** | Depends on hardware | Network + inference | Network + inference |
| **Privacy** | Data stays local | Data goes to HF | Data goes to provider |
| **Rate limits** | None | Yes (free tier) | Yes |
| **Model variety** | Any model on Hub | Any model on Hub | Provider's models only |
| **Setup effort** | Install libraries + GPU | API call | API call |
| **Scalability** | You manage it | HF manages it | Provider manages it |
| **Quality** | Model-dependent | Same model | Frontier models |

**Decision framework:**

```
Need frontier quality (GPT-4o, Claude level)?
  → Cloud API — open source hasn't caught up for the hardest tasks

Need data privacy?
  → Local inference — nothing leaves your machine

Processing high volume on a budget?
  → Local inference — amortize the hardware cost

Just prototyping?
  → Inference API — zero setup

Need production reliability without managing GPUs?
  → Inference Endpoints (paid HF) or cloud API
```

### 7.3 Model Caching

When you first load a model, Hugging Face downloads it to your local cache. Subsequent loads are instant.

```python
import os

# Default cache directory
cache_dir = os.path.expanduser("~/.cache/huggingface/hub")

# Check cache size
if os.path.exists(cache_dir):
    total_size = 0
    for dirpath, dirnames, filenames in os.walk(cache_dir):
        for f in filenames:
            fp = os.path.join(dirpath, f)
            total_size += os.path.getsize(fp)
    print(f"Cache directory: {cache_dir}")
    print(f"Total cache size: {total_size / 1e9:.2f} GB")
else:
    print("No cache directory found yet — load a model first!")

# You can specify a custom cache directory
# model = AutoModel.from_pretrained("gpt2", cache_dir="/my/custom/cache")

# Or clear the cache (careful — you'll need to re-download models)
# import shutil
# shutil.rmtree(cache_dir)
```

---

## 8. A Complete Example: Building a Text Analysis Pipeline

Let us put everything together and build a practical text analysis pipeline that combines multiple models.

```python
from transformers import pipeline
import torch
import json

class TextAnalyzer:
    """Analyze text using multiple Hugging Face models."""
    
    def __init__(self):
        device = 0 if torch.cuda.is_available() else -1
        print("Loading models...")
        
        self.sentiment = pipeline(
            "sentiment-analysis",
            device=device
        )
        self.ner = pipeline(
            "ner",
            aggregation_strategy="simple",
            device=device
        )
        self.zero_shot = pipeline(
            "zero-shot-classification",
            model="facebook/bart-large-mnli",
            device=device
        )
        print("All models loaded!")
    
    def analyze(self, text, categories=None):
        """Run full analysis on a piece of text."""
        if categories is None:
            categories = ["technology", "business", "science", "politics", "entertainment"]
        
        # Sentiment
        sentiment_result = self.sentiment(text)[0]
        
        # Named entities
        entities = self.ner(text)
        
        # Topic classification
        topic_result = self.zero_shot(text, categories)
        
        return {
            "text": text,
            "sentiment": {
                "label": sentiment_result["label"],
                "score": round(sentiment_result["score"], 4),
            },
            "entities": [
                {
                    "text": e["word"],
                    "type": e["entity_group"],
                    "score": round(e["score"], 4),
                }
                for e in entities
            ],
            "topics": {
                label: round(score, 4)
                for label, score in zip(
                    topic_result["labels"][:3],
                    topic_result["scores"][:3]
                )
            },
        }


# Use it
analyzer = TextAnalyzer()

texts = [
    "Apple CEO Tim Cook announced new AI features for the iPhone at WWDC in San Francisco.",
    "Researchers at MIT published a breakthrough paper on protein folding prediction.",
    "The Federal Reserve raised interest rates by 25 basis points, surprising Wall Street analysts.",
]

for text in texts:
    result = analyzer.analyze(text)
    print(f"\nText: {result['text'][:70]}...")
    print(f"  Sentiment: {result['sentiment']['label']} ({result['sentiment']['score']})")
    print(f"  Entities:  {[e['text'] + ' (' + e['type'] + ')' for e in result['entities']]}")
    print(f"  Topics:    {result['topics']}")
```

This is a real, useful tool. No API keys, no costs per request, runs entirely on your machine. The total model size is under 2GB, and it processes text in milliseconds on a GPU.

---

## 9. Key Takeaways

1. **Hugging Face is the ecosystem.** The Hub has 500K+ models. The `transformers` library loads any of them. The `pipeline` API wraps everything in one line.

2. **Seven tasks, seven lines.** Sentiment, generation, zero-shot classification, fill-mask, QA, summarization, and NER are all one pipeline call away.

3. **Model cards matter.** Check the task, downloads, license, size, training data, and evaluation results before using a model in production.

4. **Auto classes handle the plumbing.** `AutoTokenizer` and `AutoModelFor*` detect the right architecture from the model config. You do not need to know whether it is BERT, GPT, or T5 under the hood.

5. **Batch for performance.** Processing multiple inputs at once is dramatically faster than one at a time on GPU.

6. **Local vs API is a real trade-off.** Local gives you privacy, no costs, and no rate limits. APIs give you frontier quality and zero infrastructure.

---

## What's Next

In **Chapter 41: Image Generation**, we leave the text domain and use the `diffusers` library to generate images locally with Stable Diffusion. Same ecosystem, same patterns, entirely new modality.

Then in **Chapter 42**, we generate embeddings with Sentence Transformers — the local equivalent of the OpenAI embeddings API you used in Part 3. And in **Chapter 43**, we synthesize everything into a comprehensive model selection framework.

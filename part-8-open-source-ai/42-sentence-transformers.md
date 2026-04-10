<!--
  CHAPTER: 42
  TITLE: Embeddings & Sentence Transformers
  PART: 8 — Open Source AI & Inference
  PHASE: 2 — Become an Expert
  PREREQS: Ch 1 (Embeddings & Similarity), Ch 14 (Semantic Search), Ch 36 (Neural Networks), Ch 40 (Hugging Face & Pipelines)
  KEY_TOPICS: sentence transformers, embeddings, MTEB benchmark, semantic search, cosine similarity, local embeddings, embedding models, bi-encoders, cross-encoders
  DIFFICULTY: Intermediate
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 42: Embeddings & Sentence Transformers

> **Part 8 — Open Source AI & Inference** | Phase 2: Become an Expert | Prerequisites: Ch 1, Ch 14, Ch 36, Ch 40 | Difficulty: Intermediate | Language: Python

In Chapter 1, you learned that embeddings are vectors that capture semantic meaning. In Chapter 14, you used OpenAI's embedding API to build semantic search. In Chapter 36, you saw how neural networks produce these vectors through learned weights. Now you generate them yourself, locally, with no API calls and no per-request costs.

The Sentence Transformers library is the standard way to generate embeddings locally. It provides pre-trained models that convert text into dense vectors — the same kind of vectors you used for semantic search in Part 3, but now you control the model, the quality, and the cost. You can process a million documents without spending a dollar on API fees.

This chapter covers generating embeddings locally, choosing the right embedding model for your task, understanding the MTEB benchmark, building a local semantic search system, and comparing local vs API embedding quality.

### In This Chapter
- Sentence Transformers: the library for local embeddings
- Generating embeddings for sentences, paragraphs, and documents
- MTEB benchmark: how embedding models are evaluated
- Choosing the right embedding model for your task
- Building local semantic search (no API calls)
- Bi-encoders vs cross-encoders: speed vs accuracy
- Comparing API embeddings (OpenAI) vs local embeddings

### Related Chapters
- **Ch 1 (Embeddings & Similarity)** — spirals from concept to generating your own
- **Ch 14 (Semantic Search)** — spirals from API-based search to local search
- **Ch 36 (Neural Networks)** — neural nets produce these vectors through learned weights
- **Ch 40 (Hugging Face & Pipelines)** — same ecosystem, specialized for embeddings
- **Ch 44 (RAG vs Fine-Tuning)** — embeddings in fine-tuning decisions

---

## 1. Sentence Transformers: The Library

### 1.1 What Are Sentence Transformers?

Regular transformer models like BERT produce token-level embeddings — one vector per token. That is useful for some tasks, but for semantic search, similarity, and clustering, you need a single vector for the entire sentence or paragraph. Sentence Transformers solve this.

A Sentence Transformer is a BERT-family model that has been fine-tuned specifically to produce meaningful sentence-level embeddings. The key property: sentences with similar meaning have similar embeddings (high cosine similarity), and sentences with different meaning have different embeddings (low cosine similarity).

```python
# Install sentence-transformers
# !pip install sentence-transformers

from sentence_transformers import SentenceTransformer
import numpy as np

# Load a model — this downloads it on first run (~90 MB)
model = SentenceTransformer("all-MiniLM-L6-v2")

# Encode a single sentence
sentence = "Machine learning is transforming how we build software."
embedding = model.encode(sentence)

print(f"Sentence: '{sentence}'")
print(f"Embedding shape: {embedding.shape}")    # (384,)
print(f"Embedding dtype: {embedding.dtype}")     # float32
print(f"First 10 values: {embedding[:10]}")
print(f"L2 norm: {np.linalg.norm(embedding):.4f}")  # ~1.0 (normalized)
```

### 1.2 Semantic Similarity in Action

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")

# Encode multiple sentences
sentences = [
    "The cat sat on the mat.",
    "A kitten was sitting on the rug.",
    "The stock market crashed yesterday.",
    "Financial markets experienced a significant downturn.",
    "I enjoy programming in Python.",
    "Python is my favorite programming language.",
]

embeddings = model.encode(sentences)

# Compute pairwise cosine similarity
similarities = cos_sim(embeddings, embeddings)

print("Pairwise cosine similarities:")
print("-" * 70)
for i in range(len(sentences)):
    for j in range(i + 1, len(sentences)):
        sim = similarities[i][j].item()
        indicator = "***" if sim > 0.7 else "   "
        print(f"  {indicator} {sim:.4f}  '{sentences[i][:35]}' vs '{sentences[j][:35]}'")

print("\n*** = high similarity (>0.7)")
```

You will see that semantically similar sentences (cat/kitten, stock market/financial markets, Python/Python) have high similarity scores, while unrelated sentences have low scores. This works even when the sentences use completely different words.

### 1.3 Encoding Options

```python
model = SentenceTransformer("all-MiniLM-L6-v2")

# Basic encoding
embeddings = model.encode(["Hello world"])

# Batch encoding (faster for many sentences)
sentences = [f"Sentence number {i}" for i in range(100)]
embeddings = model.encode(
    sentences,
    batch_size=32,           # Process 32 at a time
    show_progress_bar=True,  # Show progress for large batches
    normalize_embeddings=True,  # L2 normalize (cosine sim = dot product)
)
print(f"Encoded {len(sentences)} sentences → shape {embeddings.shape}")

# Convert to different formats
import torch

# NumPy array (default)
np_embeddings = model.encode(sentences, convert_to_numpy=True)
print(f"NumPy: {type(np_embeddings)}, dtype={np_embeddings.dtype}")

# PyTorch tensor
pt_embeddings = model.encode(sentences, convert_to_tensor=True)
print(f"PyTorch: {type(pt_embeddings)}, device={pt_embeddings.device}")

# GPU encoding (if available)
if torch.cuda.is_available():
    model = model.to("cuda")
    gpu_embeddings = model.encode(sentences, convert_to_tensor=True)
    print(f"GPU embeddings device: {gpu_embeddings.device}")
```

---

## 2. Choosing an Embedding Model

### 2.1 The MTEB Benchmark

The Massive Text Embedding Benchmark (MTEB) evaluates embedding models across multiple tasks. It is the standard way to compare models.

```python
# MTEB evaluates models on these task categories:
mteb_tasks = {
    "Classification":    "Assign labels to text (sentiment, topic, etc.)",
    "Clustering":        "Group similar texts together",
    "PairClassification": "Determine if two texts are similar/related",
    "Reranking":         "Reorder a list of texts by relevance to a query",
    "Retrieval":         "Find relevant documents for a query (semantic search)",
    "STS":               "Semantic Textual Similarity — rate similarity of pairs",
    "Summarization":     "Evaluate summary quality via embedding similarity",
}

print("MTEB Task Categories:")
print("-" * 70)
for task, description in mteb_tasks.items():
    print(f"  {task:20s}: {description}")

print("\nFor RAG and semantic search, focus on the 'Retrieval' scores.")
print("For general-purpose use, look at the overall MTEB average.")
print("Leaderboard: https://huggingface.co/spaces/mteb/leaderboard")
```

### 2.2 Popular Embedding Models

```python
from sentence_transformers import SentenceTransformer
import time

# Popular embedding models ranked by use case
models_info = {
    "all-MiniLM-L6-v2": {
        "params": "22M",
        "dims": 384,
        "speed": "very fast",
        "quality": "good",
        "best_for": "General-purpose, when speed matters",
    },
    "all-mpnet-base-v2": {
        "params": "109M",
        "dims": 768,
        "speed": "fast",
        "quality": "very good",
        "best_for": "General-purpose, best quality in the small model range",
    },
    "BAAI/bge-small-en-v1.5": {
        "params": "33M",
        "dims": 384,
        "speed": "very fast",
        "quality": "very good",
        "best_for": "Retrieval/search, strong MTEB scores for its size",
    },
    "BAAI/bge-base-en-v1.5": {
        "params": "109M",
        "dims": 768,
        "speed": "fast",
        "quality": "excellent",
        "best_for": "Retrieval/search, top MTEB scores",
    },
    "BAAI/bge-large-en-v1.5": {
        "params": "335M",
        "dims": 1024,
        "speed": "moderate",
        "quality": "excellent",
        "best_for": "When quality matters more than speed",
    },
}

print("Popular Embedding Models:")
print("=" * 80)
for name, info in models_info.items():
    print(f"\n  {name}")
    print(f"    Parameters: {info['params']:>6s}  |  Dimensions: {info['dims']:>5d}  |  Speed: {info['speed']}")
    print(f"    Quality: {info['quality']}  |  Best for: {info['best_for']}")
```

### 2.3 Benchmarking Models Head-to-Head

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim
import time
import numpy as np

# Test sentences: some pairs are similar, some are not
test_pairs = [
    # Similar pairs (should have high similarity)
    ("How do I reset my password?", "I forgot my password and need to change it"),
    ("The restaurant has great Italian food", "Amazing pasta and pizza at this place"),
    ("Machine learning model training", "Training neural networks on data"),
    # Dissimilar pairs (should have low similarity)
    ("How do I reset my password?", "The restaurant has great Italian food"),
    ("Machine learning model training", "I forgot my password and need to change it"),
    ("The weather is nice today", "Quantum computing uses qubits"),
]

models_to_test = [
    "all-MiniLM-L6-v2",
    "all-mpnet-base-v2",
    "BAAI/bge-small-en-v1.5",
]

for model_name in models_to_test:
    model = SentenceTransformer(model_name)
    
    sentences_a = [pair[0] for pair in test_pairs]
    sentences_b = [pair[1] for pair in test_pairs]
    
    # Benchmark speed
    start = time.time()
    embeddings_a = model.encode(sentences_a)
    embeddings_b = model.encode(sentences_b)
    elapsed = time.time() - start
    
    # Compute similarities
    similarities = [
        cos_sim(embeddings_a[i:i+1], embeddings_b[i:i+1]).item()
        for i in range(len(test_pairs))
    ]
    
    print(f"\nModel: {model_name}")
    print(f"  Encoding time: {elapsed*1000:.0f}ms for {len(test_pairs)*2} sentences")
    print(f"  Dimensions: {embeddings_a.shape[1]}")
    print(f"  Similarities:")
    for i, (a, b) in enumerate(test_pairs):
        tag = "SIMILAR" if i < 3 else "DIFFERENT"
        print(f"    [{tag:9s}] {similarities[i]:.4f}  '{a[:30]}' vs '{b[:30]}'")
    
    # Quality metric: gap between similar and dissimilar pairs
    avg_similar = np.mean(similarities[:3])
    avg_dissimilar = np.mean(similarities[3:])
    print(f"  Avg similar: {avg_similar:.4f}, Avg dissimilar: {avg_dissimilar:.4f}")
    print(f"  Separation gap: {avg_similar - avg_dissimilar:.4f} (higher = better)")
```

---

## 3. Building Local Semantic Search

### 3.1 The Complete Pipeline

This is the local equivalent of the semantic search you built in Ch 14 with OpenAI's API. No API calls, no costs.

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim
import numpy as np
import json
from typing import Optional


class LocalSemanticSearch:
    """A complete semantic search system with no API dependencies."""
    
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)
        self.documents: list[str] = []
        self.metadata: list[dict] = []
        self.embeddings: Optional[np.ndarray] = None
        print(f"Loaded model: {model_name} ({self.model.get_sentence_embedding_dimension()}d)")
    
    def add_documents(self, documents: list[str], metadata: Optional[list[dict]] = None):
        """Add documents to the search index."""
        new_embeddings = self.model.encode(
            documents,
            show_progress_bar=len(documents) > 100,
            normalize_embeddings=True,  # Pre-normalize for fast cosine similarity
        )
        
        self.documents.extend(documents)
        self.metadata.extend(metadata or [{}] * len(documents))
        
        if self.embeddings is None:
            self.embeddings = new_embeddings
        else:
            self.embeddings = np.vstack([self.embeddings, new_embeddings])
        
        print(f"Added {len(documents)} documents. Total: {len(self.documents)}")
    
    def search(self, query: str, top_k: int = 5) -> list[dict]:
        """Search for documents similar to the query."""
        if self.embeddings is None or len(self.documents) == 0:
            return []
        
        # Encode the query
        query_embedding = self.model.encode(
            [query], 
            normalize_embeddings=True
        )
        
        # Cosine similarity (dot product since embeddings are normalized)
        scores = np.dot(self.embeddings, query_embedding.T).flatten()
        
        # Get top-k indices
        top_indices = np.argsort(scores)[::-1][:top_k]
        
        results = []
        for idx in top_indices:
            results.append({
                "document": self.documents[idx],
                "score": float(scores[idx]),
                "metadata": self.metadata[idx],
                "index": int(idx),
            })
        
        return results
    
    def save_index(self, path: str):
        """Save the index to disk."""
        np.save(f"{path}_embeddings.npy", self.embeddings)
        with open(f"{path}_data.json", "w") as f:
            json.dump({
                "documents": self.documents,
                "metadata": self.metadata,
            }, f)
        print(f"Index saved to {path}_embeddings.npy and {path}_data.json")
    
    def load_index(self, path: str):
        """Load the index from disk."""
        self.embeddings = np.load(f"{path}_embeddings.npy")
        with open(f"{path}_data.json") as f:
            data = json.load(f)
            self.documents = data["documents"]
            self.metadata = data["metadata"]
        print(f"Loaded {len(self.documents)} documents from {path}")


# Build a search index
search = LocalSemanticSearch("all-MiniLM-L6-v2")

# Add a knowledge base
documents = [
    "Python is a high-level programming language known for its readability and versatility.",
    "JavaScript is the language of the web, running in browsers and on Node.js servers.",
    "Rust provides memory safety without garbage collection through its ownership system.",
    "Docker containers package applications with their dependencies for consistent deployment.",
    "Kubernetes orchestrates container deployments across clusters of machines.",
    "PostgreSQL is a powerful open-source relational database with JSON support.",
    "Redis is an in-memory data store used for caching and real-time applications.",
    "GraphQL is a query language for APIs that lets clients request exactly the data they need.",
    "REST APIs use HTTP methods to perform CRUD operations on resources.",
    "Git is a distributed version control system for tracking changes in source code.",
    "Machine learning models learn patterns from data to make predictions.",
    "Neural networks are computing systems inspired by biological neural networks.",
    "Transformers use self-attention mechanisms to process sequential data in parallel.",
    "Fine-tuning adapts a pre-trained model to a specific task with task-specific data.",
    "RAG combines retrieval with generation to ground LLM responses in source documents.",
]

metadata = [{"category": "lang"} if i < 3 
            else {"category": "infra"} if i < 6
            else {"category": "data"} if i < 9
            else {"category": "tools"} if i < 10
            else {"category": "ml"} for i in range(len(documents))]

search.add_documents(documents, metadata)

# Search
queries = [
    "How do I deploy my application?",
    "What database should I use?",
    "How do neural networks work?",
    "best language for web development",
]

for query in queries:
    print(f"\nQuery: '{query}'")
    print("-" * 60)
    results = search.search(query, top_k=3)
    for i, result in enumerate(results):
        print(f"  {i+1}. [{result['score']:.4f}] {result['document'][:70]}...")
        print(f"     Category: {result['metadata'].get('category', 'unknown')}")
```

### 3.2 Scaling to Large Document Sets

```python
import time

# Performance benchmarking
model = SentenceTransformer("all-MiniLM-L6-v2")

# Simulate a large document set
num_docs = 10000
fake_docs = [f"This is document number {i} about topic {i % 50}" for i in range(num_docs)]

# Encoding speed
start = time.time()
embeddings = model.encode(
    fake_docs, 
    batch_size=128, 
    show_progress_bar=True,
    normalize_embeddings=True,
)
encode_time = time.time() - start

print(f"\nEncoding {num_docs} documents:")
print(f"  Time: {encode_time:.2f}s ({num_docs/encode_time:.0f} docs/sec)")
print(f"  Memory: {embeddings.nbytes / 1e6:.1f} MB")

# Search speed
query_embedding = model.encode(["deploy application"], normalize_embeddings=True)

start = time.time()
for _ in range(100):
    scores = np.dot(embeddings, query_embedding.T).flatten()
    top_k = np.argsort(scores)[::-1][:10]
search_time = (time.time() - start) / 100

print(f"\nSearch over {num_docs} documents:")
print(f"  Time per query: {search_time*1000:.2f}ms")
print(f"  Queries per second: {1/search_time:.0f}")
print(f"\nFor >100K documents, consider FAISS or Annoy for approximate nearest neighbors")
```

### 3.3 Using FAISS for Larger Indexes

```python
# For production-scale search (100K+ documents), use FAISS
# !pip install faiss-cpu  # or faiss-gpu

import faiss
import numpy as np

def build_faiss_index(embeddings: np.ndarray) -> faiss.IndexFlatIP:
    """Build a FAISS index for fast similarity search."""
    dimension = embeddings.shape[1]
    
    # Normalize for cosine similarity
    faiss.normalize_L2(embeddings)
    
    # IndexFlatIP = exact inner product search (cosine sim on normalized vectors)
    index = faiss.IndexFlatIP(dimension)
    index.add(embeddings)
    
    return index

def search_faiss(index, query_embedding: np.ndarray, top_k: int = 5):
    """Search the FAISS index."""
    faiss.normalize_L2(query_embedding)
    scores, indices = index.search(query_embedding, top_k)
    return scores[0], indices[0]


# Build index
model = SentenceTransformer("all-MiniLM-L6-v2")

documents = [
    "Python is great for machine learning",
    "JavaScript powers the modern web",
    "Rust is fast and memory-safe",
    "Go is excellent for microservices",
    "Docker simplifies deployment",
    "Kubernetes manages container orchestration",
    "PostgreSQL is a robust relational database",
    "MongoDB is a popular document database",
    "Redis provides in-memory caching",
    "GraphQL offers flexible API queries",
]

embeddings = model.encode(documents, convert_to_numpy=True)

# Build FAISS index
index = build_faiss_index(embeddings.copy())
print(f"FAISS index built: {index.ntotal} vectors, {index.d} dimensions")

# Search
query = "What's good for building APIs?"
query_embedding = model.encode([query], convert_to_numpy=True)

scores, indices = search_faiss(index, query_embedding.copy(), top_k=3)

print(f"\nQuery: '{query}'")
for score, idx in zip(scores, indices):
    print(f"  [{score:.4f}] {documents[idx]}")
```

---

## 4. Bi-Encoders vs Cross-Encoders

### 4.1 Two Approaches to Similarity

Sentence Transformers supports two architectures for text similarity:

**Bi-encoders** encode each text independently. Fast, scalable, but less accurate.
**Cross-encoders** process both texts together. Slow, not scalable, but more accurate.

```
Bi-encoder (SentenceTransformer):
  Text A → Encoder → Embedding A  ─┐
                                     ├─→ Cosine Similarity → 0.87
  Text B → Encoder → Embedding B  ─┘
  
  Speed: O(1) per comparison (embeddings are pre-computed)
  Use: Search over millions of documents

Cross-encoder (CrossEncoder):
  [Text A, Text B] → Encoder → Score: 0.92
  
  Speed: O(n) — must re-encode for every pair
  Use: Re-rank top-k results from a bi-encoder
```

### 4.2 Cross-Encoders in Practice

```python
from sentence_transformers import CrossEncoder

# Load a cross-encoder for re-ranking
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

# Score pairs directly
pairs = [
    ("How do I deploy my app?", "Docker containers package applications for deployment"),
    ("How do I deploy my app?", "Python is a programming language"),
    ("How do I deploy my app?", "Kubernetes orchestrates container deployments"),
]

scores = cross_encoder.predict(pairs)

print("Cross-Encoder Scores:")
print("-" * 70)
for (query, doc), score in zip(pairs, scores):
    print(f"  {score:7.4f}  Q: '{query[:35]}' → D: '{doc[:40]}'")
```

### 4.3 The Two-Stage Retrieval Pattern

The production pattern: use a bi-encoder to retrieve candidates quickly, then use a cross-encoder to re-rank them accurately.

```python
from sentence_transformers import SentenceTransformer, CrossEncoder
from sentence_transformers.util import cos_sim
import numpy as np

# Stage 1: Bi-encoder for fast retrieval
bi_encoder = SentenceTransformer("all-MiniLM-L6-v2")

# Stage 2: Cross-encoder for accurate re-ranking
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

# Knowledge base
documents = [
    "To deploy a web application, you can use Docker containers for consistent environments.",
    "Python virtual environments isolate project dependencies from the system Python.",
    "Kubernetes provides automatic scaling, load balancing, and self-healing for containers.",
    "Git branches allow parallel development streams that can be merged later.",
    "CI/CD pipelines automate testing and deployment when code is pushed to a repository.",
    "Environment variables store configuration that varies between deployment environments.",
    "Load balancers distribute incoming traffic across multiple server instances.",
    "SSL certificates encrypt data in transit between clients and servers.",
    "Database migrations track schema changes and apply them consistently across environments.",
    "Health checks monitor application status and restart failed instances automatically.",
]

# Encode all documents with bi-encoder
doc_embeddings = bi_encoder.encode(documents, normalize_embeddings=True)

query = "How do I get my app running in production?"

# Stage 1: Bi-encoder retrieval (fast — works on millions of docs)
query_embedding = bi_encoder.encode([query], normalize_embeddings=True)
bi_scores = np.dot(doc_embeddings, query_embedding.T).flatten()
top_k = 5
candidate_indices = np.argsort(bi_scores)[::-1][:top_k]

print(f"Query: '{query}'")
print(f"\nStage 1 — Bi-Encoder Top {top_k}:")
for idx in candidate_indices:
    print(f"  [{bi_scores[idx]:.4f}] {documents[idx][:70]}")

# Stage 2: Cross-encoder re-ranking (slow but accurate — only on top-k)
candidate_docs = [documents[idx] for idx in candidate_indices]
pairs = [(query, doc) for doc in candidate_docs]
cross_scores = cross_encoder.predict(pairs)

# Re-rank
reranked = sorted(
    zip(candidate_indices, cross_scores, candidate_docs),
    key=lambda x: x[1],
    reverse=True
)

print(f"\nStage 2 — Cross-Encoder Re-Ranked:")
for idx, score, doc in reranked:
    print(f"  [{score:.4f}] {doc[:70]}")

print("\nNotice how the cross-encoder changes the ranking!")
```

---

## 5. Comparing API vs Local Embeddings

### 5.1 Quality Comparison

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim
import numpy as np

# Local model
local_model = SentenceTransformer("all-MiniLM-L6-v2")

# For comparison with OpenAI, you would do:
# import openai
# client = openai.OpenAI()
# response = client.embeddings.create(
#     model="text-embedding-3-small",
#     input=sentences
# )
# api_embeddings = [e.embedding for e in response.data]

# Let's compare two local models of different sizes
small_model = SentenceTransformer("all-MiniLM-L6-v2")     # 22M params, 384d
large_model = SentenceTransformer("all-mpnet-base-v2")     # 109M params, 768d

# Test data: pairs with known similarity
test_data = [
    # Clearly similar pairs
    {"a": "The cat sat on the mat", "b": "A feline rested on a rug", "expected": "high"},
    {"a": "How much does it cost?", "b": "What's the price?", "expected": "high"},
    {"a": "The server is down", "b": "Our backend systems are experiencing an outage", "expected": "high"},
    # Clearly dissimilar pairs
    {"a": "The cat sat on the mat", "b": "The stock market crashed", "expected": "low"},
    {"a": "How much does it cost?", "b": "The weather is sunny today", "expected": "low"},
    {"a": "The server is down", "b": "I like chocolate ice cream", "expected": "low"},
    # Tricky pairs (related but different)
    {"a": "Apple released a new iPhone", "b": "I ate an apple for lunch", "expected": "low"},
    {"a": "Python is a great language", "b": "I saw a python at the zoo", "expected": "low"},
    {"a": "The bank is closed", "b": "We walked along the river bank", "expected": "low"},
]

for model_name, model in [("MiniLM-L6 (22M)", small_model), ("MPNet-base (109M)", large_model)]:
    print(f"\nModel: {model_name}")
    print("-" * 70)
    
    correct = 0
    for item in test_data:
        emb_a = model.encode([item["a"]])
        emb_b = model.encode([item["b"]])
        sim = cos_sim(emb_a, emb_b).item()
        
        predicted = "high" if sim > 0.5 else "low"
        match = "OK" if predicted == item["expected"] else "MISS"
        if match == "OK":
            correct += 1
        
        print(f"  [{match:4s}] {sim:.4f} (expect {item['expected']:4s})  "
              f"'{item['a'][:25]}' vs '{item['b'][:25]}'")
    
    print(f"  Accuracy: {correct}/{len(test_data)} ({100*correct/len(test_data):.0f}%)")
```

### 5.2 Cost Comparison

```python
# Cost comparison: API vs local embeddings

# OpenAI text-embedding-3-small pricing (as of 2026)
api_cost_per_million_tokens = 0.02  # $0.02 per 1M tokens
avg_tokens_per_document = 150  # Typical for a paragraph

# Local model costs (assuming cloud GPU)
gpu_cost_per_hour = 0.50  # T4 on Google Cloud ~$0.50/hr
docs_per_second_local = 500  # all-MiniLM-L6-v2 on T4

# Scenario: embed 1 million documents
num_documents = 1_000_000
total_tokens = num_documents * avg_tokens_per_document

# API cost
api_cost = (total_tokens / 1_000_000) * api_cost_per_million_tokens
print("Embedding 1,000,000 documents:")
print("=" * 50)
print(f"\nAPI (OpenAI text-embedding-3-small):")
print(f"  Total tokens: {total_tokens:,.0f}")
print(f"  Cost: ${api_cost:.2f}")
print(f"  Time: ~5-10 minutes (rate-limited)")

# Local cost
local_time_seconds = num_documents / docs_per_second_local
local_time_hours = local_time_seconds / 3600
local_cost = local_time_hours * gpu_cost_per_hour
print(f"\nLocal (all-MiniLM-L6-v2 on T4 GPU):")
print(f"  Time: {local_time_seconds:.0f} seconds ({local_time_hours:.1f} hours)")
print(f"  GPU cost: ${local_cost:.2f}")

# Break-even analysis
print(f"\nBreak-even analysis:")
print(f"  API cost per doc: ${api_cost / num_documents * 1000:.4f} per 1K docs")
print(f"  Local cost per doc: ${local_cost / num_documents * 1000:.4f} per 1K docs")
print(f"  Local is {api_cost / max(local_cost, 0.01):.1f}x cheaper at this volume")
print(f"\n  For small volumes (<10K docs), the API is simpler.")
print(f"  For large volumes (>100K docs), local is much cheaper.")
```

---

## 6. Advanced Embedding Techniques

### 6.1 Instruction-Tuned Embeddings

Some newer models accept instructions that improve embedding quality for specific tasks.

```python
from sentence_transformers import SentenceTransformer

# BGE models support instruction prefixes
model = SentenceTransformer("BAAI/bge-base-en-v1.5")

# For retrieval tasks, prefix the query (not the documents!)
query_prefix = "Represent this sentence for searching relevant passages: "

query = query_prefix + "How do I handle errors in Python?"
documents = [
    "Python uses try/except blocks for exception handling.",
    "Error handling in Python involves catching exceptions with try-except.",
    "Python is a programming language created by Guido van Rossum.",
]

query_embedding = model.encode([query], normalize_embeddings=True)
doc_embeddings = model.encode(documents, normalize_embeddings=True)

similarities = np.dot(doc_embeddings, query_embedding.T).flatten()

print("With instruction prefix:")
for doc, sim in sorted(zip(documents, similarities), key=lambda x: -x[1]):
    print(f"  [{sim:.4f}] {doc}")
```

### 6.2 Multilingual Embeddings

```python
from sentence_transformers import SentenceTransformer
from sentence_transformers.util import cos_sim

# Multilingual model — supports 50+ languages
model = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")

# Same meaning in different languages
sentences = [
    "The weather is beautiful today",           # English
    "Le temps est magnifique aujourd'hui",       # French
    "Das Wetter ist heute wunderschoen",         # German
    "El tiempo es hermoso hoy",                  # Spanish
    "I love programming in Python",              # English (different topic)
]

embeddings = model.encode(sentences)
similarities = cos_sim(embeddings, embeddings)

print("Cross-Lingual Similarity:")
print("-" * 70)
for i in range(len(sentences)):
    for j in range(i + 1, len(sentences)):
        sim = similarities[i][j].item()
        flag = "***" if sim > 0.7 else "   "
        print(f"  {flag} {sim:.4f}  '{sentences[i][:35]}' vs '{sentences[j][:35]}'")

print("\n*** = high similarity (>0.7)")
print("Notice: same meaning in different languages clusters together!")
```

### 6.3 Document Chunking for Long Texts

Embedding models have a maximum input length (typically 256-512 tokens). For longer documents, you need to chunk them.

```python
from sentence_transformers import SentenceTransformer
import numpy as np


def chunk_text(text: str, chunk_size: int = 200, overlap: int = 50) -> list[str]:
    """Split text into overlapping chunks by word count."""
    words = text.split()
    chunks = []
    
    start = 0
    while start < len(words):
        end = min(start + chunk_size, len(words))
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        start += chunk_size - overlap
    
    return chunks


def embed_long_document(model, text: str, method: str = "max_pool") -> np.ndarray:
    """Embed a long document using chunking and pooling."""
    chunks = chunk_text(text)
    chunk_embeddings = model.encode(chunks, normalize_embeddings=True)
    
    if method == "mean_pool":
        # Average all chunk embeddings
        doc_embedding = np.mean(chunk_embeddings, axis=0)
    elif method == "max_pool":
        # Take the max across chunks for each dimension
        doc_embedding = np.max(chunk_embeddings, axis=0)
    elif method == "first_chunk":
        # Just use the first chunk (simplest)
        doc_embedding = chunk_embeddings[0]
    
    # Re-normalize
    doc_embedding = doc_embedding / np.linalg.norm(doc_embedding)
    return doc_embedding


model = SentenceTransformer("all-MiniLM-L6-v2")

long_text = """
Machine learning is a subset of artificial intelligence that focuses on
building systems that learn from data. Instead of being explicitly programmed
with rules, these systems identify patterns in data and use those patterns
to make predictions or decisions. The field has grown enormously in recent
years, driven by increases in computing power, availability of large datasets,
and advances in algorithms. Deep learning, a subset of machine learning
that uses neural networks with many layers, has been particularly
transformative. Applications range from image recognition and natural
language processing to autonomous vehicles and drug discovery.
""" * 3  # Make it longer by repeating

chunks = chunk_text(long_text)
print(f"Document: {len(long_text.split())} words → {len(chunks)} chunks")

embedding = embed_long_document(model, long_text, method="mean_pool")
print(f"Document embedding: {embedding.shape}")
```

---

## 7. Key Takeaways

1. **Sentence Transformers generate meaningful embeddings.** Similar sentences get similar vectors. This enables semantic search, clustering, and similarity without any API calls.

2. **Model choice matters.** `all-MiniLM-L6-v2` for speed, `all-mpnet-base-v2` for quality, `bge-*` models for retrieval. Check MTEB scores for your specific task.

3. **Bi-encoders for search, cross-encoders for re-ranking.** The two-stage pattern (retrieve with bi-encoder, re-rank with cross-encoder) gives you both speed and accuracy.

4. **Local embeddings are dramatically cheaper at scale.** For 100K+ documents, local embedding is 10-100x cheaper than API embedding.

5. **Normalize your embeddings.** With L2-normalized embeddings, cosine similarity equals the dot product, which is much faster to compute.

6. **Chunk long documents.** Embedding models have token limits. Split documents into overlapping chunks and pool the embeddings.

---

## What's Next

In **Chapter 43: Model Selection & Architecture**, we synthesize everything from Part 8 into a decision framework. You have run text models (Ch 40), image models (Ch 41), and embedding models (Ch 42). Now you need to understand when to use BERT, when to use GPT, when to use T5, and how to choose among the hundreds of thousands of models available. Chapter 43 gives you that framework.

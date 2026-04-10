<!--
  CHAPTER: 47
  TITLE: Quantization & Deployment
  PART: 9 — Fine-Tuning & Training
  PHASE: 2 — Become an Expert
  PREREQS: Ch 43 (Model Selection), Ch 45 (Fine-Tuning with LoRA)
  KEY_TOPICS: quantization, GGUF, GPTQ, AWQ, bitsandbytes, llama.cpp, Ollama, inference optimization, KV cache, batching, speculative decoding, deployment, vLLM, TGI
  DIFFICULTY: Advanced
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 47: Quantization & Deployment

> **Part 9 — Fine-Tuning & Training** | Phase 2: Become an Expert | Prerequisites: Ch 43, Ch 45 | Difficulty: Advanced | Language: Python

In Chapter 43, you previewed quantization — the technique of reducing model precision to shrink models and speed them up. In Chapter 45, you fine-tuned a model with LoRA. Now you need to get that model into production: small enough to deploy efficiently, fast enough to serve users, and cheap enough to run at scale.

Quantization is the bridge between "I have a model" and "I have a product." A 7B parameter model in full precision needs 14GB of memory and a beefy GPU. That same model quantized to 4-bit needs 3.5GB and runs on a laptop. The quality loss is usually small — 1-3% on benchmarks — while the speed and cost improvements are dramatic.

This chapter covers everything you need to deploy models efficiently: quantization formats (GGUF, GPTQ, AWQ), inference optimization (KV cache, batching, speculative decoding), and deployment platforms (Ollama, vLLM, cloud services).

### In This Chapter
- What quantization is and why it matters
- GGUF: the format for CPU/hybrid inference (llama.cpp)
- GPTQ: GPU-optimized quantization
- AWQ: activation-aware quantization
- Running quantized models with the Transformers library
- Inference optimization: KV cache, continuous batching, speculative decoding
- The quality vs size vs speed trade-off
- Deployment platforms: Ollama, vLLM, TGI, cloud services

### Related Chapters
- **Ch 15 (The RAG Pipeline)** — spirals back: quantized embedding models power efficient RAG retrieval
- **Ch 21 (Eval-Driven Development)** — spirals back: eval before and after quantization to measure quality impact
- **Ch 43 (Model Selection)** — spirals from model selection into deployment
- **Ch 45 (Fine-Tuning with LoRA)** — deploy the model you fine-tuned
- **Ch 57 (Scaling & Cost at Infra Level)** — infrastructure-level scaling

---

## 1. What Quantization Is

### 1.1 The Precision Spectrum

Every number in a neural network (weights, activations, gradients) is stored as a floating point number. The precision determines how many bits are used to represent each number — and therefore how much memory the model requires.

```python
import numpy as np
import struct

# Demonstrate precision levels
weight = 0.123456789

# fp32: 32 bits (4 bytes) — full precision
fp32 = np.float32(weight)
print(f"fp32: {fp32:.10f}  ({32} bits, {4} bytes)")

# fp16: 16 bits (2 bytes) — half precision
fp16 = np.float16(weight)
print(f"fp16: {fp16:.10f}  ({16} bits, {2} bytes)")

# bf16: 16 bits (2 bytes) — brain floating point (wider range, less precision)
# NumPy doesn't support bf16 natively, but PyTorch does
# bf16 has the same range as fp32 but with fp16-level precision
print(f"bf16: ~{fp16:.6f}       ({16} bits, {2} bytes, wider range than fp16)")

# int8: 8 bits (1 byte) — integer quantization
int8_scaled = np.int8(round(weight * 127))  # Scale to [-127, 127]
int8_reconstructed = int8_scaled / 127.0
print(f"int8: {int8_reconstructed:.10f}  ({8} bits, {1} byte)")

# int4: 4 bits (0.5 bytes) — aggressive quantization
int4_scaled = round(weight * 7)  # Scale to [-7, 7]
int4_reconstructed = int4_scaled / 7.0
print(f"int4: {int4_reconstructed:.10f}  ({4} bits, {0.5} bytes)")

print(f"\nPrecision loss:")
print(f"  fp32 -> fp16: {abs(fp32 - fp16):.10f}")
print(f"  fp32 -> int8: {abs(fp32 - int8_reconstructed):.10f}")
print(f"  fp32 -> int4: {abs(fp32 - int4_reconstructed):.10f}")
```

### 1.2 Memory Impact

```python
# Memory requirements for different model sizes and precisions

def memory_table(params_billions: float, name: str):
    """Calculate memory for different precisions."""
    params = params_billions * 1e9
    
    precisions = {
        "fp32 (32-bit)": 4,
        "fp16 (16-bit)": 2,
        "int8 (8-bit)": 1,
        "int4 (4-bit)": 0.5,
    }
    
    print(f"\n  {name} ({params_billions}B parameters):")
    for precision, bytes_per_param in precisions.items():
        memory_gb = params * bytes_per_param / 1e9
        print(f"    {precision:16s}: {memory_gb:>6.1f} GB")

print("Model Memory by Precision:")
print("=" * 50)
memory_table(1, "Phi-3 Mini / Llama 3.2 1B")
memory_table(7, "Llama 3.1 8B / Mistral 7B")
memory_table(13, "Llama 2 13B")
memory_table(70, "Llama 3.1 70B")

print(f"\n  Hardware Reference:")
print(f"    MacBook Air M2:  8-24 GB unified memory")
print(f"    RTX 4090:        24 GB VRAM")
print(f"    Google Colab T4: 16 GB VRAM")
print(f"    A100:            40/80 GB VRAM")
```

---

## 2. Quantization Formats

### 2.1 GGUF: The Local Inference Standard

GGUF (GPT-Generated Unified Format) is the format used by llama.cpp, Ollama, LM Studio, and most local inference tools. It supports mixed CPU+GPU inference, making it perfect for consumer hardware.

```python
# GGUF quantization levels — from highest quality to most compressed
gguf_variants = {
    "Q8_0": {
        "bits": 8,
        "quality": "Excellent — nearly lossless",
        "size_ratio": 0.50,  # Compared to fp16
        "use_case": "When you have enough memory and want near-perfect quality",
    },
    "Q6_K": {
        "bits": 6,
        "quality": "Very good — minimal quality loss",
        "size_ratio": 0.38,
        "use_case": "Good balance for larger models",
    },
    "Q5_K_M": {
        "bits": 5,
        "quality": "Good — slight quality loss, hard to notice",
        "size_ratio": 0.33,
        "use_case": "Recommended general-purpose quantization",
    },
    "Q4_K_M": {
        "bits": 4,
        "quality": "Good — noticeable on benchmarks, fine for most tasks",
        "size_ratio": 0.25,
        "use_case": "Most popular — great quality/size balance",
    },
    "Q3_K_M": {
        "bits": 3,
        "quality": "Acceptable — visible quality loss on complex tasks",
        "size_ratio": 0.19,
        "use_case": "When memory is very tight",
    },
    "Q2_K": {
        "bits": 2,
        "quality": "Poor — significant quality loss, use only if desperate",
        "size_ratio": 0.13,
        "use_case": "Absolute minimum — not recommended for production",
    },
}

print("GGUF Quantization Variants:")
print("=" * 70)
print(f"  {'Variant':<10} {'Bits':<6} {'Size vs fp16':<14} {'Quality':<20}")
print("-" * 70)
for variant, info in gguf_variants.items():
    # Calculate actual sizes for a 7B model
    fp16_size = 7 * 2  # 7B params * 2 bytes
    quant_size = fp16_size * info["size_ratio"]
    print(f"  {variant:<10} {info['bits']:<6} {quant_size:.1f} GB ({info['size_ratio']*100:.0f}%)  "
          f"   {info['quality'][:30]}")

print(f"\n  Recommendation: Q4_K_M for most use cases")
print(f"  It gives ~25% of fp16 size with ~97% of the quality")
```

### 2.2 Using GGUF with Ollama

```python
# Ollama is the easiest way to run GGUF models locally
# Install: https://ollama.com

# Using Ollama from Python
import requests
import json

def ollama_generate(prompt: str, model: str = "llama3.1:8b", stream: bool = False) -> str:
    """Generate text using Ollama's local API."""
    response = requests.post(
        "http://localhost:11434/api/generate",
        json={
            "model": model,
            "prompt": prompt,
            "stream": stream,
        }
    )
    
    if response.status_code == 200:
        return response.json()["response"]
    else:
        return f"Error: {response.status_code}"

# Common Ollama commands (run in terminal)
ollama_commands = """
# Install and run a model
ollama pull llama3.1:8b            # Download the model (GGUF format)
ollama pull llama3.1:8b-q4_K_M    # Specific quantization level
ollama run llama3.1:8b             # Interactive chat

# List models
ollama list                         # Show downloaded models

# Model sizes (7B Llama 3.1 example):
#   llama3.1:8b-fp16     — 16 GB (full precision)
#   llama3.1:8b-q8_0     — 8.5 GB
#   llama3.1:8b-q4_K_M   — 4.9 GB (recommended)
#   llama3.1:8b-q2_K     — 3.2 GB (low quality)

# Run a custom/fine-tuned model
# Create a Modelfile:
#   FROM ./your-model.gguf
#   PARAMETER temperature 0.7
#   SYSTEM "You are a helpful assistant."
#
# Then:
#   ollama create your-model -f Modelfile
#   ollama run your-model
"""
print("Ollama Quick Reference:")
print(ollama_commands)
```

### 2.3 Ollama Advanced Usage Patterns

Ollama goes beyond simple chat. Here are the patterns that matter for production-adjacent work.

```python
import requests
import json

# Pattern 1: Modelfile for custom models
# A Modelfile defines everything about your model's behavior.
modelfile_example = """
# Modelfile — complete specification for a custom Ollama model

# Base model (GGUF file or an existing Ollama model)
FROM llama3.1:8b

# System prompt — baked into the model
SYSTEM \"\"\"You are a senior Python developer. You give concise, correct answers.
When showing code, always include type hints. Prefer standard library solutions.
Never apologize or use filler phrases.\"\"\"

# Generation parameters
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER num_predict 500
PARAMETER stop "```"
PARAMETER stop "###"

# Template — how messages are formatted (model-specific)
# Most models use the default chat template, but you can customize:
# TEMPLATE \"\"\"{{ .System }}\\n\\n{{ .Prompt }}\"\"\"
"""
print("Modelfile Example:")
print(modelfile_example)

# Pattern 2: Ollama REST API for application integration
def ollama_chat(messages: list[dict], model: str = "llama3.1:8b") -> str:
    """Chat with Ollama using the chat API (supports message history)."""
    response = requests.post(
        "http://localhost:11434/api/chat",
        json={
            "model": model,
            "messages": messages,
            "stream": False,
        }
    )
    return response.json()["message"]["content"]

# Example: multi-turn conversation
# messages = [
#     {"role": "system", "content": "You are a Python expert."},
#     {"role": "user", "content": "How do I read a CSV file?"},
# ]
# response = ollama_chat(messages)

# Pattern 3: Embedding generation with Ollama
def ollama_embed(text: str, model: str = "llama3.1:8b") -> list[float]:
    """Generate embeddings using Ollama (any model supports this)."""
    response = requests.post(
        "http://localhost:11434/api/embeddings",
        json={"model": model, "prompt": text}
    )
    return response.json()["embedding"]

# Pattern 4: Model management
print("""
Model Management Commands:
  ollama list                    # List all downloaded models
  ollama show llama3.1:8b       # Show model details (size, parameters, template)
  ollama cp llama3.1:8b my-bot  # Copy a model (to customize without re-downloading)
  ollama rm old-model            # Delete a model to free disk space
  ollama ps                      # Show currently running models
""")
```

### 2.3 GPTQ: GPU-Optimized Quantization

GPTQ quantizes models for GPU inference, producing models that run efficiently with libraries like vLLM and HuggingFace Transformers.

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

# Load a GPTQ-quantized model from the Hub
# Many models have GPTQ variants uploaded by the community

# Example: Loading a GPTQ model
# model_name = "TheBloke/Llama-2-7B-GPTQ"  # Popular GPTQ model
# For a smaller demo:
# model_name = "TheBloke/phi-2-GPTQ"

# When loading GPTQ models with transformers:
# model = AutoModelForCausalLM.from_pretrained(
#     model_name,
#     device_map="auto",
#     torch_dtype=torch.float16,
# )

print("GPTQ (GPU Post-Training Quantization):")
print("-" * 50)
print("  How it works:")
print("    1. Analyze weight distributions layer by layer")
print("    2. Quantize weights to int4 using calibration data")
print("    3. Compensate for quantization error using Hessian information")
print("    4. Result: int4 weights with minimal quality loss")
print()
print("  Advantages:")
print("    - Fast GPU inference (optimized CUDA kernels)")
print("    - Good quality at 4-bit")
print("    - Widely supported (vLLM, TGI, transformers)")
print()
print("  Disadvantages:")
print("    - GPU-only (no CPU inference)")
print("    - Quantization process needs calibration data")
print("    - Slower to quantize than other methods")
```

### 2.4 AWQ: Activation-Aware Quantization

AWQ achieves better quality than GPTQ at the same bit-width by considering activation patterns.

```python
# AWQ uses activation distributions to decide which weights matter most
# Important weights get higher precision, less important ones get lower

print("AWQ (Activation-Aware Weight Quantization):")
print("-" * 50)
print("  Key insight: not all weights are equally important")
print("  Some weights have large activations — they matter more")
print("  AWQ preserves precision for important weights")
print()

# Loading an AWQ model
# !pip install autoawq

# from awq import AutoAWQForCausalLM
# model_name = "TheBloke/Llama-2-7B-AWQ"
# model = AutoAWQForCausalLM.from_quantized(
#     model_name,
#     fuse_layers=True,
#     trust_remote_code=True,
# )

comparison = """
Format Comparison (7B model, 4-bit):
  
  Format    | Quality | Speed  | CPU? | Size  | Best For
  ----------|---------|--------|------|-------|------------------
  GGUF Q4   | Good    | Fast   | Yes  | 4 GB  | Local/edge, Ollama
  GPTQ 4bit | Good+   | V.Fast | No   | 3.5GB | GPU servers, vLLM
  AWQ 4bit  | Best    | V.Fast | No   | 3.5GB | Quality-sensitive GPU
  BnB 4bit  | Good    | Fast   | No   | 3.5GB | Fine-tuning (QLoRA)
"""
print(comparison)
```

### 2.5 Quantizing Your Own Model

```python
# Method 1: Using bitsandbytes (simplest — for inference and QLoRA)
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "gpt2",
    quantization_config=bnb_config,
    device_map="auto",
)

print(f"Model memory: {model.get_memory_footprint() / 1e6:.1f} MB (4-bit)")
print("bitsandbytes: easiest quantization, great for QLoRA fine-tuning")

# Method 2: Converting to GGUF (for llama.cpp / Ollama)
print("""
Converting to GGUF (for Ollama/llama.cpp):

  # Install llama.cpp
  git clone https://github.com/ggerganov/llama.cpp
  cd llama.cpp && make
  
  # Convert HuggingFace model to GGUF
  python convert_hf_to_gguf.py /path/to/your/model --outtype f16
  
  # Quantize to desired level
  ./llama-quantize model-f16.gguf model-q4_K_M.gguf Q4_K_M
  
  # Test with llama.cpp
  ./llama-cli -m model-q4_K_M.gguf -p "Hello, how are you?" -n 50
  
  # Or use with Ollama
  # Create Modelfile:
  #   FROM ./model-q4_K_M.gguf
  # Then: ollama create my-model -f Modelfile
""")

# Method 3: GPTQ quantization (for GPU deployment)
print("""
GPTQ Quantization:

  # !pip install auto-gptq optimum
  
  from transformers import AutoTokenizer
  from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig
  
  quantize_config = BaseQuantizeConfig(
      bits=4,
      group_size=128,
      desc_act=False,
  )
  
  model = AutoGPTQForCausalLM.from_pretrained(
      "your-model",
      quantize_config,
  )
  
  # Quantize (needs calibration data)
  model.quantize(calibration_dataset)
  
  # Save
  model.save_quantized("your-model-gptq-4bit")
""")
```

---

## 3. Inference Optimization

### 3.1 KV Cache

The KV (Key-Value) cache stores computed attention keys and values so they do not need to be recomputed for each new token. This is the single most impactful optimization for autoregressive generation.

```python
# Without KV cache: recompute attention for ALL tokens at each step
# With KV cache: only compute attention for the NEW token

print("KV Cache: The Most Important Inference Optimization")
print("=" * 60)

# Demonstration of computational savings
def kv_cache_savings(seq_length: int, new_tokens: int):
    """Calculate KV cache computational savings."""
    
    # Without cache: at step i, process all (seq_length + i) tokens
    without_cache = sum(seq_length + i for i in range(new_tokens))
    
    # With cache: at step i, only process 1 new token
    # (but look up cached keys/values for previous tokens)
    with_cache = new_tokens  # Simplified — each step processes 1 token
    
    return without_cache, with_cache

prompt_length = 100
gen_tokens = 200

without, with_cache = kv_cache_savings(prompt_length, gen_tokens)

print(f"  Prompt: {prompt_length} tokens, Generating: {gen_tokens} tokens")
print(f"  Without KV cache: {without:,} token computations")
print(f"  With KV cache:    {with_cache:,} token computations")
print(f"  Speedup:          {without / with_cache:.0f}x")

# Memory cost of KV cache
def kv_cache_memory(num_layers: int, hidden_size: int, num_heads: int, 
                     seq_length: int, dtype_bytes: int = 2):
    """Calculate KV cache memory in bytes."""
    head_dim = hidden_size // num_heads
    # K and V for each layer, each head
    memory = 2 * num_layers * num_heads * seq_length * head_dim * dtype_bytes
    return memory

# Llama 3.1 8B KV cache
memory = kv_cache_memory(
    num_layers=32, hidden_size=4096, num_heads=32, 
    seq_length=4096, dtype_bytes=2
)
print(f"\n  KV cache memory for Llama 8B (4K context): {memory / 1e9:.2f} GB")

memory_128k = kv_cache_memory(
    num_layers=32, hidden_size=4096, num_heads=32,
    seq_length=128_000, dtype_bytes=2
)
print(f"  KV cache memory for Llama 8B (128K context): {memory_128k / 1e9:.2f} GB")
print(f"  This is why long contexts are expensive!")
```

### 3.2 Continuous Batching

```python
# Traditional batching: wait for all requests to finish before starting new ones
# Continuous batching: start new requests as soon as old ones finish

print("Continuous Batching:")
print("=" * 60)
print("""
Traditional (static) batching:
  Request A: ████████████████                (long output)
  Request B: ████████                         (short output)
  Request C: ████                             (very short)
  GPU idle:                   ████████████    (wasted compute!)
  
  GPU utilization: ~50% — short requests block the batch

Continuous (dynamic) batching:
  Request A: ████████████████
  Request B: ████████  → Request D: ██████████
  Request C: ████  → Request E: ████████████████████
  
  GPU utilization: ~95% — new requests fill freed slots
  
  Throughput improvement: 2-5x over static batching
""")

# Continuous batching is implemented by:
# - vLLM (most popular)
# - TGI (HuggingFace Text Generation Inference)
# - TensorRT-LLM
```

### 3.3 Speculative Decoding

```python
print("Speculative Decoding:")
print("=" * 60)
print("""
The idea: use a SMALL, FAST model to draft tokens,
then use the BIG model to verify them in one pass.

  Step 1: Small model (1B) drafts 5 tokens quickly
          "The weather is nice today"
          
  Step 2: Big model (70B) verifies all 5 at once
          (parallel verification is fast — similar to prefill)
          
  Step 3: Accept matching tokens, reject mismatched
          If 4/5 match, you saved 3 serial big-model steps
          
  Speedup: 2-3x for long generations
  No quality loss: you always get the big model's quality
  
  Trade-off: uses extra memory for the small model
  Best when: big model is slow, small model has high agreement
""")
```

### 3.4 Other Optimizations

```python
optimizations = {
    "Flash Attention": {
        "what": "Memory-efficient attention that avoids materializing the full attention matrix",
        "speedup": "2-4x faster attention, especially for long sequences",
        "how": "Fuses attention computation into a single GPU kernel",
        "usage": "model = AutoModelForCausalLM.from_pretrained(..., attn_implementation='flash_attention_2')",
    },
    "Paged Attention (vLLM)": {
        "what": "Manages KV cache like virtual memory — pages of fixed size",
        "speedup": "2-4x better GPU memory utilization for serving",
        "how": "KV cache is stored in non-contiguous memory pages",
        "usage": "Built into vLLM — just use vLLM for serving",
    },
    "Tensor Parallelism": {
        "what": "Split model layers across multiple GPUs",
        "speedup": "Near-linear scaling with number of GPUs",
        "how": "Each GPU handles a portion of each layer's computation",
        "usage": "vLLM: --tensor-parallel-size N",
    },
    "Prefix Caching": {
        "what": "Cache KV values for shared prompt prefixes",
        "speedup": "Huge for applications where many requests share the same system prompt",
        "how": "Detect common prefixes and reuse their KV cache",
        "usage": "vLLM: --enable-prefix-caching",
    },
}

print("Inference Optimizations:")
print("=" * 60)
for name, info in optimizations.items():
    print(f"\n  {name}")
    print(f"    What:    {info['what']}")
    print(f"    Speedup: {info['speedup']}")
    print(f"    Usage:   {info['usage']}")
```

---

## 4. Deployment Platforms

### 4.1 Local Deployment

```python
local_options = {
    "Ollama": {
        "best_for": "Simplest local deployment, GGUF models",
        "platform": "macOS, Linux, Windows",
        "format": "GGUF",
        "setup": "Download from ollama.com, then: ollama run llama3.1",
        "api": "REST API on localhost:11434",
        "gpu": "Automatic GPU detection and offloading",
    },
    "llama.cpp": {
        "best_for": "Maximum control, custom builds, C++ performance",
        "platform": "Any (compile from source)",
        "format": "GGUF",
        "setup": "git clone + make, then: ./llama-cli -m model.gguf",
        "api": "./llama-server for OpenAI-compatible API",
        "gpu": "CUDA, Metal, Vulkan, OpenCL",
    },
    "LM Studio": {
        "best_for": "GUI-based local inference, non-technical users",
        "platform": "macOS, Windows, Linux",
        "format": "GGUF",
        "setup": "Download from lmstudio.ai, click to download models",
        "api": "OpenAI-compatible API on localhost",
        "gpu": "Automatic",
    },
}

print("Local Deployment Options:")
print("=" * 60)
for name, info in local_options.items():
    print(f"\n  {name}")
    for key, value in info.items():
        print(f"    {key:10s}: {value}")
```

### 4.2 Cloud Deployment

```python
cloud_options = {
    "vLLM": {
        "best_for": "High-throughput GPU serving with continuous batching",
        "setup": "pip install vllm; python -m vllm.entrypoints.openai.api_server --model model_name",
        "features": "PagedAttention, continuous batching, tensor parallelism",
        "format": "HuggingFace, GPTQ, AWQ",
        "api": "OpenAI-compatible",
    },
    "TGI (Text Generation Inference)": {
        "best_for": "HuggingFace ecosystem, Docker deployment",
        "setup": "docker run ghcr.io/huggingface/text-generation-inference --model-id model_name",
        "features": "Continuous batching, quantization, streaming",
        "format": "HuggingFace, GPTQ, AWQ",
        "api": "REST + gRPC",
    },
    "Replicate": {
        "best_for": "Serverless GPU, pay-per-second, no infrastructure",
        "setup": "Push model to Replicate, call via API",
        "features": "Auto-scaling, cold start optimization",
        "format": "Any (containerized)",
        "api": "REST API, Python SDK",
    },
    "Modal": {
        "best_for": "Python-native serverless GPU with fine control",
        "setup": "Define model in Python, deploy with modal deploy",
        "features": "Auto-scaling, persistent volumes, web endpoints",
        "format": "Any (Python environment)",
        "api": "Python SDK, web endpoints",
    },
    "AWS SageMaker": {
        "best_for": "Enterprise, AWS ecosystem, managed endpoints",
        "setup": "SageMaker JumpStart or custom container",
        "features": "Auto-scaling, monitoring, A/B testing",
        "format": "HuggingFace, custom containers",
        "api": "SageMaker SDK",
    },
}

print("Cloud Deployment Options:")
print("=" * 60)
for name, info in cloud_options.items():
    print(f"\n  {name}")
    for key, value in info.items():
        print(f"    {key:10s}: {value}")
```

### 4.3 vLLM Setup and Configuration

vLLM is the gold standard for high-throughput LLM serving. Here is a complete setup guide.

```python
# Step 1: Installation
# !pip install vllm
# Note: vLLM requires CUDA-capable GPU. Does not work on CPU or Apple Silicon.

# Step 2: Verify your environment
import torch
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"VRAM: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")
```

```python
# vLLM configuration parameters explained

vllm_config = {
    "--model": "HuggingFace model ID or local path",
    "--dtype": "float16 or bfloat16 (auto detects best)",
    "--max-model-len": "Maximum sequence length (reduce if OOM). Defaults to model's max.",
    "--gpu-memory-utilization": "Fraction of GPU memory to use (0.9 = 90%). Lower to avoid OOM.",
    "--tensor-parallel-size": "Number of GPUs for tensor parallelism (for large models)",
    "--quantization": "awq, gptq, or squeezellm for pre-quantized models",
    "--enable-prefix-caching": "Cache common prefixes (great if many requests share system prompt)",
    "--max-num-seqs": "Max concurrent sequences in a batch (affects throughput vs latency)",
    "--port": "API port (default 8000)",
    "--host": "Bind address (0.0.0.0 for external access)",
}

print("vLLM Configuration Reference:")
print("=" * 70)
for param, desc in vllm_config.items():
    print(f"  {param:<30} {desc}")

# Common configurations by hardware
print("""
Recommended vLLM Configurations:

  T4 (16GB) — Llama 3.1 8B:
    --model meta-llama/Llama-3.1-8B-Instruct
    --dtype float16
    --max-model-len 4096
    --gpu-memory-utilization 0.90

  A10G (24GB) — Llama 3.1 8B with long context:
    --model meta-llama/Llama-3.1-8B-Instruct
    --dtype float16
    --max-model-len 16384
    --gpu-memory-utilization 0.92
    --enable-prefix-caching

  A100 80GB — Llama 3.1 70B:
    --model meta-llama/Llama-3.1-70B-Instruct
    --dtype bfloat16
    --tensor-parallel-size 2
    --max-model-len 8192
    --gpu-memory-utilization 0.95
""")
```

### 4.4 Deploying with vLLM

```python
# vLLM is the gold standard for high-throughput LLM serving

# !pip install vllm

# Option 1: Command-line server
vllm_commands = """
# Start a vLLM server (OpenAI-compatible API)
python -m vllm.entrypoints.openai.api_server \\
    --model meta-llama/Llama-3.1-8B-Instruct \\
    --dtype float16 \\
    --max-model-len 4096 \\
    --gpu-memory-utilization 0.90 \\
    --port 8000

# With quantization
python -m vllm.entrypoints.openai.api_server \\
    --model TheBloke/Llama-2-7B-AWQ \\
    --quantization awq \\
    --port 8000

# With tensor parallelism (multi-GPU)
python -m vllm.entrypoints.openai.api_server \\
    --model meta-llama/Llama-3.1-70B-Instruct \\
    --tensor-parallel-size 4 \\
    --port 8000
"""
print("vLLM Server Commands:")
print(vllm_commands)

# Option 2: Python API
print("vLLM Python API:")
print("""
from vllm import LLM, SamplingParams

# Load model
llm = LLM(model="meta-llama/Llama-3.1-8B-Instruct")

# Configure sampling
sampling = SamplingParams(
    temperature=0.7,
    max_tokens=200,
    top_p=0.9,
)

# Generate (batched automatically)
prompts = [
    "What is machine learning?",
    "Explain transformers in simple terms.",
    "How does LoRA work?",
]

outputs = llm.generate(prompts, sampling)

for output in outputs:
    print(f"Prompt: {output.prompt[:40]}...")
    print(f"Output: {output.outputs[0].text[:80]}...")
""")

# Option 3: Client calling the OpenAI-compatible API
print("\nCalling vLLM's OpenAI-compatible API:")
print("""
import openai

client = openai.OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",  # vLLM doesn't require auth by default
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Hello!"}],
    temperature=0.7,
    max_tokens=200,
)

print(response.choices[0].message.content)

# This means you can switch between OpenAI and self-hosted
# by changing just the base_url and model name!
""")
```

### 4.4 Deploying with Ollama (Simplest Path)

```python
# For quick local deployment, Ollama is unbeatable

print("Deploying a Custom Model with Ollama:")
print("=" * 60)
print("""
Step 1: Convert your fine-tuned model to GGUF
  # Using llama.cpp's conversion script
  python convert_hf_to_gguf.py ./your-finetuned-model --outtype f16
  ./llama-quantize model-f16.gguf model-q4_K_M.gguf Q4_K_M

Step 2: Create an Ollama Modelfile
  # Create a file called 'Modelfile' with:
  FROM ./model-q4_K_M.gguf
  PARAMETER temperature 0.7
  PARAMETER top_p 0.9
  PARAMETER stop "### Instruction:"
  SYSTEM "You are a helpful customer support assistant. Be empathetic and solution-oriented."

Step 3: Create the Ollama model
  ollama create my-support-bot -f Modelfile

Step 4: Run it
  ollama run my-support-bot

Step 5: Use the API
  curl http://localhost:11434/api/chat -d '{
    "model": "my-support-bot",
    "messages": [{"role": "user", "content": "My order is late!"}]
  }'
""")
```

---

## 5. The Quality vs Size vs Speed Trade-Off

### 5.1 Benchmark Comparison

```python
# Real-world quality comparison (approximate benchmark scores)
# Based on typical results from MMLU, HumanEval, etc.

benchmarks = {
    "Llama 3.1 8B (fp16)": {
        "mmlu": 65.0,
        "humaneval": 45.0,
        "memory_gb": 16,
        "speed_tok_s": 40,
        "quality_note": "Baseline — full precision",
    },
    "Llama 3.1 8B (GPTQ 4-bit)": {
        "mmlu": 63.5,
        "humaneval": 43.0,
        "memory_gb": 4.5,
        "speed_tok_s": 55,
        "quality_note": "~2% quality loss, 3.5x less memory",
    },
    "Llama 3.1 8B (AWQ 4-bit)": {
        "mmlu": 64.0,
        "humaneval": 44.0,
        "memory_gb": 4.5,
        "speed_tok_s": 55,
        "quality_note": "~1.5% quality loss, best 4-bit quality",
    },
    "Llama 3.1 8B (GGUF Q4_K_M)": {
        "mmlu": 63.0,
        "humaneval": 42.0,
        "memory_gb": 4.9,
        "speed_tok_s": 35,
        "quality_note": "~3% quality loss, runs on CPU+GPU",
    },
    "Llama 3.1 8B (GGUF Q2_K)": {
        "mmlu": 55.0,
        "humaneval": 30.0,
        "memory_gb": 3.2,
        "speed_tok_s": 45,
        "quality_note": "~15% quality loss — not recommended",
    },
}

print("Quality vs Size vs Speed (Llama 3.1 8B):")
print("=" * 80)
print(f"  {'Configuration':<30} {'MMLU':<7} {'HumanEval':<11} {'Memory':<9} {'Speed':<10}")
print("-" * 80)
for name, info in benchmarks.items():
    print(f"  {name:<30} {info['mmlu']:<7.1f} {info['humaneval']:<11.1f} "
          f"{info['memory_gb']:<9.1f} {info['speed_tok_s']:<10}")

print(f"\n  Key takeaway: Q4_K_M and AWQ 4-bit give excellent quality/size ratios.")
print(f"  Below 4-bit, quality degrades rapidly. Above 4-bit, diminishing returns.")
```

### 5.2 Edge Deployment: Running Models on Constrained Hardware

Edge deployment — running models on phones, tablets, Raspberry Pi, or IoT devices — has unique constraints.

```python
# Edge deployment hardware profiles
edge_hardware = {
    "iPhone 15 Pro (8GB RAM)": {
        "usable_memory": "~4 GB (OS and apps take the rest)",
        "suitable_models": "Llama 3.2 1B (Q4_K_M: 0.7 GB), Phi-3 mini (Q4: 2.3 GB)",
        "inference_speed": "10-20 tokens/sec with Core ML / MLX",
        "framework": "llama.cpp (via MLX or Core ML), Swift native",
    },
    "Android flagship (12GB RAM)": {
        "usable_memory": "~6 GB",
        "suitable_models": "Llama 3.2 3B (Q4_K_M: 2.0 GB), Phi-3 mini (Q4: 2.3 GB)",
        "inference_speed": "8-15 tokens/sec on Snapdragon 8 Gen 3",
        "framework": "llama.cpp via JNI, MediaPipe LLM",
    },
    "Raspberry Pi 5 (8GB)": {
        "usable_memory": "~6 GB",
        "suitable_models": "Llama 3.2 1B (Q4_K_M: 0.7 GB), Phi-3 mini (Q3: 1.8 GB)",
        "inference_speed": "2-5 tokens/sec (CPU only, no GPU)",
        "framework": "llama.cpp compiled for ARM",
    },
    "Laptop CPU (16GB RAM, no GPU)": {
        "usable_memory": "~10 GB",
        "suitable_models": "Llama 3.1 8B (Q4_K_M: 4.9 GB), Mistral 7B (Q4: 4.4 GB)",
        "inference_speed": "5-15 tokens/sec on Apple M2, 3-8 on Intel i7",
        "framework": "Ollama, llama.cpp, LM Studio",
    },
}

print("Edge Deployment Hardware Guide:")
print("=" * 70)
for device, info in edge_hardware.items():
    print(f"\n  {device}")
    for key, value in info.items():
        print(f"    {key:20s}: {value}")

print("""
Edge deployment rules:
  1. Always quantize to Q4_K_M or lower
  2. Target models with <3B parameters for phones, <8B for laptops
  3. Test with the actual device — benchmarks vary wildly
  4. Consider latency requirements: 10 tok/s is conversational, 2 tok/s is painful
  5. Battery impact is real: GPU inference drains batteries 3-5x faster than idle
""")
```

### 5.3 When to Quantize

```python
quantization_decision = """
WHEN TO QUANTIZE:

  Always quantize when:
    - Deploying to consumer hardware (laptop, phone, edge)
    - Running locally with Ollama/llama.cpp
    - Model doesn't fit in GPU memory at fp16
    - Cost-optimizing high-volume inference
  
  Don't quantize when:
    - You have plenty of GPU memory and need maximum quality
    - You are fine-tuning (keep weights in fp16/bf16 for training)
    - The task requires very precise numerical reasoning
    - You are benchmarking/evaluating (use fp16 for fair comparison)

  WHICH FORMAT:
    - Local/edge deployment    -> GGUF (Q4_K_M or Q5_K_M)
    - GPU server (vLLM/TGI)   -> AWQ 4-bit or GPTQ 4-bit
    - Fine-tuning              -> bitsandbytes 4-bit (QLoRA)
    - Maximum quality          -> fp16 (no quantization)
"""
print(quantization_decision)
```

---

## 6. End-to-End Deployment Example

```python
# Complete workflow: fine-tune -> quantize -> deploy

print("""
End-to-End: From Fine-Tuning to Production
==========================================

Step 1: Fine-tune with LoRA (Ch 45)
  - Train LoRA adapter on your dataset
  - Merge adapter into base model
  - Save: ./my-finetuned-model/

Step 2: Quantize
  Option A — For local/Ollama:
    python convert_hf_to_gguf.py ./my-finetuned-model --outtype f16
    ./llama-quantize model-f16.gguf model-q4_K_M.gguf Q4_K_M
    ollama create my-model -f Modelfile
    
  Option B — For GPU server (vLLM):
    # Upload to HuggingFace Hub or keep locally
    # vLLM handles quantization at load time with --quantization flag
    # Or pre-quantize with auto-gptq/autoawq

Step 3: Deploy
  Option A — Local:
    ollama run my-model
    # API at localhost:11434
    
  Option B — Single GPU server:
    python -m vllm.entrypoints.openai.api_server \\
        --model ./my-finetuned-model \\
        --quantization awq
    # API at localhost:8000
    
  Option C — Serverless:
    # Push to Replicate, Modal, or SageMaker
    # API endpoint provided by the platform

Step 4: Integrate
  import openai
  
  client = openai.OpenAI(
      base_url="http://your-server:8000/v1",
      api_key="your-key",
  )
  
  # Same code works with OpenAI, vLLM, Ollama, or any
  # OpenAI-compatible API!

Step 5: Monitor & Iterate
  - Track latency, throughput, error rates
  - Monitor quality with evals (Ch 51)
  - Retrain with better data as needed (Ch 46)
""")
```

---

## 7. Key Takeaways

1. **4-bit is the sweet spot.** Q4_K_M (GGUF) and AWQ 4-bit give roughly 97% of full-precision quality at 25% of the memory. Below 4-bit, quality degrades sharply.

2. **Format depends on deployment.** GGUF for local/CPU inference (Ollama, llama.cpp). GPTQ/AWQ for GPU servers (vLLM, TGI). bitsandbytes for training (QLoRA).

3. **KV cache is the biggest bottleneck.** Long contexts consume massive amounts of memory. This is why context length matters for deployment sizing.

4. **vLLM is the production standard.** Continuous batching, PagedAttention, and OpenAI-compatible API make vLLM the default choice for GPU serving.

5. **Ollama is the local standard.** For development, testing, and local deployment, Ollama provides the simplest path from model to API.

6. **The OpenAI-compatible API is the integration layer.** Whether you use vLLM, Ollama, TGI, or a cloud provider, the API is the same. Write your application code once, swap providers freely.

7. **Quantize after fine-tuning.** Train in fp16/bf16 for best results, then quantize the merged model for deployment.

---

## What's Next

You have completed Part 9: you can decide between RAG and fine-tuning (Ch 44), fine-tune models efficiently with LoRA (Ch 45), build high-quality training datasets (Ch 46), and quantize and deploy the result (this chapter).

In **Part 10: Production AI Systems**, you will harden everything for production: security and guardrails (Ch 48), cost engineering (Ch 49), advanced context strategies (Ch 50), production eval pipelines (Ch 51), and observability (Ch 52). The models are trained and deployed — now you make them reliable.

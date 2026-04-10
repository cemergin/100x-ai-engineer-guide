<!--
  TYPE: part-index
  PART: 8
  TITLE: Open Source AI & Inference
  PHASE: 2 — Become an Expert
  CHAPTERS: 40, 41, 42, 43
  UPDATED: 2026-04-10
-->

# Part 8 — Open Source AI & Inference

> **Phase 2: Become an Expert** | 4 chapters | Intermediate-Advanced | Language: Python

Run your own models. This Part spirals Part 1 from calling APIs to local inference — you go from "send a request to OpenAI" to "load a model on your GPU and run it yourself."

In Part 7, you built neural networks from scratch with numpy. You understand weights, attention, tokenization, and decoding. Now you use that understanding to run real models locally. Hugging Face gives you access to hundreds of thousands of pre-trained models with a single line of code. You will generate text, create images, produce embeddings, and learn how to choose the right architecture for any task — all without sending a single API call to a cloud provider.

The practical value is enormous. Local inference means no per-token costs, no rate limits, complete data privacy, and the ability to customize models for your specific use case. The trade-off is that you need hardware (or a Colab GPU) and you need to understand what you are running. By the end of this Part, you will.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 40 | [Hugging Face & Pipelines](./40-hugging-face.md) | Intermediate | Pipeline API, model hub, tasks, running inference locally |
| 41 | [Image Generation](./41-image-generation.md) | Intermediate | Stable Diffusion, text-to-image, image-to-image, DreamBooth |
| 42 | [Embeddings & Sentence Transformers](./42-sentence-transformers.md) | Intermediate | Generate your own embeddings, MTEB benchmarks, model selection |
| 43 | [Model Selection & Architecture](./43-model-selection.md) | Inter-Adv | BERT vs GPT vs T5 vs Llama, sizes, quantization, local vs cloud |

## The Within-Part Spiral

```
Ch 40: Hugging Face & Pipelines (one-line inference for any NLP task)
  └──> Ch 41: Image Generation (from text tasks to visual generation)
        └──> Ch 42: Embeddings & Sentence Transformers (from generation to representation)
              └──> Ch 43: Model Selection & Architecture (synthesize everything into decisions)
```

This spiral mirrors how you naturally explore open-source AI: start with the easiest path (pipelines), branch into new modalities (images), deepen your understanding of representations (embeddings), then step back and develop a framework for choosing the right tool for any job.

## How This Part Connects

```
Part 1 (Building with LLM APIs — the first pass)
  │
  └──> Part 8 (you are here — the local inference pass)
         Ch 40 spirals "API calls" from Ch 3 into local inference
         Ch 41 spirals "multimodal" from Ch 7 into local image generation
         Ch 42 spirals "embeddings" from Ch 1/14 into generating your own
         Ch 43 spirals "model landscape" from Ch 2 into architecture decisions
         │
         ├──> Part 9: Fine-Tuning & Training
         │      Ch 44: RAG vs fine-tuning (you now know both sides)
         │      Ch 45: LoRA fine-tunes the models from Ch 40/43
         │      Ch 47: quantize and deploy the models from Ch 43
         │
         └──> Part 10: Production AI
                Ch 49: cost engineering (local vs API costs from Ch 43)
```

## Reading Time

Expect about 6-8 hours to work through all four chapters. Every chapter includes runnable Python code — most examples are designed to run on Google Colab with a free GPU. You will need `transformers`, `diffusers`, `sentence-transformers`, and `torch` installed. By the end, you will have run text generation, image generation, and embedding models locally, and developed a decision framework for choosing models in production.

---

*Previous: [Part 7 — The Hard Parts](../part-7-hard-parts/)* | *Next: [Part 9 — Fine-Tuning & Training](../part-9-fine-tuning-training/)*

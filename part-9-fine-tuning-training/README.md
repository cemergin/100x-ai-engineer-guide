<!--
  TYPE: part-index
  PART: 9
  TITLE: Fine-Tuning & Training
  PHASE: 2 — Become an Expert
  CHAPTERS: 44, 45, 46, 47
  UPDATED: 2026-04-10
-->

# Part 9 — Fine-Tuning & Training

> **Phase 2: Become an Expert** | 4 chapters | Advanced | Language: Python

Bake knowledge into weights. This Part spirals Part 3 from retrieving knowledge at query time to training it into the model itself — you go from "give the model documents to read" to "make the model already know your domain."

In Part 8, you ran open-source models locally. In Part 3, you built RAG pipelines to feed external knowledge into those models. Now you face the fundamental question: should you retrieve knowledge at query time (RAG) or bake it into the model's weights (fine-tuning)? The answer depends on your data, your task, your budget, and your latency requirements. This Part gives you the framework to decide — and then teaches you how to actually do it.

You will fine-tune a language model with LoRA (adding less than 1% new parameters), engineer datasets that make fine-tuning work, and quantize models for deployment on hardware that would normally be too small. By the end, you can take any open-source model from Part 8 and customize it for your specific domain.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 44 | [RAG vs Fine-Tuning](./44-rag-vs-finetuning.md) | Inter-Adv | When to retrieve vs train, cost/quality/latency trade-offs |
| 45 | [Fine-Tuning with LoRA](./45-lora-finetuning.md) | Advanced | Low-rank adaptation, PEFT, fine-tune GPT-2 on custom data |
| 46 | [Dataset Engineering](./46-dataset-engineering.md) | Advanced | Curating data, quality, formats, synthetic data, augmentation |
| 47 | [Quantization & Deployment](./47-quantization-deployment.md) | Advanced | GGUF, GPTQ, AWQ, inference optimization, when to quantize |

## The Within-Part Spiral

```
Ch 44: RAG vs Fine-Tuning (should you fine-tune at all?)
  └──> Ch 45: Fine-Tuning with LoRA (yes — here's how)
        └──> Ch 46: Dataset Engineering (your fine-tune is only as good as your data)
              └──> Ch 47: Quantization & Deployment (ship the result efficiently)
```

This spiral mirrors the real workflow: first you decide whether fine-tuning is the right approach, then you do it, then you improve your data to make it better, then you optimize and deploy the result. Each chapter builds directly on the previous one.

## How This Part Connects

```
Part 3 (RAG & Knowledge — the retrieval approach)
  │
  └──> Part 9 (you are here — the training approach)
         Ch 44 spirals RAG from Ch 15 against fine-tuning
         Ch 45 spirals "weights" from Ch 36 into LoRA adapters
         Ch 46 spirals "eval datasets" from Ch 18 into training datasets
         Ch 47 spirals "model selection" from Ch 43 into deployment
         │
         ├──> Part 10: Production AI
         │      Ch 48: secure the fine-tuned model
         │      Ch 49: cost engineering for custom models
         │
         └──> Part 11: Deployment & Infrastructure
                Ch 57: scale the deployed model
```

## Reading Time

Expect about 6-8 hours to work through all four chapters. Chapters 45 and 46 are heavily hands-on — you will fine-tune an actual model and build a real dataset. Google Colab with a T4 GPU is sufficient for all examples. You will need `transformers`, `peft`, `datasets`, `bitsandbytes`, and `trl` installed. By the end, you will have fine-tuned a model, built a training dataset, quantized the result, and deployed it for inference.

---

*Previous: [Part 8 — Open Source AI & Inference](../part-8-open-source-ai/)* | *Next: [Part 10 — Production AI Systems](../part-10-production-ai/)*

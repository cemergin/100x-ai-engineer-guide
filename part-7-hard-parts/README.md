<!--
  TYPE: part-index
  PART: 7
  TITLE: The Hard Parts
  PHASE: 2 — Become an Expert
  CHAPTERS: 34, 35, 36, 37, 38, 39
  UPDATED: 2026-04-10
-->

# Part 7 — The Hard Parts

> **Phase 2: Become an Expert** | 6 chapters | Inter-Adv | Language: Python

Build a neural network by hand. Understand transformers from scratch. This Part spirals Part 0 all the way down — from "tokens cost money" to building the engine that produces them.

In Part 0, you learned that LLMs predict the next token. You learned that temperature controls randomness and embeddings capture meaning. You learned enough to be dangerous. Now you learn enough to be *right*. These six chapters take you from "I know what a model does" to "I know what a model *is*" — and you will build one, piece by piece, in Python with nothing but numpy.

The pedagogical approach follows Will Sentance's "Hard Parts" style: start with intuition, build mental models step by step, use concrete examples. You will detect DoorDash refund fraud, classify smile pixels, tokenize text by hand, implement attention, and decode tokens into sentences. By the end, when someone says "the transformer uses multi-head self-attention with learned positional encodings," you will know exactly what every word means — because you built it.

---

## Chapters

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 34 | [ML Decision Making](./34-ml-decision-making.md) | Intermediate | Prediction, features, decision boundaries, the DoorDash refund example |
| 35 | [Data & Preprocessing](./35-data-preprocessing.md) | Intermediate | Sample populations, normalization, train/test splits, the pixel grid |
| 36 | [Neural Networks from Scratch](./36-neural-networks.md) | Inter-Adv | Weights, sigmoid, gradient descent, backpropagation, smile detector |
| 37 | [Tokenization Deep Dive](./37-tokenization.md) | Inter-Adv | BPE, WordPiece, encoding/decoding, batching, attention masks |
| 38 | [Transformers & Attention](./38-transformers-attention.md) | Advanced | Self-attention, multi-head, positional encoding, encoder/decoder |
| 39 | [Decoding & Generation](./39-decoding-generation.md) | Advanced | Greedy, beam search, top-k, top-p, temperature as math |

## The Within-Part Spiral

```
Ch 34: ML Decision Making (prediction basics — what IS a model?)
  └──> Ch 35: Data & Preprocessing (a model needs data — what kind? how?)
        └──> Ch 36: Neural Networks from Scratch (feed data into weights — learn)
              └──> Ch 37: Tokenization Deep Dive (how text enters the network)
                    └──> Ch 38: Transformers & Attention (the architecture that changed everything)
                          └──> Ch 39: Decoding & Generation (how text comes out the other side)
```

This spiral mirrors the actual data flow: raw problem (Ch 34) becomes structured data (Ch 35), flows into a neural network (Ch 36), enters as tokens (Ch 37), gets processed by attention (Ch 38), and emerges as generated text (Ch 39).

## How This Part Connects

```
Part 0 (LLM Fundamentals — the first pass)
  │
  └──> Part 7 (you are here — the deep pass)
         Ch 34 spirals "models predict" from Ch 0 into building a predictor
         Ch 35 spirals "embeddings are vectors" from Ch 1 into pixel grids
         Ch 36 spirals embeddings from Ch 1 into learned weights
         Ch 37 spirals tokens from Ch 0 into BPE mechanics
         Ch 38 spirals "transformers" from Ch 0 into Q/K/V math
         Ch 39 spirals temperature from Ch 0 into probability math
         │
         ├──> Part 8: Open Source AI
         │      Ch 40: run the models you now understand (Hugging Face)
         │      Ch 42: generate embeddings from Ch 36 (sentence transformers)
         │      Ch 43: choose architectures from Ch 38 (BERT vs GPT vs T5)
         │
         ├──> Part 9: Fine-Tuning & Training
         │      Ch 44: RAG vs fine-tuning (now you know what "tuning weights" means)
         │      Ch 45: LoRA adjusts the weights from Ch 36
         │      Ch 46: dataset engineering from Ch 35
         │
         └──> Part 10: Production AI
                Ch 49: cost engineering (tokens from Ch 37, generation from Ch 39)
```

## Reading Time

Expect about 8-10 hours to work through all six chapters. Unlike Part 6, this Part is *hands-on* — you should run the Python code, modify the examples, and watch the numbers change. The code uses only numpy and the Python standard library (no PyTorch yet — that is Part 8). By the end, you will have built a working neural network, implemented BPE tokenization, computed self-attention, and run a complete text generation pipeline.

---

*Previous: [Part 6 — Anatomy of AI Developer Tools](../part-6-anatomy-of-ai-tools/)* | *Next: [Part 8 — Open Source AI & Inference](../part-8-open-source-ai/)*

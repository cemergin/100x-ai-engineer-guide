<!--
  TYPE: appendix
  TITLE: Resources
  UPDATED: 2026-04-10
-->

# Appendix: Resources

> Essential papers, books, courses, blogs, tools, and articles for the practicing AI engineer. Organized by category with notes on when each resource is most useful.

---

## 1. Essential Papers

These are the papers that define the field. You do not need to read them all, but knowing what they say and why they matter will make you a better AI engineer.

### Foundation

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **Attention Is All You Need** (Vaswani et al.) | 2017 | Introduced the transformer architecture. Everything in modern AI descends from this paper. | Ch 38 |
| **BERT: Pre-training of Deep Bidirectional Transformers** (Devlin et al.) | 2018 | Showed that pre-training on masked language modeling creates powerful general-purpose representations. | Ch 38, Ch 43 |
| **Language Models are Few-Shot Learners (GPT-3)** (Brown et al.) | 2020 | Demonstrated that large language models can perform tasks with just a few examples in the prompt. | Ch 0, Ch 4 |
| **Training Language Models to Follow Instructions (InstructGPT)** (Ouyang et al.) | 2022 | Introduced RLHF for aligning language models to human instructions. The technique that made ChatGPT possible. | Ch 0 |
| **Constitutional AI** (Bai et al., Anthropic) | 2022 | Anthropic's approach to alignment: train the model with principles rather than just human feedback. | Ch 48 |

### Retrieval & RAG

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **Retrieval-Augmented Generation** (Lewis et al.) | 2020 | The original RAG paper. Showed that retrieving documents and conditioning generation on them improves factual accuracy. | Ch 15 |
| **Dense Passage Retrieval** (Karpathy et al.) | 2020 | Showed that learned dense embeddings outperform BM25 for open-domain question answering. | Ch 14 |
| **REALM: Retrieval-Augmented Language Model** (Guu et al.) | 2020 | Pre-training with a retrieval component. Foundational to understanding RAG architectures. | Ch 15 |
| **Sentence-BERT** (Reimers & Gurevych) | 2019 | Adapted BERT for generating semantically meaningful sentence embeddings efficiently. | Ch 42 |

### Fine-Tuning & Efficiency

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **LoRA: Low-Rank Adaptation** (Hu et al.) | 2021 | The technique that made fine-tuning accessible. Train tiny adapter matrices instead of the full model. | Ch 45 |
| **QLoRA** (Dettmers et al.) | 2023 | Combine 4-bit quantization with LoRA. Fine-tune a 65B model on a single GPU. | Ch 45 |
| **GPTQ** (Frantar et al.) | 2022 | Post-training quantization using approximate second-order information. Run large models on consumer hardware. | Ch 47 |
| **Scaling Laws for Neural Language Models** (Kaplan et al.) | 2020 | Discovered power-law relationships between model size, data, compute, and performance. Guides model selection decisions. | Ch 43 |

### Agents & Tools

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **ReAct: Synergizing Reasoning and Acting** (Yao et al.) | 2022 | The Reasoning + Acting pattern. Foundation of modern agent architectures. | Ch 13 |
| **Toolformer** (Schick et al.) | 2023 | Showed that LLMs can learn to use tools (search, calculator, etc.) by self-supervised training. | Ch 8 |
| **HuggingGPT** (Shen et al.) | 2023 | Using an LLM to coordinate multiple specialist models. Early multi-agent orchestration. | Ch 58 |

### Evaluation

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **Judging LLM-as-a-Judge** (Zheng et al.) | 2023 | Systematic study of using LLMs to evaluate other LLMs. Establishes when this works and when it doesn't. | Ch 20 |
| **MTEB: Massive Text Embedding Benchmark** (Muennighoff et al.) | 2022 | The standard benchmark for comparing embedding models. Essential for model selection. | Ch 42 |

---

## 2. Books

### For Practitioners (Recommended)

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Designing Machine Learning Systems** | Chip Huyen | The best book on production ML systems. Covers the engineering around the model, not just the model. |
| **Building LLM Apps** | Valentina Alto | Practical guide to building applications with LLMs. Good TypeScript/Python code examples. |
| **Natural Language Processing with Transformers** | Lewis Tunstall, Leandro von Werra, Thomas Wolf | By the Hugging Face team. Deep dive into transformers, fine-tuning, and the Hugging Face ecosystem. |
| **AI Engineering** | Chip Huyen | Comprehensive guide to the emerging AI engineering discipline. Covers the full lifecycle. |

### For Deeper Understanding

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Deep Learning** | Ian Goodfellow, Yoshua Bengio, Aaron Courville | The definitive textbook. Dense but comprehensive. Best for understanding fundamentals. |
| **The Hundred-Page Machine Learning Book** | Andriy Burkov | Exactly what it says. Best for quickly reviewing ML concepts. |
| **Speech and Language Processing** | Dan Jurafsky, James H. Martin | The classic NLP textbook. Free online. Excellent for understanding the progression from traditional NLP to modern approaches. |
| **Grokking Deep Learning** | Andrew Trask | Build neural networks from scratch. Approachable for engineers without ML background. Aligns with Ch 36. |

### For Strategy and Organization

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Prediction Machines** | Ajay Agrawal, Joshua Gans, Avi Goldfarb | Economic framework for thinking about AI as a drop in the cost of prediction. Useful for the cost arguments in Ch 62. |
| **The AI Organization** | David De Cremer | How companies should structure themselves around AI capabilities. Relevant to Ch 62. |

---

## 3. Courses

### Referenced in This Guide

| Course | Platform | Instructor | Guide Connection |
|--------|----------|-----------|-----------------|
| **AI Agents with LLMs** | Frontend Masters | Scott Moss | Agent loops, tool calling, multi-agent patterns. Aligns with Part 2 and Ch 58. |
| **Machine Learning with Hugging Face** | Frontend Masters | Steve Kinney | Pipelines, Sentence Transformers, model selection. Aligns with Part 8. |
| **Neural Networks and Deep Learning** | Frontend Masters | Will Sentance | Building neural nets from scratch. Aligns with Part 7. |

### Additional Recommended Courses

| Course | Platform | Why Take It |
|--------|----------|------------|
| **Practical Deep Learning for Coders** | fast.ai | Jeremy Howard's legendary course. Top-down approach: build things first, understand later. Mirrors this guide's philosophy. |
| **CS224N: NLP with Deep Learning** | Stanford (free) | Chris Manning's graduate NLP course. Goes deep into transformers, attention, and modern NLP. |
| **Full Stack LLM Bootcamp** | The Full Stack | Practical end-to-end LLM application development. Good for bridging theory and practice. |
| **LLM University** | Cohere | Free comprehensive course covering embeddings, RAG, and LLM fundamentals. |
| **DeepLearning.AI Short Courses** | DeepLearning.AI | Andrew Ng's platform. Many focused 1-2 hour courses on specific topics (RAG, agents, fine-tuning). |
| **Hugging Face NLP Course** | Hugging Face (free) | Official course covering the Hugging Face ecosystem. Essential for Part 8-9. |

---

## 4. Blogs & Newsletters

### Must-Follow

| Blog / Newsletter | Author | Why Follow |
|-------------------|--------|-----------|
| **Simon Willison's Blog** | Simon Willison | The single best blog on practical AI engineering. Covers tools, techniques, and implications with deep technical rigor and intellectual honesty. |
| **Lilian Weng's Blog** | Lilian Weng (OpenAI) | Extraordinary technical deep-dives. Her posts on attention, agents, and prompting are reference-quality. |
| **The Batch** | Andrew Ng / DeepLearning.AI | Weekly newsletter covering AI news, research, and applications. Good for staying current. |
| **Anthropic Blog** | Anthropic | Research publications, model releases, and safety research. Essential for understanding Claude's capabilities. |
| **OpenAI Blog** | OpenAI | Model announcements, research, and best practices. |
| **The Gradient** | Various | In-depth articles on ML research for practitioners. |

### Also Worth Reading

| Blog / Newsletter | Focus |
|-------------------|-------|
| **Jay Alammar's Blog** | Visual explanations of transformers, attention, and NLP concepts. The best visual guides in the field. |
| **Chip Huyen's Blog** | Production ML systems, AI engineering, industry trends. |
| **Eugene Yan's Blog** | Applied ML, RecSys, LLM applications. Practical and well-written. |
| **Sebastian Raschka's Blog** | Deep learning, LLMs, fine-tuning. Author of "Machine Learning with PyTorch and Scikit-Learn." |
| **Ahead of AI Newsletter** | Sebastian Raschka's newsletter. Monthly deep-dives into LLM research. |
| **Latent Space Podcast/Newsletter** | swyx & Alessio | Interviews with AI practitioners. Great for understanding how companies actually use AI. |

---

## 5. Tools & Libraries

### SDKs and Frameworks (Ch 3, Ch 6, Ch 8, Ch 13)

| Tool | Language | What It Does |
|------|----------|-------------|
| **Anthropic SDK** (`@anthropic-ai/sdk`) | TypeScript, Python | Official SDK for Claude. Messages API, tool calling, streaming. |
| **OpenAI SDK** (`openai`) | TypeScript, Python | Official SDK for GPT models. Chat completions, tool calling, structured output. |
| **Vercel AI SDK** (`ai`) | TypeScript | Framework-agnostic SDK for building AI-powered UIs. Streaming, tool calling, multi-provider. |
| **LangChain** | TypeScript, Python | Comprehensive framework for LLM applications. Chains, agents, memory, retrieval. |
| **Mastra** | TypeScript | TypeScript-first AI framework. Agents, workflows, RAG, integrations. |
| **LlamaIndex** | Python | Data framework for LLM applications. Excels at RAG and document processing. |

### Vector Databases (Ch 14)

| Tool | Type | Best For |
|------|------|---------|
| **Pinecone** | Managed SaaS | Fastest time to production. Zero ops. |
| **Weaviate** | Self-hosted or cloud | Hybrid search, GraphQL API. |
| **Qdrant** | Self-hosted or cloud | High performance, rich filtering. |
| **ChromaDB** | Self-hosted | Simple, developer-friendly, great for prototyping. |
| **pgvector** | PostgreSQL extension | Adding vector search to existing Postgres. |

### Evaluation (Ch 18-21, Ch 51)

| Tool | What It Does |
|------|-------------|
| **Braintrust** | Eval framework with logging, scoring, and experiment tracking. |
| **Laminar** | OpenTelemetry-native LLM observability. Traces, evals, cost tracking. |
| **Arize Phoenix** | Open-source LLM observability. Traces, evaluations, datasets. |
| **Promptfoo** | Open-source prompt testing and eval framework. CI-friendly. |

### Observability (Ch 22, Ch 52)

| Tool | What It Does |
|------|-------------|
| **Datadog LLM Observability** | Enterprise-grade monitoring for LLM applications. APM integration. |
| **Langfuse** | Open-source LLM observability. Traces, scores, prompt management. |
| **Helicone** | LLM observability proxy. Logs all requests, tracks cost, caches. |

### MCP (Ch 24)

| Tool | What It Does |
|------|-------------|
| **MCP SDK** (`@modelcontextprotocol/sdk`) | Official SDK for building MCP servers and clients. |
| **MCP server registry** | Community registry of pre-built MCP servers for common services. |

### Open-Source Models (Ch 40-43, Ch 47)

| Tool | What It Does |
|------|-------------|
| **Hugging Face Transformers** | The standard library for working with transformer models. |
| **Hugging Face Hub** | Model repository with 500K+ models. |
| **Sentence Transformers** | Generate sentence embeddings. |
| **llama.cpp** | Run quantized LLMs on CPU. |
| **Ollama** | Run open-source LLMs locally with a simple CLI. |
| **vLLM** | High-throughput LLM inference engine. |
| **TGI (Text Generation Inference)** | Hugging Face's optimized inference server. |

### Fine-Tuning (Ch 44-46)

| Tool | What It Does |
|------|-------------|
| **PEFT** | Hugging Face's library for parameter-efficient fine-tuning (LoRA, etc.). |
| **TRL** | Transformer Reinforcement Learning. Fine-tune with RLHF, DPO, SFT. |
| **Axolotl** | Streamlined fine-tuning toolkit. Supports LoRA, QLoRA, full fine-tuning. |
| **Unsloth** | 2x faster fine-tuning with 80% less memory. |

### Automation (Ch 25, Ch 59, Ch 61)

| Tool | What It Does |
|------|-------------|
| **n8n** | Open-source workflow automation. Connect triggers to AI agent workflows. |
| **Temporal** | Durable workflow orchestration. Good for long-running agent processes. |
| **Inngest** | Event-driven AI workflows with retries and scheduling. |

---

## 6. Key Articles

### The Ramp Case Study

| Article | Key Takeaway |
|---------|-------------|
| **"How Ramp built Glass"** (Ramp Engineering Blog) | Architecture of Glass: Agent SDK, SSO-driven tool discovery, 30+ pre-connected integrations. The technical blueprint for Ch 59. |
| **"Building Dojo: A Skills Marketplace"** (Ramp Engineering Blog) | How Ramp built the skills marketplace with 350+ Git-backed skills. The blueprint for Ch 60. |
| **"AI Adoption at Ramp: From 0 to 99.5%"** (Ramp Engineering Blog) | The organizational playbook: L0-L3 levels, leaderboards, hub-and-spoke, removing constraints. The blueprint for Ch 62. |
| **"Glass: Lessons from Building an Internal AI Tool"** (Ramp Engineering Blog) | Retrospective on what worked and what didn't. "If the user has to debug, we've already lost." |

### The Anthropic Perspective

| Article | Key Takeaway |
|---------|-------------|
| **"The Anthropic Hive Mind"** (Steve Yegge) | How Anthropic uses their own tools internally. Meta-commentary on AI tool usage at an AI company. |
| **"How Claude Code Works"** (Anthropic) | Official architecture documentation for Claude Code. Foundation for Part 6. |
| **"Prompt Engineering Guide"** (Anthropic Docs) | Official best practices for prompting Claude. Foundation for Ch 4. |
| **"Tool Use Guide"** (Anthropic Docs) | Official documentation for Claude's tool calling capabilities. Foundation for Ch 8. |

### Industry Perspectives

| Article | Key Takeaway |
|---------|-------------|
| **"Building Effective Agents"** (Anthropic) | Anthropic's recommendations for agent architecture. Keep agents simple; complex frameworks often hurt more than help. |
| **"Stripe Minions"** (Stripe Engineering Blog) | How Stripe uses AI agents for code review, migration, and documentation. Real-world multi-agent system. |
| **"What We Learned from a Year of Building with LLMs"** (Various) | Practical lessons from companies building LLM applications. Common pitfalls and solutions. |

---

## 7. Model References

### Current Models (as of April 2026)

| Model | Provider | Context | Strengths | Guide Usage |
|-------|----------|---------|-----------|-------------|
| **Claude Opus 4** | Anthropic | 200K | Strongest reasoning, long-form analysis, coding | High-stakes tasks, architecture review |
| **Claude Sonnet 4** | Anthropic | 200K | Best balance of quality, speed, and cost | Default for most tasks in this guide |
| **Claude Haiku 3.5** | Anthropic | 200K | Fast and cheap | Bulk processing, classification, summarization |
| **GPT-4o** | OpenAI | 128K | Strong all-around, vision, audio | Multimodal tasks, second opinion in council |
| **GPT-4o mini** | OpenAI | 128K | Very fast, very cheap | High-volume, low-complexity tasks |
| **Gemini 2.5 Pro** | Google | 1M | Massive context window | Document analysis, long-form reasoning |
| **Llama 3.3 70B** | Meta (open) | 128K | Best open-source large model | Self-hosted inference, privacy-sensitive |
| **Mistral Large** | Mistral | 128K | Strong multilingual, coding | European deployments, multilingual |
| **Qwen 2.5** | Alibaba (open) | 128K | Strong coding, multilingual | Alternative open-source option |

---

## 8. Where to Start

**If you are just beginning:** Read Simon Willison's blog, take the fast.ai course, and work through Parts 0-2 of this guide.

**If you want to go deep on transformers:** Read "Attention Is All You Need," Jay Alammar's visual guides, and work through Part 7 of this guide.

**If you are building production systems:** Read Chip Huyen's "Designing Machine Learning Systems," set up Datadog or Langfuse, and work through Parts 10-11 of this guide.

**If you are driving AI adoption at your company:** Read the Ramp blog posts, work through Part 12, and start with the week-by-week playbook in Ch 62, Section 11.

---

*Previous: [Glossary](./appendix-glossary.md)* | *Next: [Cheat Sheet](./appendix-cheatsheet.md)*

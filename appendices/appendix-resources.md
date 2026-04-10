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
| **Language Models are Few-Shot Learners (GPT-3)** (Brown et al.) | 2020 | Demonstrated that large language models can perform tasks with just a few examples in the prompt. Introduced the era of in-context learning. | Ch 0, Ch 4 |
| **Training Language Models to Follow Instructions (InstructGPT)** (Ouyang et al.) | 2022 | Introduced RLHF for aligning language models to human instructions. The technique that made ChatGPT possible. | Ch 0 |
| **Constitutional AI** (Bai et al., Anthropic) | 2022 | Anthropic's approach to alignment: train the model with principles rather than just human feedback. Foundation of Claude's safety architecture. | Ch 48 |
| **Chain-of-Thought Prompting Elicits Reasoning in Large Language Models** (Wei et al.) | 2022 | Showed that prompting LLMs to "think step by step" dramatically improves reasoning on complex tasks. The technique is now standard practice. | Ch 4 |
| **Self-Instruct: Aligning Language Models with Self-Generated Instructions** (Wang et al.) | 2022 | Demonstrated that LLMs can generate their own training data from a small seed set, enabling low-cost instruction tuning. Key to synthetic data generation. | Ch 46 |

### Retrieval & RAG

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **Retrieval-Augmented Generation** (Lewis et al.) | 2020 | The original RAG paper. Showed that retrieving documents and conditioning generation on them improves factual accuracy. | Ch 15 |
| **Dense Passage Retrieval** (Karpukhin et al.) | 2020 | Showed that learned dense embeddings outperform BM25 for open-domain question answering. | Ch 14 |
| **REALM: Retrieval-Augmented Language Model** (Guu et al.) | 2020 | Pre-training with a retrieval component. Foundational to understanding RAG architectures. | Ch 15 |
| **Sentence-BERT** (Reimers & Gurevych) | 2019 | Adapted BERT for generating semantically meaningful sentence embeddings efficiently using siamese networks. | Ch 42 |
| **HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels** (Gao et al.) | 2022 | Generate a hypothetical answer, embed it, and use it for retrieval. Bridges the query-document gap for complex questions. | Ch 17 |
| **Lost in the Middle** (Liu et al.) | 2023 | Discovered that LLMs struggle to use information placed in the middle of long contexts. Important for RAG chunk ordering. | Ch 15, Ch 50 |

### Fine-Tuning & Efficiency

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **LoRA: Low-Rank Adaptation** (Hu et al.) | 2021 | The technique that made fine-tuning accessible. Train tiny adapter matrices instead of the full model. | Ch 45 |
| **QLoRA** (Dettmers et al.) | 2023 | Combine 4-bit quantization with LoRA. Fine-tune a 65B model on a single GPU. | Ch 45 |
| **GPTQ** (Frantar et al.) | 2022 | Post-training quantization using approximate second-order information. Run large models on consumer hardware. | Ch 47 |
| **Scaling Laws for Neural Language Models** (Kaplan et al.) | 2020 | Discovered power-law relationships between model size, data, compute, and performance. Guides model selection decisions. | Ch 43 |
| **Chinchilla: Training Compute-Optimal Large Language Models** (Hoffmann et al.) | 2022 | Showed that most models were undertrained: more data with fewer parameters often beats larger models with less data. Changed how the industry sizes models. | Ch 43 |
| **Speculative Decoding** (Leviathan et al.) | 2022 | Use a small draft model to generate candidate tokens, then verify in parallel with the large model. 2-3x speedup with no quality loss. | Ch 57 |

### Agents & Tools

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **ReAct: Synergizing Reasoning and Acting** (Yao et al.) | 2022 | The Reasoning + Acting pattern: interleave reasoning traces with actions. Foundation of modern agent architectures including Claude Code. | Ch 13 |
| **Toolformer** (Schick et al.) | 2023 | Showed that LLMs can learn to use tools (search, calculator, APIs) by self-supervised training on tool-augmented data. Demonstrated that tool use emerges naturally. | Ch 8 |
| **HuggingGPT** (Shen et al.) | 2023 | Using an LLM to coordinate multiple specialist models. Early multi-agent orchestration. | Ch 58 |
| **Reflexion: Language Agents with Verbal Reinforcement Learning** (Shinn et al.) | 2023 | Agents that reflect on their failures and improve in subsequent attempts. Introduces verbal self-reflection as a form of agent learning. | Ch 13, Ch 61 |

### Evaluation

| Paper | Year | Why It Matters | Guide Chapter |
|-------|------|---------------|---------------|
| **Judging LLM-as-a-Judge** (Zheng et al.) | 2023 | Systematic study of using LLMs to evaluate other LLMs. Establishes when this works and when it doesn't. | Ch 20 |
| **MTEB: Massive Text Embedding Benchmark** (Muennighoff et al.) | 2022 | The standard benchmark for comparing embedding models. Essential for model selection. | Ch 42 |
| **Holistic Evaluation of Language Models (HELM)** (Liang et al.) | 2022 | Comprehensive evaluation framework across many scenarios and metrics. Sets the standard for model evaluation rigor. | Ch 51 |

---

## 2. Books

### For Practitioners (Recommended)

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **AI Engineering** | Chip Huyen | Comprehensive guide to the emerging AI engineering discipline. Covers the full lifecycle from prototyping to production. The closest book to this guide's philosophy. |
| **Designing Machine Learning Systems** | Chip Huyen | The best book on production ML systems. Covers the engineering around the model, not just the model. Essential reading for Part 10-11. |
| **Building LLM Apps** | Valentina Alto | Practical guide to building applications with LLMs. Good TypeScript/Python code examples. |
| **Natural Language Processing with Transformers** | Lewis Tunstall, Leandro von Werra, Thomas Wolf | By the Hugging Face team. Deep dive into transformers, fine-tuning, and the Hugging Face ecosystem. Essential companion for Part 8-9. |
| **Prompt Engineering for Generative AI** | James Phoenix, Mike Taylor | Practical techniques for prompt engineering across different model providers. Good supplementary reading for Ch 4. |

### For Deeper Understanding

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Deep Learning** | Ian Goodfellow, Yoshua Bengio, Aaron Courville | The definitive textbook. Dense but comprehensive. Best for understanding fundamentals. Free online. |
| **The Hundred-Page Machine Learning Book** | Andriy Burkov | Exactly what it says. Best for quickly reviewing ML concepts when you need a refresher. |
| **Speech and Language Processing** | Dan Jurafsky, James H. Martin | The classic NLP textbook. Free online. Excellent for understanding the progression from traditional NLP to modern approaches. |
| **Grokking Deep Learning** | Andrew Trask | Build neural networks from scratch in Python. Approachable for engineers without ML background. Aligns with Ch 36. |
| **Build a Large Language Model (From Scratch)** | Sebastian Raschka | Step-by-step guide to building a GPT-style model. Covers data preparation, attention, pre-training, and fine-tuning. The code companion for Part 7. |

### For Strategy and Organization

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Prediction Machines** | Ajay Agrawal, Joshua Gans, Avi Goldfarb | Economic framework for thinking about AI as a drop in the cost of prediction. Useful for the cost arguments in Ch 62. |
| **The AI Organization** | David De Cremer | How companies should structure themselves around AI capabilities. Relevant to Ch 62. |
| **Co-Intelligence: Living and Working with AI** | Ethan Mollick | Practical frameworks for human-AI collaboration from Wharton's leading AI researcher. Useful for the adoption strategies in Ch 62. |
| **Team of Teams** | General Stanley McChrystal | While not about AI, this book on decentralized organizational structure directly informs the hub-and-spoke model for AI adoption in Ch 62. |

### For Reference

| Book | Author(s) | Why Read It |
|------|-----------|------------|
| **Hands-On Large Language Models** | Jay Alammar, Maarten Grootendorst | Practical guide with excellent visualizations. Covers embeddings, RAG, fine-tuning, and generation. By the author of the best transformer visualizations. |
| **What Is ChatGPT Doing... and Why Does It Work?** | Stephen Wolfram | Short, accessible explanation of how LLMs work for a technical audience. Good starting point for Part 0. |

---

## 3. Courses

### Referenced in This Guide

| Course | Platform | Instructor | Guide Connection |
|--------|----------|-----------|-----------------|
| **AI Agents v2** | Frontend Masters | Scott Moss | Agent loops, tool calling, multi-agent patterns. The hands-on companion for Part 2 and Ch 58. |
| **Agents in Production** | Frontend Masters | Scott Moss | Production deployment of AI agents: scaling, monitoring, and reliability. Companion for Part 10-11. |
| **Open Source AI** | Frontend Masters | Steve Kinney | Hugging Face Pipelines, Sentence Transformers, model hub, inference. Aligns with Part 8. |
| **Hard Parts: Neural Networks** | Frontend Masters | Will Sentance | Building neural nets from scratch in JavaScript. The pedagogical companion for Part 7. |
| **Fullstack Next.js v4** | Frontend Masters | Brian Holt | Full-stack Next.js with AI features. Relevant to deploying AI-powered applications (Ch 53). |

### Additional Recommended Courses

| Course | Platform | Why Take It |
|--------|----------|------------|
| **Neural Networks: Zero to Hero** | YouTube | Andrej Karpathy | Build a GPT from scratch, step by step. The single best video series for understanding transformers. Free. Start here if you want to understand how LLMs actually work. |
| **Practical Deep Learning for Coders** | fast.ai | Jeremy Howard's legendary course. Top-down approach: build things first, understand later. Mirrors this guide's philosophy. |
| **CS224N: NLP with Deep Learning** | Stanford (free) | Chris Manning's graduate NLP course. Goes deep into transformers, attention, and modern NLP. |
| **Full Stack LLM Bootcamp** | The Full Stack | Practical end-to-end LLM application development. Good for bridging theory and practice. |
| **LLM University** | Cohere | Free comprehensive course covering embeddings, RAG, and LLM fundamentals. |
| **DeepLearning.AI Short Courses** | DeepLearning.AI | Andrew Ng's platform. Many focused 1-2 hour courses on specific topics (RAG, agents, fine-tuning). |
| **Hugging Face NLP Course** | Hugging Face (free) | Official course covering the Hugging Face ecosystem. Essential for Part 8-9. |
| **fast.ai Part 2: Deep Learning Foundations** | fast.ai | Goes deeper into the foundations: builds a training framework from scratch. Good for Part 7 and 9. |
| **Machine Learning Engineering for Production (MLOps)** | Coursera (Andrew Ng) | Covers the production lifecycle: data pipelines, model serving, monitoring, drift detection. |
| **Stanford CS25: Transformers United** | Stanford (free) | Guest lectures from transformer researchers. Covers vision transformers, scaling, and more. |

---

## 4. Blogs & Newsletters

### Must-Follow

| Blog / Newsletter | Author | Why Follow |
|-------------------|--------|-----------|
| **Simon Willison's Blog** | Simon Willison | The single best blog on practical AI engineering. Covers tools, techniques, and implications with deep technical rigor and intellectual honesty. His "lethal trifecta" concept (Ch 48) is essential reading for AI security. |
| **Lilian Weng's Blog** | Lilian Weng (OpenAI) | Extraordinary technical deep-dives. Her posts on attention, agents, prompting, and LLM-powered agents are reference-quality. Treat these as supplementary textbook chapters. |
| **Jay Alammar's Blog** | Jay Alammar | The best visual explanations of transformers, attention, and NLP concepts in existence. "The Illustrated Transformer" and "The Illustrated BERT" should be mandatory reading alongside Ch 38. |
| **The Batch** | Andrew Ng / DeepLearning.AI | Weekly newsletter covering AI news, research, and applications. Good for staying current. |
| **Anthropic Blog** | Anthropic | Research publications, model releases, and safety research. Essential for understanding Claude's capabilities. |
| **OpenAI Blog** | OpenAI | Model announcements, research, and best practices. |
| **The Gradient** | Various | In-depth articles on ML research for practitioners. |

### Also Worth Reading

| Blog / Newsletter | Focus |
|-------------------|-------|
| **Chip Huyen's Blog** | Production ML systems, AI engineering, industry trends. Author of "Designing Machine Learning Systems" and "AI Engineering." Her writing bridges the gap between research and production. |
| **Eugene Yan's Blog** | Applied ML, RecSys, LLM applications. Practical, well-written, and opinionated. Excellent posts on evaluation, RAG, and production patterns. |
| **Hamel Husain's Blog** | Fine-tuning, evaluation, and practical LLM engineering. Co-creator of CodeSearchNet. Deep expertise in making LLMs work in practice. Excellent posts on systematic evaluation. |
| **Sebastian Raschka's Blog** | Deep learning, LLMs, fine-tuning. Author of "Machine Learning with PyTorch and Scikit-Learn." |
| **Ahead of AI Newsletter** | Sebastian Raschka's newsletter. Monthly deep-dives into LLM research with practical takeaways. |
| **Latent Space Podcast/Newsletter** | swyx & Alessio. Interviews with AI practitioners. Great for understanding how companies actually use AI. |
| **Interconnects** | Nathan Lambert (ex-HuggingFace). RLHF, alignment, and open-source model training. Deep technical analysis of new model releases. |
| **The AI Engineer Newsletter** | Ben Tossell / Various. Industry news focused on building with AI, not just AI research. |
| **Elicit Blog** | The research assistant company's blog. Practical insights on using AI for research at scale. |
| **Braintrust Blog** | Evaluation and observability best practices from the Braintrust team. |
| **Modal Blog** | GPU compute, inference optimization, and serverless AI deployment patterns. |
| **Vercel Blog (AI section)** | AI SDK updates, deployment patterns, and production AI architecture posts. |

---

## 5. Tools & Libraries

### SDKs and Frameworks (Ch 3, Ch 6, Ch 8, Ch 13)

| Tool | Language | What It Does |
|------|----------|-------------|
| **Anthropic SDK** (`@anthropic-ai/sdk`) | TypeScript, Python | Official SDK for Claude. Messages API, tool calling, streaming. |
| **Claude Agent SDK** | TypeScript, Python | Anthropic's framework for building production agents. Provides the agent loop, tool management, and orchestration primitives. |
| **OpenAI SDK** (`openai`) | TypeScript, Python | Official SDK for GPT models. Chat completions, tool calling, structured output. |
| **Vercel AI SDK** (`ai`) | TypeScript | Framework-agnostic SDK for building AI-powered UIs. Streaming, tool calling, multi-provider. |
| **LangChain** | TypeScript, Python | Comprehensive framework for LLM applications. Chains, agents, memory, retrieval. Large ecosystem of integrations. |
| **Mastra** | TypeScript | TypeScript-first AI framework. Agents, workflows, RAG, integrations. |
| **CrewAI** | Python | Framework for building multi-agent systems with role-based agent collaboration. |
| **LlamaIndex** | Python | Data framework for LLM applications. Excels at RAG and document processing. |

### Vector Databases (Ch 14)

| Tool | Type | Best For |
|------|------|---------|
| **Pinecone** | Managed SaaS | Fastest time to production. Zero ops. Serverless and pod-based options. |
| **Upstash Vector** | Serverless SaaS | Pay-per-request pricing. Perfect for serverless AI applications on Vercel. |
| **Weaviate** | Self-hosted or cloud | Hybrid search, GraphQL API, multi-tenancy. |
| **Qdrant** | Self-hosted or cloud | High performance, rich filtering, payload indexing. |
| **ChromaDB** | Self-hosted | Simple, developer-friendly, great for prototyping and local development. |
| **pgvector** | PostgreSQL extension | Adding vector search to existing Postgres. Pragmatic when you already use PG. |
| **Milvus** | Self-hosted or cloud | Distributed vector database for billion-scale datasets. |

### Evaluation (Ch 18-21, Ch 51)

| Tool | What It Does |
|------|-------------|
| **Braintrust** | Eval framework with logging, scoring, experiment tracking, and dataset management. |
| **Laminar** | OpenTelemetry-native LLM observability. Traces, evals, cost tracking. |
| **Arize Phoenix** | Open-source LLM observability. Traces, evaluations, datasets. |
| **Promptfoo** | Open-source prompt testing and eval framework. CI-friendly. Compare prompts side-by-side. |
| **Evalite** | Lightweight eval library for TypeScript. Simple API for running evals locally. |

### Observability (Ch 22, Ch 52)

| Tool | What It Does |
|------|-------------|
| **Datadog LLM Observability** | Enterprise-grade monitoring for LLM applications. APM integration, trace correlation. |
| **LangSmith** | LangChain's observability platform. Trace debugging, dataset management, annotation queues. |
| **Langfuse** | Open-source LLM observability. Traces, scores, prompt management. Self-hostable. |
| **Helicone** | LLM observability proxy. Logs all requests, tracks cost, caches. Drop-in integration. |

### MCP (Ch 24)

| Tool | What It Does |
|------|-------------|
| **MCP SDK** (`@modelcontextprotocol/sdk`) | Official SDK for building MCP servers and clients. TypeScript. |
| **MCP Python SDK** (`mcp`) | Official Python SDK for MCP servers. |
| **MCP server registry** | Community registry of pre-built MCP servers for common services (Slack, GitHub, databases, etc.). |
| **MCP Inspector** | Debugging tool for testing MCP servers interactively. |
| **Smithery** | Marketplace for discovering and installing pre-built MCP servers. |

### Open-Source Models (Ch 40-43, Ch 47)

| Tool | What It Does |
|------|-------------|
| **Hugging Face Transformers** | The standard library for working with transformer models. 200K+ models. |
| **Hugging Face Hub** | Model repository with 500K+ models. Model cards, benchmarks, and usage examples. |
| **Sentence Transformers** | Generate sentence embeddings. 5000+ pre-trained models. |
| **llama.cpp** | Run quantized LLMs on CPU. GGUF format. Powers most local inference. |
| **Ollama** | Run open-source LLMs locally with a simple CLI. `ollama run llama3.3` and go. |
| **vLLM** | High-throughput LLM inference engine with PagedAttention and continuous batching. |
| **TGI (Text Generation Inference)** | Hugging Face's optimized inference server. Production-grade. |
| **FAISS** | Facebook's library for efficient similarity search and clustering of dense vectors. |

### Fine-Tuning (Ch 44-46)

| Tool | What It Does |
|------|-------------|
| **PEFT** | Hugging Face's library for parameter-efficient fine-tuning (LoRA, QLoRA, adapters, prefix tuning). |
| **TRL** | Transformer Reinforcement Learning. Fine-tune with RLHF, DPO, SFT, PPO. |
| **Axolotl** | Streamlined fine-tuning toolkit. YAML config-driven. Supports LoRA, QLoRA, full fine-tuning. |
| **Unsloth** | 2x faster fine-tuning with 80% less memory. Custom CUDA kernels. |
| **AutoTrain** | Hugging Face's no-code fine-tuning tool. Point at a dataset and go. |

### Automation (Ch 25, Ch 59, Ch 61)

| Tool | What It Does |
|------|-------------|
| **n8n** | Open-source workflow automation. Connect triggers to AI agent workflows. |
| **Temporal** | Durable workflow orchestration. Good for long-running agent processes with retry. |
| **Inngest** | Event-driven AI workflows with retries, scheduling, and step functions. |

### Deployment & Infrastructure (Ch 53-57)

| Tool | What It Does |
|------|-------------|
| **Vercel** | Serverless deployment platform. Fluid Compute for AI workloads. Native AI SDK integration. |
| **Docker** | Container runtime. Essential for agent sandboxing (OpenClaw pattern). |
| **Fly.io** | Edge compute platform. Good for deploying AI workloads close to users. |
| **Modal** | Serverless GPU compute. Run inference and fine-tuning without managing infrastructure. |
| **Replicate** | Run open-source models via API. Pay per prediction. No GPU management. |
| **BentoML** | Framework for serving ML models as APIs. Supports batching, async, and custom logic. |

### Provider Abstraction & Routing (Ch 54)

| Tool | What It Does |
|------|-------------|
| **Vercel AI Gateway** | Model routing, provider failover, cost tracking across multiple providers. |
| **LiteLLM** | Unified API for 100+ LLM providers. Drop-in OpenAI replacement. |
| **Portkey** | AI gateway with caching, retries, load balancing, and cost tracking. |
| **Martian** | Intelligent model router that selects the optimal model per request. |

---

## 6. Key Articles & Blog Posts

### The Ramp Case Study

| Article | Key Takeaway |
|---------|-------------|
| **"How Ramp built Glass"** (Ramp Engineering Blog) | Architecture of Glass: Agent SDK, SSO-driven tool discovery, 30+ pre-connected integrations. The technical blueprint for Ch 59. |
| **"We Built Every Employee Their Own AI Coworker"** (Ramp Blog) | The vision and strategy behind giving every Ramp employee a personalized AI assistant. How Glass became the daily companion for 99.5% of the company. |
| **"How to Get Your Company AI Pilled"** (Ramp Blog) | The organizational change management playbook: how Ramp drove adoption from skeptical engineers to enthusiastic power users. Tactical advice on leaderboards, champions, and removing friction. |
| **"Building Dojo: A Skills Marketplace"** (Ramp Engineering Blog) | How Ramp built the skills marketplace with 350+ Git-backed skills. The blueprint for Ch 60. |
| **"AI Adoption at Ramp: From 0 to 99.5%"** (Ramp Engineering Blog) | The organizational playbook: L0-L3 levels, leaderboards, hub-and-spoke, removing constraints. The blueprint for Ch 62. |
| **"Glass: Lessons from Building an Internal AI Tool"** (Ramp Engineering Blog) | Retrospective on what worked and what didn't. "If the user has to debug, we've already lost." |

### The Anthropic Perspective

| Article | Key Takeaway |
|---------|-------------|
| **"The Anthropic Hive Mind"** (Steve Yegge) | How Anthropic uses their own tools internally. Meta-commentary on AI tool usage at an AI company. Yegge's signature blend of technical depth and cultural observation. Essential context for understanding why developer tools matter. |
| **"How Claude Code Works"** (Anthropic) | Official architecture documentation for Claude Code. Foundation for Part 6. |
| **"Building Effective Agents"** (Anthropic) | Anthropic's recommendations for agent architecture. Keep agents simple; complex frameworks often hurt more than help. The philosophical foundation for this guide's approach to agent engineering. |
| **"Prompt Engineering Guide"** (Anthropic Docs) | Official best practices for prompting Claude. Foundation for Ch 4. |
| **"Tool Use Guide"** (Anthropic Docs) | Official documentation for Claude's tool calling capabilities. Foundation for Ch 8. |

### Industry Perspectives

| Article | Key Takeaway |
|---------|-------------|
| **"Stripe Minions: Multi-Agent System for Code at Scale"** (Stripe Engineering Blog) | How Stripe uses AI agents for code review, migration, and documentation at massive scale. Real-world multi-agent system with practical architecture lessons. Multiple blog posts cover the evolution of the system. |
| **"What We Learned from a Year of Building with LLMs"** (Various) | Practical lessons from companies building LLM applications. Common pitfalls and solutions. Covers the full lifecycle from prototype to production. |
| **"Emerging Architectures for LLM Applications"** (a16z) | Andreessen Horowitz's survey of LLM application patterns: RAG, fine-tuning, agents, evaluations. Good high-level overview of the landscape. |
| **"Patterns for Building LLM-based Systems & Products"** (Eugene Yan) | Practical engineering patterns: evals, RAG, fine-tuning, caching, guardrails. One of the best practitioner-written pieces on production LLM systems. |
| **"The AI-Powered Developer Workflow"** (GitHub Engineering) | How GitHub built Copilot's architecture. Insights into code completion, context gathering, and prompt optimization at scale. |
| **"Large Language Models Explained"** (Mark Riedl) | Accessible explanation of how transformers and LLMs work. Good companion to Part 7. |

### Anthropic Documentation (Essential Reference)

| Document | What It Covers |
|----------|---------------|
| **Anthropic API Docs** | Complete API reference: Messages, Tool Use, Streaming, Batches, Vision, Extended Thinking. |
| **Claude Model Card** | Official model capabilities, limitations, and safety information. |
| **Prompt Engineering Guide** | Best practices: system prompts, XML tags, chain-of-thought, few-shot, role prompting. |
| **Tool Use (Function Calling) Guide** | How to define tools, handle tool calls, and build agent loops with Claude. |
| **Prompt Caching Guide** | How to cache system prompts and frequently-repeated content for cost and latency savings. |
| **Extended Thinking Guide** | How to use Claude's extended thinking mode for complex reasoning tasks. |
| **MCP Specification** | The full Model Context Protocol specification for building servers and clients. |

---

## 7. Podcasts & Video Channels

### Must-Watch / Must-Listen

| Resource | Host(s) | Why Follow |
|----------|---------|-----------|
| **Latent Space Podcast** | swyx & Alessio Fanelli | The best podcast for AI engineering practitioners. Long-form interviews with builders at Anthropic, OpenAI, Vercel, and startups. |
| **Andrej Karpathy's YouTube** | Andrej Karpathy | "Neural Networks: Zero to Hero" is the definitive video course. "Let's build GPT" has 15M+ views for a reason. |
| **Lex Fridman Podcast** | Lex Fridman | Long-form interviews with AI researchers (Altman, Amodei, LeCun, Hinton). Good for big-picture context. |
| **3Blue1Brown** | Grant Sanderson | The best visual math explanations. His neural network and linear algebra series are essential for Part 7. |
| **Yannic Kilcher** | Yannic Kilcher | Paper walkthroughs. Good for understanding new research papers without reading the full paper yourself. |
| **Two Minute Papers** | Karoly Zsolnai-Feher | Quick, accessible summaries of new AI research. Good for staying current in 2 minutes per paper. |

---

## 8. Key GitHub Repositories

### Models & Inference

| Repository | Stars | What It Is |
|-----------|-------|-----------|
| **huggingface/transformers** | 140K+ | The standard library for working with transformer models. If you work with open-source AI, you use this. |
| **ggerganov/llama.cpp** | 75K+ | Run LLMs on CPU with quantization. The engine behind Ollama and most local LLM tools. |
| **vllm-project/vllm** | 40K+ | High-throughput LLM serving with PagedAttention. The standard for production self-hosted inference. |
| **ollama/ollama** | 120K+ | Run open-source LLMs locally with a single command. Downloads, configures, and serves models. |
| **huggingface/peft** | 17K+ | Parameter-efficient fine-tuning library. LoRA, QLoRA, prefix tuning, and more. |
| **unslothai/unsloth** | 25K+ | 2x faster fine-tuning with 80% less memory. Makes fine-tuning large models accessible. |

### Frameworks & SDKs

| Repository | Stars | What It Is |
|-----------|-------|-----------|
| **vercel/ai** | 15K+ | Vercel AI SDK. Framework-agnostic TypeScript SDK for building AI-powered applications. |
| **langchain-ai/langchain** | 100K+ | Comprehensive LLM application framework. Chains, agents, memory, retrieval. |
| **langchain-ai/langchainjs** | 14K+ | TypeScript port of LangChain. Aligns with this guide's TypeScript-first approach. |
| **openai/openai-cookbook** | 65K+ | Official OpenAI examples and best practices. Covers embeddings, RAG, function calling, and more. |
| **anthropics/anthropic-cookbook** | 10K+ | Official Anthropic examples. Tool use, prompt caching, multimodal, and Claude best practices. |
| **modelcontextprotocol/sdk** | 5K+ | Official MCP SDK. Build MCP servers and clients. |

### Tools & Applications

| Repository | Stars | What It Is |
|-----------|-------|-----------|
| **anthropics/claude-code** | 30K+ | Claude Code CLI source and documentation. The reference implementation for agent harness architecture. |
| **anthropics/anthropic-cookbook** | 10K+ | Official Anthropic examples: tool use, prompt caching, multimodal, RAG, agents. |
| **BerriAI/litellm** | 18K+ | Unified API for 100+ LLM providers. Useful for provider abstraction and failover. |
| **chroma-core/chroma** | 18K+ | Open-source embedding database. Simple, developer-friendly. |
| **run-llama/llama_index** | 40K+ | Data framework for LLM applications. RAG pipelines, document loaders, query engines. |
| **promptfoo/promptfoo** | 5K+ | Open-source prompt testing and eval framework. CI-friendly. YAML config. |
| **mastra-ai/mastra** | 8K+ | TypeScript-first AI framework for agents, workflows, and RAG. |

---

## 9. Model References

### Current Models (as of April 2026)

| Model | Provider | Context | Strengths | Guide Usage |
|-------|----------|---------|-----------|-------------|
| **Claude Opus 4** | Anthropic | 200K | Strongest reasoning, long-form analysis, complex coding | High-stakes tasks, architecture review, agent orchestration |
| **Claude Sonnet 4** | Anthropic | 200K | Best quality/cost ratio, strong coding, tools | Default for most tasks in this guide |
| **Claude Haiku 3.5** | Anthropic | 200K | Fast and cheap | Bulk processing, classification, summarization, routing |
| **GPT-4o** | OpenAI | 128K | Strong all-around, vision, audio | Multimodal tasks, second opinion in council pattern |
| **GPT-4o mini** | OpenAI | 128K | Very fast, very cheap | High-volume, low-complexity tasks |
| **Gemini 2.5 Pro** | Google | 1M | Massive context window, strong reasoning | Whole-codebase analysis, long documents |
| **Gemini 2.5 Flash** | Google | 1M | Fast with huge context | Bulk processing with long context |
| **Llama 3.3 70B** | Meta (open) | 128K | Best open-source large model | Self-hosted inference, privacy-sensitive |
| **Llama 3.2 8B** | Meta (open) | 128K | Small but capable | Local development, edge deployment |
| **Mistral Large** | Mistral | 128K | Strong multilingual, coding | European deployments, multilingual |
| **Qwen 2.5 72B** | Alibaba (open) | 128K | Strong coding, multilingual | Alternative open-source option |

### Model Selection Quick Guide

| Scenario | Recommended Model | Why |
|----------|------------------|-----|
| Daily coding agent | Claude Sonnet 4 | Best coding quality per dollar |
| Bulk classification | Claude Haiku 3.5 | Cheapest per token with good accuracy |
| Architecture decisions | Claude Opus 4 | Deepest reasoning for complex trade-offs |
| Analyzing a 500-page doc | Gemini 2.5 Pro | 1M context window fits entire documents |
| Self-hosted, private data | Llama 3.3 70B | Best open-source quality, full data control |
| Local development | Llama 3.2 8B via Ollama | Runs on laptop, fast iteration |
| Multi-model council | Mix of providers | Different perspectives improve decision quality |

---

## 10. Tools by Category

### AI-Powered IDEs & Coding Agents

| Tool | Type | Best For |
|------|------|---------|
| **Claude Code** | CLI agent | Terminal-native. Best for complex tasks, multi-file changes, agent workflows. |
| **Cursor** | IDE (VS Code fork) | Tab completion + chat. Best for interactive coding with visual context. |
| **Windsurf** | IDE (VS Code fork) | Similar to Cursor with different UX philosophy. Strong autocomplete. |
| **GitHub Copilot** | IDE extension | Inline suggestions. Best for pure code completion in existing editors. |

### AI Frameworks

| Tool | Language | Strengths |
|------|----------|----------|
| **Vercel AI SDK** | TypeScript | Multi-provider, streaming, tool calling, structured output. Framework-agnostic. |
| **LangChain** | TS, Python | Comprehensive. Large ecosystem of integrations. Good for complex RAG. |
| **Mastra** | TypeScript | TypeScript-native. Agents, workflows, RAG, integrations. |
| **CrewAI** | Python | Multi-agent collaboration with role-based agents. |
| **LlamaIndex** | Python | Data-focused. Best for document processing and RAG pipelines. |
| **Semantic Kernel** | C#, Python | Microsoft's framework. Good for .NET shops. |

### Eval & Testing Tools

| Tool | Type | Best For |
|------|------|---------|
| **Braintrust** | Platform | Full eval lifecycle: datasets, scoring, experiments, production monitoring. |
| **Laminar** | Platform | OpenTelemetry-native. Traces, evals, cost tracking. |
| **Promptfoo** | Open-source CLI | CI-friendly prompt testing. Side-by-side comparison. |
| **Arize Phoenix** | Open-source | Tracing, evaluations, dataset management. Self-hostable. |

### Observability & Monitoring

| Tool | Type | Best For |
|------|------|---------|
| **Datadog LLM Observability** | Enterprise | APM integration, trace correlation, dashboards. Best if you already use Datadog. |
| **LangSmith** | Platform | Deep LangChain integration. Trace debugging, annotation queues. |
| **Langfuse** | Open-source | Self-hostable. Traces, scores, prompt management. |
| **Helicone** | Proxy | Drop-in. Logs all LLM requests, tracks cost, caches responses. |

---

## 11. Where to Start

### By Experience Level

**If you are just beginning:**
1. Read Simon Willison's blog (especially his LLM and tool use posts)
2. Watch Andrej Karpathy's "Neural Networks: Zero to Hero" series
3. Work through Parts 0-2 of this guide
4. Set up Claude Code and start using it daily (Ch 23)

**If you want to go deep on transformers:**
1. Read "Attention Is All You Need" (the paper)
2. Read Jay Alammar's "The Illustrated Transformer" (the visual guide)
3. Watch Andrej Karpathy's "Let's build GPT from scratch" video
4. Work through Part 7 of this guide (Ch 36-39)

**If you are building production systems:**
1. Read Chip Huyen's "Designing Machine Learning Systems"
2. Read "What We Learned from a Year of Building with LLMs"
3. Set up Datadog or Langfuse for observability
4. Work through Parts 10-11 of this guide

**If you are driving AI adoption at your company:**
1. Read the Ramp blog posts ("We Built Every Employee Their Own AI Coworker" and "How to Get Your Company AI Pilled")
2. Read Steve Yegge's "The Anthropic Hive Mind"
3. Work through Part 12, especially Ch 62
4. Start with the week-by-week playbook in Ch 62, Section 11

**If you want to fine-tune models:**
1. Read the LoRA and QLoRA papers
2. Take fast.ai Part 2
3. Work through Part 9 of this guide (Ch 44-47)
4. Start with Unsloth or Axolotl for your first fine-tuning run

### By Time Available

| Time | Path |
|------|------|
| **1 hour** | Ch 0 (How LLMs Work) + Ch 3 (Your First LLM Call) |
| **1 day** | Parts 0-1 (Fundamentals + Building with LLMs) |
| **1 week** | Phase 1 (Parts 0-5, "Get Dangerous") |
| **1 month** | Full guide (Parts 0-12, both phases) |
| **Ongoing** | Subscribe to Simon Willison, Lilian Weng, and Latent Space |

---

*Previous: [Glossary](./appendix-glossary.md)* | *Next: [Cheat Sheet](./appendix-cheatsheet.md)*

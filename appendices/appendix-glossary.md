<!--
  TYPE: appendix
  TITLE: Glossary
  UPDATED: 2026-04-10
-->

# Appendix: Glossary

> 200+ AI/ML/LLM terms for the practicing engineer. Each term includes a brief definition and the chapter(s) where it is covered in depth.

---

## A

**A/B Testing (for AI)** — Running two variants of an AI system simultaneously to measure which performs better. Differs from traditional A/B testing because AI outputs are non-deterministic. Ch 51, Ch 56, Ch 61.

**Activation Function** — A mathematical function applied to a neuron's output that introduces non-linearity, allowing neural networks to learn complex patterns. Common examples: sigmoid, ReLU, tanh. Ch 36.

**Adapter (LoRA)** — A small set of trainable parameters added to a frozen pre-trained model. Instead of fine-tuning all weights, only the adapter weights are trained, drastically reducing compute and memory. Ch 45.

**Agent** — An AI system that uses an LLM to decide which actions to take, executes those actions via tools, observes the results, and repeats until a task is complete. Ch 9, Ch 13, Ch 27, Ch 58.

**Agent Loop** — The core cycle of an AI agent: prompt the LLM, receive a tool call or response, execute the tool, append the result, repeat. Ch 9, Ch 27, Ch 58.

**Agent SDK** — Anthropic's official SDK for building AI agents. Provides the agent loop, tool calling, streaming, and multi-turn conversation management. Ch 59.

**Alignment** — The challenge of ensuring AI systems behave according to human values and intentions. Includes techniques like RLHF, Constitutional AI, and red teaming. Ch 48.

**Attention Mechanism** — The core innovation of the transformer architecture. Allows each token to "attend to" (compute relevance scores with) every other token in the sequence. Ch 38.

**AutoDream** — Claude Code's background memory consolidation system. During idle time, the agent reviews recent interactions, identifies patterns, and consolidates them into long-term memory. Ch 28, Ch 61.

**Agentic Workflow** — A workflow where an AI agent autonomously decides which tools to use, in what order, and when to stop. Contrasts with deterministic pipelines where each step is pre-defined. Ch 9, Ch 13.

**API Gateway** — A server that sits between clients and AI providers, handling rate limiting, caching, failover, and cost tracking. Ch 54.

**Approval Flow** — A mechanism where certain AI actions (e.g., writing files, executing code, sending messages) require explicit human confirmation before execution. Ch 11, Ch 30.

**AWQ (Activation-Aware Weight Quantization)** — A quantization method that preserves important weights at higher precision based on activation patterns. Ch 47.

## B

**Backpropagation** — The algorithm for computing gradients of the loss function with respect to each weight in a neural network, enabling gradient descent to update weights. Ch 36.

**Batch Inference** — Processing multiple inputs through a model simultaneously rather than one at a time. Increases throughput and reduces per-item cost. Ch 57.

**Beam Search** — A decoding strategy that maintains multiple candidate sequences (beams) at each generation step and selects the highest-probability complete sequence. Ch 39.

**BERT** — Bidirectional Encoder Representations from Transformers. An encoder-only transformer trained with masked language modeling. Excellent for classification, NER, and embedding generation but not for text generation. Ch 38, Ch 43.

**BPE (Byte Pair Encoding)** — A tokenization algorithm that iteratively merges the most frequent pairs of characters or subwords. Used by GPT models. Ch 0, Ch 37.

**Bias (ML)** — A constant added to a neuron's weighted sum before the activation function. Allows the model to shift the decision boundary. Ch 36.

**Bias (Fairness)** — Systematic errors in AI outputs that reflect prejudices in training data or model design. Ch 48.

**BM25** — A probabilistic ranking function for keyword-based document retrieval. The standard baseline for sparse retrieval systems. Ch 17.

**Brownfield** — Integrating AI into an existing codebase or product, as opposed to greenfield (building from scratch). Most real-world AI engineering is brownfield. Ch 3.

## C

**Cache-Break Vector** — A technique used in context window management where specific tokens or patterns are inserted to prevent prompt caching from reusing stale context. Ch 29.

**Canary Deployment** — A deployment strategy where a change is rolled out to a small percentage of traffic first, monitored for issues, then gradually expanded. Essential for prompt and model changes in AI systems. Ch 56.

**Chain-of-Thought (CoT)** — A prompting technique that asks the LLM to show its reasoning step by step before giving a final answer. Improves accuracy on complex tasks. Ch 4.

**Chunking** — Splitting documents into smaller pieces (chunks) for embedding and retrieval in RAG systems. Chunk size and overlap significantly affect retrieval quality. Ch 15.

**CLAUDE.md** — A markdown file placed in a project root that configures Claude Code's behavior for that project. Contains project context, conventions, and instructions. Ch 23, Ch 28.

**Claude Code** — Anthropic's CLI-based AI coding agent. Uses the agent loop with tools for file operations, search, and shell commands. Ch 23, Ch 27.

**Classification** — A machine learning task that assigns inputs to predefined categories. Ch 34.

**Compaction** — The process of summarizing or condensing conversation history to free up context window space. Used when conversations approach the token limit. Ch 12, Ch 29, Ch 50.

**Connector** — In platform engineering, a module that integrates an external service (Salesforce, Slack, GitHub) into the AI tool's capabilities. Ch 59.

**Constitutional AI** — Anthropic's approach to AI alignment where the model is trained to follow a set of principles (a "constitution") that guide its behavior. Ch 48.

**Context Window** — The maximum number of tokens an LLM can process in a single request (input + output combined). Determines how much information the model can consider at once. Ch 0, Ch 12, Ch 29, Ch 50.

**Coordinator Pattern** — A multi-agent pattern where one agent (the coordinator) decomposes tasks, delegates to worker agents, and synthesizes results. Ch 32, Ch 58.

**Cosine Similarity** — A measure of similarity between two vectors, computed as the dot product divided by the product of their magnitudes. Range: -1 to 1. Widely used for comparing embeddings. Ch 1, Ch 14.

**Cross-Attention** — An attention mechanism where the queries come from one sequence and the keys/values come from another. Used in encoder-decoder transformers. Ch 38.

**Content Filtering** — Automated screening of AI inputs and outputs to block harmful, offensive, or policy-violating content. Ch 48.

**Council Pattern** — A multi-agent deliberation pattern where multiple models with different perspectives evaluate the same question, cross-review each other, and a synthesizer produces a verdict. Ch 58.

**Cron Job (AI)** — A scheduled background task that triggers an AI agent on a recurring schedule. Used for digests, monitoring, and automated reporting. Ch 25, Ch 59.

**Cursor** — An AI-powered IDE that integrates LLM capabilities directly into the code editor. Ch 27.

## D

**Data Augmentation** — Techniques for expanding training datasets by creating modified versions of existing data points. Ch 46.

**Decoding** — The process of generating text from an LLM by selecting tokens one at a time based on predicted probability distributions. Ch 39.

**Dense Retrieval** — Using embedding similarity (rather than keyword matching) to find relevant documents. Ch 14, Ch 17.

**Diffusion Model** — A generative model that learns to reverse a noise-adding process: starting from pure noise, it iteratively denoises to produce images guided by text prompts. Stable Diffusion is the most widely used open-source example. Ch 41.

**Distillation** — Training a smaller "student" model to mimic the behavior of a larger "teacher" model. Ch 47.

**DreamBooth** — A technique for personalizing diffusion models by fine-tuning on 3-10 images of a specific subject. Combined with LoRA for memory efficiency. Ch 41.

**Dojo** — Ramp's internal skills marketplace with 350+ Git-backed, versioned, code-reviewed AI skills. Ch 60.

**Decoder** — The part of a transformer that generates output text token by token. GPT models are decoder-only architectures. Ch 38, Ch 43.

**Decision Boundary** — In ML, the surface that separates different classes in the feature space. The goal of training is to find the optimal boundary. Ch 34.

**DPO (Direct Preference Optimization)** — An alternative to RLHF that directly optimizes model behavior based on human preferences without needing a separate reward model. Ch 44.

**Dropout** — A regularization technique that randomly sets a fraction of neuron outputs to zero during training to prevent overfitting. Ch 36.

## E

**Embedding** — A dense numerical vector representation of text (or other data) that captures semantic meaning. Similar meanings produce similar vectors. Ch 1, Ch 14, Ch 42.

**Embedding Model** — A model specifically designed to generate embeddings from text. Examples: text-embedding-3-small, all-MiniLM-L6-v2. Ch 1, Ch 42.

**Encoder** — The part of a transformer that processes input text and produces contextual representations. BERT is an encoder-only model. Ch 38.

**Encoder-Decoder** — A transformer architecture with both an encoder (processes input) and a decoder (generates output). T5 and BART use this architecture. Ch 38, Ch 43.

**Eval / Evaluation** — The process of measuring an AI system's quality on a dataset of test cases. The foundation of reliable AI development. Ch 18-21, Ch 51, Ch 61.

**Eval-Driven Development** — A development methodology where you write evaluations first, then iteratively improve the system until evals pass. Analogous to test-driven development. Ch 21, Ch 61.

**Egress Proxy** — A network proxy that controls which external services an AI agent can reach. Essential for sandboxed agent deployments. Ch 55.

**Epoch** — One complete pass through the entire training dataset during model training. Fine-tuning typically uses 1-5 epochs. Ch 36, Ch 45.

**Escape Velocity (Adoption)** — The point at which a skills marketplace or AI platform generates new content faster than content is deprecated, creating self-sustaining growth. Ch 60.

## F

**Feature** — In ML, an individual measurable property of an input. In text, features might be token counts, embeddings, or TF-IDF scores. Ch 34.

**Few-Shot Prompting** — Including examples of desired input-output pairs in the prompt to guide the LLM's behavior. Ch 4.

**Fine-Tuning** — Continuing the training of a pre-trained model on a domain-specific dataset to specialize its behavior. Ch 44-47.

**Feedback Loop** — A system where outputs are fed back as inputs to improve future outputs. In AI platforms, user ratings improve prompts which improve outputs which earn better ratings. Ch 61.

**Flywheel** — A self-reinforcing cycle where each component accelerates the others. In skills marketplaces: more skills attract more users, more users create more skills. Ch 60, Ch 61.

**Feature Flag** — A configuration mechanism that controls which features are enabled for which users without code deployment. Used extensively in AI systems for canary rollouts, A/B testing, and kill switches. Ch 56.

**Fluid Compute** — Vercel's infrastructure feature that keeps serverless functions warm between invocations, reducing cold starts for AI workloads. Ch 53.

**Foundation Model** — A large pre-trained model (GPT-4, Claude, Llama) that serves as the base for downstream applications via prompting, fine-tuning, or RAG. Ch 0, Ch 43.

**Function Calling** — See "Tool Calling."

## G

**Generative AI** — AI systems that create new content (text, images, audio, code) rather than just analyzing or classifying existing data. Ch 0.

**GGUF** — A file format for quantized language models optimized for CPU inference. Used by llama.cpp and related tools. Ch 47.

**Glass** — Ramp's internal AI tool built on Anthropic's Agent SDK. Features SSO-driven zero-config setup, 30+ pre-connected tools, and a multi-pane workspace. Ch 59, Ch 62.

**GPT (Generative Pre-trained Transformer)** — OpenAI's family of decoder-only transformer models trained for text generation. Ch 0, Ch 38, Ch 43.

**GPTQ** — A post-training quantization method that compresses model weights using approximate second-order information. Ch 47.

**Gradient Descent** — An optimization algorithm that iteratively adjusts model parameters in the direction that minimizes the loss function. Ch 36.

**Grounding** — Connecting AI outputs to verifiable facts, typically by providing source documents (RAG) or tool results. Reduces hallucination. Ch 15, Ch 48.

**Guardrails** — Safety mechanisms that constrain AI system behavior. Includes input validation, output filtering, and tool permission systems. Ch 48.

## H

**Hallucination** — When an LLM generates text that is factually incorrect, fabricated, or not supported by the input context. A fundamental limitation of generative models. Ch 0, Ch 15, Ch 48.

**Headless Mode** — Running an AI agent without a UI. Tasks are triggered by API calls, schedules, or events, and results are delivered asynchronously. Ch 59.

**Hook** — In Claude Code, a script that runs before or after specific tool calls. Used for validation, logging, and custom behavior. Ch 23, Ch 33.

**Hub-and-Spoke** — An organizational model for AI adoption where a small central team (hub) builds the platform and functional teams (spokes) build domain-specific skills and workflows. Ch 62.

**Hugging Face** — The dominant platform for open-source AI models, datasets, and tools. Hosts the model hub and provides the Transformers library. Ch 40-43.

**Human-in-the-Loop** — A design pattern where certain AI actions require human approval before execution. Essential for high-risk operations. Ch 11, Ch 30, Ch 59.

**HyDE (Hypothetical Document Embeddings)** — A retrieval technique where the query is used to generate a hypothetical answer, which is then embedded and used for similarity search. Ch 17.

**Hyperparameter** — A configuration value set before training begins (learning rate, batch size, number of layers). Contrasts with parameters (weights), which are learned during training. Ch 36, Ch 45.

**Hybrid Search** — Combining dense (embedding) and sparse (keyword) retrieval to leverage the strengths of both approaches. Ch 17.

## I

**Idempotent** — An operation that produces the same result whether executed once or multiple times. Important for agent tool calls that may be retried on failure. Ch 8.

**Input Tokens** — Tokens in the prompt sent to the LLM (system prompt + conversation history + user message). Cheaper than output tokens for most providers. Ch 0, Ch 49.

**Inference** — The process of running a trained model on new inputs to generate predictions or outputs. The "using" phase as opposed to the "training" phase. Ch 0, Ch 40, Ch 57.

**Instruction Tuning** — Fine-tuning a model on a dataset of instruction-response pairs to improve its ability to follow instructions. Ch 44.

## J

**Jailbreak** — An attack that tricks an AI model into bypassing its safety restrictions. Distinct from prompt injection, which overrides instructions. Ch 48.

**JSON Mode** — A provider-specific setting that constrains the LLM to output valid JSON. More reliable than asking for JSON in the prompt. Ch 5.

## K

**KAIROS** — Claude Code's three-layer memory system: session memory, project memory (CLAUDE.md), and global memory. Ch 28.

**Key-Value Cache (KV Cache)** — A mechanism that caches intermediate attention computations to avoid redundant work during autoregressive generation. Speeds up inference. Ch 29, Ch 57.

**Knowledge Compounding** — The phenomenon where shared AI skills create superlinear value: more skills attract more users, more users create more skills, and each skill raises the floor for everyone. Ch 60, Ch 62.

**Knowledge Distillation** — See "Distillation."

## L

**Label** — In supervised learning, the correct answer associated with a training example. Used to compute the loss during training. Ch 34, Ch 46.

**Layer** — A single transformation step in a neural network. Each layer contains weights and applies a computation (linear transformation + activation function). Ch 36.

**L0-L3 Proficiency Levels** — Ramp's framework for measuring AI adoption maturity: L0 (Unaware), L1 (Experimenter), L2 (Practitioner), L3 (Builder). Ch 62.

**Latency** — The time between sending a request and receiving the first (or complete) response. Critical for user experience. Ch 0, Ch 53.

**Leaderboard** — A public ranking of AI tool usage across a team or organization. Used to drive adoption through social proof and competitive dynamics. Ch 62.

**LLM (Large Language Model)** — A neural network trained on massive text datasets that can generate, analyze, and transform text. Examples: GPT-4, Claude, Gemini, Llama. Ch 0.

**LLM-as-Judge** — Using one LLM to evaluate the output of another LLM. The foundation of automated evaluation for AI systems. Ch 20, Ch 51.

**LoRA (Low-Rank Adaptation)** — A parameter-efficient fine-tuning technique that adds small trainable matrices to a frozen model. Dramatically reduces the compute needed for fine-tuning. Ch 45.

**Learning Rate** — A hyperparameter that controls how large each weight update step is during training. Too high causes divergence; too low causes slow training. Ch 36, Ch 45.

**Lethal Trifecta** — Simon Willison's term for the three conditions that create genuinely dangerous AI vulnerabilities: (1) access to private data, (2) ability to take actions, and (3) exposure to untrusted input. Any two are manageable; all three together are explosive. Ch 48, Ch 55.

**Llama** — Meta's family of open-source large language models. Llama 3 is one of the most capable open models. Ch 43, Ch 47.

**Loss Function** — A mathematical function that measures how far a model's predictions are from the correct answers. Training minimizes this function. Ch 36.

## M

**Masked Language Modeling (MLM)** — A pre-training objective where random tokens in the input are masked and the model learns to predict them. Used by BERT. Ch 38.

**Map-Reduce (Agent Pattern)** — A multi-agent pattern where multiple agents process items in parallel (map) and a synthesis agent combines results (reduce). Ch 58.

**MCP (Model Context Protocol)** — An open standard for connecting AI tools to external services. Defines how tools, resources, and prompts are exposed. Ch 24, Ch 33.

**Memory (Agent)** — The ability of an agent to retain and recall information across turns (short-term), sessions (working memory), and time (long-term). Ch 10, Ch 28.

**Model Routing** — Directing different types of requests to different models based on complexity, cost, or quality requirements. Ch 49, Ch 54.

**MTEB (Massive Text Embedding Benchmark)** — A benchmark for evaluating embedding models across diverse tasks. The standard for comparing embedding quality. Ch 42.

**Multi-Head Attention** — Running multiple attention mechanisms in parallel, each learning different aspects of token relationships. Ch 38.

**Multi-Modal** — AI models that process multiple types of input (text, images, audio, video). Ch 7.

**Mixture of Experts (MoE)** — A model architecture where only a subset of parameters (experts) are activated for each input, allowing larger total capacity at lower inference cost. Ch 43.

**Multi-Turn** — A conversation with multiple back-and-forth exchanges between user and AI. Distinct from single-turn (one question, one answer). Ch 6, Ch 20.

## N

**Next-Token Prediction** — The fundamental task of autoregressive language models: given a sequence of tokens, predict the most likely next token. This simple objective produces complex capabilities. Ch 0, Ch 39.

**N+1 Query Problem** — A database anti-pattern where an initial query triggers N additional queries. Often identified in AI-generated code. Ch 58.

**Named Entity Recognition (NER)** — A task that identifies and classifies named entities (people, places, organizations) in text. Ch 40.

**Neural Network** — A computational model inspired by biological neural networks, consisting of layers of interconnected nodes (neurons) that learn to transform inputs into outputs. Ch 36.

**Non-Deterministic** — Producing different outputs for the same input. LLMs are non-deterministic by default due to sampling. This is why evals, not unit tests, are needed. Ch 0, Ch 18.

**Normalization** — Scaling input features to a standard range (e.g., 0-1 or mean-0, std-1) to improve training stability and speed. Ch 35.

## O

**Observability** — The ability to understand an AI system's internal state by examining its outputs: logs, traces, metrics, and eval results. Ch 22, Ch 52.

**Ollama** — A tool for running open-source LLMs locally with a simple command-line interface. Ch 43, Ch 47.

**One-Shot Prompting** — Including a single example in the prompt. Between zero-shot (no examples) and few-shot (multiple examples). Ch 4.

**OpenClaw** — A deployment pattern for internal AI agents using Docker-based sandboxing with VPC isolation and egress proxy controls. Ch 55, Ch 59.

**Output Tokens** — Tokens generated by the LLM in its response. Typically more expensive than input tokens. Ch 0, Ch 49.

**OpenTelemetry** — An open standard for distributed tracing and metrics collection. Used for AI system observability. Ch 22, Ch 52.

**Overfitting** — When a model performs well on training data but poorly on unseen data. Indicates the model has memorized rather than learned. Ch 36, Ch 46.

## P

**Parallel Execution (Multi-Agent)** — Running multiple agents simultaneously on independent tasks. Fan-out/fan-in pattern for throughput. Ch 58.

**Perplexity** — A measure of how well a language model predicts a sequence. Lower perplexity = better prediction. Used to evaluate language model quality. Ch 39.

**Persona Prompt** — A system prompt that assigns the AI a specific role or character to improve output quality for specialized tasks. Ch 4.

**PII (Personally Identifiable Information)** — Data that can identify individuals (names, emails, SSNs). AI systems must detect and handle PII carefully. Ch 48.

**PEFT (Parameter-Efficient Fine-Tuning)** — A family of techniques (LoRA, QLoRA, adapters) that fine-tune only a small subset of model parameters. Ch 45.

**Pipeline (Agent Pattern)** — A multi-agent pattern where agents are chained sequentially, with each agent's output becoming the next agent's input. Ch 58.

**Pipeline (Hugging Face)** — A high-level API in the Transformers library that wraps models for common tasks (text generation, classification, summarization, etc.). Ch 40.

**Positional Encoding** — Information added to token embeddings that encodes their position in the sequence, since transformers have no inherent notion of order. Ch 38.

**Pre-Training** — The initial phase of training a language model on a large corpus of text to learn general language understanding. Happens before fine-tuning. Ch 0, Ch 44.

**Prompt** — The text input sent to an LLM, including system prompts, user messages, and conversation history. Ch 4.

**Prompt Caching** — Storing and reusing the model's computation of the system prompt or common prefix to reduce latency and cost on subsequent requests. Ch 49.

**Prompt Engineering** — The practice of crafting effective prompts to get desired behavior from LLMs. Includes techniques like few-shot, chain-of-thought, and persona prompts. Ch 4.

**Prompt Injection** — An attack where malicious text in the input overrides the system prompt's instructions. The primary security vulnerability in LLM applications. Ch 48.

**Prompt Template** — A reusable prompt structure with placeholders for dynamic content. Enables consistent, maintainable prompts across an application. Ch 4.

**Provider** — A company that offers LLM access via API (Anthropic, OpenAI, Google, Mistral, etc.). Different providers have different models, pricing, and capabilities. Ch 2.

## Q

**Query Expansion** — Transforming a user query into multiple variant queries to improve retrieval recall. Techniques include synonym expansion and HyDE. Ch 17.

**QLoRA** — Quantized LoRA: combines 4-bit quantization of the base model with LoRA fine-tuning. Enables fine-tuning large models on consumer hardware. Ch 45.

**Quantization** — Reducing the numerical precision of model weights (e.g., from 16-bit to 4-bit) to decrease model size and increase inference speed. Ch 47.

**Quorum (Agent Pattern)** — A multi-agent error handling strategy that succeeds if a minimum number of agents succeed, even if others fail. Ch 58.

## R

**RAG (Retrieval-Augmented Generation)** — A pattern that grounds LLM responses in retrieved documents. Combines a retrieval system with a generative model. Ch 15-17, Ch 44.

**Ralph Loop** — An eval-driven development cycle named after a case study in Ch 21. Write eval, run system, analyze failures, improve, repeat. Ch 21, Ch 61.

**Rate Limiting** — Controlling the number of requests sent to an API within a time period. Essential for production AI systems. Ch 49, Ch 54.

**ReAct** — A prompting pattern that alternates between Reasoning and Acting. The agent thinks about what to do, does it, observes the result, and repeats. Ch 13.

**Reranking** — A second-stage retrieval step that reorders initial retrieval results using a more sophisticated model. Improves RAG quality. Ch 15, Ch 17.

**RLHF (Reinforcement Learning from Human Feedback)** — A training technique where the model is fine-tuned using human preference judgments. Key to making LLMs helpful and safe. Ch 0.

**Red Teaming** — Deliberately attempting to break an AI system by finding adversarial inputs, bypasses, and failure modes. Ch 48.

**Regression (AI)** — When an AI system's performance on a metric decreases, often after a model update, prompt change, or dependency change. Ch 51, Ch 56.

**Reward Model** — In RLHF, a model trained on human preferences that scores outputs. Used to guide reinforcement learning. Ch 0.

**Risk Tier** — A classification of tool operations by their potential for harm. Read-only tools are low risk; destructive tools (delete, send, deploy) are high risk. Ch 30.

**Rollback** — Reverting an AI system to a previous known-good version when a new version degrades. Ch 56, Ch 61.

**RoPE (Rotary Position Embedding)** — A positional encoding scheme that encodes relative position through rotation of embedding vectors. Used in Llama and many modern models. Ch 38.

## S

**Scaffold** — See "Harness." The system surrounding an LLM that makes it useful: tool calling, memory, context management, permissions. Ch 27.

**Scorer** — A function that evaluates the quality of an AI output, producing a numerical score. Can be deterministic (regex match, exact match) or LLM-based (LLM-as-judge). Ch 19, Ch 20.

**Sampling** — The process of randomly selecting the next token from the LLM's probability distribution, controlled by parameters like temperature and top-p. Ch 0, Ch 39.

**Sandboxing** — Running AI agents in isolated environments (containers, VMs, restricted networks) that limit their ability to affect production systems. Ch 55, Ch 59.

**Self-Attention** — An attention mechanism where all queries, keys, and values come from the same sequence. Each token attends to every other token in the input. Ch 38.

**Self-Reinforcing System** — A system where the output feeds back to improve the system itself. Includes automated eval loops, feedback-driven improvement, and agents improving agents. Ch 61.

**Semantic Search** — Finding documents based on meaning rather than keyword matching. Uses embeddings to compare the semantic similarity of queries and documents. Ch 14.

**Sensei** — Ramp's AI-powered skill recommendation engine that suggests relevant skills based on user role, tools, and current activity. Ch 60.

**Sentence Transformers** — A library for generating sentence-level embeddings using pre-trained transformer models. Ch 42.

**Sigmoid** — An activation function that maps any input to a value between 0 and 1. Commonly used in the output layer for binary classification. Ch 36.

**Skill** — A packaged set of instructions (usually in markdown) that customizes how an AI agent behaves for a specific task. Ch 25, Ch 33, Ch 60.

**Skills Marketplace** — A platform for discovering, sharing, and managing AI skills across an organization. Ch 60.

**Sparse Retrieval** — Finding documents using keyword matching (e.g., TF-IDF, BM25) rather than embedding similarity. Ch 17.

**Speculative Decoding** — An inference optimization where a small draft model generates candidate tokens that a larger model then verifies in parallel. Speeds up generation by 2-3x with no quality loss. Ch 47, Ch 57.

**Streaming** — Sending LLM output to the user token-by-token as it is generated, rather than waiting for the complete response. Reduces perceived latency. Ch 3, Ch 6.

**Structured Output** — Constraining LLM output to a specific format (usually JSON matching a schema) for reliable parsing. Ch 5.

**Stop Condition** — The criterion that determines when an agent loop should terminate. Can be a special token, maximum iterations, LLM decision (end_turn), or quality threshold. Ch 9.

**Supervised Fine-Tuning (SFT)** — Fine-tuning with labeled examples (input-output pairs). The most common form of fine-tuning for LLMs. Ch 45, Ch 46.

**Synthetic Data** — Training data generated by an AI model rather than collected from real-world sources. Used to augment datasets when real data is scarce. Ch 46.

**System Prompt** — A special message at the beginning of a conversation that sets the AI's behavior, personality, and constraints. Ch 4.

## T

**T5 (Text-to-Text Transfer Transformer)** — An encoder-decoder model that frames every NLP task as text-to-text. Input: text, output: text. Ch 38, Ch 43.

**Task Decomposition** — Breaking a complex task into smaller subtasks that can be handled by individual agents or sequential steps. The coordinator pattern relies on this. Ch 58.

**Temperature** — A parameter that controls the randomness of LLM output. Higher temperature = more random/creative. Lower = more focused/deterministic. Ch 0, Ch 39.

**Telemetry** — Collecting and transmitting data about AI system performance, usage, and errors. Includes token counts, latency, tool calls, and cost. Ch 22.

**TF-IDF (Term Frequency-Inverse Document Frequency)** — A classic text scoring method that weights words by how frequent they are in a document vs. the corpus. Ch 17.

**Tick Loop** — In Claude Code, a background process that checks for work to do during idle time. Enables idle-time agency (AutoDream, background maintenance). Ch 27, Ch 32.

**Token** — The atomic unit of text that LLMs process. Typically 0.75 words per token in English. Determines cost, speed, and context window usage. Ch 0, Ch 37.

**Token Budget** — A limit on the total tokens an agent or fleet of agents can consume. Used for cost control in multi-agent systems. Ch 49, Ch 58.

**Tokenizer** — The component that converts text into tokens (encoding) and tokens back into text (decoding). Different models use different tokenizers. Ch 0, Ch 37.

**Tool Calling** — A protocol where the LLM requests that your application execute a function on its behalf. The LLM generates the function name and arguments; your code executes. Ch 8, Ch 24, Ch 30.

**Top-k Sampling** — A decoding strategy that restricts the next token selection to the k most probable tokens. Ch 0, Ch 39.

**Top-p Sampling (Nucleus Sampling)** — A decoding strategy that selects from the smallest set of tokens whose cumulative probability exceeds p. Ch 0, Ch 39.

**Tracing** — Recording the sequence of operations (LLM calls, tool calls, decisions) within an AI system for debugging and analysis. Ch 22, Ch 52.

**Training** — The process of adjusting model parameters to minimize a loss function on a dataset. Includes pre-training and fine-tuning. Ch 0, Ch 36, Ch 44-46.

**Transformer** — The neural network architecture behind all modern LLMs. Based on self-attention mechanisms and positional encoding. Introduced in "Attention Is All You Need" (2017). Ch 38.

**Transfer Learning** — Using knowledge from a model trained on one task to improve performance on a different task. Pre-training + fine-tuning is the dominant form. Ch 44.

**Trust Spectrum** — A model for categorizing AI tool operations from fully trusted (no confirmation) to fully distrusted (requires approval). Ch 11, Ch 30.

## U

**Upsert** — Insert a record if it does not exist, or update it if it does. Common operation in vector databases when refreshing embeddings. Ch 14.

**Underfitting** — When a model is too simple to capture patterns in the data. Performs poorly on both training and test data. Ch 36.

## V

**Vector Database / Vector Store** — A database optimized for storing and querying embedding vectors using similarity search. Examples: Pinecone, Weaviate, Qdrant, pgvector. Ch 14.

**Vector Similarity Search** — Finding the nearest vectors to a query vector in a vector database. The core operation of semantic search and RAG retrieval. Ch 14.

**Vercel AI SDK** — A TypeScript SDK for building AI-powered applications. Provides streaming, tool calling, and multi-provider support. Ch 6, Ch 13.

**vLLM** — A high-throughput inference engine for LLMs that uses PagedAttention for efficient memory management. Ch 57.

## W

**Warm-Up** — Gradually increasing the learning rate at the start of training to improve stability. Commonly used with transformers. Ch 36, Ch 45.

**WebSocket** — A persistent bidirectional communication protocol. Used to stream AI responses to client applications in real-time. Ch 6, Ch 59.

**Window (Sliding)** — A technique for managing context by keeping only the most recent N messages or tokens. Simple but loses early context. Ch 12.

**Weight** — A numerical parameter in a neural network that is learned during training. Weights determine how inputs are transformed into outputs. Ch 36.

**WordPiece** — A tokenization algorithm used by BERT. Similar to BPE but uses a likelihood-based criterion for merging. Ch 37.

## Z

**Zero-Config Setup** — An onboarding pattern where a tool requires no manual configuration. Authentication is handled via SSO, integrations are pre-connected, and defaults are sensible. Ch 59, Ch 62.

**Zero-Shot Prompting** — Asking an LLM to perform a task with no examples. Relies entirely on the model's pre-trained knowledge. Ch 4.

**Zero-Shot Classification** — Classifying text into categories that were not part of the model's fine-tuning. Uses the model's general understanding to classify by description rather than label. Ch 40.

**Zod** — A TypeScript-first schema validation library used extensively for defining tool parameters and structured output schemas in AI applications. Ch 5, Ch 8.

---

*[Back to Guide](../README.md)* | *Next: [Resources](./appendix-resources.md)*

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

**AWQ (Activation-Aware Weight Quantization)** — A quantization method that preserves important weights at higher precision based on activation patterns. Ch 47.

## B

**Backpropagation** — The algorithm for computing gradients of the loss function with respect to each weight in a neural network, enabling gradient descent to update weights. Ch 36.

**Batch Inference** — Processing multiple inputs through a model simultaneously rather than one at a time. Increases throughput and reduces per-item cost. Ch 57.

**Beam Search** — A decoding strategy that maintains multiple candidate sequences (beams) at each generation step and selects the highest-probability complete sequence. Ch 39.

**BERT** — Bidirectional Encoder Representations from Transformers. An encoder-only transformer trained with masked language modeling. Excellent for classification, NER, and embedding generation but not for text generation. Ch 38, Ch 43.

**BPE (Byte Pair Encoding)** — A tokenization algorithm that iteratively merges the most frequent pairs of characters or subwords. Used by GPT models. Ch 0, Ch 37.

**Bias (ML)** — A constant added to a neuron's weighted sum before the activation function. Allows the model to shift the decision boundary. Ch 36.

**Bias (Fairness)** — Systematic errors in AI outputs that reflect prejudices in training data or model design. Ch 48.

## C

**Cache-Break Vector** — A technique used in context window management where specific tokens or patterns are inserted to prevent prompt caching from reusing stale context. Ch 29.

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

**Cursor** — An AI-powered IDE that integrates LLM capabilities directly into the code editor. Ch 27.

## D

**Data Augmentation** — Techniques for expanding training datasets by creating modified versions of existing data points. Ch 46.

**Decoding** — The process of generating text from an LLM by selecting tokens one at a time based on predicted probability distributions. Ch 39.

**Dense Retrieval** — Using embedding similarity (rather than keyword matching) to find relevant documents. Ch 14, Ch 17.

**Distillation** — Training a smaller "student" model to mimic the behavior of a larger "teacher" model. Ch 47.

**Dojo** — Ramp's internal skills marketplace with 350+ Git-backed, versioned, code-reviewed AI skills. Ch 60.

**Dropout** — A regularization technique that randomly sets a fraction of neuron outputs to zero during training to prevent overfitting. Ch 36.

## E

**Embedding** — A dense numerical vector representation of text (or other data) that captures semantic meaning. Similar meanings produce similar vectors. Ch 1, Ch 14, Ch 42.

**Embedding Model** — A model specifically designed to generate embeddings from text. Examples: text-embedding-3-small, all-MiniLM-L6-v2. Ch 1, Ch 42.

**Encoder** — The part of a transformer that processes input text and produces contextual representations. BERT is an encoder-only model. Ch 38.

**Encoder-Decoder** — A transformer architecture with both an encoder (processes input) and a decoder (generates output). T5 and BART use this architecture. Ch 38, Ch 43.

**Eval / Evaluation** — The process of measuring an AI system's quality on a dataset of test cases. The foundation of reliable AI development. Ch 18-21, Ch 51, Ch 61.

**Eval-Driven Development** — A development methodology where you write evaluations first, then iteratively improve the system until evals pass. Analogous to test-driven development. Ch 21, Ch 61.

## F

**Feature** — In ML, an individual measurable property of an input. In text, features might be token counts, embeddings, or TF-IDF scores. Ch 34.

**Few-Shot Prompting** — Including examples of desired input-output pairs in the prompt to guide the LLM's behavior. Ch 4.

**Fine-Tuning** — Continuing the training of a pre-trained model on a domain-specific dataset to specialize its behavior. Ch 44-47.

**Function Calling** — See "Tool Calling."

## G

**Generative AI** — AI systems that create new content (text, images, audio, code) rather than just analyzing or classifying existing data. Ch 0.

**GGUF** — A file format for quantized language models optimized for CPU inference. Used by llama.cpp and related tools. Ch 47.

**Glass** — Ramp's internal AI tool built on Anthropic's Agent SDK. Features SSO-driven zero-config setup, 30+ pre-connected tools, and a multi-pane workspace. Ch 59, Ch 62.

**GPT (Generative Pre-trained Transformer)** — OpenAI's family of decoder-only transformer models trained for text generation. Ch 0, Ch 38, Ch 43.

**GPTQ** — A post-training quantization method that compresses model weights using approximate second-order information. Ch 47.

**Gradient Descent** — An optimization algorithm that iteratively adjusts model parameters in the direction that minimizes the loss function. Ch 36.

**Guardrails** — Safety mechanisms that constrain AI system behavior. Includes input validation, output filtering, and tool permission systems. Ch 48.

## H

**Hallucination** — When an LLM generates text that is factually incorrect, fabricated, or not supported by the input context. A fundamental limitation of generative models. Ch 0, Ch 15, Ch 48.

**Headless Mode** — Running an AI agent without a UI. Tasks are triggered by API calls, schedules, or events, and results are delivered asynchronously. Ch 59.

**Hook** — In Claude Code, a script that runs before or after specific tool calls. Used for validation, logging, and custom behavior. Ch 23, Ch 33.

**Hub-and-Spoke** — An organizational model for AI adoption where a small central team (hub) builds the platform and functional teams (spokes) build domain-specific skills and workflows. Ch 62.

**Hugging Face** — The dominant platform for open-source AI models, datasets, and tools. Hosts the model hub and provides the Transformers library. Ch 40-43.

**Human-in-the-Loop** — A design pattern where certain AI actions require human approval before execution. Essential for high-risk operations. Ch 11, Ch 30, Ch 59.

**HyDE (Hypothetical Document Embeddings)** — A retrieval technique where the query is used to generate a hypothetical answer, which is then embedded and used for similarity search. Ch 17.

**Hybrid Search** — Combining dense (embedding) and sparse (keyword) retrieval to leverage the strengths of both approaches. Ch 17.

## I

**Inference** — The process of running a trained model on new inputs to generate predictions or outputs. The "using" phase as opposed to the "training" phase. Ch 0, Ch 40, Ch 57.

**Instruction Tuning** — Fine-tuning a model on a dataset of instruction-response pairs to improve its ability to follow instructions. Ch 44.

## J

**JSON Mode** — A provider-specific setting that constrains the LLM to output valid JSON. More reliable than asking for JSON in the prompt. Ch 5.

## K

**KAIROS** — Claude Code's three-layer memory system: session memory, project memory (CLAUDE.md), and global memory. Ch 28.

**Knowledge Compounding** — The phenomenon where shared AI skills create superlinear value: more skills attract more users, more users create more skills, and each skill raises the floor for everyone. Ch 60, Ch 62.

## L

**L0-L3 Proficiency Levels** — Ramp's framework for measuring AI adoption maturity: L0 (Unaware), L1 (Experimenter), L2 (Practitioner), L3 (Builder). Ch 62.

**Latency** — The time between sending a request and receiving the first (or complete) response. Critical for user experience. Ch 0, Ch 53.

**Leaderboard** — A public ranking of AI tool usage across a team or organization. Used to drive adoption through social proof and competitive dynamics. Ch 62.

**LLM (Large Language Model)** — A neural network trained on massive text datasets that can generate, analyze, and transform text. Examples: GPT-4, Claude, Gemini, Llama. Ch 0.

**LLM-as-Judge** — Using one LLM to evaluate the output of another LLM. The foundation of automated evaluation for AI systems. Ch 20, Ch 51.

**LoRA (Low-Rank Adaptation)** — A parameter-efficient fine-tuning technique that adds small trainable matrices to a frozen model. Dramatically reduces the compute needed for fine-tuning. Ch 45.

**Loss Function** — A mathematical function that measures how far a model's predictions are from the correct answers. Training minimizes this function. Ch 36.

## M

**Map-Reduce (Agent Pattern)** — A multi-agent pattern where multiple agents process items in parallel (map) and a synthesis agent combines results (reduce). Ch 58.

**MCP (Model Context Protocol)** — An open standard for connecting AI tools to external services. Defines how tools, resources, and prompts are exposed. Ch 24, Ch 33.

**Memory (Agent)** — The ability of an agent to retain and recall information across turns (short-term), sessions (working memory), and time (long-term). Ch 10, Ch 28.

**Model Routing** — Directing different types of requests to different models based on complexity, cost, or quality requirements. Ch 49, Ch 54.

**MTEB (Massive Text Embedding Benchmark)** — A benchmark for evaluating embedding models across diverse tasks. The standard for comparing embedding quality. Ch 42.

**Multi-Head Attention** — Running multiple attention mechanisms in parallel, each learning different aspects of token relationships. Ch 38.

**Multi-Modal** — AI models that process multiple types of input (text, images, audio, video). Ch 7.

**Multi-Turn** — A conversation with multiple back-and-forth exchanges between user and AI. Distinct from single-turn (one question, one answer). Ch 6, Ch 20.

## N

**N+1 Query Problem** — A database anti-pattern where an initial query triggers N additional queries. Often identified in AI-generated code. Ch 58.

**Named Entity Recognition (NER)** — A task that identifies and classifies named entities (people, places, organizations) in text. Ch 40.

**Neural Network** — A computational model inspired by biological neural networks, consisting of layers of interconnected nodes (neurons) that learn to transform inputs into outputs. Ch 36.

**Normalization** — Scaling input features to a standard range (e.g., 0-1 or mean-0, std-1) to improve training stability and speed. Ch 35.

## O

**One-Shot Prompting** — Including a single example in the prompt. Between zero-shot (no examples) and few-shot (multiple examples). Ch 4.

**OpenClaw** — A deployment pattern for internal AI agents using Docker-based sandboxing with VPC isolation and egress proxy controls. Ch 55, Ch 59.

**Output Tokens** — Tokens generated by the LLM in its response. Typically more expensive than input tokens. Ch 0, Ch 49.

**Overfitting** — When a model performs well on training data but poorly on unseen data. Indicates the model has memorized rather than learned. Ch 36, Ch 46.

## P

**PEFT (Parameter-Efficient Fine-Tuning)** — A family of techniques (LoRA, QLoRA, adapters) that fine-tune only a small subset of model parameters. Ch 45.

**Pipeline (Agent Pattern)** — A multi-agent pattern where agents are chained sequentially, with each agent's output becoming the next agent's input. Ch 58.

**Pipeline (Hugging Face)** — A high-level API in the Transformers library that wraps models for common tasks (text generation, classification, summarization, etc.). Ch 40.

**Positional Encoding** — Information added to token embeddings that encodes their position in the sequence, since transformers have no inherent notion of order. Ch 38.

**Pre-Training** — The initial phase of training a language model on a large corpus of text to learn general language understanding. Happens before fine-tuning. Ch 0, Ch 44.

**Prompt** — The text input sent to an LLM, including system prompts, user messages, and conversation history. Ch 4.

**Prompt Caching** — Storing and reusing the model's computation of the system prompt or common prefix to reduce latency and cost on subsequent requests. Ch 49.

**Prompt Engineering** — The practice of crafting effective prompts to get desired behavior from LLMs. Includes techniques like few-shot, chain-of-thought, and persona prompts. Ch 4.

**Prompt Injection** — An attack where malicious text in the input overrides the system prompt's instructions. The primary security vulnerability in LLM applications. Ch 48.

## Q

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

**Rollback** — Reverting an AI system to a previous known-good version when a new version degrades. Ch 56, Ch 61.

## S

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

**Streaming** — Sending LLM output to the user token-by-token as it is generated, rather than waiting for the complete response. Reduces perceived latency. Ch 3, Ch 6.

**Structured Output** — Constraining LLM output to a specific format (usually JSON matching a schema) for reliable parsing. Ch 5.

**System Prompt** — A special message at the beginning of a conversation that sets the AI's behavior, personality, and constraints. Ch 4.

## T

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

## U

**Underfitting** — When a model is too simple to capture patterns in the data. Performs poorly on both training and test data. Ch 36.

## V

**Vector Database / Vector Store** — A database optimized for storing and querying embedding vectors using similarity search. Examples: Pinecone, Weaviate, Qdrant, pgvector. Ch 14.

**Vercel AI SDK** — A TypeScript SDK for building AI-powered applications. Provides streaming, tool calling, and multi-provider support. Ch 6, Ch 13.

## W

**Weight** — A numerical parameter in a neural network that is learned during training. Weights determine how inputs are transformed into outputs. Ch 36.

**WordPiece** — A tokenization algorithm used by BERT. Similar to BPE but uses a likelihood-based criterion for merging. Ch 37.

## Z

**Zero-Config Setup** — An onboarding pattern where a tool requires no manual configuration. Authentication is handled via SSO, integrations are pre-connected, and defaults are sensible. Ch 59, Ch 62.

**Zero-Shot Prompting** — Asking an LLM to perform a task with no examples. Relies entirely on the model's pre-trained knowledge. Ch 4.

**Zero-Shot Classification** — Classifying text into categories that were not part of the model's fine-tuning. Uses the model's general understanding to classify by description rather than label. Ch 40.

---

*[Back to Guide](../README.md)* | *Next: [Resources](./appendix-resources.md)*

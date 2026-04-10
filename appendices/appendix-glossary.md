<!--
  TYPE: appendix
  TITLE: Glossary
  UPDATED: 2026-04-10
-->

# Appendix: Glossary

> 310+ AI/ML/LLM terms for the practicing engineer. Each term includes a brief definition and the chapter(s) where it is covered in depth.

---

## A

**A/B Testing (for AI)** — Running two variants of an AI system simultaneously to measure which performs better. Differs from traditional A/B testing because AI outputs are non-deterministic. Ch 51, Ch 56, Ch 61.

**Accuracy** — The fraction of predictions that exactly match the expected output. Simple but can be misleading for imbalanced datasets. In AI engineering, accuracy is just one metric alongside precision, recall, and task-specific scorers. For LLM evals, fuzzy matching and LLM-as-judge are often more appropriate than exact accuracy. Ch 19, Ch 34.

**Activation Function** — A mathematical function applied to a neuron's output that introduces non-linearity, allowing neural networks to learn complex patterns. Common examples: sigmoid, ReLU, GELU, tanh. Without activation functions, a neural network would reduce to a single linear transformation. Ch 36.

**Adapter (LoRA)** — A small set of trainable parameters added to a frozen pre-trained model. Instead of fine-tuning all weights, only the adapter weights are trained, drastically reducing compute and memory. Ch 45.

**Agent** — An AI system that uses an LLM to decide which actions to take, executes those actions via tools, observes the results, and repeats until a task is complete. Ch 9, Ch 13, Ch 27, Ch 58.

**Agent Loop** — The core cycle of an AI agent: prompt the LLM, receive a tool call or response, execute the tool, append the result, repeat. Ch 9, Ch 27, Ch 58.

**Agent SDK** — See "Claude Agent SDK."

**Alignment** — The challenge of ensuring AI systems behave according to human values and intentions. Includes techniques like RLHF, Constitutional AI, and red teaming. Ch 48.

**Attention Mask** — A binary tensor that tells the transformer which tokens to attend to and which to ignore. Used to handle padding in batched inputs and to enforce causal masking during generation. Ch 37, Ch 38.

**Attention Mechanism** — The core innovation of the transformer architecture. Allows each token to "attend to" (compute relevance scores with) every other token in the sequence. Ch 38.

**AutoDream** — Claude Code's background memory consolidation system. During idle time, the agent reviews recent interactions, identifies patterns, and consolidates them into long-term memory. Ch 28, Ch 61.

**Autoregressive** — A generation strategy where each new token is predicted based on all previously generated tokens. GPT and Claude are autoregressive models: they generate text one token at a time, left to right. Ch 0, Ch 38, Ch 39.

**Agentic Workflow** — A workflow where an AI agent autonomously decides which tools to use, in what order, and when to stop. Contrasts with deterministic pipelines where each step is pre-defined. Ch 9, Ch 13.

**API Gateway** — A server that sits between clients and AI providers, handling rate limiting, caching, failover, and cost tracking. Ch 54.

**Approval Flow** — A mechanism where certain AI actions (e.g., writing files, executing code, sending messages) require explicit human confirmation before execution. Ch 11, Ch 30.

**ANN (Approximate Nearest Neighbor)** — An algorithm that finds vectors "close enough" to a query vector without exhaustively comparing every vector. Essential for making vector search fast at scale. Libraries: FAISS, Annoy, ScaNN. Ch 14.

**AWQ (Activation-Aware Weight Quantization)** — A quantization method that preserves important weights at higher precision based on activation patterns. Ch 47.

## B

**Backpropagation** — The algorithm for computing gradients of the loss function with respect to each weight in a neural network, enabling gradient descent to update weights. Ch 36.

**Batch Inference** — Processing multiple inputs through a model simultaneously rather than one at a time. Increases throughput and reduces per-item cost. Batch APIs from providers like Anthropic and OpenAI offer up to 50% cost savings for non-time-sensitive workloads. Ch 57.

**Beam Search** — A decoding strategy that maintains multiple candidate sequences (beams) at each generation step and selects the highest-probability complete sequence. More thorough than greedy decoding but more expensive. Beam width controls the trade-off. Ch 39.

**BERT** — Bidirectional Encoder Representations from Transformers. An encoder-only transformer trained with masked language modeling. Processes text bidirectionally (left-to-right and right-to-left simultaneously). Excellent for classification, NER, and embedding generation but not for text generation. Ch 38, Ch 43.

**Bi-Encoder** — A model architecture that encodes queries and documents independently into embeddings, enabling pre-computation and fast retrieval via vector similarity. Less accurate than cross-encoders for relevance scoring but orders of magnitude faster. The standard architecture for the first stage of retrieval. Ch 14, Ch 17, Ch 42.

**BPE (Byte Pair Encoding)** — A tokenization algorithm that iteratively merges the most frequent pairs of characters or subwords into new tokens. Used by GPT models and many modern LLMs. The vocabulary is built bottom-up from characters. Ch 0, Ch 37.

**Bias (ML)** — A constant added to a neuron's weighted sum before the activation function. Allows the model to shift the decision boundary. Ch 36.

**Bias (Fairness)** — Systematic errors in AI outputs that reflect prejudices in training data or model design. Ch 48.

**BM25** — A probabilistic ranking function for keyword-based document retrieval. The standard baseline for sparse retrieval systems. Ch 17.

**Brownfield** — Integrating AI into an existing codebase or product, as opposed to greenfield (building from scratch). Most real-world AI engineering is brownfield. Ch 3.

**Batch Size** — The number of training examples processed together in one forward/backward pass. Larger batch sizes give more stable gradient estimates but require more memory. In inference, batch size affects throughput vs. latency trade-offs. Ch 36, Ch 57.

**Benchmarking** — Evaluating a model's performance on standardized tests to compare against other models. Common benchmarks: MMLU (general knowledge), HumanEval (code), MTEB (embeddings), GSM8K (math). Benchmark scores guide model selection but do not replace task-specific evals. Ch 42, Ch 43.

## C

**Cache-Break Vector** — A technique used in context window management where specific tokens or patterns are inserted to prevent prompt caching from reusing stale context. Forces the model to recompute attention for the modified portion of the prompt. Ch 29.

**Causal Masking** — A masking pattern in transformer decoders that prevents each token from attending to future tokens. This ensures the model can only use information from the past when predicting the next token, which is what makes generation autoregressive. Ch 38, Ch 39.

**Canary Deployment** — A deployment strategy where a change is rolled out to a small percentage of traffic first (e.g., 5%), monitored for regressions using eval metrics, then gradually expanded. Essential for prompt and model changes in AI systems because small prompt edits can have outsized effects. Ch 56.

**Chain-of-Thought (CoT)** — A prompting technique that asks the LLM to show its reasoning step by step before giving a final answer. Significantly improves accuracy on complex reasoning, math, and multi-step tasks. Can be triggered by adding "Think step by step" or by providing worked examples. Ch 4.

**Checkpoint** — A saved snapshot of model weights during training. Allows resuming training after interruption and selecting the best-performing version of a model. In fine-tuning, you evaluate each checkpoint and pick the one with the lowest validation loss. Ch 36, Ch 45.

**Chunk Overlap** — The number of tokens shared between adjacent chunks when splitting documents. Prevents information loss at chunk boundaries. Typical overlap: 10-20% of chunk size (e.g., 100 tokens for 500-token chunks). Ch 15.

**Classifier** — A model that assigns inputs to predefined categories. In AI engineering, classifiers are used for intent routing (directing user queries to the right handler), content moderation, and model routing (deciding which LLM to use for a given query). Ch 34, Ch 49.

**Chunking** — Splitting documents into smaller pieces (chunks) for embedding and retrieval in RAG systems. Chunk size and overlap significantly affect retrieval quality. Common strategies: fixed-size, sentence-based, semantic, and recursive character splitting. Ch 15.

**CLAUDE.md** — A markdown file placed in a project root that configures Claude Code's behavior for that project. Contains project context, conventions, and instructions. Ch 23, Ch 28.

**Claude Agent SDK** — Anthropic's official SDK for building AI agents. Provides the agent loop, tool calling, streaming, and multi-turn conversation management. The foundation of Ramp's Glass and other internal AI tools. Ch 59.

**Claude Code** — Anthropic's CLI-based AI coding agent. Uses the agent loop with tools for file operations, search, and shell commands. Ch 23, Ch 27.

**Classification** — A machine learning task that assigns inputs to predefined categories. Ch 34.

**Compaction** — The process of summarizing or condensing conversation history to free up context window space. Used when conversations approach the token limit. Techniques include recursive summarization, selective pruning, and sub-agent delegation. Ch 12, Ch 29, Ch 50.

**Connector** — In platform engineering, a module that integrates an external service (Salesforce, Slack, GitHub) into the AI tool's capabilities. In Ramp's Glass, connectors are pre-configured based on SSO role so users have tools ready without setup. Ch 59.

**Constitutional AI** — Anthropic's approach to AI alignment where the model is trained to follow a set of principles (a "constitution") rather than relying solely on human feedback. The model critiques and revises its own outputs based on these principles during training. Produces more consistent, principled behavior than pure RLHF. Ch 48.

**Context Stuffing** — The practice of inserting as much relevant information as possible into the prompt context to improve the model's responses. In RAG, this means including the top-k retrieved chunks. The trade-off: more context gives the model more information but costs more tokens and may dilute signal with noise. Ch 15, Ch 50.

**Context Window** — The maximum number of tokens an LLM can process in a single request (input + output combined). Determines how much information the model can consider at once. Ranges from 32K (small models) to 1M (Gemini). Ch 0, Ch 12, Ch 29, Ch 50.

**Coordinator Pattern** — A multi-agent pattern where one agent (the coordinator) decomposes tasks, delegates to worker agents, and synthesizes results. Ch 32, Ch 58.

**Cosine Similarity** — A measure of similarity between two vectors, computed as the dot product divided by the product of their magnitudes. Range: -1 to 1. Widely used for comparing embeddings. Ch 1, Ch 14.

**Cross-Attention** — An attention mechanism where the queries come from one sequence and the keys/values come from another. Used in encoder-decoder transformers (e.g., T5) where the decoder attends to encoder outputs. Ch 38.

**Cross-Encoder** — A model that takes two texts as a single input and outputs a relevance score. More accurate than bi-encoders for reranking but much slower since pairs cannot be pre-computed. Used as the second stage in retrieve-then-rerank pipelines. Ch 15, Ch 17.

**Completion** — The text generated by an LLM in response to a prompt. In API terminology, a "chat completion" is the model's response in a conversation. The term originates from the original GPT paradigm of completing a text prefix. Ch 0, Ch 3.

**Content Filtering** — Automated screening of AI inputs and outputs to block harmful, offensive, or policy-violating content. Ch 48.

**Council Pattern** — A multi-agent deliberation pattern where multiple models with different perspectives evaluate the same question, cross-review each other, and a synthesizer produces a verdict. Ch 58.

**Continuous Batching** — A serving optimization where new requests are inserted into a running batch as soon as existing requests complete, rather than waiting for the entire batch to finish. Maximizes GPU utilization. Used by vLLM and TGI. Ch 57.

**Cron Job (AI)** — A scheduled background task that triggers an AI agent on a recurring schedule. Used for digests, monitoring, and automated reporting. Ch 25, Ch 59.

**Cursor** — An AI-powered IDE that integrates LLM capabilities directly into the code editor. Uses a tab-completion model for inline suggestions and a chat panel for larger tasks. Ch 27.

## D

**Data Flywheel** — A virtuous cycle where usage generates data, data improves the model, and the improved model generates more usage. The core dynamic behind AI product moats. Ch 61, Ch 62.

**Data Loader** — A component that reads documents from a source (PDFs, web pages, YouTube transcripts, databases) and converts them into text for chunking and embedding. Handles format parsing, OCR, and content extraction. Ch 15, Ch 16.

**Data Augmentation** — Techniques for expanding training datasets by creating modified versions of existing data points. In NLP, includes paraphrasing, back-translation, synonym replacement, and synthetic generation. Ch 46.

**Decoding** — The process of generating text from an LLM by selecting tokens one at a time based on predicted probability distributions. Strategies include greedy, beam search, top-k, and top-p (nucleus) sampling. Ch 39.

**Dense Retrieval** — Using learned embedding similarity (rather than keyword matching) to find relevant documents. Dense vectors capture semantic meaning, so "canine nutrition" matches "dog food" even with no keyword overlap. Ch 14, Ch 17.

**Diffusion Model** — A generative model that learns to reverse a noise-adding process: starting from pure noise, it iteratively denoises to produce images guided by text prompts. Operates in latent space for efficiency. Stable Diffusion is the most widely used open-source example. Ch 41.

**Distillation** — Training a smaller "student" model to mimic the behavior of a larger "teacher" model. The student learns from the teacher's output probability distributions (soft labels) rather than hard labels, capturing richer information. Used to create faster, cheaper models that retain most of the teacher's capability. Ch 47.

**DreamBooth** — A technique for personalizing diffusion models by fine-tuning on 3-10 images of a specific subject, teaching the model to associate a unique identifier token with that subject. Combined with LoRA for memory efficiency. Enables "a photo of [sks] person in front of the Eiffel Tower" generation. Ch 41.

**Dojo** — Ramp's internal skills marketplace with 350+ Git-backed, versioned, code-reviewed AI skills. Ch 60.

**Decoder** — The part of a transformer that generates output text token by token. Uses causal masking so each position can only attend to previous positions. GPT and Claude are decoder-only architectures. Ch 38, Ch 43.

**Decision Boundary** — In ML, the surface that separates different classes in the feature space. The goal of training is to find the optimal boundary. Ch 34.

**DPO (Direct Preference Optimization)** — An alternative to RLHF that directly optimizes model behavior based on human preferences without needing a separate reward model. Ch 44.

**Dropout** — A regularization technique that randomly sets a fraction of neuron outputs to zero during training to prevent overfitting. During inference, dropout is disabled and all neurons contribute. Ch 36.

**Dot Product** — A mathematical operation that multiplies corresponding elements of two vectors and sums the results. The core operation in attention: the attention score between a query and key is their dot product (scaled by the square root of the key dimension). Also used in cosine similarity computation. Ch 1, Ch 38.

## E

**Embedding** — A dense numerical vector representation of text (or other data) that captures semantic meaning. Similar meanings produce similar vectors. Ch 1, Ch 14, Ch 42.

**Early Stopping** — A regularization technique that halts training when the validation loss stops improving, preventing overfitting. The training checkpoint with the lowest validation loss is saved as the final model. Common patience: 3-5 epochs of no improvement. Ch 36, Ch 45.

**Embedding Dimension** — The size (number of elements) of an embedding vector. Larger dimensions can capture more nuance but cost more to store and search. Common sizes: 384 (MiniLM), 1024 (BGE), 1536 (OpenAI small), 3072 (OpenAI large). Ch 1, Ch 42.

**Embedding Model** — A model specifically designed to generate embeddings from text. Examples: text-embedding-3-small, all-MiniLM-L6-v2. Ch 1, Ch 42.

**Encoder** — The part of a transformer that processes input text bidirectionally, producing contextual representations for each token. BERT is an encoder-only model. Good for classification and embedding but not generation. Ch 38.

**Encoder-Decoder** — A transformer architecture with both an encoder (processes input bidirectionally) and a decoder (generates output autoregressively). The encoder's output is fed to the decoder via cross-attention. T5 and BART use this architecture. Ch 38, Ch 43.

**Eval / Evaluation** — The process of measuring an AI system's quality on a dataset of test cases. The foundation of reliable AI development. Ch 18-21, Ch 51, Ch 61.

**Eval-Driven Development** — A development methodology where you write evaluations first, then iteratively improve the system until evals pass. Analogous to test-driven development but designed for non-deterministic systems. The "Ralph Loop": write eval, run system, analyze failures, improve, repeat. Ch 21, Ch 61.

**Egress Proxy** — A network proxy that controls which external services an AI agent can reach. Essential for sandboxed agent deployments. Prevents agents from exfiltrating data or accessing unauthorized services. Ch 55.

**Emergent Capabilities** — Abilities that appear in large language models at scale that were not explicitly trained for: arithmetic, translation, code generation, reasoning. These capabilities emerge from the simple objective of next-token prediction applied at sufficient scale. Ch 0, Ch 43.

**End Token** — A special token that signals the model to stop generating. Different providers use different names: `end_turn` (Anthropic), `stop` (OpenAI), `<|endoftext|>` (GPT tokenizer). When the model emits this token, the agent loop knows the response is complete. Ch 9, Ch 39.

**Epoch** — One complete pass through the entire training dataset during model training. Fine-tuning typically uses 1-5 epochs. More epochs risk overfitting; fewer risk underfitting. Ch 36, Ch 45.

**Escape Velocity (Adoption)** — The point at which a skills marketplace or AI platform generates new content faster than content is deprecated, creating self-sustaining growth. Ch 60.

**Extended Thinking** — A mode where the model explicitly allocates computation to reasoning before generating a response. Anthropic's Claude supports extended thinking with configurable token budgets. Improves accuracy on complex tasks at the cost of latency and tokens. Ch 4, Ch 39.

## F

**Fan-Out / Fan-In** — A multi-agent execution pattern where a coordinator sends the same or different tasks to multiple workers simultaneously (fan-out), then collects and merges their results (fan-in). Enables parallel processing of independent subtasks. Ch 58.

**Feature** — In ML, an individual measurable property of an input. In text, features might be token counts, embeddings, or TF-IDF scores. Feature selection and engineering determine model quality. Ch 34.

**Few-Shot Prompting** — Including examples of desired input-output pairs in the prompt to guide the LLM's behavior. Typically 3-10 examples. More examples improve consistency but consume context window. Ch 4.

**Fill-Mask** — A Hugging Face pipeline task where the model predicts a masked token in a sentence (e.g., "The capital of France is [MASK]"). Used by encoder models like BERT. Useful for probing model knowledge and data augmentation. Ch 40.

**Fine-Tuning** — Continuing the training of a pre-trained model on a domain-specific dataset to specialize its behavior. Ch 44-47.

**Feedback Loop** — A system where outputs are fed back as inputs to improve future outputs. In AI platforms, user ratings improve prompts which improve outputs which earn better ratings. Ch 61.

**Flywheel** — A self-reinforcing cycle where each component accelerates the others. In skills marketplaces: more skills attract more users, more users create more skills. Ch 60, Ch 61.

**FAISS (Facebook AI Similarity Search)** — An open-source library by Meta for efficient similarity search and clustering of dense vectors. Supports multiple index types (flat, IVF, HNSW) with different speed/accuracy trade-offs. Commonly used as the backend for local vector search. Ch 14.

**Feature Flag** — A configuration mechanism that controls which features are enabled for which users without code deployment. Used extensively in AI systems for canary rollouts, A/B testing, and kill switches. Ch 56.

**Fluid Compute** — Vercel's infrastructure feature that keeps serverless functions warm between invocations, reducing cold starts for AI workloads. Particularly important for AI routes that load models or maintain WebSocket connections. Ch 53.

**FP16 / BF16** — Half-precision floating-point formats (16 bits) used to store model weights. BF16 (bfloat16) has the same exponent range as FP32 but with less mantissa precision, making it more numerically stable for training. Using half-precision roughly halves memory usage compared to FP32. Ch 47, Ch 57.

**Foundation Model** — A large pre-trained model (GPT-4, Claude, Llama) that serves as the base for downstream applications via prompting, fine-tuning, or RAG. Ch 0, Ch 43.

**Function Calling** — See "Tool Calling."

## G

**Gate / Gating Mechanism** — A neural network component that controls information flow by learning which inputs to pass through and which to suppress. Uses sigmoid activation to produce values between 0 (block) and 1 (pass). Found in LSTMs, GRUs, and mixture-of-experts routing. Ch 36.

**Generative AI** — AI systems that create new content (text, images, audio, code) rather than just analyzing or classifying existing data. Contrasts with discriminative AI, which classifies or categorizes existing data. Ch 0.

**GGUF** — A file format for quantized language models optimized for CPU inference. Successor to GGML. Stores model weights, tokenizer, and metadata in a single file. Used by llama.cpp, Ollama, and related tools. Common quantization levels: Q4_K_M (good balance), Q5_K_M (higher quality), Q8_0 (near-lossless). Ch 47.

**GELU (Gaussian Error Linear Unit)** — An activation function used in most modern transformers (GPT, BERT, Llama). Smoother than ReLU, approximated as x * sigmoid(1.702 * x). Provides slight accuracy improvements in transformer training. Ch 36, Ch 38.

**Glass** — Ramp's internal AI tool built on Anthropic's Agent SDK. Features SSO-driven zero-config setup, 30+ pre-connected tools, and a multi-pane workspace. The blueprint for building internal AI platforms. Ch 59, Ch 62.

**Gemini** — Google's family of multimodal LLMs. Gemini 2.5 Pro offers a 1M-token context window, the largest among frontier models. Ch 2, Ch 43.

**GPT (Generative Pre-trained Transformer)** — OpenAI's family of decoder-only transformer models trained for text generation. The "Pre-trained" refers to unsupervised next-token prediction on internet text. Ch 0, Ch 38, Ch 43.

**GPTQ** — A post-training quantization method that compresses model weights using approximate second-order information. Produces models that run efficiently on GPU. Ch 47.

**Gradient** — The vector of partial derivatives of the loss function with respect to each model parameter. Points in the direction of steepest increase; training moves weights in the opposite direction. Ch 36.

**Gradient Descent** — An optimization algorithm that iteratively adjusts model parameters in the direction that minimizes the loss function. Variants: stochastic (SGD, one sample), mini-batch (subset), and full-batch (all data). Ch 36.

**Greedy Decoding** — The simplest decoding strategy: always select the token with the highest probability at each step. Fast and deterministic but tends to produce repetitive, bland text. Equivalent to temperature=0. Ch 39.

**Grounding** — Connecting AI outputs to verifiable facts, typically by providing source documents (RAG) or tool results. Reduces hallucination. Ch 15, Ch 48.

**Guardrails** — Safety mechanisms that constrain AI system behavior. Includes input validation, output filtering, and tool permission systems. Ch 48.

**Guidance Scale** — In diffusion models, a parameter that controls how strongly the generated image follows the text prompt. Higher values produce images more faithful to the prompt but potentially less natural. Also called classifier-free guidance (CFG). Ch 41.

## H

**Hallucination** — When an LLM generates text that is factually incorrect, fabricated, or not supported by the input context. A fundamental limitation of generative models. Mitigated by RAG, grounding, and eval pipelines that detect confabulation. Ch 0, Ch 15, Ch 48.

**Harness** — The complete system surrounding an LLM: tool definitions, memory management, context window management, permission system, and user interface. The thesis of this guide is that the harness matters more than the model. Ch 27.

**Headless Mode** — Running an AI agent without a UI. Tasks are triggered by API calls, schedules, or events, and results are delivered asynchronously. Enables background agents, cron-triggered workflows, and CI/CD integration. Ch 59.

**Hidden State** — The internal vector representation of a token at a given layer in a neural network. In a transformer, each layer transforms hidden states through attention and feed-forward operations. The final hidden state is used for predictions. The dimensionality of hidden states (e.g., 768 for BERT-base, 4096 for GPT-3) is a key model architecture parameter. Ch 36, Ch 38.

**Hook** — In Claude Code, a script that runs before or after specific tool calls (PreToolUse, PostToolUse, Stop). Used for validation, logging, auto-formatting, and custom behavior. Hooks can allow, block, or modify tool calls. Ch 23, Ch 33.

**Hub-and-Spoke** — An organizational model for AI adoption where a small central team (hub) builds the platform and functional teams (spokes) build domain-specific skills and workflows. Ch 62.

**Hugging Face** — The dominant platform for open-source AI models, datasets, and tools. Hosts the model hub (500K+ models), provides the Transformers library, and offers inference APIs. The "GitHub of machine learning." Ch 40-43.

**Hugging Face Hub** — The model and dataset repository hosted by Hugging Face. Models can be downloaded, used locally, or accessed via Inference API. Every model card includes documentation, benchmarks, and usage examples. Ch 40, Ch 43.

**Human-in-the-Loop** — A design pattern where certain AI actions require human approval before execution. Essential for high-risk operations. Ch 11, Ch 30, Ch 59.

**HyDE (Hypothetical Document Embeddings)** — A retrieval technique where the LLM first generates a hypothetical answer to the query, then that answer is embedded and used for similarity search instead of the original query. Bridges the gap between how users ask questions and how documents are written. Improves retrieval quality for complex queries at the cost of an extra LLM call. Ch 17.

**Hyperparameter** — A configuration value set before training begins (learning rate, batch size, number of layers, dropout rate). Contrasts with parameters (weights), which are learned during training. Hyperparameter tuning is the process of finding optimal values. Ch 36, Ch 45.

**Hybrid Search** — Combining dense (embedding) and sparse (keyword) retrieval to leverage the strengths of both approaches. Typically uses reciprocal rank fusion (RRF) or learned score combination to merge results. Generally outperforms either approach alone. Ch 17.

**HNSW (Hierarchical Navigable Small World)** — The most popular algorithm for approximate nearest neighbor search in vector databases. Builds a multi-layered graph structure that enables fast similarity search with tunable accuracy. Used by Pinecone, Qdrant, pgvector, and most major vector stores. Ch 14.

## I

**Idempotent** — An operation that produces the same result whether executed once or multiple times. Important for agent tool calls that may be retried on failure. Design tools to be idempotent when possible: use upserts instead of inserts, check state before acting. Ch 8.

**Image-to-Image** — A diffusion model pipeline that takes an existing image plus a text prompt and generates a modified version of the image. The strength parameter controls how much the output deviates from the input. Used for style transfer, inpainting, and image editing. Ch 41.

**Inpainting** — A diffusion model technique that fills in a masked region of an image based on a text prompt while keeping the rest of the image unchanged. Useful for editing specific parts of images. Ch 41.

**Input Tokens** — Tokens in the prompt sent to the LLM (system prompt + conversation history + user message). Cheaper than output tokens for most providers (typically 3-5x cheaper). Ch 0, Ch 49.

**Inference** — The process of running a trained model on new inputs to generate predictions or outputs. The "using" phase as opposed to the "training" phase. Every API call to Claude or GPT is an inference request. Ch 0, Ch 40, Ch 57.

**Instruction Tuning** — Fine-tuning a model on a dataset of instruction-response pairs to improve its ability to follow instructions. Ch 44.

**INT4 / INT8** — Integer quantization formats that represent model weights using 4-bit or 8-bit integers instead of floating-point numbers. INT4 models are roughly 4x smaller than FP16 but may lose some accuracy. The sweet spot for most deployment scenarios. Ch 47.

## J

**Jailbreak** — An attack that tricks an AI model into bypassing its safety restrictions. Distinct from prompt injection, which overrides task instructions. Jailbreak attacks target safety training; prompt injection attacks target application-level system prompts. Ch 48.

**JSON Mode** — A provider-specific setting that constrains the LLM to output valid JSON. More reliable than asking for JSON in the prompt. Does not guarantee the JSON matches a specific schema; for that, use structured output with a schema definition. Ch 5.

**JSON Schema** — A vocabulary for annotating and validating JSON documents. Used to define the expected structure of tool parameters and structured outputs. In AI engineering, Zod schemas are often compiled to JSON Schema for use with provider APIs. Ch 5, Ch 8.

## K

**Kill Switch** — An emergency mechanism to immediately disable an AI feature or agent without a full deployment. Typically implemented via feature flags. Essential for production AI systems where a bad model or prompt can cause widespread issues. Ch 56, Ch 62.

**KNN (K-Nearest Neighbors)** — A simple algorithm that classifies a new data point based on the majority class of its K nearest neighbors in the feature space. In vector search, exact KNN is precise but slow; approximate nearest neighbor (ANN) algorithms trade slight accuracy for massive speed gains. Ch 14.

**KAIROS** — Claude Code's three-layer memory system: session memory (current conversation), project memory (CLAUDE.md), and global memory (cross-session preferences). Enables persistent learning across interactions. Ch 28.

**Key-Value Cache (KV Cache)** — A mechanism that caches the key and value matrices from previous attention computations to avoid redundant work during autoregressive generation. Without KV cache, each new token would require recomputing attention over the entire sequence. Speeds up inference significantly but consumes GPU memory proportional to sequence length. Ch 29, Ch 57.

**Knowledge Compounding** — The phenomenon where shared AI skills create superlinear value: more skills attract more users, more users create more skills, and each skill raises the floor for everyone. Ch 60, Ch 62.

**Knowledge Cutoff** — The date after which a model has no training data. Any events, APIs, or facts that changed after this date may be incorrect in the model's responses. RAG is the primary solution for providing post-cutoff knowledge. Ch 0, Ch 15.

**Knowledge Distillation** — See "Distillation."

## L

**Label** — In supervised learning, the correct answer associated with a training example. Used to compute the loss during training. Ch 34, Ch 46.

**Latent Space** — The compressed, abstract representation space learned by a model. In diffusion models, images are encoded into a lower-dimensional latent space where the denoising process operates, then decoded back to pixel space. In LLMs, the hidden states form a latent space that captures semantic meaning. Ch 41, Ch 42.

**Layer** — A single transformation step in a neural network. Each layer contains weights and applies a computation (linear transformation + activation function). Transformer models stack many layers: BERT-base has 12, GPT-3 has 96, Llama 3 70B has 80. Ch 36.

**L0-L3 Proficiency Levels** — Ramp's framework for measuring AI adoption maturity: L0 (Unaware), L1 (Experimenter), L2 (Practitioner), L3 (Builder). Ch 62.

**LangChain** — A popular framework for building LLM applications. Provides abstractions for chains (sequential LLM calls), agents (tool-using LLMs), memory, and retrieval. Available in TypeScript and Python. Useful for complex applications but adds abstraction overhead for simple use cases. Ch 13.

**Latency** — The time between sending a request and receiving the first (or complete) response. Critical for user experience. Two key metrics: time-to-first-token (TTFT) and total latency. Streaming helps perceived latency even when total latency is high. Ch 0, Ch 53.

**Leaderboard** — A public ranking of AI tool usage across a team or organization. Used to drive adoption through social proof and competitive dynamics. Ch 62.

**LLM (Large Language Model)** — A neural network trained on massive text datasets that can generate, analyze, and transform text. Examples: GPT-4, Claude, Gemini, Llama. Ch 0.

**LLM-as-Judge** — Using one LLM to evaluate the output of another LLM. The foundation of automated evaluation for AI systems. Works best with structured rubrics, chain-of-thought scoring, and calibrated score ranges. Correlation with human judgment varies: strong for factual accuracy, weaker for creativity and style. Ch 20, Ch 51.

**Logits** — The raw, unnormalized scores output by the final layer of a neural network before applying softmax. In language models, the logit vector has one value per token in the vocabulary. Higher logits correspond to higher probability of being selected. Temperature and sampling parameters operate on logits. Ch 39.

**LoRA (Low-Rank Adaptation)** — A parameter-efficient fine-tuning technique that adds small low-rank decomposition matrices (A and B) to a frozen model's attention layers. Instead of updating a full d x d weight matrix, LoRA updates a d x r and r x d pair where r is much smaller (typically 4-64). Dramatically reduces trainable parameters from billions to millions. Ch 45.

**Learning Rate** — A hyperparameter that controls how large each weight update step is during training. Too high causes divergence; too low causes slow training. For LoRA fine-tuning, typical values are 1e-4 to 2e-4. Often used with a scheduler that adjusts the rate during training. Ch 36, Ch 45.

**Learning Rate Scheduler** — A strategy for adjusting the learning rate during training. Common patterns: linear warmup then decay, cosine annealing, and step-based reduction. Proper scheduling can significantly improve final model quality. Ch 36, Ch 45.

**Lethal Trifecta** — Simon Willison's term for the three conditions that create genuinely dangerous AI vulnerabilities: (1) access to private data, (2) ability to take actions, and (3) exposure to untrusted input. Any two are manageable; all three together are explosive. Ch 48, Ch 55.

**Llama** — Meta's family of open-source large language models. Llama 3.3 is one of the most capable open models. Available in 8B and 70B parameter sizes. Ch 43, Ch 47.

**Loss Function** — A mathematical function that measures how far a model's predictions are from the correct answers. Training minimizes this function. Common examples: cross-entropy loss (classification), mean squared error (regression). Ch 36.

## M

**Mastra** — A TypeScript-first AI framework for building agents, workflows, and RAG pipelines. Provides integrations with vector databases, LLM providers, and external APIs. An alternative to LangChain with a more TypeScript-native API. Ch 13.

**Masked Language Modeling (MLM)** — A pre-training objective where random tokens in the input are masked and the model learns to predict them from surrounding context. Used by BERT and other encoder models. This bidirectional training is what makes BERT powerful for understanding tasks but unable to generate text. Ch 38.

**Map-Reduce (Agent Pattern)** — A multi-agent pattern where multiple agents process items in parallel (map) and a synthesis agent combines results (reduce). Ch 58.

**MCP (Model Context Protocol)** — An open standard for connecting AI tools to external services. Defines how tools, resources, and prompts are exposed to AI agents through a client-server architecture. MCP servers wrap APIs, databases, and services; MCP clients (like Claude Code) discover and invoke them. Ch 24, Ch 33.

**Memory (Agent)** — The ability of an agent to retain and recall information across turns (short-term), sessions (working memory), and time (long-term). Ch 10, Ch 28.

**Model Routing** — Directing different types of requests to different models based on complexity, cost, or quality requirements. A simple classifier or heuristic sends easy tasks to cheap models (Haiku) and hard tasks to expensive models (Opus). Can reduce costs 40-60% with minimal quality impact. Ch 49, Ch 54.

**MTEB (Massive Text Embedding Benchmark)** — A comprehensive benchmark for evaluating embedding models across 8 task categories (classification, clustering, retrieval, reranking, etc.) in multiple languages. The standard leaderboard for comparing embedding model quality. Check MTEB scores when selecting an embedding model. Ch 42.

**Multi-Head Attention** — Running multiple attention mechanisms (heads) in parallel, each with its own learned projection matrices, allowing the model to attend to information from different representation subspaces at different positions. Results are concatenated and linearly projected. Typical configurations: 12 heads (BERT-base), 32 heads (GPT-3), 128 heads (Llama 3 70B). Ch 38.

**Mean Pooling** — An operation that averages all token embeddings in a sequence to produce a single fixed-size vector representing the entire input. The standard approach for creating sentence embeddings from transformer output. Ch 42.

**Multi-Modal** — AI models that process multiple types of input (text, images, audio, video). GPT-4o and Gemini natively support vision; Claude supports image input. Ch 7.

**Mixture of Experts (MoE)** — A model architecture where only a subset of parameters (experts) are activated for each input via a routing mechanism, allowing larger total capacity at lower inference cost. Mixtral, for example, has 45B total parameters but only activates 12B per token. Ch 43.

**Multi-Turn** — A conversation with multiple back-and-forth exchanges between user and AI. Distinct from single-turn (one question, one answer). Multi-turn evaluation is harder because quality depends on conversation history. Ch 6, Ch 20.

## N

**Next-Token Prediction** — The fundamental training objective of autoregressive language models: given a sequence of tokens, predict the probability distribution over the next token. This deceptively simple objective, scaled up with massive data and compute, produces the emergent capabilities of modern LLMs. Ch 0, Ch 39.

**N+1 Query Problem** — A database anti-pattern where an initial query triggers N additional queries (one for each result). Often identified by AI code review tools in generated code. Ch 58.

**Named Entity Recognition (NER)** — A task that identifies and classifies named entities (people, places, organizations, dates, monetary values) in text. Typically uses encoder models like BERT with token classification heads. Available as a Hugging Face pipeline task. Ch 40.

**Negative Prompt** — In diffusion models, text describing what you do not want in the generated image (e.g., "blurry, low quality, distorted"). The model steers generation away from the concepts described. Ch 41.

**Nucleus Sampling** — See "Top-p Sampling."

**Neural Network** — A computational model inspired by biological neural networks, consisting of layers of interconnected nodes (neurons) that learn to transform inputs into outputs. Modern LLMs are neural networks with billions of parameters arranged in transformer layers. Ch 36.

**NLI (Natural Language Inference)** — A task where the model determines the relationship between two sentences: entailment, contradiction, or neutral. NLI models are the foundation of zero-shot classification: "This movie is great" entails "This review is positive." Ch 40.

**Non-Deterministic** — Producing different outputs for the same input. LLMs are non-deterministic by default due to sampling. This is why evals (statistical measurement), not unit tests (exact comparison), are needed. Ch 0, Ch 18.

**Normalization** — Scaling input features to a standard range (e.g., 0-1 or mean-0, std-1) to improve training stability and speed. In transformers, layer normalization is applied within each layer. Ch 35.

## O

**One-Hot Encoding** — A representation where a token is encoded as a vector of all zeros except for a single 1 at the token's index position. The starting point before embeddings: a 50,000-word vocabulary produces 50,000-dimensional sparse vectors. Embeddings compress this to dense, meaningful vectors. Ch 36, Ch 37.

**Observability** — The ability to understand an AI system's internal state by examining its outputs: logs, traces, metrics, and eval results. For AI systems, includes token usage, latency per tool call, cost per request, and output quality scores. Ch 22, Ch 52.

**Ollama** — A tool for running open-source LLMs locally with a simple command-line interface and REST API. Downloads, configures, and serves GGUF-quantized models. `ollama run llama3.3` downloads and runs Llama 3.3 locally. Ch 43, Ch 47.

**One-Shot Prompting** — Including a single example in the prompt to demonstrate the expected behavior. Between zero-shot (no examples) and few-shot (multiple examples). Ch 4.

**OpenClaw** — A deployment pattern for internal AI agents using Docker-based sandboxing with VPC isolation and egress proxy controls. Ch 55, Ch 59.

**Output Tokens** — Tokens generated by the LLM in its response. Typically 3-5x more expensive than input tokens because each output token requires a full forward pass through the model. The `max_tokens` parameter caps output length. Ch 0, Ch 49.

**OpenTelemetry** — An open standard for distributed tracing and metrics collection. Provides vendor-neutral APIs, SDKs, and tooling for collecting telemetry data. In AI systems, used to trace the full lifecycle of a request through LLM calls, tool executions, and retrieval steps. Ch 22, Ch 52.

**Optimizer** — The algorithm that updates model weights based on computed gradients. AdamW is the standard optimizer for transformer training. The optimizer maintains state (momentum, variance estimates) that affects convergence speed and stability. Ch 36, Ch 45.

**Overfitting** — When a model performs well on training data but poorly on unseen data. Indicates the model has memorized rather than learned. Mitigated by regularization (dropout, weight decay), more data, or early stopping. Ch 36, Ch 46.

## P

**PagedAttention** — A memory management technique used by vLLM that pages KV cache memory like an operating system pages virtual memory. Allows efficient batching of requests with different sequence lengths and reduces GPU memory waste from fragmentation. Ch 57.

**Padding** — Adding special tokens to shorter sequences in a batch so all sequences have the same length, enabling parallel processing. Attention masks indicate which tokens are real and which are padding so the model ignores padding tokens. Ch 37.

**Parallel Execution (Multi-Agent)** — Running multiple agents simultaneously on independent tasks. Fan-out/fan-in pattern for throughput. Ch 58.

**Parameter Count** — The total number of learnable weights in a model. GPT-3 has 175B parameters; Llama 3.3 70B has 70B. Larger models generally perform better but require more memory, compute, and cost to serve. The "B" in "70B" stands for billion. Ch 43, Ch 47.

**Persona Prompt** — A system prompt that assigns the AI a specific role or character to improve output quality for specialized tasks. Ch 4.

**pgvector** — A PostgreSQL extension that adds vector similarity search capabilities to an existing Postgres database. Supports exact and approximate (HNSW, IVFFlat) nearest neighbor search. The pragmatic choice when you already use PostgreSQL and want to avoid adding a separate vector database. Ch 14.

**Pinecone** — A fully managed vector database designed for production AI applications. Offers serverless and pod-based deployment options, hybrid search, metadata filtering, and namespace isolation. Zero infrastructure management. Ch 14.

**PII (Personally Identifiable Information)** — Data that can identify individuals (names, emails, SSNs). AI systems must detect and handle PII carefully. Automated PII detection using NER models is a common guardrail. Ch 48.

**PEFT (Parameter-Efficient Fine-Tuning)** — A family of techniques (LoRA, QLoRA, adapters, prefix tuning) that fine-tune only a small subset of model parameters. Typically trains less than 1% of the total weights while achieving 90%+ of full fine-tuning quality. Ch 45.

**Perplexity** — A measure of how well a language model predicts a sequence of tokens. Mathematically, it is the exponentiated average negative log-likelihood per token. Lower perplexity means the model is more confident and accurate in its predictions. Used to compare language models and detect distribution shifts. Ch 39.

**Pipeline (Agent Pattern)** — A multi-agent pattern where agents are chained sequentially, with each agent's output becoming the next agent's input. Ch 58.

**Pipeline (Hugging Face)** — A high-level API in the Transformers library that wraps models for common tasks (text generation, classification, summarization, NER, fill-mask, etc.) behind a simple `pipeline("task")` call. Handles tokenization, model loading, and post-processing automatically. Ch 40.

**Positional Encoding** — Information added to token embeddings that encodes their position in the sequence, since the self-attention operation is permutation-invariant and has no inherent notion of order. Original transformers used sinusoidal functions; modern models use learned embeddings or RoPE. Ch 38.

**Pre-Training** — The initial phase of training a language model on a large corpus of text (often trillions of tokens from the internet, books, and code) to learn general language understanding. Costs millions of dollars in compute. Happens before fine-tuning. Most engineers use pre-trained models rather than training from scratch. Ch 0, Ch 44.

**Plan-and-Execute** — An agent pattern where the LLM first creates a plan (list of steps), then executes each step sequentially, checking progress and replanning as needed. More structured than ReAct; useful for complex, multi-step tasks. Ch 13.

**Prompt** — The text input sent to an LLM, including system prompts, user messages, and conversation history. The complete prompt determines cost, latency, and output quality. Ch 4.

**Prompt Versioning** — Tracking changes to prompts over time using version control (Git) or a prompt management system. Essential for reproducing results, debugging regressions, and rolling back bad changes. Treat prompts as code. Ch 4, Ch 56.

**Prompt Caching** — Storing and reusing the model's computation of the system prompt or common prefix to reduce latency and cost on subsequent requests. Anthropic's prompt caching can reduce costs by up to 90% and latency by up to 85% for repeated prefixes. Ch 49.

**Prompt Engineering** — The practice of crafting effective prompts to get desired behavior from LLMs. Includes techniques like few-shot, chain-of-thought, persona prompts, and structured templates. Ch 4.

**Prompt Injection** — An attack where malicious text in the input overrides the system prompt's instructions. The primary security vulnerability in LLM applications. Distinct from jailbreaking, which targets safety training. Ch 48.

**Prompt Template** — A reusable prompt structure with placeholders for dynamic content. Enables consistent, maintainable prompts across an application. Version-control your templates and track which versions produced which results. Ch 4.

**Precision** — In retrieval, the fraction of retrieved documents that are actually relevant. High precision means few irrelevant documents were returned. In classification, the fraction of positive predictions that are correct. Ch 17, Ch 19.

**Provider** — A company that offers LLM access via API (Anthropic, OpenAI, Google, Mistral, etc.). Different providers have different models, pricing, and capabilities. Ch 2.

**Prefix Caching** — See "Prompt Caching."

## Q

**Query** — In the context of attention, a vector derived from the current token that is compared against key vectors to determine attention weights. In the context of RAG, the user's input that is embedded and used to search the vector store. Ch 14, Ch 38.

**Query Expansion** — Transforming a user query into multiple variant queries to improve retrieval recall. Techniques include synonym expansion, HyDE (hypothetical document generation), and LLM-powered query rewriting. Each variant is searched independently and results are merged. Ch 17.

**QLoRA** — Quantized LoRA: combines 4-bit NormalFloat quantization of the base model with LoRA fine-tuning adapters. Enables fine-tuning a 65B-parameter model on a single 48GB GPU. Ch 45.

**Quantization** — Reducing the numerical precision of model weights (e.g., from FP16 to INT4) to decrease model size and increase inference speed. A 4-bit quantized model uses roughly 4x less memory than its FP16 version. Common formats: GGUF (CPU), GPTQ (GPU), AWQ (GPU). Ch 47.

**Quorum (Agent Pattern)** — A multi-agent error handling strategy that succeeds if a minimum number of agents succeed, even if others fail. Provides fault tolerance in parallel agent execution. Ch 58.

## R

**R-squared** — A statistical measure of how well predictions match actual values (0 to 1). In eval contexts, used to measure how well an LLM-as-judge's scores correlate with human ratings. Ch 20.

**Rank** — In LoRA, the dimension of the low-rank decomposition matrices. Lower rank (r=4-8) uses fewer parameters but may underfit; higher rank (r=32-64) captures more but approaches full fine-tuning cost. A key hyperparameter when configuring LoRA. Ch 45.

**RAG (Retrieval-Augmented Generation)** — A pattern that grounds LLM responses in retrieved documents. Combines a retrieval system (embeddings + vector store) with a generative model. The retrieved context is injected into the prompt so the model can answer based on real data rather than training-time knowledge. Ch 15-17, Ch 44.

**Ralph Loop** — An eval-driven development cycle named after a case study in Ch 21. Write eval, run system, analyze failures, improve, repeat. Ch 21, Ch 61.

**Rate Limiting** — Controlling the number of requests sent to an API within a time period. Essential for production AI systems to stay within provider quotas and manage costs. Implement with exponential backoff and retry logic. Ch 49, Ch 54.

**Recall** — In retrieval, the fraction of all relevant documents that were actually retrieved. High recall means few relevant documents were missed. In RAG, recall is often more important than precision because the LLM can filter out irrelevant chunks but cannot use chunks it never received. Ch 17, Ch 19.

**ReAct** — A prompting pattern that alternates between Reasoning and Acting. The agent thinks about what to do (Thought), does it (Action), observes the result (Observation), and repeats. The foundational pattern behind modern agent architectures including Claude Code. Ch 13.

**Reranking** — A second-stage retrieval step that reorders initial retrieval results using a more sophisticated cross-encoder model. Typically applied to the top 20-50 candidates from the initial retrieval to select the best 3-5. Improves RAG quality significantly. Ch 15, Ch 17.

**ReLU (Rectified Linear Unit)** — An activation function that outputs the input directly if positive, or zero if negative: f(x) = max(0, x). The most common activation function in modern neural networks due to its simplicity and effectiveness. Variants include Leaky ReLU and GELU. Ch 36.

**RLHF (Reinforcement Learning from Human Feedback)** — A training technique where the model is fine-tuned using human preference judgments to align behavior with human expectations. The three-step process: collect human preference data, train a reward model, optimize the LLM with reinforcement learning. Key to making LLMs helpful and safe. Ch 0.

**Reciprocal Rank Fusion (RRF)** — A method for combining results from multiple retrieval systems (e.g., dense + sparse) into a single ranked list. Each document's score is the sum of 1/(k + rank) across all systems, where k is a constant (typically 60). Simple, effective, and requires no training. Ch 17.

**Red Teaming** — Deliberately attempting to break an AI system by finding adversarial inputs, bypasses, and failure modes. Essential before production launch. Ch 48.

**Regression (AI)** — When an AI system's performance on a metric decreases, often after a model update, prompt change, or dependency change. Detected by continuous eval pipelines running in CI/CD. Ch 51, Ch 56.

**Reward Model** — In RLHF, a model trained on human preferences that scores outputs. Takes a prompt and response as input, outputs a scalar score. Used to guide reinforcement learning optimization of the base model. Ch 0.

**Risk Tier** — A classification of tool operations by their potential for harm. Read-only tools are low risk; destructive tools (delete, send, deploy) are high risk. Determines whether a tool call needs human approval, logging, or sandboxing. Ch 30.

**Rollback** — Reverting an AI system to a previous known-good version (prompt, model, or system configuration) when a new version degrades performance. Prompt rollbacks should be as easy as code rollbacks. Version-control everything. Ch 56, Ch 61.

**RoPE (Rotary Position Embedding)** — A positional encoding scheme that encodes relative position through rotation of embedding vectors in complex number space. Enables length extrapolation and efficient relative position awareness. Used in Llama and many modern models. Ch 38.

**Routing (Model)** — See "Model Routing."

**Run (Eval)** — A single execution of an evaluation suite against a system configuration. Each run produces scores that can be compared to previous runs to detect regressions. Store run metadata (prompt version, model, timestamp) for reproducibility. Ch 18, Ch 51.

## S

**Scaffold** — See "Harness." The system surrounding an LLM that makes it useful: tool calling, memory, context management, permissions. The thesis of this guide is that the scaffold matters more than the model. Ch 27.

**Scorer** — A function that evaluates the quality of an AI output, producing a numerical score. Can be deterministic (regex match, exact match) or LLM-based (LLM-as-judge). Ch 19, Ch 20.

**Sampling** — The process of randomly selecting the next token from the LLM's probability distribution, controlled by parameters like temperature, top-k, and top-p. The source of non-determinism in LLM outputs. Ch 0, Ch 39.

**Scaling Laws** — Empirically discovered power-law relationships between model size, dataset size, compute budget, and model performance. Larger models trained on more data with more compute perform predictably better. These laws guide decisions about model training and selection. Ch 43.

**Sandboxing** — Running AI agents in isolated environments (containers, VMs, restricted networks) that limit their ability to affect production systems. Isolation prevents agents from accessing production databases, sending unauthorized network requests, or modifying critical files. The OpenClaw pattern uses Docker + VPC + egress proxy. Ch 55, Ch 59.

**Self-Attention** — An attention mechanism where all queries, keys, and values come from the same sequence. Each token computes attention scores with every other token, producing context-aware representations. The core operation of transformer models. Ch 38.

**Self-Reinforcing System** — A system where the output feeds back to improve the system itself. Includes automated eval loops, feedback-driven improvement, and agents improving agents. Ch 61.

**Semantic Search** — Finding documents based on meaning rather than keyword matching. Uses embeddings to compare the semantic similarity of queries and documents. "Dog food" matches "pet nutrition" even though no keywords overlap. Ch 14.

**Seed** — A fixed random number used to make LLM outputs reproducible. When a seed is set, the same prompt with the same seed should produce the same output (though this is not guaranteed across all providers). Useful for debugging and eval reproducibility. Ch 39.

**Self-Instruct** — A technique where an LLM generates instruction-response pairs from a small seed set, which are then used to fine-tune the model or another model. A key method for creating synthetic training data without expensive human annotation. Ch 46.

**Sensei** — Ramp's AI-powered skill recommendation engine that suggests relevant skills based on user role, tools, and current activity. Ch 60.

**Sentence Transformer** — A model architecture (and Python library) for generating fixed-size sentence-level embeddings by mean-pooling the output of a pre-trained transformer. Enables efficient semantic similarity computation. Models like all-MiniLM-L6-v2 produce 384-dimensional embeddings. Ch 42.

**SentencePiece** — A language-independent tokenization library by Google that treats the input as a raw stream of Unicode characters, with no need for pre-tokenization or whitespace splitting. Supports both BPE and unigram models. Used by Llama, T5, and many multilingual models. Ch 37.

**Sigmoid** — An activation function that maps any input to a value between 0 and 1, following the formula 1/(1+e^-x). Commonly used in the output layer for binary classification and as a gating mechanism inside neural networks. Ch 36.

**Softmax** — A function that converts a vector of raw scores (logits) into a probability distribution where all values are positive and sum to 1. Used in the final layer of classifiers and in attention mechanisms to compute attention weights. Ch 36, Ch 38.

**Skill** — A packaged set of instructions (usually in markdown) that customizes how an AI agent behaves for a specific task. Ch 25, Ch 33, Ch 60.

**Skills Marketplace** — A platform for discovering, sharing, and managing AI skills across an organization. Skills are Git-backed, versioned, code-reviewed, and searchable. Ramp's Dojo marketplace has 350+ skills. Ch 60.

**Skillify** — Ramp's internal tool that converts existing internal documentation (runbooks, playbooks, wikis) into AI skills. Reduces the barrier to contributing new skills from "write markdown from scratch" to "point at existing docs." Ch 33, Ch 60.

**Sparse Retrieval** — Finding documents using keyword matching (e.g., TF-IDF, BM25) rather than embedding similarity. Better for exact keyword matches, acronyms, and product names. Worse for paraphrased queries. Best results often come from hybrid search combining sparse and dense. Ch 17.

**Speculative Decoding** — An inference optimization where a small draft model generates candidate tokens that a larger model then verifies in parallel. Speeds up generation by 2-3x with no quality loss because the verification can happen in a single forward pass. Ch 47, Ch 57.

**Spiral Curriculum** — An educational design pattern where concepts are introduced simply and revisited at increasing depth throughout the material. This guide uses a fractal spiral: embeddings appear in Ch 1 (concept), Ch 14 (search), Ch 36 (math), Ch 42 (generate), and Ch 44 (vs fine-tuning). Ch 0.

**Stable Diffusion** — The most widely used open-source diffusion model for image generation. Operates in latent space for efficiency and supports text-to-image, image-to-image, and inpainting. Can be personalized with DreamBooth and LoRA. Ch 41.

**Streaming** — Sending LLM output to the user token-by-token as it is generated, rather than waiting for the complete response. Reduces perceived latency dramatically (users see content in milliseconds rather than waiting seconds). Uses Server-Sent Events (SSE) or WebSockets. Ch 3, Ch 6.

**Source Attribution** — Including references to the source documents that informed an AI-generated answer. Essential for RAG systems where users need to verify claims. Typically implemented by injecting chunk metadata into the prompt and instructing the model to cite sources. Ch 15, Ch 16.

**Structured Output** — Constraining LLM output to a specific format (usually JSON matching a schema) for reliable parsing. Ch 5.

**Stop Condition** — The criterion that determines when an agent loop should terminate. Can be a special token, maximum iterations, LLM decision (end_turn), quality threshold, or a combination. Well-designed stop conditions prevent infinite loops and runaway costs. Ch 9.

**Stop Sequence** — A specific string that, when generated by the model, causes generation to halt. Configurable via API parameters. Different from the model's natural end token. Useful for constraining output format (e.g., stop at "\n\n" to get single-paragraph answers). Ch 39.

**Subword** — A token that represents part of a word rather than a complete word. Tokenizers break rare or unknown words into subword pieces: "unhappiness" might become ["un", "happiness"] or ["un", "happi", "ness"]. This is why all tokenizers have a fixed vocabulary but can handle any input text. Ch 37.

**Supervised Fine-Tuning (SFT)** — Fine-tuning with labeled examples (input-output pairs). The most common form of fine-tuning for LLMs. Requires high-quality training data: garbage in, garbage out. Ch 45, Ch 46.

**Synthetic Data** — Training data generated by an AI model rather than collected from real-world sources. Used to augment datasets when real data is scarce or expensive. Quality filtering is critical: synthetic data can amplify model biases if not carefully curated. Ch 46.

**System Prompt** — A special message at the beginning of a conversation that sets the AI's behavior, personality, and constraints. Not visible to the user but strongly influences all subsequent responses. Ch 4.

## T

**T5 (Text-to-Text Transfer Transformer)** — An encoder-decoder model that frames every NLP task as a text-to-text problem. Input: text (e.g., "translate English to French: Hello"), output: text (e.g., "Bonjour"). Demonstrates the power of unified task formatting. Ch 38, Ch 43.

**Task Decomposition** — Breaking a complex task into smaller subtasks that can be handled by individual agents or sequential steps. The coordinator pattern relies on this. A key skill for effective prompt engineering and agent architecture. Ch 58.

**Temperature** — A parameter that controls the randomness of LLM output by scaling the logits before sampling. Higher temperature (e.g., 1.0) = more random/creative. Lower (e.g., 0.1) = more focused/deterministic. At temperature=0, equivalent to greedy decoding. Ch 0, Ch 39.

**Telemetry** — Collecting and transmitting data about AI system performance, usage, and errors. For AI systems: token counts per request, latency (TTFT and total), tool call frequency and duration, cost per request, and quality scores. The foundation of observability and cost engineering. Ch 22.

**Tensor** — A multi-dimensional array of numbers. Scalars (0D), vectors (1D), matrices (2D), and higher-dimensional arrays are all tensors. Model weights, embeddings, and attention scores are all stored as tensors. PyTorch and TensorFlow are named for this fundamental data structure. Ch 36.

**Tensor Parallelism** — Splitting a single model's layers across multiple GPUs to run models that do not fit in a single GPU's memory. Different from data parallelism (same model, different data). Used by vLLM and TGI for large model serving. Ch 57.

**TF-IDF (Term Frequency-Inverse Document Frequency)** — A classic text scoring method that weights words by how frequent they are in a document vs. the corpus. Words that appear in many documents get lower weight. Simple, interpretable, and still competitive for keyword retrieval. Ch 17.

**Tick Loop** — In Claude Code, a background process that periodically checks for work to do during idle time. Enables idle-time agency (AutoDream, background maintenance, memory consolidation). The "heartbeat" of an always-on agent. Ch 27, Ch 32.

**Token** — The atomic unit of text that LLMs process. Typically ~0.75 words per token in English. Tokens can be whole words, subwords, or individual characters depending on the tokenizer. Determines cost, speed, and context window usage. Ch 0, Ch 37.

**Token Budget** — A limit on the total tokens an agent or fleet of agents can consume. Used for cost control in multi-agent systems. Ch 49, Ch 58.

**Tokenizer** — The component that converts text into token IDs (encoding) and token IDs back into text (decoding). Different models use different tokenizers and vocabularies (GPT uses BPE, BERT uses WordPiece, Llama uses SentencePiece). Text tokenized for one model cannot be used with another. Ch 0, Ch 37.

**TTFT (Time to First Token)** — The latency between sending a request and receiving the first output token. A critical UX metric for streaming applications. Prompt caching and smaller models improve TTFT. Ch 53, Ch 57.

**Tool Calling** — A protocol where the LLM requests that your application execute a function on its behalf. The LLM generates the function name and arguments in a structured format; your code executes the function and returns the result. Also called "function calling." The foundation of agent capabilities. Ch 8, Ch 24, Ch 30.

**Tool Description** — The natural language description provided with a tool definition that tells the LLM when and how to use the tool. The quality of tool descriptions directly impacts tool selection accuracy. Clear, specific descriptions with examples outperform vague ones. Ch 8, Ch 19.

**Top-k Sampling** — A decoding strategy that restricts the next token selection to the k most probable tokens, then samples from that set. With k=1, equivalent to greedy decoding. Typical values: k=40 to k=100. Ch 0, Ch 39.

**Top-p Sampling (Nucleus Sampling)** — A decoding strategy that selects from the smallest set of tokens whose cumulative probability exceeds the threshold p. Adapts dynamically: when the model is confident, fewer tokens are considered; when uncertain, more are included. Typical value: p=0.9 to p=0.95. Ch 0, Ch 39.

**Tracing** — Recording the sequence of operations (LLM calls, tool calls, decisions) within an AI system for debugging and analysis. Essential for understanding why an agent made a particular decision or where a pipeline failed. Ch 22, Ch 52.

**Training** — The process of adjusting model parameters to minimize a loss function on a dataset. Includes pre-training (learning general knowledge from large corpora) and fine-tuning (specializing on a specific task or domain). Ch 0, Ch 36, Ch 44-46.

**Training Loop** — The repeated cycle of: forward pass (compute predictions), loss computation, backward pass (compute gradients), optimizer step (update weights). One complete pass through all training data is an epoch. Ch 36, Ch 45.

**Transformer** — The neural network architecture behind all modern LLMs. Based on self-attention mechanisms and positional encoding rather than recurrence or convolution. Introduced in "Attention Is All You Need" (2017). Comes in three variants: encoder-only (BERT), decoder-only (GPT, Claude), and encoder-decoder (T5). Ch 38.

**Transfer Learning** — Using knowledge from a model trained on one task to improve performance on a different task. Pre-training + fine-tuning is the dominant form. The entire modern AI stack is built on this principle. Ch 44.

**Transformers Library** — Hugging Face's open-source Python library for working with transformer models. Provides a unified API for 200K+ models, including model loading, tokenization, inference, and fine-tuning. The `pipeline()` function provides the simplest entry point. Ch 40.

**Trust Spectrum** — A model for categorizing AI tool operations from fully trusted (no confirmation) to fully distrusted (requires approval). Read operations are generally trusted; write, delete, and send operations require escalating levels of confirmation. Ch 11, Ch 30.

**TRL (Transformer Reinforcement Learning)** — Hugging Face's library for training language models with reinforcement learning techniques (RLHF, DPO, PPO). Built on top of the Transformers library. Used for preference-based alignment and reward model training. Ch 44.

## U

**Unsloth** — An open-source library that accelerates LoRA and QLoRA fine-tuning by 2x with 80% less memory through custom CUDA kernels and memory-efficient backpropagation. Makes fine-tuning 70B models feasible on a single GPU. Ch 45.

**Upsert** — Insert a record if it does not exist, or update it if it does. Common operation in vector databases when refreshing embeddings. Ch 14.

**Underfitting** — When a model is too simple to capture patterns in the data. Performs poorly on both training and test data. Remedy: increase model capacity, train longer, or add features. Ch 36.

**Unigram Model** — A tokenization algorithm (used in SentencePiece) that starts with a large vocabulary and iteratively removes tokens that contribute least to the training data's likelihood. An alternative to BPE that tends to produce more linguistically meaningful subwords. Ch 37.

**Upstash** — A serverless data platform offering Redis and Vector database-as-a-service. Upstash Vector is a serverless vector database with pay-per-request pricing, suitable for RAG applications deployed on serverless infrastructure. Ch 14.

## V

**Vector Database / Vector Store** — A database optimized for storing and querying embedding vectors using approximate nearest neighbor (ANN) similarity search. Examples: Pinecone (managed), Weaviate (self-hosted/cloud), Qdrant (self-hosted/cloud), pgvector (PostgreSQL extension), Chroma (local), Upstash Vector (serverless). Ch 14.

**Vector Similarity Search** — Finding the nearest vectors to a query vector in a vector database using distance metrics (cosine similarity, Euclidean distance, dot product). The core operation of semantic search and RAG retrieval. At scale, uses approximate algorithms (HNSW, IVF) for speed. Ch 14.

**Validation Loss** — The loss computed on a held-out validation dataset during training. Used to monitor for overfitting: when validation loss starts increasing while training loss continues decreasing, the model is memorizing rather than learning. Ch 36, Ch 45.

**Vercel AI SDK** — A TypeScript SDK for building AI-powered applications. Provides streaming, tool calling, structured output, and multi-provider support. Framework-agnostic: works with Next.js, Svelte, Vue, and plain Node.js. Ch 6, Ch 13.

**Vocabulary** — The fixed set of all tokens a model can process. Determined during tokenizer training. GPT-4's vocabulary contains ~100K tokens; BERT's contains ~30K. Tokens not in the vocabulary are broken into smaller subword pieces. Vocabulary size is a trade-off: larger vocabularies encode text more efficiently but increase model size. Ch 37.

**vLLM** — A high-throughput inference engine for LLMs that uses PagedAttention for efficient memory management. Supports continuous batching, tensor parallelism, and many model architectures. The standard for self-hosted LLM serving. Ch 57.

## W

**Weight Decay** — A regularization technique that penalizes large weight values by adding a fraction of the weight magnitude to the loss function. Prevents the model from relying too heavily on any single feature. Ch 36, Ch 45.

**Windsurf** — An AI-powered IDE (formerly Codeium) that integrates LLM capabilities into the code editor, similar to Cursor. Ch 27.

**Warm-Up** — Gradually increasing the learning rate at the start of training to improve stability. Prevents the model from making large, destructive weight updates before it has seen enough data. Commonly used with transformers. Ch 36, Ch 45.

**WebSocket** — A persistent bidirectional communication protocol. Used to stream AI responses to client applications in real-time. Unlike HTTP, keeps the connection open for ongoing data flow. Ch 6, Ch 59.

**Weight** — A numerical parameter in a neural network that is learned during training. Weights determine how inputs are transformed into outputs. A 70B model has 70 billion weights. Ch 36.

**Window (Sliding)** — A technique for managing context by keeping only the most recent N messages or tokens. Simple but loses early context. Often combined with summarization of dropped content. Ch 12.

**Worker Agent** — In multi-agent systems, an agent that receives a specific subtask from a coordinator agent, executes it, and returns results. Workers are typically simpler and more focused than coordinators. The coordinator-worker pattern enables parallel task execution. Ch 32, Ch 58.

**WordPiece** — A tokenization algorithm used by BERT and related models. Similar to BPE but uses a likelihood-based criterion for merging subword pairs rather than frequency. Subword tokens that are continuations are prefixed with `##` (e.g., "playing" might become ["play", "##ing"]). Ch 37.

**Worktree** — A Git feature that allows multiple working directories linked to the same repository. In Claude Code, worktrees enable multiple agents to work on different branches simultaneously without interfering with each other. Each agent gets its own filesystem view. Combined with sandboxing for safe parallel agent execution. Ch 23, Ch 55.

## X

**XAI (Explainable AI)** — Techniques for making AI system decisions interpretable to humans. Includes attention visualization, feature attribution, and chain-of-thought reasoning as a form of built-in explainability. Ch 48.

## Y

**YAML** — A human-readable data serialization format commonly used for configuration files. In AI engineering, used for dataset definitions, eval configurations, fine-tuning configs (Axolotl), and Docker Compose files. Ch 46, Ch 55.

**You Only Look Once (YOLO)** — A family of real-time object detection models. Not a transformer but relevant because modern AI agents sometimes integrate vision models for screen analysis and image understanding tasks. Ch 7.

## Z

**Zero-Config Setup** — An onboarding pattern where a tool requires no manual configuration. Authentication is handled via SSO, integrations are pre-connected, and defaults are sensible. Ramp achieved 99.5% AI adoption partly by making Glass zero-config. Ch 59, Ch 62.

**Zero-Shot Prompting** — Asking an LLM to perform a task with no examples in the prompt. Relies entirely on the model's pre-trained knowledge and instruction tuning. Works surprisingly well for frontier models on common tasks. Ch 4.

**Zero-Shot Classification** — Classifying text into categories that were not part of the model's fine-tuning. Uses the model's general understanding to classify by description rather than label. Available as a Hugging Face pipeline task using natural language inference (NLI) models. Ch 40.

**Zod** — A TypeScript-first schema validation library used extensively for defining tool parameters and structured output schemas in AI applications. Zod schemas compile to JSON Schema, which providers use to constrain LLM output. Ch 5, Ch 8.

---

*[Back to Guide](../README.md)* | *Next: [Resources](./appendix-resources.md)*

<!--
  CHAPTER: 45
  TITLE: Fine-Tuning with LoRA
  PART: 9 — Fine-Tuning & Training
  PHASE: 2 — Become an Expert
  PREREQS: Ch 36 (Neural Networks), Ch 43 (Model Selection), Ch 44 (RAG vs Fine-Tuning)
  KEY_TOPICS: LoRA, low-rank adaptation, PEFT, fine-tuning, GPT-2, training loop, loss curves, adapters, learning rate, epochs, QLoRA
  DIFFICULTY: Advanced
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 45: Fine-Tuning with LoRA

> **Part 9 — Fine-Tuning & Training** | Phase 2: Become an Expert | Prerequisites: Ch 36, Ch 43, Ch 44 | Difficulty: Advanced | Language: Python

In Chapter 36, you built a neural network from scratch and watched weights update through backpropagation. In Chapter 44, you decided that fine-tuning is the right approach for your task. Now you do it for real.

Full fine-tuning of a large language model means updating every single weight — billions of parameters. That requires massive amounts of GPU memory, takes hours or days, and produces a full copy of the model. LoRA (Low-Rank Adaptation) makes this practical by adding tiny trainable layers on top of frozen weights. Instead of modifying 7 billion parameters, you train 1-10 million new parameters — less than 1% of the model — and get surprisingly good results.

This chapter is hands-on. You will fine-tune GPT-2 on a custom dataset using LoRA and the PEFT library. The complete training runs in under 30 minutes on a Google Colab T4 GPU. By the end, you will have a working fine-tuned model that generates text in a specific style, and you will understand every line of the training code.

### In This Chapter
- Why full fine-tuning is impractical for most teams
- How LoRA works: low-rank matrix decomposition
- The PEFT library: parameter-efficient fine-tuning
- Hands-on: fine-tune GPT-2 on a quotes dataset
- Training configuration: learning rate, epochs, batch size
- Monitoring training: loss curves, what good training looks like
- Saving and loading LoRA adapters
- QLoRA: quantized base model + LoRA adapters

### Related Chapters
- **Ch 21 (Eval-Driven Development)** — spirals back: use evals to measure whether fine-tuning improves your system
- **Ch 36 (Neural Networks)** — you understand weights and backpropagation; now you use them
- **Ch 43 (Model Selection)** — you picked the model; now you customize it
- **Ch 44 (RAG vs Fine-Tuning)** — you decided to fine-tune; here is how
- **Ch 46 (Dataset Engineering)** — better data makes better fine-tunes
- **Ch 47 (Quantization & Deployment)** — deploy the result

---

## 1. Why Full Fine-Tuning Is Impractical

### 1.1 The Scale Problem

```python
# Let's calculate what full fine-tuning actually requires

model_sizes = {
    "GPT-2 (124M)": 124_000_000,
    "GPT-2 Medium (345M)": 345_000_000,
    "Llama 3.1 8B": 8_000_000_000,
    "Llama 3.1 70B": 70_000_000_000,
}

print("Full Fine-Tuning Requirements:")
print("=" * 70)
for name, params in model_sizes.items():
    # Memory requirements for training (approximate):
    # Model weights (fp16): 2 bytes/param
    # Gradients: 2 bytes/param
    # Optimizer states (Adam): 8 bytes/param (2 momentum + 2 variance)
    # Activations: ~2-4x model size (depends on batch size and sequence length)
    
    model_memory = params * 2 / 1e9       # fp16 weights
    gradient_memory = params * 2 / 1e9     # fp16 gradients
    optimizer_memory = params * 8 / 1e9    # Adam states
    activation_memory = params * 4 / 1e9   # Rough activation estimate
    
    total = model_memory + gradient_memory + optimizer_memory + activation_memory
    
    print(f"\n  {name}")
    print(f"    Model weights:    {model_memory:>6.1f} GB")
    print(f"    Gradients:        {gradient_memory:>6.1f} GB")
    print(f"    Optimizer states: {optimizer_memory:>6.1f} GB")
    print(f"    Activations:      {activation_memory:>6.1f} GB")
    print(f"    TOTAL:            {total:>6.1f} GB")

print("\nA T4 GPU has 16GB. An A100 has 40/80GB.")
print("Full fine-tuning of even a 7B model needs specialized infrastructure.")
```

### 1.2 The LoRA Solution

LoRA adds small trainable matrices alongside the frozen model weights. Instead of updating W (a large weight matrix), it adds two small matrices A and B such that the update is W + BA, where B and A are much smaller than W.

```python
import numpy as np

# Conceptual illustration of LoRA
# Original weight matrix: d x k (e.g., 4096 x 4096 for a layer in Llama)
d = 4096  # Input dimension
k = 4096  # Output dimension

# Full fine-tuning: update ALL of W
W = np.random.randn(d, k)
full_params = d * k
print(f"Full weight matrix: {d} x {k} = {full_params:,} parameters")

# LoRA: decompose the update into two small matrices
r = 8  # Rank (typically 4, 8, 16, or 32)

A = np.random.randn(d, r)  # Down-projection: d x r
B = np.random.randn(r, k)  # Up-projection: r x k

lora_params = d * r + r * k
print(f"LoRA matrices: ({d} x {r}) + ({r} x {k}) = {lora_params:,} parameters")
print(f"Compression ratio: {full_params / lora_params:.0f}x fewer parameters")
print(f"Parameter percentage: {100 * lora_params / full_params:.2f}%")

# The effective weight becomes: W + B @ A (scaled by alpha/r)
alpha = 16  # LoRA scaling factor
scaling = alpha / r

W_effective = W + scaling * (A @ B)
print(f"\nEffective weight shape: {W_effective.shape} (same as original)")
print(f"But we only trained {lora_params:,} parameters instead of {full_params:,}")
```

The mathematical insight: weight updates during fine-tuning tend to have low rank. The model does not need to change dramatically — it just needs small, targeted adjustments. LoRA exploits this by constraining the update to a low-rank decomposition.

```
Original transformer layer:
  Input --> [Frozen W (d x k)] --> Output

With LoRA:
  Input --> [Frozen W (d x k)] -----------------------------> (+) --> Output
  Input --> [Trainable A (d x r)] --> [Trainable B (r x k)] -> (scale) -^

  Only A and B are trained. W stays frozen.
  At inference: merge W' = W + (alpha/r) * A @ B
```

---

## 2. The PEFT Library

### 2.1 Setup

```python
# Install the required libraries
# !pip install peft transformers datasets accelerate torch bitsandbytes trl

from peft import LoraConfig, get_peft_model, TaskType, PeftModel
from transformers import AutoTokenizer, AutoModelForCausalLM, TrainingArguments
import torch

print("PEFT (Parameter-Efficient Fine-Tuning) library loaded")
print("This library implements LoRA, QLoRA, and other efficient fine-tuning methods")
```

### 2.2 Configuring LoRA

```python
# LoRA configuration — every parameter explained
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,  # We're fine-tuning for text generation
    
    r=8,                    # Rank of the LoRA matrices
                            # Higher = more capacity but more parameters
                            # 4-8 for simple tasks, 16-32 for complex tasks
    
    lora_alpha=16,          # Scaling factor
                            # Effective scaling = alpha / r
                            # Rule of thumb: alpha = 2 * r
    
    lora_dropout=0.05,      # Dropout on LoRA layers
                            # Helps prevent overfitting on small datasets
    
    target_modules=[        # Which layers to apply LoRA to
        "c_attn",           # GPT-2's attention projection
        "c_proj",           # GPT-2's output projection
    ],
    # For Llama models, use: ["q_proj", "v_proj", "k_proj", "o_proj"]
    # For more thorough fine-tuning, also add: ["gate_proj", "up_proj", "down_proj"]
    
    bias="none",            # Don't train bias terms
)

print("LoRA Configuration:")
print(f"  Rank (r):        {lora_config.r}")
print(f"  Alpha:           {lora_config.lora_alpha}")
print(f"  Scaling:         {lora_config.lora_alpha / lora_config.r}")
print(f"  Dropout:         {lora_config.lora_dropout}")
print(f"  Target modules:  {lora_config.target_modules}")
```

### 2.3 Applying LoRA to a Model

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import LoraConfig, get_peft_model, TaskType

# Load GPT-2 (small enough to fine-tune on a free Colab GPU)
model_name = "gpt2"  # 124M parameters

tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token  # GPT-2 doesn't have a pad token

model = AutoModelForCausalLM.from_pretrained(model_name)

# Check original model size
original_params = sum(p.numel() for p in model.parameters())
print(f"Original model parameters: {original_params:,}")

# Apply LoRA
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=8,
    lora_alpha=16,
    lora_dropout=0.05,
    target_modules=["c_attn", "c_proj"],
)

peft_model = get_peft_model(model, lora_config)

# Check what changed
peft_model.print_trainable_parameters()
# Output: trainable params: 294,912 || all params: 124,734,464 || trainable%: 0.2364%

# That's the magic: we're only training 0.24% of the parameters
trainable = sum(p.numel() for p in peft_model.parameters() if p.requires_grad)
total = sum(p.numel() for p in peft_model.parameters())
print(f"\nTrainable parameters: {trainable:,} ({100*trainable/total:.2f}%)")
print(f"Frozen parameters:   {total - trainable:,} ({100*(total-trainable)/total:.2f}%)")
print(f"\nMemory saved: training {trainable:,} params vs {total:,} params")
```

---

## 3. Hands-On: Fine-Tune GPT-2 on a Quotes Dataset

### 3.1 Prepare the Dataset

```python
from datasets import Dataset

# We'll create a quotes dataset for fine-tuning
# The model will learn to generate inspirational quotes in a specific style

training_quotes = [
    "The only way to do great work is to love what you do.",
    "Innovation distinguishes between a leader and a follower.",
    "Stay hungry, stay foolish.",
    "The people who are crazy enough to think they can change the world are the ones who do.",
    "Your time is limited, don't waste it living someone else's life.",
    "Design is not just what it looks like and feels like. Design is how it works.",
    "Great things in business are never done by one person. They're done by a team of people.",
    "Quality is more important than quantity. One home run is much better than two doubles.",
    "I think if you do something and it turns out pretty good, then you should go do something else wonderful.",
    "Be a yardstick of quality. Some people aren't used to an environment where excellence is expected.",
    "Creativity is just connecting things.",
    "Technology is nothing. What's important is that you have a faith in people.",
    "The journey is the reward.",
    "I want to put a ding in the universe.",
    "Simple can be harder than complex.",
    "We're here to put a dent in the universe. Otherwise why else even be here?",
    "My favorite things in life don't cost any money. The most precious resource we all have is time.",
    "Details matter, it's worth waiting to get it right.",
    "The people who are doing the work are the moving force behind the company.",
    "Sometimes life hits you in the head with a brick. Don't lose faith.",
    "Let's go invent tomorrow rather than worrying about what happened yesterday.",
    "Have the courage to follow your heart and intuition.",
    "Going to bed at night saying we've done something wonderful, that's what matters to me.",
    "I'm as proud of many of the things we haven't done as the things we have done.",
    "You can't connect the dots looking forward; you can only connect them looking backwards.",
    "Your work is going to fill a large part of your life. Do what you believe is great work.",
    "Things don't have to change the world to be important.",
    "If you haven't found it yet, keep looking. Don't settle.",
    "Do not try to do everything. Do one thing well.",
    "Focus is about saying no.",
    "Every once in a while a revolutionary product comes along that changes everything.",
    "When you first start off trying to solve a problem, the first solutions you come up with are very complex.",
    "Most people make the mistake of thinking design is what it looks like.",
    "It's technology married with liberal arts that yields results that make our hearts sing.",
    "A lot of companies have chosen to downsize, and maybe that was the right thing for them. We chose a different path.",
    "What a computer is to me is the most remarkable tool that we have ever come up with.",
    "Getting fired from Apple was the best thing that could have ever happened to me.",
    "The engineering is long gone in most PC companies.",
]

# Format for fine-tuning: each quote as a training example
formatted_data = []
for quote in training_quotes:
    formatted_data.append({"text": f"Quote: {quote}\n"})

dataset = Dataset.from_list(formatted_data)
print(f"Dataset size: {len(dataset)} examples")
print(f"Example: {dataset[0]['text']}")
```

### 3.2 Tokenize the Dataset

```python
def tokenize_function(examples):
    """Tokenize the text data for causal language modeling."""
    tokenized = tokenizer(
        examples["text"],
        truncation=True,
        max_length=128,
        padding="max_length",
    )
    # For causal LM, labels = input_ids (the model predicts the next token)
    tokenized["labels"] = tokenized["input_ids"].copy()
    return tokenized

# Tokenize the dataset
tokenized_dataset = dataset.map(
    tokenize_function,
    batched=True,
    remove_columns=["text"],
)

print(f"Tokenized dataset: {len(tokenized_dataset)} examples")
print(f"Features: {tokenized_dataset.column_names}")
print(f"Example input_ids length: {len(tokenized_dataset[0]['input_ids'])}")

# Show what the tokens look like
example_tokens = tokenized_dataset[0]["input_ids"][:20]
example_text = tokenizer.decode(example_tokens)
print(f"First 20 tokens decoded: '{example_text}'")
```

### 3.3 Configure Training

```python
from transformers import TrainingArguments, Trainer, DataCollatorForLanguageModeling

# Training arguments — every parameter explained
training_args = TrainingArguments(
    output_dir="./gpt2-quotes-lora",      # Where to save checkpoints
    
    # Training hyperparameters
    num_train_epochs=3,                     # Number of passes through the data
    per_device_train_batch_size=4,          # Batch size per GPU
    gradient_accumulation_steps=4,          # Effective batch = 4 * 4 = 16
    
    # Learning rate
    learning_rate=2e-4,                     # LoRA can use higher LR than full fine-tuning
    lr_scheduler_type="cosine",             # Cosine decay schedule
    warmup_steps=10,                        # Gradual warmup from 0 to learning_rate
    
    # Optimization
    optim="adamw_torch",                    # Adam with weight decay
    weight_decay=0.01,                      # L2 regularization
    max_grad_norm=1.0,                      # Gradient clipping
    
    # Logging
    logging_steps=5,                        # Log every 5 steps
    logging_dir="./logs",                   # TensorBoard logs
    
    # Saving
    save_strategy="epoch",                  # Save after each epoch
    save_total_limit=2,                     # Keep only last 2 checkpoints
    
    # Performance
    fp16=torch.cuda.is_available(),         # Use fp16 on GPU
    dataloader_num_workers=2,               # Data loading workers
    
    # Misc
    remove_unused_columns=False,
    report_to="none",                       # Disable W&B etc. for simplicity
)

# Data collator handles dynamic batching
data_collator = DataCollatorForLanguageModeling(
    tokenizer=tokenizer,
    mlm=False,  # We're doing causal LM, not masked LM
)

print("Training configuration:")
print(f"  Epochs: {training_args.num_train_epochs}")
print(f"  Batch size: {training_args.per_device_train_batch_size}")
print(f"  Gradient accumulation: {training_args.gradient_accumulation_steps}")
print(f"  Effective batch size: {training_args.per_device_train_batch_size * training_args.gradient_accumulation_steps}")
print(f"  Learning rate: {training_args.learning_rate}")
print(f"  fp16: {training_args.fp16}")
```

### 3.4 Train

```python
# Create the trainer
trainer = Trainer(
    model=peft_model,
    args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=data_collator,
)

# Train the model
print("Starting training...")
print("=" * 50)

train_result = trainer.train()

# Print results
print("\nTraining complete!")
print(f"  Training loss: {train_result.training_loss:.4f}")
print(f"  Training time: {train_result.metrics.get('train_runtime', 0):.0f} seconds")
print(f"  Samples/second: {train_result.metrics.get('train_samples_per_second', 0):.1f}")
```

### 3.5 Generate with the Fine-Tuned Model

```python
# Test the fine-tuned model
peft_model.config.use_cache = True

def generate_quote(prompt="Quote:", max_new_tokens=50, temperature=0.8):
    """Generate a quote using the fine-tuned model."""
    inputs = tokenizer(prompt, return_tensors="pt")
    
    if torch.cuda.is_available():
        inputs = {k: v.to("cuda") for k, v in inputs.items()}
        peft_model.to("cuda")
    
    with torch.no_grad():
        outputs = peft_model.generate(
            **inputs,
            max_new_tokens=max_new_tokens,
            temperature=temperature,
            do_sample=True,
            top_p=0.9,
            repetition_penalty=1.2,
            pad_token_id=tokenizer.eos_token_id,
        )
    
    generated = tokenizer.decode(outputs[0], skip_special_tokens=True)
    return generated

# Generate several quotes
print("Generated Quotes (fine-tuned model):")
print("=" * 60)
for i in range(5):
    quote = generate_quote(temperature=0.8 + i * 0.1)
    print(f"\n  {quote}")

# Compare with base GPT-2 (without fine-tuning)
print("\n\nBase GPT-2 (not fine-tuned):")
print("=" * 60)
base_model = AutoModelForCausalLM.from_pretrained("gpt2")
if torch.cuda.is_available():
    base_model = base_model.to("cuda")

base_inputs = tokenizer("Quote:", return_tensors="pt")
if torch.cuda.is_available():
    base_inputs = {k: v.to("cuda") for k, v in base_inputs.items()}

with torch.no_grad():
    base_outputs = base_model.generate(
        **base_inputs,
        max_new_tokens=50,
        temperature=0.8,
        do_sample=True,
        top_p=0.9,
        pad_token_id=tokenizer.eos_token_id,
    )
base_text = tokenizer.decode(base_outputs[0], skip_special_tokens=True)
print(f"\n  {base_text}")
print("\nNotice the difference in style between fine-tuned and base model!")
```

---

## 4. Understanding the Training Process

### 4.1 Loss Curves: What Good Training Looks Like

```python
import matplotlib
matplotlib.use('Agg')  # Non-interactive backend
import matplotlib.pyplot as plt

# Parse training logs
log_history = trainer.state.log_history

# Extract loss values
steps = [entry["step"] for entry in log_history if "loss" in entry]
losses = [entry["loss"] for entry in log_history if "loss" in entry]

if steps:
    plt.figure(figsize=(10, 5))
    plt.plot(steps, losses, "b-", linewidth=2, label="Training Loss")
    plt.xlabel("Step")
    plt.ylabel("Loss")
    plt.title("Training Loss Over Time")
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.savefig("training_loss.png", dpi=100, bbox_inches="tight")
    plt.close()
    print("Loss curve saved to training_loss.png")
    
    print(f"\n  Starting loss: {losses[0]:.4f}")
    print(f"  Final loss:    {losses[-1]:.4f}")
    print(f"  Improvement:   {(losses[0] - losses[-1]) / losses[0] * 100:.1f}%")
else:
    print("No training logs found — run training first")

# What to look for in loss curves:
print("""
Good training signs:
  - Loss decreases steadily, then levels off
  - No sudden spikes (learning rate too high)
  - Final loss is significantly lower than starting loss
  
Bad training signs:
  - Loss oscillates wildly (learning rate too high)
  - Loss plateaus immediately (learning rate too low)
  - Loss decreases to near-zero (overfitting to training data)
  - Loss increases (something is very wrong)
""")
```

### 4.2 Hyperparameter Tuning Guide

```python
# The most important hyperparameters and how to tune them

hyperparameters = {
    "learning_rate": {
        "default": "2e-4 for LoRA (10x higher than full fine-tuning)",
        "too_high": "Loss oscillates or diverges",
        "too_low": "Training is very slow, loss barely decreases",
        "range": "1e-5 to 1e-3",
        "tip": "Start with 2e-4. If loss is unstable, halve it.",
    },
    "r (LoRA rank)": {
        "default": "8",
        "too_high": "More parameters to train, slower, risk of overfitting",
        "too_low": "Model can't learn enough, underfitting",
        "range": "4, 8, 16, 32",
        "tip": "8 works for most tasks. Use 16-32 for complex tasks.",
    },
    "num_train_epochs": {
        "default": "3-5 for small datasets, 1-2 for large datasets",
        "too_high": "Overfitting — model memorizes training data",
        "too_low": "Underfitting — model hasn't learned enough",
        "range": "1 to 10",
        "tip": "Watch the loss curve. Stop when it plateaus.",
    },
    "batch_size (effective)": {
        "default": "16-32",
        "too_high": "Faster training but may miss fine-grained patterns",
        "too_low": "Noisy gradients, unstable training",
        "range": "8 to 64",
        "tip": "Use gradient accumulation if GPU memory is limited.",
    },
}

print("Hyperparameter Tuning Guide:")
print("=" * 60)
for param, info in hyperparameters.items():
    print(f"\n  {param}")
    print(f"    Default:  {info['default']}")
    print(f"    Too high: {info['too_high']}")
    print(f"    Too low:  {info['too_low']}")
    print(f"    Range:    {info['range']}")
    print(f"    Tip:      {info['tip']}")
```

### 4.3 Hyperparameter Selection: A Deeper Guide

Choosing the right hyperparameters is the difference between a model that works and one that does not. Here is how each key parameter interacts with the others.

```python
# Learning rate and rank interact — here is the practical guide

lr_rank_guide = {
    "Simple style transfer (e.g., tone change)": {
        "rank": 4,
        "alpha": 8,
        "learning_rate": "3e-4",
        "epochs": "3-5",
        "rationale": "Small behavioral change needs few parameters and trains quickly.",
    },
    "Domain adaptation (e.g., legal, medical)": {
        "rank": 8,
        "alpha": 16,
        "learning_rate": "2e-4",
        "epochs": "3-5",
        "rationale": "Moderate change — model needs to learn new vocabulary patterns.",
    },
    "Complex task learning (e.g., structured extraction)": {
        "rank": 16,
        "alpha": 32,
        "learning_rate": "1e-4",
        "epochs": "5-10",
        "rationale": "Significant behavioral change. Higher rank captures more patterns. "
                     "Lower LR prevents instability with more trainable parameters.",
    },
    "Near-full fine-tuning equivalent": {
        "rank": 64,
        "alpha": 128,
        "learning_rate": "5e-5",
        "epochs": "2-3",
        "rationale": "Maximum capacity. Approaches full fine-tuning quality. "
                     "Low LR essential — too many parameters to be aggressive.",
    },
}

print("Hyperparameter Recipes by Task Complexity:")
print("=" * 70)
for task, config in lr_rank_guide.items():
    print(f"\n  {task}")
    print(f"    rank={config['rank']}, alpha={config['alpha']}, "
          f"lr={config['learning_rate']}, epochs={config['epochs']}")
    print(f"    Why: {config['rationale']}")

# The alpha/rank relationship explained
print("""
The alpha/rank ratio controls the effective learning rate of LoRA layers.
  - scaling factor = alpha / rank
  - alpha=16, rank=8 → scaling=2.0 (standard)
  - alpha=16, rank=16 → scaling=1.0 (conservative)
  - alpha=32, rank=8 → scaling=4.0 (aggressive)

Rule of thumb: keep alpha = 2 * rank for most tasks.
Only increase the ratio if the model is not learning enough.
""")
```

### 4.4 Evaluating Your Fine-Tuned Model

After training, you need a systematic way to check whether the fine-tune actually improved things.

```python
# Before/after comparison framework

def evaluate_fine_tune(base_model, fine_tuned_model, tokenizer, test_prompts):
    """Compare base model and fine-tuned model on the same prompts."""
    
    results = []
    
    for prompt in test_prompts:
        inputs = tokenizer(prompt, return_tensors="pt")
        
        # Generate from base model
        with torch.no_grad():
            base_output = base_model.generate(
                **inputs, max_new_tokens=100, temperature=0.7,
                do_sample=True, pad_token_id=tokenizer.eos_token_id
            )
        base_text = tokenizer.decode(base_output[0], skip_special_tokens=True)
        
        # Generate from fine-tuned model
        with torch.no_grad():
            ft_output = fine_tuned_model.generate(
                **inputs, max_new_tokens=100, temperature=0.7,
                do_sample=True, pad_token_id=tokenizer.eos_token_id
            )
        ft_text = tokenizer.decode(ft_output[0], skip_special_tokens=True)
        
        results.append({
            "prompt": prompt,
            "base": base_text[len(prompt):].strip(),
            "fine_tuned": ft_text[len(prompt):].strip(),
        })
    
    return results

# What to look for in the comparison:
evaluation_criteria = {
    "Style match": "Does the fine-tuned model adopt the target style/tone?",
    "Coherence": "Are fine-tuned outputs well-formed and coherent?",
    "Task accuracy": "Does it perform the intended task correctly?",
    "No regression": "Can it still handle general prompts that it could before?",
    "Diversity": "Does it generate varied outputs, or repeat training examples?",
}

print("Fine-Tune Evaluation Criteria:")
for criterion, question in evaluation_criteria.items():
    print(f"  {criterion:16s}: {question}")
```

### 4.5 Common Fine-Tuning Failures and Diagnosis

```python
# Failure patterns and how to fix them

failures = {
    "Model repeats training examples verbatim": {
        "diagnosis": "Overfitting — model memorized instead of generalizing",
        "evidence": "Generated text matches training examples word-for-word",
        "fixes": [
            "Reduce epochs (try 1-2 instead of 5)",
            "Increase training data diversity",
            "Increase LoRA dropout to 0.1",
            "Reduce rank (try 4 instead of 16)",
        ],
    },
    "Loss decreases but output quality does not improve": {
        "diagnosis": "Training format mismatch — model learns wrong thing",
        "evidence": "Loss curve looks healthy but generated text is still bad",
        "fixes": [
            "Check that training format matches inference format exactly",
            "Verify tokenizer adds correct special tokens",
            "Inspect training data for hidden formatting issues",
        ],
    },
    "Model loses general capability (catastrophic forgetting)": {
        "diagnosis": "Too aggressive fine-tuning — base knowledge is damaged",
        "evidence": "Model handles fine-tuned task OK but fails on basic questions",
        "fixes": [
            "Use LoRA instead of full fine-tuning (it freezes base weights)",
            "Reduce learning rate by 2-5x",
            "Reduce rank to limit parameter changes",
            "Mix in some general-purpose training data",
        ],
    },
    "Loss spikes or diverges during training": {
        "diagnosis": "Learning rate too high or data issue",
        "evidence": "Loss suddenly jumps up and does not recover",
        "fixes": [
            "Reduce learning rate by half",
            "Increase warmup steps",
            "Check for corrupted/malformed training examples",
            "Reduce batch size and increase gradient accumulation",
        ],
    },
    "Model generates endless repetitive text": {
        "diagnosis": "Repetition penalty needed or training data is repetitive",
        "evidence": "Output contains loops like 'the the the' or repeated phrases",
        "fixes": [
            "Add repetition_penalty=1.2 at inference time",
            "Check training data for repetitive patterns",
            "Try higher temperature (0.8-1.0) at inference",
        ],
    },
}

print("Fine-Tuning Failure Diagnosis Guide:")
print("=" * 60)
for symptom, info in failures.items():
    print(f"\n  Symptom: {symptom}")
    print(f"  Diagnosis: {info['diagnosis']}")
    print(f"  Fixes:")
    for fix in info['fixes']:
        print(f"    - {fix}")
```

---

## 5. Saving and Loading LoRA Adapters

### 5.1 Save the Adapter

LoRA adapters are tiny — typically a few megabytes compared to the multi-gigabyte base model.

```python
# Save just the LoRA adapter (NOT the full model)
adapter_path = "./gpt2-quotes-lora/adapter"
peft_model.save_pretrained(adapter_path)

# Check the size
import os

adapter_size = sum(
    os.path.getsize(os.path.join(adapter_path, f))
    for f in os.listdir(adapter_path)
    if os.path.isfile(os.path.join(adapter_path, f))
)

base_model_size = sum(
    p.numel() * p.element_size() for p in AutoModelForCausalLM.from_pretrained("gpt2").parameters()
)

print(f"Adapter size:    {adapter_size / 1e6:.2f} MB")
print(f"Base model size: {base_model_size / 1e6:.0f} MB")
print(f"Adapter is {base_model_size / adapter_size:.0f}x smaller than the base model!")
print(f"\nFiles saved:")
for f in os.listdir(adapter_path):
    size = os.path.getsize(os.path.join(adapter_path, f))
    print(f"  {f}: {size / 1e3:.1f} KB")
```

### 5.2 Load the Adapter

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

# Load base model
base_model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

# Load LoRA adapter on top
loaded_model = PeftModel.from_pretrained(base_model, adapter_path)

print("Model loaded: base GPT-2 + LoRA adapter")
print(f"Total parameters: {sum(p.numel() for p in loaded_model.parameters()):,}")

# Generate to verify it works
inputs = tokenizer("Quote:", return_tensors="pt")
with torch.no_grad():
    outputs = loaded_model.generate(
        **inputs,
        max_new_tokens=50,
        temperature=0.8,
        do_sample=True,
        pad_token_id=tokenizer.eos_token_id,
    )
print(f"Generated: {tokenizer.decode(outputs[0], skip_special_tokens=True)}")
```

### 5.3 Merge Adapter into Base Model

For deployment, you can merge the LoRA weights into the base model to avoid the adapter overhead.

```python
# Merge LoRA weights into the base model
merged_model = loaded_model.merge_and_unload()

# Now it's a regular model — no adapter needed
print(f"Merged model parameters: {sum(p.numel() for p in merged_model.parameters()):,}")
print("This is the same as the original GPT-2 but with LoRA weights baked in")

# Save the merged model
merged_model.save_pretrained("./gpt2-quotes-merged")
tokenizer.save_pretrained("./gpt2-quotes-merged")

# The merged model can be loaded without PEFT
final_model = AutoModelForCausalLM.from_pretrained("./gpt2-quotes-merged")
print("Merged model loaded — no PEFT library needed for inference!")
```

### 5.4 Multiple Adapters

One of LoRA's superpowers: you can train multiple adapters for different tasks and swap between them at inference time.

```python
# Conceptual: multiple adapters on one base model

print("""
Multiple LoRA Adapters:

  Base Model (frozen, loaded once)
    |
    |-- Adapter A: "quotes" style (1.2 MB)
    |-- Adapter B: "technical" style (1.2 MB)
    |-- Adapter C: "casual" style (1.2 MB)
    +-- Adapter D: "formal" style (1.2 MB)

  Total storage: ~500 MB (base) + 4.8 MB (all adapters)
  vs Full fine-tuning: 4 x 500 MB = 2 GB

  Swap adapters in milliseconds:
    model.load_adapter("adapter_a")
    model.set_adapter("adapter_a")
    # Now generates in "quotes" style
    
    model.set_adapter("adapter_b")
    # Now generates in "technical" style
""")
```

---

## 6. QLoRA: Quantized LoRA

### 6.1 What Is QLoRA?

QLoRA combines quantization with LoRA: the base model is loaded in 4-bit precision (using bitsandbytes), and LoRA adapters are trained in fp16 on top. This makes it possible to fine-tune much larger models on limited hardware.

```python
from transformers import AutoTokenizer, AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training, TaskType
import torch

# QLoRA configuration
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

# Load a larger model in 4-bit
# A 7B model that would normally need 14GB VRAM now fits in ~4GB
model_name = "gpt2-medium"  # Using GPT-2 medium for demo (345M params)

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=quantization_config,
    device_map="auto",
)
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# Prepare for training
model = prepare_model_for_kbit_training(model)

# Apply LoRA
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                    # Can use higher rank since base model is quantized
    lora_alpha=32,
    lora_dropout=0.05,
    target_modules=["c_attn", "c_proj"],
)

peft_model = get_peft_model(model, lora_config)
peft_model.print_trainable_parameters()

print(f"\nMemory footprint: {model.get_memory_footprint() / 1e9:.2f} GB")
print("This is a 4-bit quantized base model + fp16 LoRA adapters")
print("For a 7B model, this reduces memory from ~14GB to ~4GB")
```

### 6.2 QLoRA Memory Savings

```python
# Memory comparison for fine-tuning different model sizes

print("Memory Requirements for Fine-Tuning:")
print("=" * 65)
print(f"{'Model':<20} {'Full FT (fp16)':<18} {'LoRA (fp16)':<18} {'QLoRA (4-bit)'}")
print("-" * 65)

models = [
    ("GPT-2 (124M)", 0.5, 0.3, 0.2),
    ("GPT-2 Medium (345M)", 1.4, 0.8, 0.5),
    ("Llama 3.1 8B", 32, 18, 6),
    ("Llama 3.1 13B", 52, 30, 10),
    ("Llama 3.1 70B", 280, 160, 40),
]

for name, full, lora, qlora in models:
    print(f"  {name:<20} {full:>6.1f} GB        {lora:>6.1f} GB        {qlora:>6.1f} GB")

print(f"\n  GPU Reference: T4 = 16 GB, A10G = 24 GB, A100 = 40/80 GB")
print(f"  QLoRA enables fine-tuning 8B models on a T4!")
```

---

## 7. Fine-Tuning with TRL (Training with Reinforcement Learning)

### 7.1 SFTTrainer for Easier Fine-Tuning

The `trl` library provides `SFTTrainer`, which simplifies the fine-tuning workflow.

```python
from trl import SFTTrainer, SFTConfig
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import LoraConfig
from datasets import Dataset
import torch

# Prepare a simple instruction dataset
instruction_data = [
    {"text": "### Instruction: Write an inspirational quote about technology.\n### Response: Technology is nothing without the human vision that shapes it into something meaningful.\n"},
    {"text": "### Instruction: Write an inspirational quote about leadership.\n### Response: A great leader doesn't create followers -- they create more leaders.\n"},
    {"text": "### Instruction: Write an inspirational quote about creativity.\n### Response: Creativity is the courage to be wrong, the patience to be confused, and the wisdom to connect the unexpected.\n"},
    {"text": "### Instruction: Write an inspirational quote about learning.\n### Response: The expert was once a beginner who refused to stop asking questions.\n"},
    {"text": "### Instruction: Write an inspirational quote about persistence.\n### Response: The wall you are staring at was built one brick at a time, and that is exactly how you will get past it.\n"},
    {"text": "### Instruction: Write an inspirational quote about innovation.\n### Response: Innovation is not about having the best ideas. It is about creating an environment where the best ideas can find you.\n"},
    {"text": "### Instruction: Write an inspirational quote about teamwork.\n### Response: Alone you can go fast. Together you can build something that lasts.\n"},
    {"text": "### Instruction: Write an inspirational quote about failure.\n### Response: Failure is not the opposite of success. It is part of the same journey.\n"},
]

dataset = Dataset.from_list(instruction_data)

# Load model
model_name = "gpt2"
model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# LoRA config
peft_config = LoraConfig(
    r=8,
    lora_alpha=16,
    lora_dropout=0.05,
    target_modules=["c_attn", "c_proj"],
    task_type="CAUSAL_LM",
)

# SFTTrainer handles tokenization, formatting, and training
sft_config = SFTConfig(
    output_dir="./sft-quotes",
    num_train_epochs=5,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    logging_steps=2,
    save_strategy="epoch",
    fp16=torch.cuda.is_available(),
    report_to="none",
    max_seq_length=256,
)

trainer = SFTTrainer(
    model=model,
    args=sft_config,
    train_dataset=dataset,
    peft_config=peft_config,
    processing_class=tokenizer,
)

# Train
print("Training with SFTTrainer...")
trainer.train()

# Test
print("\nGeneration test:")
test_prompt = "### Instruction: Write an inspirational quote about simplicity.\n### Response:"
inputs = tokenizer(test_prompt, return_tensors="pt")
if torch.cuda.is_available():
    inputs = {k: v.to("cuda") for k, v in inputs.items()}
    trainer.model.to("cuda")

with torch.no_grad():
    outputs = trainer.model.generate(
        **inputs,
        max_new_tokens=50,
        temperature=0.8,
        do_sample=True,
        pad_token_id=tokenizer.eos_token_id,
    )
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

---

## 8. Key Takeaways

1. **LoRA makes fine-tuning practical.** By training less than 1% of parameters, LoRA reduces memory requirements by 10x or more and training time proportionally. The quality is surprisingly close to full fine-tuning.

2. **The adapter is tiny.** A LoRA adapter is typically 1-10MB, while the base model is 1-100GB. You can train dozens of adapters and swap between them instantly.

3. **QLoRA goes further.** 4-bit quantized base model + fp16 LoRA adapters lets you fine-tune 7B models on a free Google Colab T4 GPU.

4. **Learning rate is higher for LoRA.** Use 1e-4 to 3e-4 for LoRA, compared to 1e-5 to 5e-5 for full fine-tuning.

5. **Watch the loss curve.** Steady decrease means good training. Oscillation means the learning rate is too high. Plateau means the rate is too low or you need more data. Near-zero means overfitting.

6. **Merge for deployment.** After training, merge the LoRA weights into the base model for simpler deployment without the PEFT dependency.

7. **Data quality matters more than model size.** A well-fine-tuned 7B model on clean data often beats a poorly fine-tuned 70B model on noisy data. Chapter 46 covers this in depth.

---

## What's Next

In **Chapter 46: Dataset Engineering**, you learn how to build the training data that makes fine-tuning work. The model is only as good as its training data — and dataset engineering is where most fine-tuning projects succeed or fail.

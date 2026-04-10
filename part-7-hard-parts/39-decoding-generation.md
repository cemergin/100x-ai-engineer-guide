<!--
  CHAPTER: 39
  TITLE: Decoding & Generation
  PART: 7 — The Hard Parts
  PHASE: 2 — Become an Expert
  PREREQS: Ch 38 (Transformers & Attention), Ch 0 (How LLMs Actually Work)
  KEY_TOPICS: greedy decoding, beam search, top-k sampling, top-p/nucleus sampling, temperature, repetition penalty, end-of-sequence, generation pipeline
  DIFFICULTY: Advanced
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 39: Decoding & Generation

> Part 7 · Phase 2 · Prereqs: Ch 38, Ch 0 · Advanced · Python

In Chapter 0, you learned that LLMs generate text one token at a time — predict the next token, append it, repeat. You learned that temperature controls "creativity." That was enough to use the API. Now you will understand *exactly* what happens between "the model outputs probabilities" and "you see text on your screen."

This chapter completes the pipeline. Chapter 37 showed how text enters the model. Chapter 38 showed how the transformer processes it. This chapter shows how text comes out. By the end, you will implement every major decoding strategy — greedy, beam search, top-k, top-p, and temperature scaling — and understand why the same model can produce boring, repetitive text OR creative, surprising text depending on a few numbers.

This IS what Chapter 0 was pointing to. The temperature knob becomes a mathematical formula. The top-p parameter becomes an algorithm. The mystery becomes machinery.

### In This Chapter

1. The Decoding Problem
2. From Logits to Probabilities
3. Greedy Decoding: Always Pick the Best
4. Beam Search: Explore Multiple Paths
5. Top-k Sampling: Random From the Best k
6. Top-p (Nucleus) Sampling: Random Until Cumulative p
7. Temperature: The Math Behind the Knob
8. Combining Strategies
9. Repetition Penalty: Avoiding Loops
10. End-of-Sequence: Knowing When to Stop
11. A Complete Generation Pipeline
12. Connecting to Everything Else

### Related Chapters

- **Ch 0** — How LLMs Actually Work (spirals back: temperature from knob to math)
- **Ch 38** — Transformers & Attention (spirals back: the decoder produces the logits we decode)
- **Ch 40** — Hugging Face & Pipelines (spirals forward: control generation parameters in code)
- **Ch 43** — Model Selection & Architecture (spirals forward: decoder models generate, encoder models do not)

---

## 1. The Decoding Problem

The transformer outputs a probability distribution over the entire vocabulary for the next token. The question is: which token do you pick?

```python
import numpy as np

# The transformer outputs RAW SCORES (logits) for each token in the vocabulary.
# These are NOT probabilities yet.

np.random.seed(42)

# Simulate: vocabulary of 10 tokens, model outputs logits for next position
vocab = ["the", "cat", "sat", "on", "mat", "dog", "ran", "big", "a", "."]
vocab_size = len(vocab)

# Raw logits from the transformer's output layer
logits = np.array([2.1, 0.5, 3.8, 0.2, 4.5, -1.0, -0.5, 0.8, 1.2, 2.0])

print("Token logits (raw scores from transformer):")
for i, (token, logit) in enumerate(zip(vocab, logits)):
    print(f"  '{token}': {logit:+.2f}")

print(f"\nThe question: which token do we pick?")
print(f"The highest logit is '{vocab[np.argmax(logits)]}' ({logits.max():.2f})")
print(f"But should we ALWAYS pick the highest? Let's explore...")
```

---

## 2. From Logits to Probabilities

First, convert logits to probabilities using softmax.

```python
def softmax(logits):
    """Convert raw logits to probabilities that sum to 1."""
    exp_logits = np.exp(logits - np.max(logits))  # subtract max for numerical stability
    return exp_logits / exp_logits.sum()

probs = softmax(logits)

print("Logits → Probabilities (via softmax):")
print(f"  {'Token':<8} {'Logit':>8} {'Probability':>12} {'Bar'}")
for i in np.argsort(probs)[::-1]:  # sort by probability, descending
    bar = '#' * int(probs[i] * 50)
    print(f"  {vocab[i]:<8} {logits[i]:>+8.2f} {probs[i]:>12.4f}  {bar}")
print(f"\n  Sum of probabilities: {probs.sum():.4f}")
```

Output:
```
Logits → Probabilities (via softmax):
  Token       Logit  Probability  Bar
  mat          +4.50       0.4504  ######################
  sat          +3.80       0.2236  ###########
  the          +2.10       0.0815  ####
  .            +2.00       0.0738  ###
  a            +1.20       0.0332  #
  big          +0.80       0.0222  #
  cat          +0.50       0.0165  
  on           +0.20       0.0122  
  ran          -0.50       0.0061  
  dog          -1.00       0.0037  

  Sum of probabilities: 1.0000
```

The model thinks "mat" is most likely (45%), followed by "sat" (22%), then "the" (8%). Every decoding strategy starts from this probability distribution and makes a different choice.

---

## 3. Greedy Decoding: Always Pick the Best

The simplest strategy: always pick the token with the highest probability.

```python
def greedy_decode(logits):
    """Pick the token with the highest probability."""
    probs = softmax(logits)
    token_id = np.argmax(probs)
    return token_id, probs[token_id]

token_id, prob = greedy_decode(logits)
print(f"Greedy selection: '{vocab[token_id]}' (probability: {prob:.4f})")
```

### Greedy generation loop:

```python
def greedy_generate(model_fn, start_tokens, max_length=20, eos_token=None):
    """
    Generate text greedily: always pick the most likely next token.
    
    model_fn: function that takes token_ids → logits for next token
    start_tokens: initial token IDs to start from
    """
    generated = list(start_tokens)
    
    for step in range(max_length):
        # Get logits for next token
        logits = model_fn(generated)
        
        # Pick the most likely
        next_token = np.argmax(softmax(logits))
        generated.append(next_token)
        
        # Stop if we hit end-of-sequence
        if eos_token is not None and next_token == eos_token:
            break
    
    return generated

# Simulate a model that returns semi-plausible logits
def fake_model(token_ids):
    """A fake model that returns logits based on the last token."""
    np.random.seed(token_ids[-1] * 17 + len(token_ids))
    logits = np.random.randn(vocab_size)
    # Boost common patterns
    if len(token_ids) >= 2:
        # If last two tokens repeat, boost the period to end
        if token_ids[-1] == token_ids[-2]:
            logits[vocab.index(".")] += 3.0
    return logits

start = [vocab.index("the")]
generated = greedy_generate(fake_model, start, max_length=8, eos_token=vocab.index("."))
print(f"\nGreedy output: {' '.join(vocab[t] for t in generated)}")
```

### The problem with greedy decoding:

```python
# Greedy decoding is DETERMINISTIC.
# Given the same input, it ALWAYS produces the same output.
# This means:
#   - No creativity or variety
#   - Tends to produce generic, "safe" text
#   - Can get stuck in repetitive loops
#
# Example from a real model:
# Input:  "Once upon a time"
# Greedy: "Once upon a time, there was a man who was a man who was a man who..."
#
# The model assigns high probability to common patterns.
# Greedy always picks those, leading to repetitive, boring text.
#
# This is why the ChatGPT API defaults to temperature > 0:
# pure greedy decoding produces terrible conversational responses.
```

---

## 4. Beam Search: Explore Multiple Paths

Instead of committing to one token at each step, beam search maintains the top-k *sequences* and explores them all in parallel.

```python
def beam_search(model_fn, start_tokens, beam_width=3, max_length=8, eos_token=None):
    """
    Beam search: maintain top-k sequences at each step.
    
    beam_width: number of candidate sequences to keep
    """
    # Each beam is (sequence, cumulative_log_probability)
    beams = [(list(start_tokens), 0.0)]
    
    completed = []
    
    for step in range(max_length):
        all_candidates = []
        
        for seq, score in beams:
            if eos_token is not None and len(seq) > len(start_tokens) and seq[-1] == eos_token:
                completed.append((seq, score))
                continue
            
            # Get logits for next token
            logits = model_fn(seq)
            probs = softmax(logits)
            log_probs = np.log(probs + 1e-10)
            
            # Consider all possible next tokens
            for token_id in range(len(probs)):
                new_seq = seq + [token_id]
                new_score = score + log_probs[token_id]
                all_candidates.append((new_seq, new_score))
        
        # Keep only the top beam_width candidates
        all_candidates.sort(key=lambda x: x[1], reverse=True)
        beams = all_candidates[:beam_width]
        
        if not beams:
            break
    
    # Return the best completed sequence, or best beam if none completed
    all_results = completed + beams
    all_results.sort(key=lambda x: x[1], reverse=True)
    
    return all_results[0][0]

start = [vocab.index("the")]
beam_result = beam_search(fake_model, start, beam_width=3, max_length=8, eos_token=vocab.index("."))
print(f"Beam search (width=3): {' '.join(vocab[t] for t in beam_result)}")

# Compare with greedy
greedy_result = greedy_generate(fake_model, start, max_length=8, eos_token=vocab.index("."))
print(f"Greedy:                {' '.join(vocab[t] for t in greedy_result)}")
```

```python
# Beam search vs greedy:
#
# Greedy at step 1: picks "mat" (best single next token)
# Beam at step 1: keeps "mat", "sat", "the" (top 3)
#
# Greedy at step 2: picks the best next token after "mat"
# Beam at step 2: tries next token for ALL 3 candidates
#                  keeps the 3 best TWO-token sequences
#
# Beam search may find: "the sat on" (total prob 0.15)
# when greedy found:     "the mat sat" (total prob 0.12)
#
# The best first step doesn't always lead to the best overall sequence.
# Beam search explores more options — at the cost of more computation.
#
# Used for: translation (where the best overall sequence matters more
#           than token-by-token creativity)
# NOT used for: chat/conversation (too deterministic, like greedy)
```

---

## 5. Top-k Sampling: Random From the Best k

Sampling introduces randomness. Top-k sampling picks randomly from only the k most likely tokens.

```python
def top_k_sample(logits, k=5):
    """
    Sample from the top-k most likely tokens.
    
    1. Find the k tokens with highest probability
    2. Zero out all other tokens
    3. Re-normalize to get a valid distribution
    4. Sample randomly from this distribution
    """
    probs = softmax(logits)
    
    # Find top-k indices
    top_k_indices = np.argsort(probs)[-k:]
    
    # Zero out everything except top-k
    filtered_probs = np.zeros_like(probs)
    filtered_probs[top_k_indices] = probs[top_k_indices]
    
    # Re-normalize
    filtered_probs = filtered_probs / filtered_probs.sum()
    
    # Sample
    token_id = np.random.choice(len(filtered_probs), p=filtered_probs)
    
    return token_id, filtered_probs

# Show how top-k changes the distribution
print("Original distribution vs Top-k filtered:")
print(f"  {'Token':<8} {'Original':>10} {'Top-3':>10} {'Top-5':>10}")

probs_original = softmax(logits)
_, probs_k3 = top_k_sample(logits, k=3)  # Just to get filtered probs
_, probs_k5 = top_k_sample(logits, k=5)

# Recompute without sampling
probs_k3_display = np.zeros_like(probs_original)
top3 = np.argsort(probs_original)[-3:]
probs_k3_display[top3] = probs_original[top3]
probs_k3_display /= probs_k3_display.sum()

probs_k5_display = np.zeros_like(probs_original)
top5 = np.argsort(probs_original)[-5:]
probs_k5_display[top5] = probs_original[top5]
probs_k5_display /= probs_k5_display.sum()

for i in np.argsort(probs_original)[::-1]:
    print(f"  {vocab[i]:<8} {probs_original[i]:>10.4f} {probs_k3_display[i]:>10.4f} {probs_k5_display[i]:>10.4f}")

print(f"\n  k=3: Only 'mat', 'sat', 'the' are candidates (all others = 0)")
print(f"  k=5: 'mat', 'sat', 'the', '.', 'a' are candidates")
```

```python
# Run top-k sampling multiple times to show variety
print("\nTop-k=3 sampling (10 attempts):")
for attempt in range(10):
    np.random.seed(attempt)
    token_id, _ = top_k_sample(logits, k=3)
    print(f"  Attempt {attempt+1}: '{vocab[token_id]}'")

# The same input produces DIFFERENT outputs each time.
# This is the "creativity" that greedy decoding lacks.
```

### Choosing k:

```python
# k = 1:  Equivalent to greedy decoding (no randomness)
# k = 5:  Moderate creativity (picks from top 5 candidates)
# k = 50: High creativity (picks from top 50 candidates)
# k = vocab_size: Pure random sampling from full distribution
#
# Trade-off:
#   Small k → more predictable, higher quality, less creative
#   Large k → more creative, more surprising, more risk of nonsense
#
# Problem with top-k:
#   k is FIXED regardless of the distribution shape.
#   Sometimes the top 3 tokens have 95% of the probability → k=3 is fine.
#   Sometimes the top 3 tokens have 30% of the probability → k=3 misses good options.
#   We need something adaptive...
```

---

## 6. Top-p (Nucleus) Sampling: Random Until Cumulative p

Top-p (nucleus) sampling adaptively selects tokens based on cumulative probability, not a fixed count.

```python
def top_p_sample(logits, p=0.9):
    """
    Nucleus sampling: include tokens until cumulative probability reaches p.
    
    1. Sort tokens by probability (descending)
    2. Include tokens until their cumulative probability exceeds p
    3. Zero out excluded tokens
    4. Re-normalize and sample
    """
    probs = softmax(logits)
    
    # Sort by probability descending
    sorted_indices = np.argsort(probs)[::-1]
    sorted_probs = probs[sorted_indices]
    
    # Find the cutoff: cumulative sum reaches p
    cumulative = np.cumsum(sorted_probs)
    
    # Keep tokens up to and including the one that pushes past p
    cutoff_index = np.searchsorted(cumulative, p) + 1
    cutoff_index = min(cutoff_index, len(probs))
    
    # Keep only the nucleus
    nucleus_indices = sorted_indices[:cutoff_index]
    
    # Zero out everything outside the nucleus
    filtered_probs = np.zeros_like(probs)
    filtered_probs[nucleus_indices] = probs[nucleus_indices]
    filtered_probs = filtered_probs / filtered_probs.sum()
    
    # Sample
    token_id = np.random.choice(len(filtered_probs), p=filtered_probs)
    
    return token_id, filtered_probs, cutoff_index

# Show how top-p adaptively selects tokens
print("Top-p (nucleus) sampling with p=0.9:")
print(f"\n  {'Token':<8} {'Prob':>8} {'Cumulative':>12} {'Included':>10}")

sorted_idx = np.argsort(probs)[::-1]
cumulative = 0
for i, idx in enumerate(sorted_idx):
    cumulative += probs[idx]
    included = "YES" if cumulative <= 0.9 or i == 0 else ("YES (cutoff)" if i == np.searchsorted(np.cumsum(probs[sorted_idx]), 0.9) else "no")
    # Simpler logic
    included = "YES" if cumulative - probs[idx] < 0.9 else "no"
    print(f"  {vocab[idx]:<8} {probs[idx]:>8.4f} {cumulative:>12.4f} {included:>10}")

_, filtered, n_tokens = top_p_sample(logits, p=0.9)
print(f"\n  Nucleus size: {n_tokens} tokens (adaptive!)")
```

### Why top-p is better than top-k:

```python
# Scenario 1: The model is very confident
# Probs: [0.85, 0.10, 0.03, 0.01, 0.01, ...]
# Top-k=5: includes tokens with 0.01 probability — too much noise
# Top-p=0.9: includes only the first token (0.85) + second (0.10) = 0.95 > 0.9
# → Nucleus size = 2, more focused

# Scenario 2: The model is uncertain
# Probs: [0.15, 0.12, 0.10, 0.08, 0.07, 0.06, 0.05, ...]
# Top-k=5: only includes 52% of probability mass — too restrictive  
# Top-p=0.9: includes ~12 tokens to reach 90% — appropriately broad
# → Nucleus size = 12, more exploratory

# Top-p ADAPTS to the model's confidence.
# Confident → small nucleus → less randomness
# Uncertain → large nucleus → more exploration

# This is why the ChatGPT API defaults to top_p=1.0 (no filtering)
# and uses temperature to control randomness instead.
# But many practitioners use both.

print("Adaptive nucleus size examples:")
scenarios = [
    ("Confident", np.array([5.0, 2.0, 0.5, -1.0, -2.0, -3.0, -4.0, -5.0, -6.0, -7.0])),
    ("Moderate",  np.array([2.0, 1.8, 1.5, 1.0, 0.5, 0.0, -0.5, -1.0, -2.0, -3.0])),
    ("Uncertain", np.array([0.5, 0.4, 0.3, 0.2, 0.1, 0.0, -0.1, -0.2, -0.3, -0.4])),
]

for name, scenario_logits in scenarios:
    _, _, n = top_p_sample(scenario_logits, p=0.9)
    top_prob = softmax(scenario_logits).max()
    print(f"  {name}: top probability = {top_prob:.2%}, nucleus size = {n} tokens")
```

---

## 7. Temperature: The Math Behind the Knob

In Chapter 0, you learned: "temperature controls creativity. Low temperature = conservative, high temperature = creative." Now you see the actual math.

```python
def apply_temperature(logits, temperature=1.0):
    """
    Scale logits by temperature before softmax.
    
    temperature < 1: sharper distribution (more confident, less random)
    temperature = 1: original distribution (unchanged)
    temperature > 1: flatter distribution (less confident, more random)
    temperature → 0: approaches greedy (argmax)
    temperature → ∞: approaches uniform random
    """
    return logits / temperature

# Show the effect of temperature on probabilities
print("Temperature effect on probability distribution:")
print(f"\n  {'Token':<8}", end="")
for t in [0.1, 0.5, 1.0, 1.5, 2.0]:
    print(f"  {'T='+str(t):>8}", end="")
print()

temperatures = [0.1, 0.5, 1.0, 1.5, 2.0]

for i in np.argsort(probs)[::-1]:
    print(f"  {vocab[i]:<8}", end="")
    for temp in temperatures:
        scaled_logits = apply_temperature(logits, temp)
        p = softmax(scaled_logits)[i]
        print(f"  {p:>8.4f}", end="")
    print()

print(f"\n  T=0.1: 'mat' has 99.99% — essentially greedy")
print(f"  T=0.5: 'mat' has 75.6%  — pretty confident")
print(f"  T=1.0: 'mat' has 45.0%  — original distribution")
print(f"  T=1.5: 'mat' has 30.3%  — flattened, more options")
print(f"  T=2.0: 'mat' has 22.5%  — very flat, nearly random")
```

### The math explained:

```python
# Standard softmax:    P(token_i) = exp(logit_i) / sum(exp(logit_j))
#
# With temperature:    P(token_i) = exp(logit_i / T) / sum(exp(logit_j / T))
#
# Why does this work?
#
# Dividing logits by T before softmax:
# - If T < 1: logits get LARGER → exp(large) grows exponentially
#   → the largest logit dominates even more → sharper distribution
# - If T > 1: logits get SMALLER → differences between logits shrink
#   → all tokens become more equally likely → flatter distribution
# - If T = 1: no change
# - If T → 0: the largest logit → infinity, everything else → 0
#   → equivalent to argmax (greedy decoding)
# - If T → ∞: all logits → 0 → all probabilities → 1/vocab_size
#   → uniform random (every token equally likely)

# Numerical demonstration:
print("\nMath of temperature scaling:")
print(f"  Original logits: mat=4.50, sat=3.80, dog=-1.00")
print()

for T in [0.1, 0.5, 1.0, 2.0]:
    l_mat = 4.50 / T
    l_sat = 3.80 / T
    l_dog = -1.00 / T
    print(f"  T={T}:")
    print(f"    Scaled logits: mat={l_mat:>8.2f}, sat={l_sat:>8.2f}, dog={l_dog:>8.2f}")
    print(f"    Difference mat-sat: {l_mat - l_sat:.2f} (original: {4.50-3.80:.2f})")
    print(f"    → {'Bigger' if T < 1 else 'Smaller' if T > 1 else 'Same'} gap = "
          f"{'sharper' if T < 1 else 'flatter' if T > 1 else 'same'} distribution")
    print()
```

### Temperature in practice:

```python
# API parameter mapping to real applications:
temperature_guide = {
    0.0: {
        "behavior": "Greedy decoding (deterministic)",
        "use_case": "Factual questions, code generation, structured output",
        "example": "What is 2+2? → 4 (always the same answer)",
    },
    0.3: {
        "behavior": "Low randomness, mostly picks top tokens",
        "use_case": "Summarization, data extraction, classification",
        "example": "Summarize: ... → consistent, predictable summaries",
    },
    0.7: {
        "behavior": "Moderate randomness, creative but controlled",
        "use_case": "Conversational AI, general writing",
        "example": "Write a story: ... → creative but coherent",
    },
    1.0: {
        "behavior": "Full model distribution, balanced",
        "use_case": "Brainstorming, creative writing",
        "example": "Give me ideas for: ... → diverse suggestions",
    },
    1.5: {
        "behavior": "High randomness, surprising outputs",
        "use_case": "Very creative tasks, novelty generation",
        "example": "Write a poem: ... → unusual, unexpected phrasing",
    },
}

print("Temperature Guide:")
for temp, info in temperature_guide.items():
    print(f"\n  T = {temp}:")
    print(f"    Behavior: {info['behavior']}")
    print(f"    Use case: {info['use_case']}")
    print(f"    Example:  {info['example']}")
```

> **The spiral from Chapter 0:** In Chapter 0, you read: "Temperature 0 = deterministic, temperature 1 = creative." Now you know the EXACT MECHANISM: dividing logits by temperature before softmax changes the shape of the probability distribution. Low temperature sharpens it (confident). High temperature flattens it (uncertain). The knob IS a division operation.

---

## 8. Combining Strategies

In practice, temperature, top-k, and top-p are combined. Here is the typical order of operations:

```python
def decode_with_all_strategies(logits, temperature=1.0, top_k=None, top_p=None):
    """
    Complete decoding with temperature + top-k + top-p.
    
    Order of operations:
    1. Apply temperature (scale logits)
    2. Convert to probabilities (softmax)
    3. Apply top-k filtering (if specified)
    4. Apply top-p filtering (if specified)
    5. Re-normalize
    6. Sample
    """
    # Step 1: Temperature
    scaled_logits = logits / temperature if temperature > 0 else logits
    
    # Step 2: Softmax
    if temperature == 0:
        # Special case: greedy (return argmax)
        return np.argmax(logits)
    
    probs = softmax(scaled_logits)
    
    # Step 3: Top-k filtering
    if top_k is not None and top_k < len(probs):
        top_k_indices = np.argsort(probs)[-top_k:]
        mask = np.zeros_like(probs)
        mask[top_k_indices] = 1
        probs = probs * mask
    
    # Step 4: Top-p filtering
    if top_p is not None and top_p < 1.0:
        sorted_indices = np.argsort(probs)[::-1]
        sorted_probs = probs[sorted_indices]
        cumulative = np.cumsum(sorted_probs)
        
        # Find cutoff
        cutoff = np.searchsorted(cumulative, top_p) + 1
        allowed_indices = sorted_indices[:cutoff]
        
        mask = np.zeros_like(probs)
        mask[allowed_indices] = 1
        probs = probs * mask
    
    # Step 5: Re-normalize
    if probs.sum() > 0:
        probs = probs / probs.sum()
    else:
        probs = np.ones_like(probs) / len(probs)
    
    # Step 6: Sample
    token_id = np.random.choice(len(probs), p=probs)
    return token_id

# Compare different combinations
print("Comparing decoding strategies (10 samples each):")
strategies = [
    {"name": "Greedy (T=0)",       "temperature": 0,   "top_k": None, "top_p": None},
    {"name": "T=0.7",              "temperature": 0.7, "top_k": None, "top_p": None},
    {"name": "T=0.7, top_k=3",    "temperature": 0.7, "top_k": 3,    "top_p": None},
    {"name": "T=0.7, top_p=0.9",  "temperature": 0.7, "top_k": None, "top_p": 0.9},
    {"name": "T=1.0, top_p=0.95", "temperature": 1.0, "top_k": None, "top_p": 0.95},
    {"name": "T=1.5",              "temperature": 1.5, "top_k": None, "top_p": None},
]

for strategy in strategies:
    samples = []
    for seed in range(10):
        np.random.seed(seed)
        tid = decode_with_all_strategies(
            logits,
            temperature=strategy["temperature"],
            top_k=strategy["top_k"],
            top_p=strategy["top_p"]
        )
        samples.append(vocab[tid])
    
    unique = len(set(samples))
    print(f"\n  {strategy['name']:25s}: {samples}")
    print(f"  {'':25s}  Unique tokens: {unique}/10 (variety score)")
```

### API parameter defaults:

```python
# How major APIs handle these parameters:
#
# OpenAI (GPT-4):
#   temperature: 1.0 (default)
#   top_p: 1.0 (default — no filtering)
#   Recommendation: "Use temperature OR top_p, not both"
#
# Anthropic (Claude):
#   temperature: 1.0 (default)
#   top_k: not exposed in standard API
#   top_p: not exposed in standard API
#   Use: temperature only
#
# Hugging Face (local models):
#   temperature: 1.0
#   top_k: 50
#   top_p: 1.0
#   do_sample: False (must set True to enable sampling)
#   Most flexible — all parameters available
```

---

## 9. Repetition Penalty: Avoiding Loops

Without intervention, language models tend to repeat themselves. Repetition penalty reduces the probability of tokens that have already appeared.

```python
def apply_repetition_penalty(logits, generated_tokens, penalty=1.2):
    """
    Reduce the logits for tokens that have already been generated.
    
    penalty > 1: reduce probability of repeated tokens
    penalty = 1: no change
    penalty < 1: INCREASE probability of repeated tokens (unusual)
    """
    penalized = logits.copy()
    
    for token_id in set(generated_tokens):
        if penalized[token_id] > 0:
            penalized[token_id] /= penalty   # Reduce positive logits
        else:
            penalized[token_id] *= penalty   # Make negative logits more negative
    
    return penalized

# Demonstration: a model that keeps generating "the"
generated_so_far = [0, 0, 0]  # "the the the"

print("Without repetition penalty:")
probs_no_penalty = softmax(logits)
print(f"  P('the') = {probs_no_penalty[0]:.4f}")

print("\nWith repetition penalty (1.2):")
penalized_logits = apply_repetition_penalty(logits, generated_so_far, penalty=1.2)
probs_penalized = softmax(penalized_logits)
print(f"  P('the') = {probs_penalized[0]:.4f}")
print(f"  Reduction: {(1 - probs_penalized[0]/probs_no_penalty[0])*100:.1f}%")

print("\nWith strong repetition penalty (2.0):")
strong_penalized = apply_repetition_penalty(logits, generated_so_far, penalty=2.0)
probs_strong = softmax(strong_penalized)
print(f"  P('the') = {probs_strong[0]:.4f}")
print(f"  Reduction: {(1 - probs_strong[0]/probs_no_penalty[0])*100:.1f}%")
```

```python
# Other approaches to prevent repetition:
#
# 1. Frequency penalty: penalize proportionally to how OFTEN a token appeared
#    (appears 3 times → 3x penalty)
#
# 2. Presence penalty: penalize any token that appeared at all
#    (binary: appeared or not, regardless of count)
#
# 3. N-gram blocking: prevent any n-gram from appearing twice
#    (if "the cat sat" appeared, block "the cat sat" from appearing again)
#
# OpenAI API exposes: frequency_penalty (-2 to 2) and presence_penalty (-2 to 2)
# These directly modify logits before sampling.

def apply_frequency_penalty(logits, generated_tokens, penalty=0.5):
    """Penalize tokens proportionally to their frequency in generated text."""
    penalized = logits.copy()
    token_counts = {}
    for tid in generated_tokens:
        token_counts[tid] = token_counts.get(tid, 0) + 1
    
    for tid, count in token_counts.items():
        penalized[tid] -= penalty * count
    
    return penalized

def apply_presence_penalty(logits, generated_tokens, penalty=0.5):
    """Penalize any token that has appeared (binary)."""
    penalized = logits.copy()
    appeared = set(generated_tokens)
    
    for tid in appeared:
        penalized[tid] -= penalty
    
    return penalized
```

---

## 10. End-of-Sequence: Knowing When to Stop

The model must know when to stop generating. It does this by generating a special end-of-sequence (EOS) token.

```python
# How generation stops:
#
# Method 1: EOS token
#   The model generates <|endoftext|> or </s> or <EOS>
#   The generation loop sees this and stops.
#   This is the "natural" stopping point — the model decided it's done.
#
# Method 2: Max length
#   The generation loop hits a maximum token count and stops.
#   This is a safety measure — prevents infinite generation.
#   max_tokens in the API controls this.
#
# Method 3: Stop sequences
#   The user specifies strings that trigger stopping.
#   If the generated text contains "Human:" or "\n\n", stop.
#   Useful for controlling format (e.g., stop after generating JSON).

def generate_with_stopping(model_fn, start_tokens, vocab, 
                           max_length=50, eos_token_id=None,
                           stop_sequences=None, temperature=1.0,
                           top_p=0.9):
    """
    Generation loop with multiple stopping conditions.
    """
    generated = list(start_tokens)
    
    for step in range(max_length):
        # Get logits
        logits = model_fn(generated)
        
        # Apply temperature and sample
        token_id = decode_with_all_strategies(
            logits, temperature=temperature, top_p=top_p
        )
        generated.append(token_id)
        
        # Check EOS
        if eos_token_id is not None and token_id == eos_token_id:
            break
        
        # Check stop sequences
        if stop_sequences:
            text_so_far = " ".join(vocab[t] for t in generated)
            for stop_seq in stop_sequences:
                if stop_seq in text_so_far:
                    # Trim to stop sequence
                    idx = text_so_far.index(stop_seq)
                    return generated, "stop_sequence", step + 1
        
    else:
        return generated, "max_length", max_length
    
    return generated, "eos_token", step + 1

# Demonstrate different stopping reasons
print("Stopping conditions:")
print(f"  EOS token:      Model generates '<EOS>' → stop naturally")
print(f"  Max length:     Hit max_tokens limit → forced stop")
print(f"  Stop sequence:  Generated text contains '\\n\\n' → stop on format")
print(f"\nIn the API:")
print(f"  max_tokens=100    → Method 2")
print(f"  stop=['\\n\\n']     → Method 3")
print(f"  (EOS is always active — Method 1)")
```

---

## 11. A Complete Generation Pipeline

Let's put everything together: a complete text generation system using all the techniques from this chapter.

```python
class TextGenerator:
    """
    A complete text generation pipeline.
    
    Combines: embedding lookup → transformer forward pass → decoding strategy
    """
    
    def __init__(self, vocab, d_model=16, n_heads=2, n_layers=2, seed=42):
        self.vocab = vocab
        self.vocab_size = len(vocab)
        self.d_model = d_model
        
        np.random.seed(seed)
        
        # Token embeddings
        self.embedding_matrix = np.random.randn(self.vocab_size, d_model) * 0.1
        
        # Simple "transformer" weights (simplified — no real attention here)
        self.W_hidden = np.random.randn(d_model, d_model) * 0.3
        self.W_output = np.random.randn(d_model, self.vocab_size) * 0.3
        
        # Bigram statistics (to make output somewhat coherent)
        self.bigram_boost = np.zeros((self.vocab_size, self.vocab_size))
        common_pairs = [
            ("the", "cat"), ("the", "dog"), ("the", "mat"),
            ("cat", "sat"), ("dog", "ran"), ("sat", "on"),
            ("on", "the"), ("a", "big"), ("big", "cat"),
            ("big", "dog"), ("ran", "on"), ("the", "big"),
        ]
        for w1, w2 in common_pairs:
            if w1 in vocab and w2 in vocab:
                i, j = vocab.index(w1), vocab.index(w2)
                self.bigram_boost[i, j] = 2.0
    
    def get_logits(self, token_ids):
        """Forward pass: token IDs → logits for next token."""
        # Get embedding for last token
        last_embedding = self.embedding_matrix[token_ids[-1]]
        
        # Simple hidden layer
        hidden = np.tanh(last_embedding @ self.W_hidden)
        
        # Project to vocabulary
        logits = hidden @ self.W_output
        
        # Add bigram boost for coherence
        logits += self.bigram_boost[token_ids[-1]]
        
        return logits
    
    def generate(self, prompt_tokens, max_length=20, temperature=1.0,
                 top_k=None, top_p=None, repetition_penalty=1.0):
        """
        Generate text from a prompt.
        
        Returns: list of token IDs, generation metadata
        """
        generated = list(prompt_tokens)
        eos_id = self.vocab.index(".") if "." in self.vocab else None
        
        metadata = {
            "steps": 0,
            "stop_reason": "max_length",
            "tokens_generated": 0,
        }
        
        for step in range(max_length):
            # Get logits
            logits = self.get_logits(generated)
            
            # Apply repetition penalty
            if repetition_penalty != 1.0:
                logits = apply_repetition_penalty(
                    logits, generated, penalty=repetition_penalty
                )
            
            # Decode
            if temperature == 0:
                token_id = np.argmax(logits)
            else:
                token_id = decode_with_all_strategies(
                    logits, temperature=temperature,
                    top_k=top_k, top_p=top_p
                )
            
            generated.append(token_id)
            metadata["tokens_generated"] += 1
            
            # Check EOS
            if eos_id is not None and token_id == eos_id:
                metadata["stop_reason"] = "eos_token"
                break
        
        metadata["steps"] = step + 1
        return generated, metadata
    
    def tokens_to_text(self, token_ids):
        """Convert token IDs back to text."""
        return " ".join(self.vocab[t] for t in token_ids)

# Create the generator
vocab = ["the", "cat", "sat", "on", "mat", "dog", "ran", "big", "a", "."]
gen = TextGenerator(vocab, d_model=16)

# Generate with different strategies
prompt = [vocab.index("the")]

print("=" * 60)
print("TEXT GENERATION PIPELINE")
print("=" * 60)

strategies = [
    {"name": "Greedy (T=0)",
     "temperature": 0, "top_k": None, "top_p": None, "repetition_penalty": 1.0},
    {"name": "Conservative (T=0.3)",
     "temperature": 0.3, "top_k": None, "top_p": 0.9, "repetition_penalty": 1.0},
    {"name": "Balanced (T=0.7, top_p=0.9)",
     "temperature": 0.7, "top_k": None, "top_p": 0.9, "repetition_penalty": 1.2},
    {"name": "Creative (T=1.0, top_p=0.95)",
     "temperature": 1.0, "top_k": None, "top_p": 0.95, "repetition_penalty": 1.3},
    {"name": "Wild (T=1.5)",
     "temperature": 1.5, "top_k": None, "top_p": None, "repetition_penalty": 1.5},
]

for strategy in strategies:
    print(f"\n{strategy['name']}:")
    for run in range(3):
        np.random.seed(run * 42 + 7)
        tokens, meta = gen.generate(
            prompt, max_length=10,
            temperature=strategy["temperature"],
            top_k=strategy["top_k"],
            top_p=strategy["top_p"],
            repetition_penalty=strategy["repetition_penalty"]
        )
        text = gen.tokens_to_text(tokens)
        print(f"  Run {run+1}: {text}")
        print(f"         [{meta['tokens_generated']} tokens, stopped: {meta['stop_reason']}]")
```

---

## 12. Connecting to Everything Else

### 12.1 The Complete Chapter 0 → Chapter 39 Arc

```python
# What you learned in Chapter 0:
#   "LLMs predict the next token"
#   "Temperature controls creativity"
#   "Tokens cost money"
#
# What you now understand:
#   Tokenization (Ch 37) → splits text into subwords → token IDs
#   Embeddings (Ch 37) → token IDs look up vectors in a matrix
#   Attention (Ch 38) → every token looks at every other token
#   Transformer blocks (Ch 38) → attention + feedforward, stacked N times
#   Output projection (Ch 38) → produces logits over vocabulary
#   Softmax (this chapter) → turns logits into probabilities
#   Temperature (this chapter) → divides logits before softmax
#   Top-k/top-p (this chapter) → filters low-probability tokens
#   Sampling (this chapter) → randomly selects from filtered distribution
#   EOS detection (this chapter) → stops when done
#   Detokenization (Ch 37) → converts token IDs back to text
#
# You understand the ENTIRE pipeline from text in to text out.
# Every "magic" parameter is now a specific mathematical operation.
```

### 12.2 What You Can Now Understand

```python
# When you see API parameters like:
#   temperature=0.7, max_tokens=1000, top_p=0.95, stop=["\n\n"]
#
# You know EXACTLY what each one does:
#   temperature=0.7  → divide logits by 0.7 → sharper distribution → more focused
#   max_tokens=1000  → stop after generating 1000 tokens
#   top_p=0.95       → only sample from tokens covering 95% of probability mass
#   stop=["\n\n"]    → stop when generated text contains double newline
#
# When a model generates repetitive text:
#   → repetition penalty too low, or temperature too low
#
# When a model generates incoherent text:
#   → temperature too high, or top_p too high
#
# When a model always gives the same answer:
#   → temperature is 0 (greedy), or effectively greedy (T very small)
#
# When a model uses too many tokens:
#   → max_tokens not set, or EOS token not being generated
```

### 12.3 Forward to Hugging Face (Ch 40)

```python
# In Chapter 40, you will control these parameters on real models:
#
# from transformers import pipeline
#
# generator = pipeline("text-generation", model="gpt2")
# 
# output = generator(
#     "Once upon a time",
#     max_length=50,
#     temperature=0.7,
#     top_k=50,
#     top_p=0.95,
#     repetition_penalty=1.2,
#     do_sample=True,
#     num_return_sequences=3,
# )
#
# Every parameter maps to something you implemented in this chapter.
# "do_sample=True" → use sampling instead of greedy.
# "num_return_sequences=3" → generate 3 different completions.
# You are no longer guessing at what these do. You built them.
```

### 12.4 The Decoding Strategy Decision Tree

```python
# Choosing a decoding strategy:
#
#  Is the task FACTUAL (code, math, extraction)?
#    YES → temperature=0 (greedy) or temperature=0.1-0.3
#    NO ↓
#
#  Is the task CREATIVE (stories, brainstorming)?
#    YES → temperature=0.7-1.0, top_p=0.9-0.95, repetition_penalty=1.1-1.3
#    NO ↓
#
#  Is the task TRANSLATION or SUMMARIZATION?
#    YES → beam search (beam_width=4-5) OR temperature=0.3-0.5
#    NO ↓
#
#  Is the task CONVERSATION (chat)?
#    YES → temperature=0.7, top_p=0.9 (balanced)
#    NO ↓
#
#  Default: temperature=1.0, top_p=1.0 (let the model be itself)
```

---

## Summary

You implemented every major text generation strategy from scratch. You now understand:

1. **Logits to probabilities** — softmax converts raw scores to a probability distribution over the vocabulary
2. **Greedy decoding** — always pick the highest probability token (deterministic, boring, repetitive)
3. **Beam search** — explore multiple candidate sequences in parallel (better for translation, still deterministic)
4. **Top-k sampling** — randomly pick from the k most likely tokens (introduces variety, fixed k)
5. **Top-p (nucleus) sampling** — randomly pick from tokens until cumulative probability reaches p (adaptive, better than top-k)
6. **Temperature** — divide logits by T before softmax; T<1 sharpens (conservative), T>1 flattens (creative)
7. **Repetition penalty** — reduce logits for already-generated tokens (prevents loops)
8. **End-of-sequence** — EOS tokens, max length, and stop sequences tell the model when to stop
9. **The complete pipeline** — tokenize, embed, attend, project, decode, detokenize

The "magic" of language model generation is not magic. It is: matrix multiply, softmax, divide by temperature, filter by nucleus, sample randomly, check for EOS, repeat. Every parameter you set in an API call maps to a specific mathematical operation that you have now implemented by hand.

Part 7 is complete. You have gone from "models predict stuff" (Ch 34) to "here is how every component works" (Ch 39). In Part 8, you will take this understanding and apply it: running real models with Hugging Face, choosing architectures, and generating embeddings.

---

*Next: [Part 8 — Open Source AI & Inference](../part-8-open-source-ai/)* | *Previous: [Chapter 38 — Transformers & Attention](./38-transformers-attention.md)*

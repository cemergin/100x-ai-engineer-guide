<!--
  CHAPTER: 38
  TITLE: Transformers & Attention
  PART: 7 — The Hard Parts
  PHASE: 2 — Become an Expert
  PREREQS: Ch 36 (Neural Networks from Scratch), Ch 37 (Tokenization Deep Dive)
  KEY_TOPICS: self-attention, query/key/value, multi-head attention, positional encoding, transformer block, encoder, decoder, encoder-decoder
  DIFFICULTY: Advanced
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 38: Transformers & Attention

> **Part 7 — The Hard Parts** | Phase 2: Become an Expert | Prerequisites: Ch 36, Ch 37 | Difficulty: Advanced | Language: Python

In Chapter 36, you built a neural network where every input pixel connects to every hidden neuron. That works for fixed-size inputs like 30-pixel faces. But language has a problem: the word "bank" means completely different things in "river bank" and "savings bank." A standard neural network treats each token independently — it cannot look at surrounding tokens to resolve ambiguity.

Attention solves this. It lets every token look at every other token and decide: "Which other tokens help me understand what I mean in this context?" This single idea — attention — is why transformers work. It is why "Attention Is All You Need" (the 2017 paper) changed everything.

In this chapter, you will implement self-attention from scratch, build multi-head attention, add positional encoding, and assemble a complete transformer block. Then you will understand the three transformer architectures: encoders (BERT), decoders (GPT), and encoder-decoders (T5). By the end, when someone says "multi-head self-attention with causal masking," you will know exactly what every word means.

### In This Chapter

1. The Problem Attention Solves
2. Self-Attention: Every Token Looks at Every Other Token
3. Query, Key, Value: The Mechanics of Attention
4. Scaled Dot-Product Attention
5. Multi-Head Attention: Multiple Perspectives
6. Positional Encoding: Teaching Word Order
7. The Full Transformer Block
8. Encoder Transformers (BERT)
9. Decoder Transformers (GPT)
10. Encoder-Decoder Transformers (T5)
11. Visualizing What Attention Learns
12. Connecting to Everything Else

### Related Chapters

- **Ch 36** — Neural Networks from Scratch (spirals back: transformers extend neural nets with attention)
- **Ch 37** — Tokenization Deep Dive (spirals back: attention operates on token sequences)
- **Ch 39** — Decoding & Generation (spirals forward: how decoders generate text token by token)
- **Ch 40** — Hugging Face & Pipelines (spirals forward: run real transformer models)
- **Ch 43** — Model Selection & Architecture (spirals forward: choose between encoder/decoder/both)

---

## 1. The Problem Attention Solves

Consider two sentences:

```python
sentence_1 = "I went to the bank to deposit my check"
sentence_2 = "I sat on the bank of the river and fished"

# The word "bank" appears in both sentences.
# In sentence_1, "bank" means a financial institution.
# In sentence_2, "bank" means the edge of a river.
#
# A standard neural network (like Ch 36) processes each token independently.
# It would give "bank" the same embedding in both sentences.
# That's wrong — the MEANING depends on the CONTEXT.

# What the model needs to do:
# In sentence_1: "bank" looks at "deposit", "check" → financial meaning
# In sentence_2: "bank" looks at "river", "fished" → geographical meaning
#
# Attention provides this mechanism.
```

Before attention, NLP models used recurrent neural networks (RNNs) that processed tokens sequentially — one after another. This was slow (couldn't parallelize) and forgetful (struggled with long-range dependencies). Attention processes all tokens simultaneously and lets any token attend to any other token, regardless of distance.

```python
# The attention intuition:
#
# For each token in the sequence:
#   1. Look at every other token
#   2. Compute a "relevance score" for each other token
#   3. Combine information from all tokens, weighted by relevance
#
# "I sat on the bank of the river"
#
# When processing "bank":
#   "I"      → relevance: 0.02  (not very relevant)
#   "sat"    → relevance: 0.05  (somewhat relevant)
#   "on"     → relevance: 0.03  (not very relevant)
#   "the"    → relevance: 0.02  (not very relevant)
#   "bank"   → relevance: 0.08  (self-reference)
#   "of"     → relevance: 0.05  (structural)
#   "the"    → relevance: 0.02  (not very relevant)
#   "river"  → relevance: 0.65  (VERY relevant — disambiguates meaning!)
#   "fished" → relevance: 0.08  (relevant — confirms river context)
#
# Weighted combination → "bank" now means "river bank", not "financial bank"
```

---

## 2. Self-Attention: Every Token Looks at Every Other Token

Let's implement basic self-attention. We start with a sequence of token embeddings and compute how much each token should attend to every other token.

```python
import numpy as np

# Simulate a sequence of 4 tokens, each with 8-dimensional embeddings
np.random.seed(42)

# Our "sentence": 4 tokens with 8-dimensional embeddings
# (In real models: 512-12288 dimensions per token)
seq_length = 4
d_model = 8  # embedding dimension

# Token embeddings (normally these come from the embedding matrix, Ch 37)
tokens = ["The", "cat", "sat", "down"]
X = np.random.randn(seq_length, d_model)  # (4, 8)

print("Token embeddings:")
for i, token in enumerate(tokens):
    print(f"  '{token}': [{', '.join(f'{v:.2f}' for v in X[i])}]")

# Simplest self-attention: dot product between all pairs of tokens
# Score(token_i, token_j) = token_i . token_j
# High score → token_i should pay attention to token_j

scores = X @ X.T  # (4, 4) — every token with every other token

print(f"\nRaw attention scores (shape {scores.shape}):")
print(f"  {'':>6}", end="")
for t in tokens:
    print(f"  {t:>6}", end="")
print()
for i, t in enumerate(tokens):
    print(f"  {t:>6}", end="")
    for j in range(len(tokens)):
        print(f"  {scores[i,j]:6.2f}", end="")
    print()
```

Now we need to turn these raw scores into attention weights (probabilities that sum to 1):

```python
def softmax(x, axis=-1):
    """Compute softmax: turn scores into probabilities."""
    exp_x = np.exp(x - np.max(x, axis=axis, keepdims=True))
    return exp_x / np.sum(exp_x, axis=axis, keepdims=True)

attention_weights = softmax(scores)

print(f"Attention weights (after softmax):")
print(f"  {'':>6}", end="")
for t in tokens:
    print(f"  {t:>6}", end="")
print()
for i, t in enumerate(tokens):
    print(f"  {t:>6}", end="")
    for j in range(len(tokens)):
        print(f"  {attention_weights[i,j]:6.3f}", end="")
    print(f"  (sum: {attention_weights[i].sum():.3f})")
```

```python
# Now use the attention weights to create new, context-aware representations
# Each token's new representation = weighted average of ALL token embeddings

output = attention_weights @ X  # (4, 4) @ (4, 8) = (4, 8)

print(f"\nOutput after attention (shape {output.shape}):")
for i, token in enumerate(tokens):
    print(f"  '{token}' (original): [{', '.join(f'{v:.2f}' for v in X[i])}]")
    print(f"  '{token}' (attended): [{', '.join(f'{v:.2f}' for v in output[i])}]")
    print()

# The output for each token is now a BLEND of all tokens,
# weighted by how relevant each other token is.
# "cat" after attention contains information from "The", "sat", and "down".
```

---

## 3. Query, Key, Value: The Mechanics of Attention

The simple dot product above has a problem: it uses the same embedding for computing relevance AND for providing information. Real attention uses three separate projections: **Query**, **Key**, and **Value**.

```python
# The library analogy:
#
# You walk into a library (the sequence of tokens).
# You have a QUESTION in mind (the Query).
# Each book has a LABEL on its spine (the Key).
# Each book has CONTENT inside (the Value).
#
# Steps:
# 1. Compare your Question to each book's Label (Query @ Key)
# 2. The best-matching books get the highest relevance scores
# 3. You read the Content of the relevant books (weighted by score)
# 4. You combine what you read into your answer
#
# Query: "What does 'bank' mean here?"
# Keys:  ["I" label, "sat" label, "on" label, "bank" label, "river" label]
# Values: ["I" content, "sat" content, "on" content, "bank" content, "river" content]
# Match: "river" key matches the query best → read "river" value → "bank" = riverbank

# In math:
# Q = X @ W_Q   (project embeddings into "question space")
# K = X @ W_K   (project embeddings into "label space")
# V = X @ W_V   (project embeddings into "content space")
# 
# Attention(Q, K, V) = softmax(Q @ K.T / sqrt(d_k)) @ V

# Implement Q, K, V attention
d_k = 4  # dimension of query/key space (can differ from d_model)
d_v = 4  # dimension of value space

# Learned projection matrices
np.random.seed(42)
W_Q = np.random.randn(d_model, d_k) * 0.5  # (8, 4)
W_K = np.random.randn(d_model, d_k) * 0.5  # (8, 4)
W_V = np.random.randn(d_model, d_v) * 0.5  # (8, 4)

# Project embeddings
Q = X @ W_Q  # (4, 4) — queries
K = X @ W_K  # (4, 4) — keys
V = X @ W_V  # (4, 4) — values

print("Projections:")
print(f"  Q (queries) shape: {Q.shape}")
print(f"  K (keys) shape:    {K.shape}")
print(f"  V (values) shape:  {V.shape}")
print()

for i, token in enumerate(tokens):
    print(f"  '{token}':")
    print(f"    Q = [{', '.join(f'{v:.2f}' for v in Q[i])}]")
    print(f"    K = [{', '.join(f'{v:.2f}' for v in K[i])}]")
    print(f"    V = [{', '.join(f'{v:.2f}' for v in V[i])}]")
```

Now why are Q, K, V separate? Because the model needs different things from the same token:

```python
# Why separate Q, K, V?
#
# Consider the sentence: "The cat that chased the mouse sat down"
# When processing "sat":
#   - Query: "Who or what is sitting?" 
#   - Keys: each token offers "I am a noun/verb/adjective..." 
#   - Values: each token offers its actual semantic content
#
# "cat" has:
#   - Key that says "I am a noun, an agent" (matches "who is sitting?")
#   - Value that says "I am a feline animal"
#
# "mouse" has:
#   - Key that says "I am a noun, a patient" (doesn't match as well)
#   - Value that says "I am a small rodent"
#
# Without separate Q/K/V, the model uses the same representation for
# both "what am I looking for?" and "what information do I provide?"
# That's too constraining.
```

---

## 4. Scaled Dot-Product Attention

The full attention computation, as described in "Attention Is All You Need":

```python
def scaled_dot_product_attention(Q, K, V, mask=None):
    """
    Compute scaled dot-product attention.
    
    Q: queries  (seq_len, d_k)
    K: keys     (seq_len, d_k)
    V: values   (seq_len, d_v)
    mask: optional mask (seq_len, seq_len)
    
    Returns: (output, attention_weights)
    """
    d_k = Q.shape[-1]
    
    # Step 1: Compute raw attention scores
    scores = Q @ K.T  # (seq_len, seq_len)
    
    # Step 2: Scale by sqrt(d_k)
    # Without scaling, large d_k → large dot products → sharp softmax → vanishing gradients
    scores = scores / np.sqrt(d_k)
    
    # Step 3: Apply mask (if provided)
    if mask is not None:
        scores = np.where(mask == 0, -1e9, scores)
    
    # Step 4: Softmax to get attention weights
    weights = softmax(scores)
    
    # Step 5: Weighted sum of values
    output = weights @ V  # (seq_len, d_v)
    
    return output, weights

# Run attention on our example
output, weights = scaled_dot_product_attention(Q, K, V)

print("Scaled Dot-Product Attention:")
print(f"\nAttention weights (who attends to whom):")
print(f"  {'':>6}", end="")
for t in tokens:
    print(f"  {t:>6}", end="")
print()
for i, t in enumerate(tokens):
    print(f"  {t:>6}", end="")
    for j in range(len(tokens)):
        print(f"  {weights[i,j]:6.3f}", end="")
    print()

print(f"\nOutput shape: {output.shape}")
print(f"(Each token now has a context-aware representation)")
```

### Why scale by sqrt(d_k)?

```python
# Demonstration: why scaling matters

d_k_small = 4
d_k_large = 512

# With small d_k, dot products are manageable
q_small = np.random.randn(d_k_small)
k_small = np.random.randn(d_k_small)
dot_small = q_small @ k_small
print(f"d_k = {d_k_small}: dot product = {dot_small:.2f}")

# With large d_k, dot products grow large
q_large = np.random.randn(d_k_large)
k_large = np.random.randn(d_k_large)
dot_large = q_large @ k_large
print(f"d_k = {d_k_large}: dot product = {dot_large:.2f}")

# Large dot products → softmax becomes very sharp (nearly one-hot)
# → gradient almost zero → training stalls
#
# Dividing by sqrt(d_k) keeps dot products in a reasonable range

print(f"\nScaled: {dot_large / np.sqrt(d_k_large):.2f}")
print(f"(Back to a reasonable magnitude)")
```

---

## 5. Multi-Head Attention: Multiple Perspectives

One attention head captures one kind of relationship. Multi-head attention runs *multiple* attention heads in parallel, each learning a different pattern.

```python
class MultiHeadAttention:
    """
    Multi-head attention: run multiple attention heads in parallel,
    then combine their outputs.
    """
    
    def __init__(self, d_model, n_heads, seed=42):
        """
        d_model: embedding dimension (e.g., 8)
        n_heads: number of attention heads (e.g., 2)
        """
        assert d_model % n_heads == 0, "d_model must be divisible by n_heads"
        
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads  # dimension per head
        
        np.random.seed(seed)
        scale = np.sqrt(2.0 / d_model)
        
        # Each head gets its own Q, K, V projection
        # We store all heads' projections concatenated for efficiency
        self.W_Q = np.random.randn(d_model, d_model) * scale
        self.W_K = np.random.randn(d_model, d_model) * scale
        self.W_V = np.random.randn(d_model, d_model) * scale
        
        # Output projection: combine all heads
        self.W_O = np.random.randn(d_model, d_model) * scale
        
        params = 4 * d_model * d_model  # Q, K, V, O
        print(f"MultiHeadAttention: {n_heads} heads, d_k={self.d_k}")
        print(f"  Parameters: {params}")
    
    def split_heads(self, X):
        """Split the last dimension into (n_heads, d_k)."""
        seq_len = X.shape[0]
        return X.reshape(seq_len, self.n_heads, self.d_k).transpose(1, 0, 2)
        # Output: (n_heads, seq_len, d_k)
    
    def forward(self, X, mask=None):
        """
        X: input embeddings (seq_len, d_model)
        mask: optional attention mask
        Returns: output (seq_len, d_model), attention weights per head
        """
        seq_len = X.shape[0]
        
        # Project to Q, K, V
        Q = X @ self.W_Q  # (seq_len, d_model)
        K = X @ self.W_K
        V = X @ self.W_V
        
        # Split into heads
        Q_heads = self.split_heads(Q)  # (n_heads, seq_len, d_k)
        K_heads = self.split_heads(K)
        V_heads = self.split_heads(V)
        
        # Compute attention for each head
        head_outputs = []
        all_weights = []
        
        for h in range(self.n_heads):
            out, w = scaled_dot_product_attention(
                Q_heads[h], K_heads[h], V_heads[h], mask
            )
            head_outputs.append(out)
            all_weights.append(w)
        
        # Concatenate all heads: (n_heads, seq_len, d_k) → (seq_len, d_model)
        concatenated = np.concatenate(head_outputs, axis=-1)
        
        # Final linear projection
        output = concatenated @ self.W_O  # (seq_len, d_model)
        
        return output, all_weights

# Create multi-head attention with 2 heads
mha = MultiHeadAttention(d_model=8, n_heads=2)

# Run on our token embeddings
output, head_weights = mha.forward(X)

print(f"\nInput shape:  {X.shape}")
print(f"Output shape: {output.shape}")

# Show attention patterns for each head
for h in range(2):
    print(f"\nHead {h+1} attention weights:")
    print(f"  {'':>6}", end="")
    for t in tokens:
        print(f"  {t:>6}", end="")
    print()
    for i, t in enumerate(tokens):
        print(f"  {t:>6}", end="")
        for j in range(len(tokens)):
            print(f"  {head_weights[h][i,j]:6.3f}", end="")
        print()
```

> **Why multiple heads?** Each head can learn a different type of relationship:
> - Head 1 might learn syntactic relationships ("sat" attends to "cat" — subject-verb)
> - Head 2 might learn positional relationships ("sat" attends to "down" — adjacent words)
> - Head 3 might learn semantic relationships ("bank" attends to "river" — meaning)
>
> With only one head, the model must compress all these relationships into a single attention pattern. Multiple heads let it maintain multiple perspectives simultaneously.

---

## 6. Positional Encoding: Teaching Word Order

Attention has no inherent concept of order. "The cat sat" and "sat cat The" produce the same attention scores (because attention is symmetric over positions). We need to inject position information.

```python
def positional_encoding(seq_length, d_model):
    """
    Generate sinusoidal positional encoding.
    
    Uses sine and cosine functions of different frequencies.
    Position 0 gets one pattern, position 1 gets a slightly different pattern, etc.
    """
    PE = np.zeros((seq_length, d_model))
    
    for pos in range(seq_length):
        for i in range(0, d_model, 2):
            # Even dimensions: sine
            PE[pos, i] = np.sin(pos / (10000 ** (i / d_model)))
            # Odd dimensions: cosine
            if i + 1 < d_model:
                PE[pos, i+1] = np.cos(pos / (10000 ** (i / d_model)))
    
    return PE

# Generate positional encoding for our sequence
PE = positional_encoding(seq_length=4, d_model=8)

print("Positional encodings (each position gets a unique pattern):")
for pos in range(4):
    print(f"  Position {pos}: [{', '.join(f'{v:+.3f}' for v in PE[pos])}]")

# Add positional encoding to token embeddings
X_with_position = X + PE

print(f"\nOriginal embeddings + positional encoding:")
for i, token in enumerate(tokens):
    print(f"  '{token}' (pos {i}):")
    print(f"    Original: [{', '.join(f'{v:+.3f}' for v in X[i])}]")
    print(f"    + PE:     [{', '.join(f'{v:+.3f}' for v in PE[i])}]")
    print(f"    = Result: [{', '.join(f'{v:+.3f}' for v in X_with_position[i])}]")
```

### Why sinusoidal?

```python
# Why sine and cosine waves for position?
#
# 1. Each position gets a UNIQUE pattern (no two positions have the same encoding)
# 2. The model can learn RELATIVE positions:
#    The difference between PE[pos] and PE[pos+k] is consistent
#    regardless of pos. This means "3 tokens apart" looks the same
#    whether it's positions (0,3) or (5,8).
# 3. Generalizes to longer sequences:
#    The encoding works for any sequence length, even longer than training.

# Demonstration: positional encoding for longer sequences
PE_long = positional_encoding(seq_length=20, d_model=8)

print("\nPositional encoding pattern (first 2 dimensions, 20 positions):")
print(f"  {'Pos':>4}  {'sin(dim0)':>10}  {'cos(dim1)':>10}")
for pos in range(20):
    print(f"  {pos:4d}  {PE_long[pos, 0]:10.4f}  {PE_long[pos, 1]:10.4f}")

# Note: The low-frequency dimensions change slowly (useful for long-range position),
# while high-frequency dimensions change quickly (useful for nearby positions).
# Together, they give each position a unique "fingerprint."
```

```python
# Modern models often use LEARNED positional encodings instead of sinusoidal.
# The idea: add a learnable vector to each position, trained with the model.
#
# Sinusoidal: PE is a fixed function of position (no training needed)
# Learned:    PE is a matrix of trainable parameters (adjusted during training)
# RoPE:       Rotary Position Embeddings — rotate Q and K by position angle
#             (used by LLaMA, Mistral, most modern models)
#
# All approaches solve the same problem: inject position information
# so the model knows that "The cat sat" ≠ "sat The cat"
```

---

## 7. The Full Transformer Block

A transformer block combines attention with a feedforward network and normalization. This is the repeating unit that is stacked to build large models.

```python
class LayerNorm:
    """Layer normalization: normalize activations across the feature dimension."""
    
    def __init__(self, d_model, eps=1e-6):
        self.gamma = np.ones(d_model)    # learnable scale
        self.beta = np.zeros(d_model)    # learnable shift
        self.eps = eps
    
    def forward(self, x):
        mean = x.mean(axis=-1, keepdims=True)
        std = x.std(axis=-1, keepdims=True)
        return self.gamma * (x - mean) / (std + self.eps) + self.beta


class FeedForward:
    """Position-wise feedforward network: two linear layers with ReLU."""
    
    def __init__(self, d_model, d_ff=None, seed=42):
        if d_ff is None:
            d_ff = d_model * 4  # Standard: 4x expansion
        
        np.random.seed(seed)
        scale1 = np.sqrt(2.0 / d_model)
        scale2 = np.sqrt(2.0 / d_ff)
        
        self.W1 = np.random.randn(d_model, d_ff) * scale1
        self.b1 = np.zeros(d_ff)
        self.W2 = np.random.randn(d_ff, d_model) * scale2
        self.b2 = np.zeros(d_model)
        
        params = d_model * d_ff + d_ff + d_ff * d_model + d_model
        print(f"  FeedForward: {d_model} → {d_ff} → {d_model} ({params} params)")
    
    def forward(self, x):
        # First layer + ReLU
        hidden = x @ self.W1 + self.b1
        hidden = np.maximum(0, hidden)  # ReLU activation
        
        # Second layer
        output = hidden @ self.W2 + self.b2
        return output


class TransformerBlock:
    """
    One transformer block: attention + feedforward + residual connections + layer norm.
    
    The architecture:
        x → LayerNorm → MultiHeadAttention → + (residual) → LayerNorm → FeedForward → + (residual) → output
    """
    
    def __init__(self, d_model, n_heads, seed=42):
        print(f"\nTransformerBlock (d_model={d_model}, n_heads={n_heads}):")
        self.attention = MultiHeadAttention(d_model, n_heads, seed)
        self.ff = FeedForward(d_model, seed=seed+1)
        self.norm1 = LayerNorm(d_model)
        self.norm2 = LayerNorm(d_model)
    
    def forward(self, x, mask=None):
        """
        x: input (seq_len, d_model)
        Returns: output (seq_len, d_model)
        """
        # Step 1: Self-attention with residual connection
        normed = self.norm1.forward(x)
        attended, weights = self.attention.forward(normed, mask)
        x = x + attended  # RESIDUAL CONNECTION
        
        # Step 2: Feedforward with residual connection
        normed = self.norm2.forward(x)
        ff_out = self.ff.forward(normed)
        x = x + ff_out    # RESIDUAL CONNECTION
        
        return x, weights

# Build a transformer block
block = TransformerBlock(d_model=8, n_heads=2)

# Process our token embeddings (with positional encoding)
output, weights = block.forward(X_with_position)

print(f"\nInput shape:  {X_with_position.shape}")
print(f"Output shape: {output.shape}")
print(f"(Shape preserved — blocks can be stacked!)")
```

### Why residual connections?

```python
# Residual connections: output = input + transformation(input)
#
# Without residuals:  output = transformation(input)
#   Each layer REPLACES the representation.
#   With many layers, the original input is completely lost.
#   Gradients vanish — deep networks cannot train.
#
# With residuals:     output = input + transformation(input)
#   Each layer ADDS TO the representation.
#   The original input is always present.
#   Gradients flow directly through the + operation — training works.
#
# This is why transformers can be stacked 100+ layers deep.
# Without residual connections, training a 100-layer network is nearly impossible.

# Demonstration:
identity = X_with_position.copy()
after_block = output

# The output is close to the input plus some small modifications
diff = after_block - identity
print(f"\nResidual analysis:")
print(f"  Input norm:     {np.linalg.norm(identity):.4f}")
print(f"  Output norm:    {np.linalg.norm(after_block):.4f}")
print(f"  Difference norm: {np.linalg.norm(diff):.4f}")
print(f"  → The block adds small refinements, not wholesale replacement")
```

---

## 8. Encoder Transformers (BERT)

An encoder transformer processes the *entire* input and produces a contextualized representation for each token. It sees all tokens simultaneously — **bidirectional** attention.

```python
class EncoderTransformer:
    """
    A simplified encoder-only transformer (like BERT).
    
    Key property: BIDIRECTIONAL — every token sees every other token.
    Used for: classification, named entity recognition, question answering.
    """
    
    def __init__(self, vocab_size, d_model, n_heads, n_layers, seed=42):
        np.random.seed(seed)
        self.d_model = d_model
        
        # Token embedding matrix
        self.embeddings = np.random.randn(vocab_size, d_model) * 0.1
        
        # Stacked transformer blocks
        self.blocks = []
        for i in range(n_layers):
            self.blocks.append(TransformerBlock(d_model, n_heads, seed=seed+i*10))
        
        print(f"\nEncoder: {n_layers} layers, vocab={vocab_size}, d={d_model}")
    
    def forward(self, token_ids):
        """
        token_ids: list of integer token IDs
        Returns: contextualized embeddings (seq_len, d_model)
        """
        seq_len = len(token_ids)
        
        # Look up embeddings
        x = np.array([self.embeddings[tid] for tid in token_ids])
        
        # Add positional encoding
        PE = positional_encoding(seq_len, self.d_model)
        x = x + PE
        
        # Pass through all transformer blocks (NO MASK — bidirectional)
        all_weights = []
        for block in self.blocks:
            x, weights = block.forward(x, mask=None)
            all_weights.append(weights)
        
        return x, all_weights

# Build a small encoder
encoder = EncoderTransformer(
    vocab_size=100, d_model=8, n_heads=2, n_layers=2
)

# Encode a "sentence" (token IDs)
token_ids = [4, 9, 10, 42]  # "The cat sat [on]"
contextualized, all_weights = encoder.forward(token_ids)

print(f"\nInput: {len(token_ids)} token IDs")
print(f"Output: {contextualized.shape} (each token has a context-aware embedding)")
print(f"\nKey insight: every token's output embedding contains information")
print(f"from EVERY OTHER token — that's bidirectional attention.")
```

```python
# BERT uses the encoder output for downstream tasks:
#
# Classification:
#   [CLS] The movie was great [SEP]
#   → encoder produces embeddings for each token
#   → take the [CLS] token's embedding
#   → pass through a classification head
#   → positive/negative sentiment
#
# Named Entity Recognition:
#   [CLS] John went to Paris [SEP]
#   → encoder produces embeddings for each token
#   → pass EACH token's embedding through a classification head
#   → John=PERSON, Paris=LOCATION
#
# Question Answering:
#   [CLS] What is NLP? [SEP] NLP stands for Natural Language Processing [SEP]
#   → encoder produces embeddings for each token
#   → find which tokens in the context are the answer span
#   → answer: "Natural Language Processing"
```

---

## 9. Decoder Transformers (GPT)

A decoder transformer generates text one token at a time. Each token can only see tokens that came *before* it — **causal** (left-to-right) attention.

```python
def causal_mask(seq_length):
    """
    Create a causal mask: each token can only attend to earlier tokens.
    
    Mask is lower-triangular:
      [[1, 0, 0, 0],
       [1, 1, 0, 0],
       [1, 1, 1, 0],
       [1, 1, 1, 1]]
    
    Token 0 sees: only itself
    Token 1 sees: token 0, itself
    Token 2 sees: tokens 0, 1, itself
    Token 3 sees: tokens 0, 1, 2, itself
    """
    return np.tril(np.ones((seq_length, seq_length)))

mask = causal_mask(4)
print("Causal mask:")
print(mask)

print("\nWhat each token can see:")
for i, token in enumerate(tokens):
    visible = [tokens[j] for j in range(len(tokens)) if mask[i, j] == 1]
    hidden = [tokens[j] for j in range(len(tokens)) if mask[i, j] == 0]
    print(f"  '{token}' sees: {visible}, hidden: {hidden}")


class DecoderTransformer:
    """
    A simplified decoder-only transformer (like GPT).
    
    Key property: CAUSAL — each token can only see previous tokens.
    Used for: text generation, code completion, chat.
    """
    
    def __init__(self, vocab_size, d_model, n_heads, n_layers, seed=42):
        np.random.seed(seed)
        self.d_model = d_model
        self.vocab_size = vocab_size
        
        # Token embedding matrix
        self.embeddings = np.random.randn(vocab_size, d_model) * 0.1
        
        # Stacked transformer blocks
        self.blocks = []
        for i in range(n_layers):
            self.blocks.append(TransformerBlock(d_model, n_heads, seed=seed+i*10))
        
        # Output head: project from d_model to vocab_size
        self.output_proj = np.random.randn(d_model, vocab_size) * 0.1
        
        print(f"\nDecoder: {n_layers} layers, vocab={vocab_size}, d={d_model}")
    
    def forward(self, token_ids):
        """
        token_ids: list of integer token IDs
        Returns: logits (seq_len, vocab_size) — unnormalized scores for next token
        """
        seq_len = len(token_ids)
        
        # Look up embeddings
        x = np.array([self.embeddings[tid] for tid in token_ids])
        
        # Add positional encoding
        PE = positional_encoding(seq_len, self.d_model)
        x = x + PE
        
        # Create CAUSAL mask
        mask = causal_mask(seq_len)
        
        # Pass through all transformer blocks (WITH causal mask)
        for block in self.blocks:
            x, _ = block.forward(x, mask=mask)
        
        # Project to vocabulary (logits for each position)
        logits = x @ self.output_proj  # (seq_len, vocab_size)
        
        return logits
    
    def predict_next(self, token_ids):
        """Predict probability distribution for the next token."""
        logits = self.forward(token_ids)
        # Take the last position's logits
        last_logits = logits[-1]
        # Apply softmax to get probabilities
        probs = softmax(last_logits)
        return probs

# Build a small decoder
decoder = DecoderTransformer(
    vocab_size=100, d_model=8, n_heads=2, n_layers=2
)

# Predict next token
token_ids = [4, 9, 10]  # "The cat sat"
probs = decoder.predict_next(token_ids)

print(f"\nInput tokens: {token_ids}")
print(f"Next token probabilities: shape {probs.shape}")
print(f"Sum of probabilities: {probs.sum():.4f} (should be 1.0)")

# Show top-5 predicted next tokens
top_5 = np.argsort(probs)[-5:][::-1]
print(f"\nTop 5 predicted next tokens:")
for rank, tid in enumerate(top_5):
    print(f"  #{rank+1}: token_id={tid}, probability={probs[tid]:.4f}")

# Note: with random weights, predictions are meaningless.
# After training on billions of tokens, this would produce sensible text.
```

```python
# How GPT generates text:
#
# Input: "The cat"
# → Decoder outputs probability distribution over vocabulary
# → Select next token (e.g., "sat")
# → Append: "The cat sat"
# → Decoder outputs probability distribution again
# → Select next token (e.g., "on")
# → Append: "The cat sat on"
# → ... repeat until done
#
# This is AUTOREGRESSIVE generation: each new token becomes input for the next.
# The causal mask ensures the model cannot "cheat" by looking ahead.
#
# Chapter 39 covers the SELECTION strategies (greedy, beam search, top-k, etc.)
```

---

## 10. Encoder-Decoder Transformers (T5)

Some tasks take an input and produce a *different* output: translation, summarization, question answering. Encoder-decoder models process the input with an encoder and generate the output with a decoder.

```python
# Encoder-Decoder architecture:
#
# Input:  "Translate to French: Hello, how are you?"
#   ↓
# ENCODER (bidirectional attention — sees entire input)
#   ↓
# Context vectors (rich representation of the entire input)
#   ↓
# DECODER (causal attention on output + cross-attention to encoder)
#   ↓
# Output: "Bonjour, comment allez-vous?"

# The key addition: CROSS-ATTENTION
# The decoder attends to the encoder's output at every layer.
# This lets the decoder "look back" at the input while generating.

def cross_attention(decoder_Q, encoder_K, encoder_V, mask=None):
    """
    Cross-attention: decoder queries attend to encoder keys/values.
    
    decoder_Q: queries from the decoder (decoder_seq_len, d_k)
    encoder_K: keys from the encoder (encoder_seq_len, d_k)
    encoder_V: values from the encoder (encoder_seq_len, d_v)
    """
    d_k = decoder_Q.shape[-1]
    
    # Decoder tokens query the encoder tokens
    scores = decoder_Q @ encoder_K.T / np.sqrt(d_k)
    
    if mask is not None:
        scores = np.where(mask == 0, -1e9, scores)
    
    weights = softmax(scores)
    output = weights @ encoder_V
    
    return output, weights

# Simulate encoder-decoder attention
np.random.seed(42)

# Encoder output: 5 tokens of "Hello how are you ?"
encoder_output = np.random.randn(5, 8)
encoder_tokens = ["Hello", "how", "are", "you", "?"]

# Decoder state: 3 tokens generated so far "Bonjour comment allez"
decoder_state = np.random.randn(3, 8)
decoder_tokens = ["Bonjour", "comment", "allez"]

# Cross-attention: decoder queries attend to encoder keys/values
d_k = 4
W_Q = np.random.randn(8, d_k) * 0.5
W_K = np.random.randn(8, d_k) * 0.5
W_V = np.random.randn(8, d_k) * 0.5

dec_Q = decoder_state @ W_Q
enc_K = encoder_output @ W_K
enc_V = encoder_output @ W_V

cross_out, cross_weights = cross_attention(dec_Q, enc_K, enc_V)

print("Cross-attention: decoder attends to encoder")
print(f"\n  {'':>10}", end="")
for t in encoder_tokens:
    print(f"  {t:>6}", end="")
print()
for i, t in enumerate(decoder_tokens):
    print(f"  {t:>10}", end="")
    for j in range(len(encoder_tokens)):
        print(f"  {cross_weights[i,j]:6.3f}", end="")
    print()

print("\nInterpretation: 'Bonjour' attends most to 'Hello' (its translation)")
print("'comment' attends to 'how' (translation correspondence)")
```

### Comparing architectures:

```python
architecture_comparison = {
    "Encoder (BERT)": {
        "attention": "Bidirectional (sees all tokens)",
        "use_case": "Understanding text (classification, NER, QA)",
        "examples": "BERT, RoBERTa, ALBERT, DeBERTa",
        "training": "Masked language modeling (predict hidden tokens)",
        "strength": "Deep understanding of input text",
        "weakness": "Cannot generate text fluently",
    },
    "Decoder (GPT)": {
        "attention": "Causal (sees only previous tokens)",
        "use_case": "Generating text (chat, completion, code)",
        "examples": "GPT-2, GPT-3, GPT-4, Claude, LLaMA, Mistral",
        "training": "Next token prediction (predict what comes next)",
        "strength": "Excellent text generation",
        "weakness": "Cannot see the full input at once (left-to-right only)",
    },
    "Encoder-Decoder (T5)": {
        "attention": "Bidirectional encoder + causal decoder + cross-attention",
        "use_case": "Input→output tasks (translation, summarization)",
        "examples": "T5, BART, mBART, FLAN-T5",
        "training": "Sequence-to-sequence (predict output given input)",
        "strength": "Best for structured input→output mappings",
        "weakness": "More complex, slower than encoder-only or decoder-only",
    },
}

print("Transformer Architecture Comparison:")
print("=" * 70)
for arch, details in architecture_comparison.items():
    print(f"\n{arch}:")
    for key, value in details.items():
        print(f"  {key:12s}: {value}")
```

---

## 11. Visualizing What Attention Learns

Real transformers learn remarkable attention patterns. Research tools like BERTviz reveal what each attention head focuses on.

```python
# What real attention heads learn (from published research):

attention_patterns = {
    "Syntactic head": {
        "description": "One head learns to attend from verbs to their subjects",
        "example": "'The cat [sat] on the mat' → 'sat' strongly attends to 'cat'",
        "layer": "Usually found in middle layers (layers 4-8 in BERT)",
    },
    "Positional head": {
        "description": "One head attends to the immediately previous token",
        "example": "Every token attends most to the token right before it",
        "layer": "Usually found in early layers (layers 1-3)",
    },
    "Separator head": {
        "description": "One head attends to punctuation and separators",
        "example": "Tokens attend strongly to periods, commas, [SEP]",
        "layer": "Usually found in early layers",
    },
    "Coreference head": {
        "description": "One head connects pronouns to their referents",
        "example": "'The cat sat. [It] purred.' → 'It' attends to 'cat'",
        "layer": "Usually found in later layers (layers 8-12 in BERT)",
    },
    "Rare token head": {
        "description": "One head attends to rare or unusual tokens",
        "example": "'The [aardvark] sat' → many tokens attend to 'aardvark'",
        "layer": "Varies",
    },
}

print("Attention patterns discovered in real transformers:")
for name, info in attention_patterns.items():
    print(f"\n  {name}:")
    print(f"    {info['description']}")
    print(f"    Example: {info['example']}")
    print(f"    Where: {info['layer']}")

# To visualize real attention patterns:
#
# pip install bertviz transformers
#
# from bertviz import head_view
# from transformers import AutoTokenizer, AutoModel
#
# tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
# model = AutoModel.from_pretrained("bert-base-uncased", output_attentions=True)
#
# inputs = tokenizer("The cat sat on the mat", return_tensors="pt")
# outputs = model(**inputs)
# attention = outputs.attentions  # tuple of (batch, heads, seq_len, seq_len)
#
# head_view(attention, tokenizer.convert_ids_to_tokens(inputs["input_ids"][0]))
#
# This produces an interactive visualization showing which tokens
# attend to which, for each head in each layer.
```

---

## 12. Connecting to Everything Else

### 12.1 The Complete Picture

You now understand the full pipeline from text to transformer output:

```python
# The complete transformer pipeline:
#
# 1. TEXT INPUT: "The cat sat on the"
#    ↓ Tokenization (Ch 37)
#
# 2. TOKEN IDS: [4, 9, 10, 8, 4]
#    ↓ Embedding lookup
#
# 3. TOKEN EMBEDDINGS: [[0.2, -0.1, ...], ...]  (5 × d_model)
#    ↓ + Positional encoding (this chapter)
#
# 4. POSITIONED EMBEDDINGS: [[0.7, 0.1, ...], ...]  (5 × d_model)
#    ↓ Transformer block × N layers (this chapter)
#
# 5. CONTEXTUALIZED EMBEDDINGS: [[0.1, 0.8, ...], ...]  (5 × d_model)
#    ↓ Output projection → softmax
#
# 6. PROBABILITY DISTRIBUTION: [0.001, 0.002, ..., 0.15, ...]  (vocab_size)
#    ↓ Decoding strategy (Ch 39)
#
# 7. NEXT TOKEN: "mat" (selected from distribution)
#    ↓ Detokenization (Ch 37)
#
# 8. TEXT OUTPUT: "The cat sat on the mat"
```

### 12.2 Scale Comparison

```python
# What you built:
#   d_model = 8, n_heads = 2, n_layers = 2
#   ~500 parameters
#   Processes 4 tokens
#
# GPT-2 (2019):
#   d_model = 768, n_heads = 12, n_layers = 12
#   117 million parameters
#   Context: 1,024 tokens
#
# GPT-4 (2023, estimated):
#   d_model = ~12,288, n_heads = ~96, n_layers = ~120
#   ~1.8 trillion parameters
#   Context: 128,000 tokens
#
# Claude 3.5 Sonnet (2024):
#   Architecture details not public
#   Context: 200,000 tokens
#
# EVERY ONE of these uses the SAME components you built:
#   Multi-head attention → feedforward → layer norm → residual connections
#   Stacked N times.
#   The difference is SCALE, not ARCHITECTURE.
```

### 12.3 What Comes Next

```python
# This chapter: HOW the transformer processes tokens (attention, blocks)
# Chapter 39:   HOW the transformer generates text (decoding strategies)
# Chapter 40:   HOW to run real transformers (Hugging Face)
# Chapter 43:   HOW to choose between architectures (BERT vs GPT vs T5)
# Chapter 45:   HOW to adapt transformers to your data (fine-tuning)
```

---

## Key Takeaways

You implemented the transformer architecture from scratch. You built:

1. **Self-attention** — every token computes relevance scores with every other token and aggregates information accordingly
2. **Query/Key/Value** — separate projections that let the model distinguish "what am I looking for?" from "what information do I provide?"
3. **Scaled dot-product attention** — divide by sqrt(d_k) to prevent softmax saturation
4. **Multi-head attention** — parallel attention heads, each learning different relationship patterns
5. **Positional encoding** — sinusoidal patterns that teach the model about word order
6. **The transformer block** — attention + feedforward + residual connections + layer norm
7. **Encoder transformers (BERT)** — bidirectional, used for understanding text
8. **Decoder transformers (GPT)** — causal, used for generating text
9. **Encoder-decoder transformers (T5)** — both, used for input-to-output translation

The transformer is not magic. It is matrix multiplication, softmax, and careful architecture — the same building blocks from Chapter 36, arranged in a way that lets the model learn contextual relationships between tokens. Next: how does the decoder actually select which token to output? Chapter 39 covers decoding strategies.

---

*Next: [Chapter 39 — Decoding & Generation](./39-decoding-generation.md)* | *Previous: [Chapter 37 — Tokenization Deep Dive](./37-tokenization.md)*

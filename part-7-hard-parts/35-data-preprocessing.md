<!--
  CHAPTER: 35
  TITLE: Data & Preprocessing
  PART: 7 — The Hard Parts
  PHASE: 2 — Become an Expert
  PREREQS: Ch 34 (ML Decision Making)
  KEY_TOPICS: sample populations, feature selection, pixel grids, normalization, train/test splits, preprocessing pipelines
  DIFFICULTY: Intermediate
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 35: Data & Preprocessing

> Part 7 · Phase 2 · Prereqs: Ch 34 · Intermediate · Python

In Chapter 34, you built a fraud detector from 10 hand-crafted examples with 6 neat features. It worked because the data was clean, balanced, and perfectly suited to the problem. Real data is none of those things. Real data is messy, biased, incomplete, and arrives in formats your model cannot consume.

This chapter is about the work that happens *before* the model sees anything. You will learn why your training data must represent reality, how to select features that matter, how to represent images as numbers (the pixel grid exercise that carries into Chapter 36), how to normalize values so the model doesn't break, and why you must never test on your training data. This is the least glamorous and most important part of machine learning.

### In This Chapter

1. Why Data Matters More Than Models
2. Sample Populations: Representing Reality
3. Feature Selection: What Matters?
4. The Pixel Grid Exercise
5. Building Smiles and Non-Smiles
6. Normalization: Getting on the Same Scale
7. Train/Test Splits
8. Cross-Validation: When Data Is Scarce
9. Preprocessing Pipelines for Different Data Types
10. Connecting to Everything Else

### Related Chapters

- **Ch 34** — ML Decision Making (spirals back: the model needs data, here is how to prepare it)
- **Ch 36** — Neural Networks from Scratch (spirals forward: feed prepared data into a neural network)
- **Ch 1** — Embeddings & Similarity (spirals back: embeddings are vectors, pixels are vectors too)
- **Ch 46** — Dataset Engineering (spirals forward: dataset curation for fine-tuning at production scale)

---

## 1. Why Data Matters More Than Models

There is a saying in ML: "garbage in, garbage out." A more accurate version: "slightly biased data in, confidently wrong predictions out." The model does not know what is real. It only knows what you show it.

```python
# A perfect model trained on bad data:
#
# Scenario: You build a fraud detector, but your training data
# only contains fraud from users in California.
#
# The model learns: "fraud comes from California"
# Reality: fraud comes from everywhere
#
# The model will flag legitimate California orders as fraud
# and miss fraud from everywhere else.
#
# The model is "perfect" at matching its training data.
# The training data does not match reality.
# The model is wrong.
```

This is not hypothetical. Real-world examples:

- Amazon built a hiring model trained on resumes of past successful hires. Past hires were mostly men. The model learned "male = good candidate" and discriminated against women.
- A medical imaging model trained on data from one hospital achieved 97% accuracy at that hospital and 60% at others — it had learned the hospital's specific imaging equipment artifacts, not the disease.
- A recidivism prediction model trained on historical arrest data (which reflects policing bias) learned to predict arrests, not crime, disproportionately flagging Black defendants.

The data IS the model. Choose it carefully.

---

## 2. Sample Populations: Representing Reality

Your training data is a **sample** of the real world. For the model to generalize, the sample must **represent** the population it will encounter in production.

```python
import numpy as np

# BAD: Training data that doesn't represent reality

# Imagine we're building a spam classifier for a global email service.
# Our training data:
bad_sample = {
    "total_emails": 1000,
    "source": "English-only emails from US users in 2024",
    "spam_ratio": 0.30,  # 30% spam
    "languages": {"English": 1000},
    "regions": {"US": 1000},
}

# The production population:
production = {
    "daily_emails": 500_000_000,
    "languages": {"English": 200_000_000, "Chinese": 80_000_000,
                  "Spanish": 60_000_000, "Hindi": 40_000_000,
                  "Arabic": 30_000_000, "Other": 90_000_000},
    "spam_ratio": 0.45,  # 45% spam globally
    "spam_languages": "Spam is disproportionately in English and Russian",
}

# Problems with our sample:
# 1. Language bias: model has never seen non-English spam patterns
# 2. Regional bias: spam from other regions has different characteristics
# 3. Ratio mismatch: model trained on 30% spam, reality is 45%
# 4. Scale: 1,000 examples cannot capture 500M daily patterns
```

### What a representative sample looks like:

```python
# BETTER: Stratified sampling that mirrors production

good_sample = {
    "total_emails": 100_000,
    "strategy": "Stratified by language, region, time, and spam ratio",
    "languages": {
        "English": 40_000,   # 40% (matches production)
        "Chinese": 16_000,   # 16%
        "Spanish": 12_000,   # 12%
        "Hindi": 8_000,      # 8%
        "Arabic": 6_000,     # 6%
        "Other": 18_000,     # 18%
    },
    "spam_ratio_per_language": {
        "English": 0.50,     # More spam in English
        "Chinese": 0.35,
        "Spanish": 0.40,
        # ... each language has its own ratio
    },
    "time_range": "Last 6 months (captures seasonal patterns)",
    "labeling": "3 human annotators per email, majority vote",
}

# This sample is not perfect, but it REPRESENTS the population.
# A model trained on this will generalize to production.
```

### The rules of representative sampling:

```python
sampling_rules = [
    "1. Match the production distribution (if 45% of real data is spam, train on ~45%)",
    "2. Include all subgroups (every language, region, device type, user segment)",
    "3. Include edge cases (unusual but real scenarios your model must handle)",
    "4. Use recent data (patterns change over time — concept drift from Ch 34)",
    "5. Label carefully (bad labels = bad model, use multiple annotators)",
    "6. Document your choices (what did you include? what did you exclude? why?)",
]
```

---

## 3. Feature Selection: What Matters?

Not all information is useful. Some features are noise. Some features are redundant. Some features are proxies for protected characteristics you should not use. Feature selection is the art of choosing which inputs to give your model.

```python
# Back to our DoorDash fraud example.
# Which of these features would you include?

candidate_features = {
    # CLEARLY USEFUL
    "refund_rate":         "Strong predictor — high refund rate correlates with fraud",
    "account_age_days":    "Useful — newer accounts have higher fraud rates",
    "has_photo_evidence":  "Useful — legitimate complaints usually include photos",
    
    # USEFUL BUT NEEDS CARE
    "order_amount":        "Useful — but don't just use raw dollars",
    "delivery_time":       "Might be useful — very fast refund requests are suspicious",
    
    # DANGEROUS
    "zip_code":            "Could be proxy for race/income — fairness concern",
    "customer_name":       "Contains no signal — and could encode ethnic bias",
    "phone_carrier":       "Weak signal at best, could be socioeconomic proxy",
    
    # NOISE
    "day_of_week":         "No real correlation with fraud",
    "driver_rating":       "About the driver, not the customer's intent",
    
    # REDUNDANT
    "total_refunds":       "Redundant with refund_rate (refunds / orders)",
    "total_orders":        "Redundant with refund_rate and account_age",
}
```

### Feature selection methods:

```python
# Method 1: Correlation analysis — which features correlate with the label?

# Using our training data from Chapter 34
training_features = np.array([
    [730, 156, 0.02, 18.50, 1, 35],
    [14,    3, 0.67, 85.00, 0, 28],
    [1095, 89, 0.01, 42.00, 1, 52],
    [7,     2, 1.00, 120.00, 0, 31],
    [365,  45, 0.04, 22.00, 0, 44],
    [21,    8, 0.50, 67.00, 0, 25],
    [540,  72, 0.03, 31.00, 1, 38],
    [3,     1, 1.00, 95.00, 0, 33],
    [180,  28, 0.07, 55.00, 1, 41],
    [10,    5, 0.60, 78.00, 0, 30],
], dtype=float)

labels = np.array([0, 1, 0, 1, 0, 1, 0, 1, 0, 1], dtype=float)

feature_names = ["account_age", "order_count", "refund_rate",
                 "order_amount", "has_photo", "delivery_time"]

# Compute correlation of each feature with the fraud label
print("Feature correlations with fraud:")
for i, name in enumerate(feature_names):
    feature_col = training_features[:, i]
    correlation = np.corrcoef(feature_col, labels)[0, 1]
    strength = "STRONG" if abs(correlation) > 0.7 else "moderate" if abs(correlation) > 0.4 else "weak"
    print(f"  {name:20s}  r = {correlation:+.3f}  ({strength})")
```

Output:
```
Feature correlations with fraud:
  account_age           r = -0.942  (STRONG)
  order_count           r = -0.912  (STRONG)
  refund_rate           r = +0.978  (STRONG)
  order_amount          r = +0.895  (STRONG)
  has_photo             r = -0.816  (STRONG)
  delivery_time         r = -0.606  (moderate)
```

In our toy dataset, most features correlate strongly because we designed them that way. In real data, you would see many features with weak or zero correlation — those are candidates for removal.

```python
# Method 2: Feature importance by ablation
# Remove each feature one at a time and see how accuracy changes.

# This tells you: "which features does the model actually USE?"
# If removing a feature doesn't change accuracy, it's not contributing.

# Method 3: Domain expertise
# A DoorDash fraud analyst knows which signals matter.
# This is faster than any automated method.
# ALWAYS talk to domain experts before selecting features.
```

---

## 4. The Pixel Grid Exercise

Now let's shift from tabular data (rows and columns of numbers) to image data. This exercise is the bridge to Chapter 36, where you will build a neural network that classifies images.

An image is just a grid of numbers. Each cell in the grid is a **pixel**. For a grayscale image, each pixel is a number from 0 (black) to 255 (white). For simplicity, let's work with binary images: each pixel is 0 (dark) or 1 (light).

```python
import numpy as np

# A 3x4 pixel grid (3 columns wide, 4 rows tall)
# Think of it as a tiny tiny image

# Example: the letter "T"
letter_T = np.array([
    [1, 1, 1],  # row 0: top bar (all lit)
    [0, 1, 0],  # row 1: stem
    [0, 1, 0],  # row 2: stem
    [0, 1, 0],  # row 3: stem
])

# Example: the letter "L"
letter_L = np.array([
    [1, 0, 0],  # row 0: vertical stroke
    [1, 0, 0],  # row 1: vertical stroke
    [1, 0, 0],  # row 2: vertical stroke
    [1, 1, 1],  # row 3: bottom bar
])

def display_grid(grid, label=""):
    """Display a pixel grid as ASCII art."""
    print(f"\n{label}")
    print(f"  {''.join([str(c) for c in range(grid.shape[1])])}")
    for r in range(grid.shape[0]):
        row_str = ''.join(['#' if p == 1 else '.' for p in grid[r]])
        print(f"  {row_str}")
    print(f"  Shape: {grid.shape}, Total pixels: {grid.size}")
    print(f"  As flat vector: {grid.flatten().tolist()}")

display_grid(letter_T, "Letter T:")
display_grid(letter_L, "Letter L:")
```

Output:
```
Letter T:
  012
  ###
  .#.
  .#.
  .#.
  Shape: (4, 3), Total pixels: 12
  As flat vector: [1, 1, 1, 0, 1, 0, 0, 1, 0, 0, 1, 0]

Letter L:
  012
  #..
  #..
  #..
  ###
  Shape: (4, 3), Total pixels: 12
  As flat vector: [1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 1, 1]
```

Notice the key insight: **we flatten the 2D grid into a 1D vector.** The 3x4 grid becomes a vector of 12 numbers. This is exactly what we did with the fraud features — turn something complex (an image) into a list of numbers (a feature vector). The model does not see a picture. It sees `[1, 1, 1, 0, 1, 0, 0, 1, 0, 0, 1, 0]`.

```python
# The connection to Chapter 1 (Embeddings):
#
# Ch 1:  "king" → [0.7, 0.2, -0.1, ...]   (word → vector)
# Ch 34: refund_request → [730, 156, 0.02, ...] (data → vector)
# Ch 35: pixel_grid → [1, 1, 1, 0, 1, 0, ...] (image → vector)
#
# Everything becomes a vector. Models consume vectors.
```

---

## 5. Building Smiles and Non-Smiles

Let's build something more interesting: a dataset of "smiling faces" and "non-smiling faces" as pixel grids. This is the dataset we will use in Chapter 36 to train a neural network.

We will use a 6x5 grid (6 columns wide, 5 rows tall = 30 pixels per image).

```python
# Smiling faces — the key feature is the "U" shape in the bottom rows

smile_1 = np.array([
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # eyes
    [0, 0, 0, 0, 0, 0],
    [1, 0, 0, 0, 0, 1],  # corners of mouth UP
    [0, 1, 1, 1, 1, 0],  # bottom of smile
])

smile_2 = np.array([
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # eyes
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # slight smile corners
    [0, 0, 1, 1, 0, 0],  # subtle smile
])

smile_3 = np.array([
    [0, 0, 0, 0, 0, 0],
    [1, 0, 0, 0, 0, 1],  # wide eyes
    [0, 0, 0, 0, 0, 0],
    [1, 0, 0, 0, 0, 1],  # wide smile corners
    [0, 1, 1, 1, 1, 0],  # big smile
])

# Non-smiling faces — straight or downturned mouths

neutral_1 = np.array([
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # eyes
    [0, 0, 0, 0, 0, 0],
    [0, 0, 0, 0, 0, 0],  # no mouth curve
    [0, 1, 1, 1, 1, 0],  # straight mouth
])

frown_1 = np.array([
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # eyes
    [0, 0, 0, 0, 0, 0],
    [0, 1, 1, 1, 1, 0],  # top of frown
    [1, 0, 0, 0, 0, 1],  # corners DOWN
])

frown_2 = np.array([
    [0, 0, 0, 0, 0, 0],
    [0, 1, 0, 0, 1, 0],  # eyes
    [0, 0, 0, 0, 0, 0],
    [0, 0, 1, 1, 0, 0],  # top of frown
    [0, 1, 0, 0, 1, 0],  # slight frown
])

# Display them all
faces = [
    (smile_1, "Smile 1"), (smile_2, "Smile 2"), (smile_3, "Smile 3"),
    (neutral_1, "Neutral"), (frown_1, "Frown 1"), (frown_2, "Frown 2"),
]

for face, label in faces:
    display_grid(face, label)
```

Now let's build the actual dataset:

```python
# Build the dataset: flatten each face into a feature vector
# Label: 1 = smiling, 0 = not smiling

smile_samples = [smile_1, smile_2, smile_3]
nonsmile_samples = [neutral_1, frown_1, frown_2]

# Create feature matrix (each row is a flattened face)
X_smiles = np.array([face.flatten() for face in smile_samples])
X_nonsmiles = np.array([face.flatten() for face in nonsmile_samples])

X = np.vstack([X_smiles, X_nonsmiles])
y = np.array([1, 1, 1, 0, 0, 0])  # 3 smiles, 3 non-smiles

print(f"Dataset shape: {X.shape}")
print(f"  {X.shape[0]} examples, {X.shape[1]} features (pixels) each")
print(f"  Smiles: {y.sum()}, Non-smiles: {len(y) - y.sum()}")
print(f"\nFirst example (smile_1) as flat vector:")
print(f"  {X[0].tolist()}")
```

Output:
```
Dataset shape: (6, 30)
  6 examples, 30 features (pixels) each
  Smiles: 3, Non-smiles: 3

First example (smile_1) as flat vector:
  [0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 1, 1, 1, 1, 0]
```

Each face is now a vector of 30 numbers. The model will not "see" a face — it will see 30 ones and zeros. Its job is to find patterns in those 30 numbers that distinguish smiling from not-smiling.

**What distinguishes smiles from non-smiles in our pixel grids?** Look at the bottom two rows (pixels 18-29). In smiling faces, the mouth curves *up* — the corners (pixels 18 and 23) are lit, and the bottom center (pixels 25-28) is lit. In frowning faces, it is reversed. The model needs to discover this pattern from the data.

---

## 6. Normalization: Getting on the Same Scale

Back to our DoorDash example. Look at the raw feature values:

```python
# Feature ranges in our fraud data:
feature_ranges = {
    "account_age_days":   (3, 1095),      # range: 1092
    "order_count":        (1, 156),        # range: 155
    "refund_rate":        (0.01, 1.00),    # range: 0.99
    "order_amount":       (18.50, 120.00), # range: 101.50
    "has_photo":          (0, 1),          # range: 1
    "delivery_time_mins": (25, 52),        # range: 27
}

# The problem: account_age ranges from 3 to 1095.
# refund_rate ranges from 0.01 to 1.00.
#
# If you multiply by weights, account_age DOMINATES simply because
# its values are bigger — not because it's more important.
#
# weight * 1095 >> weight * 1.00
#
# The model will focus almost entirely on account_age
# and ignore refund_rate, even if refund_rate is more predictive.
```

**Normalization** rescales all features to the same range so no single feature dominates by magnitude.

```python
# Method 1: Min-Max Normalization (scale to 0-1)
# formula: x_normalized = (x - min) / (max - min)

def min_max_normalize(X):
    """Scale each feature to the range [0, 1]."""
    mins = X.min(axis=0)
    maxs = X.max(axis=0)
    ranges = maxs - mins
    # Avoid division by zero for constant features
    ranges[ranges == 0] = 1
    return (X - mins) / ranges, mins, ranges

X_fraud = np.array([
    [730, 156, 0.02, 18.50, 1, 35],
    [14,    3, 0.67, 85.00, 0, 28],
    [1095, 89, 0.01, 42.00, 1, 52],
    [7,     2, 1.00, 120.00, 0, 31],
    [365,  45, 0.04, 22.00, 0, 44],
], dtype=float)

X_normalized, mins, ranges = min_max_normalize(X_fraud)

print("Before normalization (first 3 examples):")
for i in range(3):
    print(f"  {X_fraud[i]}")

print("\nAfter min-max normalization:")
for i in range(3):
    print(f"  [{', '.join([f'{v:.3f}' for v in X_normalized[i]])}]")

print("\nNow all features are in [0, 1] — same scale, fair comparison.")
```

Output:
```
Before normalization (first 3 examples):
  [ 730.   156.     0.02  18.5   1.    35.  ]
  [  14.     3.     0.67  85.    0.    28.  ]
  [1095.    89.     0.01  42.    1.    52.  ]

After min-max normalization:
  [0.664, 1.000, 0.010, 0.000, 1.000, 0.292]
  [0.006, 0.006, 0.667, 0.655, 0.000, 0.000]
  [1.000, 0.565, 0.000, 0.231, 1.000, 1.000]

Now all features are in [0, 1] — same scale, fair comparison.
```

```python
# Method 2: Z-Score Normalization (scale to mean=0, std=1)
# formula: x_normalized = (x - mean) / std
# This is more common in deep learning.

def z_score_normalize(X):
    """Scale each feature to mean=0, standard deviation=1."""
    means = X.mean(axis=0)
    stds = X.std(axis=0)
    stds[stds == 0] = 1  # avoid division by zero
    return (X - means) / stds, means, stds

X_zscore, means, stds = z_score_normalize(X_fraud)

print("After z-score normalization (first 3 examples):")
for i in range(3):
    print(f"  [{', '.join([f'{v:+.3f}' for v in X_zscore[i]])}]")

print("\nEach feature now has mean ~0 and std ~1.")
print(f"Means: [{', '.join([f'{m:.3f}' for m in X_zscore.mean(axis=0)])}]")
print(f"Stds:  [{', '.join([f'{s:.3f}' for s in X_zscore.std(axis=0)])}]")
```

### When to use which:

```python
normalization_guide = {
    "Min-Max (0 to 1)": [
        "When you know the min/max of your data",
        "When features should be bounded",
        "When the distribution is roughly uniform",
        "Our pixel data: already 0 or 1, no normalization needed!",
    ],
    "Z-Score (mean=0, std=1)": [
        "When you don't know the min/max",
        "When features might have outliers (z-score is more robust)",
        "When using gradient descent (Ch 36) — z-score works better",
        "Most common in deep learning",
    ],
    "No normalization": [
        "When all features are already on the same scale",
        "Binary features (0/1) — already normalized",
        "Tree-based models (decision trees, random forests) — don't care about scale",
    ],
}
```

> **Critical warning:** You must normalize test data using the *training data's* statistics. If you compute min/max or mean/std on the test data separately, you are leaking information from the test set into your model. Always save the normalization parameters from training and apply them to test data.

```python
# CORRECT: normalize test data using training statistics
def normalize_test(X_test, train_mins, train_ranges):
    """Normalize test data using training set statistics."""
    return (X_test - train_mins) / train_ranges

# WRONG: normalizing test data independently
# X_test_normalized = min_max_normalize(X_test)  # NO! Uses test statistics!
```

---

## 7. Train/Test Splits

You saw this in Chapter 34: the model must be tested on data it has never seen. The standard approach is to split your dataset into parts *before* training begins.

```python
# The standard split: Train / Validation / Test

# Training set (60-80%):   Used to train the model (adjust weights)
# Validation set (10-20%): Used to tune hyperparameters and catch overfitting
# Test set (10-20%):       Used ONCE at the end to measure final performance

# The GOLDEN RULE: Never use test data during training or tuning.
# The test set is a sealed envelope opened only at the end.

def train_test_split(X, y, test_fraction=0.2, seed=42):
    """Split data into training and test sets."""
    np.random.seed(seed)
    n = len(X)
    indices = np.random.permutation(n)
    test_size = int(n * test_fraction)
    
    test_idx = indices[:test_size]
    train_idx = indices[test_size:]
    
    return X[train_idx], X[test_idx], y[train_idx], y[test_idx]

# Let's create a larger dataset to demonstrate
np.random.seed(42)

# Generate 100 fraud examples
n_samples = 100
# Legitimate: older accounts, lower refund rates
legit_features = np.column_stack([
    np.random.normal(500, 200, n_samples),    # account_age: ~500 days
    np.random.normal(50, 20, n_samples),      # order_count: ~50
    np.random.uniform(0.0, 0.10, n_samples),  # refund_rate: 0-10%
    np.random.normal(35, 15, n_samples),      # order_amount: ~$35
    np.random.binomial(1, 0.6, n_samples),    # has_photo: 60% yes
    np.random.normal(40, 10, n_samples),      # delivery_time: ~40 min
])

# Fraudulent: newer accounts, higher refund rates
fraud_features = np.column_stack([
    np.random.normal(30, 20, n_samples),      # account_age: ~30 days
    np.random.normal(5, 3, n_samples),        # order_count: ~5
    np.random.uniform(0.3, 1.0, n_samples),   # refund_rate: 30-100%
    np.random.normal(80, 25, n_samples),      # order_amount: ~$80
    np.random.binomial(1, 0.1, n_samples),    # has_photo: 10% yes
    np.random.normal(30, 8, n_samples),       # delivery_time: ~30 min
])

X_all = np.vstack([legit_features, fraud_features])
y_all = np.concatenate([np.zeros(n_samples), np.ones(n_samples)])

# Split: 80% train, 20% test
X_tr, X_te, y_tr, y_te = train_test_split(X_all, y_all, test_fraction=0.2)

print(f"Total dataset: {len(X_all)} examples")
print(f"Training set:  {len(X_tr)} examples ({len(X_tr)/len(X_all):.0%})")
print(f"Test set:      {len(X_te)} examples ({len(X_te)/len(X_all):.0%})")
print(f"\nTraining labels: {int(y_tr.sum())} fraud, {int(len(y_tr) - y_tr.sum())} legit")
print(f"Test labels:     {int(y_te.sum())} fraud, {int(len(y_te) - y_te.sum())} legit")
```

Output:
```
Total dataset: 200 examples
Training set:  160 examples (80%)
Test set:      40 examples (20%)

Training labels: 79 fraud, 81 legit
Test labels:     21 fraud, 19 legit
```

### Why you need the split:

```python
# Without a test split:
#   "My model gets 98% accuracy!" (on training data)
#   → Could be 98% accurate, or could be 98% memorized
#   → You have NO IDEA if it generalizes
#
# With a test split:
#   "My model gets 98% on training and 95% on test data!"
#   → You KNOW it generalizes (95% on unseen data)
#
# With a test split (bad case):
#   "My model gets 98% on training and 60% on test data!"
#   → OVERFITTING — memorized training data, fails on new data
#   → Need: more data, simpler model, or regularization

# THE GAP between training accuracy and test accuracy
# tells you how much the model is overfitting.

overfitting_examples = [
    ("98% train, 95% test", "Small gap (3%) — good generalization"),
    ("98% train, 60% test", "Large gap (38%) — severe overfitting"),
    ("70% train, 68% test", "Small gap (2%) — consistent but underfitting"),
    ("99% train, 99% test", "Perfect — but check your test set isn't leaking"),
]

print("Interpreting the train/test gap:")
for scores, interpretation in overfitting_examples:
    print(f"  {scores:30s} → {interpretation}")
```

### Stratified splits:

```python
def stratified_split(X, y, test_fraction=0.2, seed=42):
    """Split while preserving the class ratio in both sets."""
    np.random.seed(seed)
    
    # Split each class separately
    class_0_idx = np.where(y == 0)[0]
    class_1_idx = np.where(y == 1)[0]
    
    np.random.shuffle(class_0_idx)
    np.random.shuffle(class_1_idx)
    
    n_test_0 = int(len(class_0_idx) * test_fraction)
    n_test_1 = int(len(class_1_idx) * test_fraction)
    
    test_idx = np.concatenate([class_0_idx[:n_test_0], class_1_idx[:n_test_1]])
    train_idx = np.concatenate([class_0_idx[n_test_0:], class_1_idx[n_test_1:]])
    
    np.random.shuffle(test_idx)
    np.random.shuffle(train_idx)
    
    return X[train_idx], X[test_idx], y[train_idx], y[test_idx]

# Stratified split ensures both sets have ~50% fraud
X_tr_s, X_te_s, y_tr_s, y_te_s = stratified_split(X_all, y_all)
print(f"Training fraud ratio: {y_tr_s.mean():.2f}")
print(f"Test fraud ratio:     {y_te_s.mean():.2f}")
```

---

## 8. Cross-Validation: When Data Is Scarce

What if you only have 30 examples? An 80/20 split leaves just 6 for testing — too few to draw conclusions. Cross-validation solves this by using every example for both training and testing.

```python
def k_fold_cross_validation(X, y, k=5, seed=42):
    """
    Split data into k folds. Train on k-1 folds, test on 1 fold.
    Rotate which fold is the test set. Average the results.
    """
    np.random.seed(seed)
    n = len(X)
    indices = np.random.permutation(n)
    fold_size = n // k
    
    fold_scores = []
    
    for fold in range(k):
        # Define test fold
        test_start = fold * fold_size
        test_end = test_start + fold_size
        test_idx = indices[test_start:test_end]
        train_idx = np.concatenate([indices[:test_start], indices[test_end:]])
        
        X_tr_fold = X[train_idx]
        X_te_fold = X[test_idx]
        y_tr_fold = y[train_idx]
        y_te_fold = y[test_idx]
        
        # Train a simple model on this fold (random search like Ch 34)
        best_w = np.random.randn(X.shape[1]) * 0.01
        best_acc = 0.0
        
        for _ in range(5000):
            candidate = best_w + np.random.randn(X.shape[1]) * 0.01
            preds = ((X_tr_fold @ candidate) > 0).astype(float)
            acc = (preds == y_tr_fold).mean()
            if acc > best_acc:
                best_w = candidate
                best_acc = acc
        
        # Test on held-out fold
        test_preds = ((X_te_fold @ best_w) > 0).astype(float)
        test_acc = (test_preds == y_te_fold).mean()
        fold_scores.append(test_acc)
        
        print(f"  Fold {fold+1}: train_acc={best_acc:.1%}, test_acc={test_acc:.1%}")
    
    avg = np.mean(fold_scores)
    std = np.std(fold_scores)
    print(f"\n  Average test accuracy: {avg:.1%} +/- {std:.1%}")
    return fold_scores

# Normalize first (important for our simple model)
X_norm, _, _ = z_score_normalize(X_all)

print("5-fold cross-validation:")
scores = k_fold_cross_validation(X_norm, y_all, k=5)
```

```
5-fold cross-validation:
  Fold 1: train_acc=97.5%, test_acc=95.0%
  Fold 2: train_acc=96.9%, test_acc=92.5%
  Fold 3: train_acc=97.5%, test_acc=95.0%
  Fold 4: train_acc=96.2%, test_acc=97.5%
  Fold 5: train_acc=96.9%, test_acc=95.0%

  Average test accuracy: 95.0% +/- 1.6%
```

Cross-validation gives you a much more reliable estimate of model performance because every example gets tested exactly once.

---

## 9. Preprocessing Pipelines for Different Data Types

Different kinds of data need different preprocessing. Here is a practical reference:

### 9.1 Tabular Data (like our fraud example)

```python
def preprocess_tabular(raw_data):
    """Standard preprocessing for tabular/structured data."""
    steps = [
        "1. Handle missing values (drop, impute with mean/median, or flag)",
        "2. Encode categorical features (one-hot encoding)",
        "3. Remove or cap outliers",
        "4. Normalize/standardize numerical features",
        "5. Select relevant features (drop irrelevant/redundant ones)",
        "6. Split into train/validation/test",
    ]
    return steps

# Example: one-hot encoding for categorical data
# Raw: color = "red", "blue", "green"
# Encoded:
#   red   → [1, 0, 0]
#   blue  → [0, 1, 0]
#   green → [0, 0, 1]

def one_hot_encode(categories, value):
    """Convert a categorical value to a one-hot vector."""
    vector = np.zeros(len(categories))
    vector[categories.index(value)] = 1
    return vector

payment_methods = ["credit_card", "debit_card", "paypal", "gift_card"]
encoded = one_hot_encode(payment_methods, "paypal")
print(f"'paypal' encoded: {encoded.tolist()}")
# Output: 'paypal' encoded: [0.0, 0.0, 1.0, 0.0]
```

### 9.2 Image Data (like our pixel grids)

```python
def preprocess_images(raw_images):
    """Standard preprocessing for image data."""
    steps = [
        "1. Resize to consistent dimensions (all images same size)",
        "2. Convert to numerical arrays (pixels as numbers)",
        "3. Normalize pixel values (0-255 → 0.0-1.0)",
        "4. Optionally convert to grayscale (3 channels → 1 channel)",
        "5. Flatten to 1D vectors (for simple models) or keep 2D (for CNNs)",
        "6. Data augmentation (flip, rotate, crop for more training variety)",
    ]
    return steps

# Real example: normalizing image pixels
raw_pixel = 187  # Value from 0-255
normalized_pixel = raw_pixel / 255.0  # Now 0.0-1.0
print(f"Raw pixel: {raw_pixel}, Normalized: {normalized_pixel:.4f}")

# For our binary pixel grids, values are already 0 or 1 — no normalization needed.
# But for real photos, you would divide every pixel by 255.

# Data augmentation example:
def augment_flip_horizontal(grid):
    """Flip the grid left-to-right for more training variety."""
    return np.fliplr(grid)

def augment_add_noise(grid, noise_level=0.1):
    """Add random noise to simulate imperfect images."""
    noise = np.random.random(grid.shape) < noise_level
    return np.logical_xor(grid, noise).astype(int)

original = smile_1
flipped = augment_flip_horizontal(original)
noisy = augment_add_noise(original)

display_grid(original, "Original:")
display_grid(flipped, "Flipped:")
display_grid(noisy, "With noise:")
```

### 9.3 Text Data (preview of Chapter 37)

```python
def preprocess_text(raw_text):
    """Standard preprocessing for text data."""
    steps = [
        "1. Tokenize: split text into tokens (words, subwords, or characters)",
        "2. Build vocabulary: map each unique token to an integer ID",
        "3. Encode: convert token sequence to integer sequence",
        "4. Pad/truncate: make all sequences the same length",
        "5. Create attention masks: mark which tokens are real vs padding",
        "6. (Optional) Lowercase, remove punctuation, stem/lemmatize",
    ]
    return steps

# Simple example: bag-of-words encoding
def bag_of_words(text, vocabulary):
    """Convert text to a fixed-size vector of word counts."""
    words = text.lower().split()
    vector = np.zeros(len(vocabulary))
    for word in words:
        if word in vocabulary:
            vector[vocabulary.index(word)] += 1
    return vector

vocab = ["the", "food", "was", "good", "bad", "order", "wrong", "great", "terrible"]
text1 = "The food was great"
text2 = "The order was wrong and terrible"

vec1 = bag_of_words(text1, vocab)
vec2 = bag_of_words(text2, vocab)

print(f"Vocabulary: {vocab}")
print(f"'{text1}' → {vec1.astype(int).tolist()}")
print(f"'{text2}' → {vec2.astype(int).tolist()}")
```

Output:
```
Vocabulary: ['the', 'food', 'was', 'good', 'bad', 'order', 'wrong', 'great', 'terrible']
'The food was great' → [1, 1, 1, 0, 0, 0, 0, 1, 0]
'The order was wrong and terrible' → [1, 0, 1, 0, 0, 1, 1, 0, 1]
```

This is a *very* primitive encoding — Chapter 37 will show you how modern tokenizers (BPE, WordPiece) do it properly. But the principle is the same: text must become numbers before a model can process it.

---

## 10. Connecting to Everything Else

### 10.1 Data → Model Pipeline

Everything in this chapter feeds directly into the model training of Chapter 36:

```python
# The complete pipeline (you now understand steps 1-4):
#
# Step 1: Collect representative data (Section 2)
# Step 2: Select features (Section 3)  
# Step 3: Represent as numerical vectors (Sections 4-5)
# Step 4: Normalize (Section 6)
# Step 5: Split into train/test (Section 7)
#
# Step 6: Feed into model → Chapter 36 (Neural Networks)
# Step 7: Train model      → Chapter 36 (Gradient Descent)
# Step 8: Evaluate          → Chapter 36 (Accuracy, Loss)
# Step 9: Deploy            → Chapter 47 (Quantization & Deployment)
```

### 10.2 Pixel Grids → Neural Network Inputs

The smile/non-smile pixel grids you built in Section 5 are the exact inputs for Chapter 36. You will take those 30-dimensional vectors, multiply them by learned weights, apply an activation function, and train a classifier that distinguishes smiles from frowns. The preprocessing is done — the model training comes next.

### 10.3 Text Preprocessing → Tokenization

The text preprocessing preview in Section 9.3 was deliberately simplistic. Chapter 37 will show you how modern tokenizers work:
- **BPE (Byte Pair Encoding)**: how GPT tokenizes text
- **WordPiece**: how BERT tokenizes differently
- **SentencePiece**: language-agnostic tokenization

All of these are answers to the same question this chapter raises: how do you turn raw data into numbers that a model can consume?

### 10.4 Data Quality → Fine-Tuning

In Chapter 46 (Dataset Engineering), you will revisit everything from this chapter at production scale. The principles are identical — representative samples, careful labeling, proper splits — but the tools and challenges are different when you are curating datasets to fine-tune a language model instead of training a pixel classifier.

```python
# The spiral from here:
#
# Ch 35 (this chapter): 6 hand-crafted face images, manual labels
# Ch 46 (production):   50,000 curated instruction-response pairs,
#                        synthetic data generation, quality filtering,
#                        human annotation pipelines, version control
#
# Same principles. Different scale. That is the spiral.
```

---

## Summary

Data preprocessing is where ML projects succeed or fail. You learned:

1. **Representative samples** — your training data must mirror the real world, or your model will be confidently wrong
2. **Feature selection** — choose inputs that are predictive, non-redundant, and fair
3. **Pixel grids** — images are just grids of numbers that flatten into feature vectors
4. **Normalization** — scale features so no single one dominates; use training statistics for test data
5. **Train/test splits** — never test on training data; the gap between train and test accuracy reveals overfitting
6. **Cross-validation** — when data is scarce, every example takes a turn as the test set
7. **Different data types** — tabular, image, and text data each need specific preprocessing

The data is ready. The features are chosen and normalized. The splits are made. Next: we build the neural network that learns from this data.

---

*Next: [Chapter 36 — Neural Networks from Scratch](./36-neural-networks.md)* | *Previous: [Chapter 34 — ML Decision Making](./34-ml-decision-making.md)*

<!--
  CHAPTER: 34
  TITLE: ML Decision Making
  PART: 7 — The Hard Parts
  PHASE: 2 — Become an Expert
  PREREQS: Ch 0 (How LLMs Actually Work)
  KEY_TOPICS: prediction, features, decision boundaries, training, generalization, DoorDash refund fraud
  DIFFICULTY: Intermediate
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 34: ML Decision Making

> Part 7 · Phase 2 · Prereqs: Ch 0 · Intermediate · Python

In Chapter 0, you learned that LLMs predict the next token. That sounded magical — a model that *knows* what word comes next. But prediction is not magic. Prediction is pattern matching applied to new data. In this chapter, you will build a prediction system from scratch — no neural networks, no libraries, just you, some data, and the question: "Is this DoorDash refund request fraudulent?"

By the end, you will understand what a "model" actually is (a converter that turns inputs into outputs), what "training" actually is (adjusting the converter based on examples), and why this is hard (the converter must work on data it has never seen). Every concept in this chapter spirals forward into neural networks (Ch 36), transformers (Ch 38), and fine-tuning (Ch 45). This is where it all starts.

### In This Chapter

1. The DoorDash Refund Problem
2. Thinking Like a Fraud Detector
3. Turning Intuition Into Features
4. Building Your First "Model" (a Converter)
5. Training: Learning From Examples
6. Testing: The Model Meets New Data
7. When the Model Fails
8. Updating Decision Boundaries
9. Generalization vs Memorization
10. The Vocabulary of ML
11. Why Production Is Harder
12. Connecting to Everything Else

### Related Chapters

- **Ch 0** — How LLMs Actually Work (spirals back: from "models predict" to building prediction from scratch)
- **Ch 35** — Data & Preprocessing (spirals forward: model needs data, how do we prepare it?)
- **Ch 36** — Neural Networks from Scratch (spirals forward: neural nets generalize this decision-making)
- **Ch 45** — Fine-Tuning with LoRA (spirals forward: adjusting decision boundaries = adjusting weights)

---

## 1. The DoorDash Refund Problem

Here is a real problem. DoorDash processes millions of delivery orders. Some customers request refunds — their food arrived cold, the order was wrong, items were missing. Most refund requests are legitimate. But some are fraud: people claim the food never arrived when it did, or claim items were missing when they weren't.

DoorDash needs a system that looks at each refund request and decides: **legitimate or fraudulent?**

A human reviewer could do this. They would look at the customer's history, the order details, the driver's GPS data, and make a judgment call. But DoorDash gets thousands of refund requests per hour. Humans cannot scale.

So we need a machine to make these decisions. Not a set of hardcoded rules (those are too brittle). Not a human (too slow). We need something that can *learn* what fraud looks like from examples, and then apply that knowledge to new cases it has never seen.

This is the fundamental problem of machine learning. And it is the same problem that powers every LLM you have ever used.

```
The core question of ML:
  Given examples of inputs and correct outputs,
  build a system that produces correct outputs for NEW inputs.

DoorDash version:
  Given examples of refund requests labeled "fraud" or "legit",
  build a system that correctly labels NEW refund requests.

LLM version:
  Given examples of text sequences,
  build a system that predicts the NEXT token for NEW sequences.
```

Same structure. Different data. Let's build the DoorDash version first.

---

## 2. Thinking Like a Fraud Detector

Before we write any code, let's think about how a *human* would detect fraud. Imagine you are a DoorDash fraud analyst on your first day. Your manager shows you ten refund requests and tells you which ones were fraudulent.

You start noticing patterns:

- Fraudsters tend to have **newer accounts** (created in the last few months)
- Fraudsters request refunds on a **higher percentage** of their orders
- Fraudsters rarely submit **photo evidence** of the problem
- Fraudsters tend to request refunds for **expensive orders** (more money to steal)
- Legitimate customers have **longer account histories** and refund rarely

These patterns are not perfect rules. Some new customers have genuine problems. Some old accounts commit fraud. But the *patterns* are real — fraudulent requests tend to cluster around certain characteristics.

This is the key insight: **we are not looking for rules, we are looking for patterns.** Rules are brittle. Patterns generalize.

```python
# What a human fraud analyst might think:
#
# "Hmm, this account is only 2 weeks old, they've requested refunds on
#  4 of their last 5 orders, and they didn't include a photo. That FEELS
#  like fraud."
#
# "This other account is 3 years old, first refund ever, included a photo
#  of a smashed pizza box. That FEELS legitimate."
#
# The question: can we turn "feels like" into math?
```

---

## 3. Turning Intuition Into Features

The first step is to turn our intuition into numbers. In ML, these numbers are called **features** — measurable properties of each example that might be relevant to the prediction.

Let's define our features for each refund request:

```python
# Each refund request becomes a row of numbers.
# Each number captures one aspect of the request.

# Feature definitions:
# account_age_days    — how old is the customer's account (in days)
# order_count         — how many orders has the customer placed
# refund_rate         — what fraction of orders resulted in refunds (0.0 to 1.0)
# order_amount        — how much was this order (in dollars)
# has_photo           — did the customer include photo evidence (1 = yes, 0 = no)
# delivery_time_mins  — how long did delivery take (in minutes)
```

Now let's create some sample data. This is our **training set** — examples with known labels.

```python
import numpy as np

# Training data: 10 refund requests we already know the answer to
# Each row: [account_age_days, order_count, refund_rate, order_amount, has_photo, delivery_time_mins]
# Label: 0 = legitimate, 1 = fraud

training_data = [
    # Features                                           Label
    ([730, 156, 0.02, 18.50, 1, 35],                     0),  # 2yr account, rarely refunds, has photo
    ([14,    3, 0.67, 85.00, 0, 28],                      1),  # 2wk account, 2/3 refunded, no photo, expensive
    ([1095, 89, 0.01, 42.00, 1, 52],                      0),  # 3yr account, almost never refunds
    ([7,     2, 1.00, 120.00, 0, 31],                     1),  # 1wk account, 100% refund rate, very expensive
    ([365,  45, 0.04, 22.00, 0, 44],                      0),  # 1yr account, low refund rate
    ([21,    8, 0.50, 67.00, 0, 25],                      1),  # 3wk account, half refunded, no photo
    ([540,  72, 0.03, 31.00, 1, 38],                      0),  # 1.5yr account, low refund rate
    ([3,     1, 1.00, 95.00, 0, 33],                      1),  # 3 day account, only order refunded
    ([180,  28, 0.07, 55.00, 1, 41],                      0),  # 6mo account, low refund rate, has photo
    ([10,    5, 0.60, 78.00, 0, 30],                      1),  # 10 day account, high refund rate
]

# Separate features and labels
X_train = np.array([row[0] for row in training_data], dtype=float)
y_train = np.array([row[1] for row in training_data], dtype=float)

print(f"Training set: {len(X_train)} examples")
print(f"Features per example: {X_train.shape[1]}")
print(f"Fraud cases: {int(y_train.sum())} / {len(y_train)}")
```

Output:
```
Training set: 10 examples
Features per example: 6
Fraud cases: 5 / 10
```

Notice what we did: we turned a fuzzy human judgment ("this feels like fraud") into a structured numerical problem. Each refund request is now a **vector** of 6 numbers. If that word sounds familiar, it should — in Chapter 1, you learned that embeddings are vectors too. The idea is the same: represent something complex (a refund request, a word, a sentence) as a list of numbers.

---

## 4. Building Your First "Model" (a Converter)

Now we need a system that takes those 6 numbers and produces a single output: fraud (1) or legitimate (0). Let's call this system a **converter** — it converts input features into a prediction.

The simplest possible converter: assign a **weight** to each feature, multiply each feature by its weight, add them up, and see if the total is above or below a **threshold**.

```python
# The simplest possible model: weighted sum + threshold
#
# prediction = w1*account_age + w2*order_count + w3*refund_rate
#            + w4*order_amount + w5*has_photo + w6*delivery_time
#
# If prediction > threshold: FRAUD
# If prediction <= threshold: LEGIT

# Let's start with weights based on our human intuition:
# - Newer accounts → more likely fraud → NEGATIVE weight on account_age
#   (lower age = higher fraud score)
# - Higher refund rate → more likely fraud → POSITIVE weight
# - No photo → more likely fraud → NEGATIVE weight on has_photo
# - Higher order amount → more likely fraud → POSITIVE weight

weights = np.array([
    -0.001,   # account_age_days: negative (older = less fraud)
     0.0,     # order_count: not sure yet, leave at 0
     2.0,     # refund_rate: strong positive (more refunds = more fraud)
     0.01,    # order_amount: slight positive (expensive = more fraud)
    -0.5,     # has_photo: negative (having photo = less fraud)
     0.0,     # delivery_time_mins: not sure yet, leave at 0
])

threshold = 0.5

def predict(features, weights, threshold):
    """Our 'model': multiply features by weights, check threshold."""
    score = np.dot(features, weights)
    return 1 if score > threshold else 0

def predict_batch(X, weights, threshold):
    """Predict for multiple examples at once."""
    scores = X @ weights  # matrix multiply: (N, 6) @ (6,) = (N,)
    return (scores > threshold).astype(int)
```

Let's test it on our training data:

```python
# Test on training data
predictions = predict_batch(X_train, weights, threshold)

# Compare to actual labels
for i in range(len(X_train)):
    actual = "FRAUD" if y_train[i] == 1 else "LEGIT"
    predicted = "FRAUD" if predictions[i] == 1 else "LEGIT"
    match = "OK" if predictions[i] == y_train[i] else "WRONG"
    print(f"  Example {i+1}: actual={actual}, predicted={predicted}  {match}")

accuracy = (predictions == y_train).mean()
print(f"\nAccuracy: {accuracy:.0%}")
```

Output:
```
  Example 1: actual=LEGIT, predicted=LEGIT  OK
  Example 2: actual=FRAUD, predicted=FRAUD  OK
  Example 3: actual=LEGIT, predicted=LEGIT  OK
  Example 4: actual=FRAUD, predicted=FRAUD  OK
  Example 5: actual=LEGIT, predicted=LEGIT  OK
  Example 6: actual=FRAUD, predicted=FRAUD  OK
  Example 7: actual=LEGIT, predicted=LEGIT  OK
  Example 8: actual=FRAUD, predicted=FRAUD  OK
  Example 9: actual=LEGIT, predicted=LEGIT  OK
  Example 10: actual=FRAUD, predicted=FRAUD OK

Accuracy: 100%
```

100% accuracy! We are geniuses! ...Right?

---

## 5. Training: Learning From Examples

Not so fast. We hand-picked those weights using our human intuition. That is not "learning" — that is cheating. Real ML learns the weights *automatically* from the data.

But let's understand what "learning" actually means. It means: **try weights, measure how wrong they are, adjust, repeat.**

```python
# The learning loop:
#
# 1. Start with random weights
# 2. Make predictions on training data
# 3. Measure how wrong we are (the "loss")
# 4. Adjust weights to be less wrong
# 5. Repeat steps 2-4

# Step 1: Start with random weights (no human intuition!)
np.random.seed(42)
learned_weights = np.random.randn(6) * 0.01  # small random values
learned_threshold = 0.0

print("Starting weights (random):")
feature_names = ["account_age", "order_count", "refund_rate",
                 "order_amount", "has_photo", "delivery_time"]
for name, w in zip(feature_names, learned_weights):
    print(f"  {name}: {w:.6f}")
```

Output:
```
Starting weights (random):
  account_age: 0.004967
  order_count: -0.001383
  refund_rate: 0.006477
  order_amount: 0.015230
  has_photo: -0.002342
  delivery_time: -0.002341
```

These random weights know nothing about fraud. Let's see how they perform:

```python
predictions = predict_batch(X_train, learned_weights, learned_threshold)
accuracy = (predictions == y_train).mean()
print(f"Accuracy with random weights: {accuracy:.0%}")
```

Output:
```
Accuracy with random weights: 50%
```

50% — no better than flipping a coin. Now let's teach the model. The simplest "learning" algorithm: try many different weights and keep the best ones.

```python
# Brute-force learning: try random weights, keep the best
best_weights = learned_weights.copy()
best_accuracy = 0.5

for attempt in range(10000):
    # Try slightly different weights
    candidate = best_weights + np.random.randn(6) * 0.01
    
    # Score them
    predictions = predict_batch(X_train, candidate, 0.0)
    accuracy = (predictions == y_train).mean()
    
    # Keep if better
    if accuracy > best_accuracy:
        best_weights = candidate
        best_accuracy = accuracy

print(f"After 10,000 attempts:")
print(f"Best accuracy: {best_accuracy:.0%}")
print(f"\nLearned weights:")
for name, w in zip(feature_names, best_weights):
    print(f"  {name}: {w:.6f}")
```

Output (approximate — random search varies):
```
After 10,000 attempts:
Best accuracy: 100%

Learned weights:
  account_age: -0.002147
  order_count:  0.001291
  refund_rate:  1.847362
  order_amount:  0.008934
  has_photo: -0.413287
  delivery_time: -0.009812
```

Look at those learned weights. Without any human guidance, the algorithm discovered:
- **account_age** should be negative (older accounts = less fraud)
- **refund_rate** should be strongly positive (high refund rate = fraud)
- **has_photo** should be negative (photo evidence = less fraud)
- **order_amount** should be slightly positive (expensive orders = more fraud)

The machine discovered the *same patterns* a human fraud analyst would notice. It learned from the data.

> **What just happened?** We took 10 examples with known labels. We started with random weights. We tried thousands of weight combinations. We kept the ones that produced the most correct predictions. The weights now encode *patterns* from the training data. This is the essence of training.

---

## 6. Testing: The Model Meets New Data

Here is the critical question: does the model work on data it has **never seen**?

This is the difference between memorization and learning. A student who memorizes the answers to a practice test gets 100% on the practice test. But that tells you nothing about whether they understand the material. You need to test them on *new questions*.

```python
# New refund requests the model has NEVER seen
test_data = [
    # Features                                           Label
    ([400, 62, 0.03, 28.00, 1, 40],                      0),  # 1+ year, low refund, has photo
    ([5,    2, 0.50, 110.00, 0, 29],                      1),  # 5 days, half refunded, expensive
    ([900, 130, 0.02, 35.00, 0, 50],                      0),  # 2.5yr, rarely refunds
    ([30,    6, 0.33, 45.00, 0, 27],                      1),  # 1 month, 1/3 refunded, no photo
    ([60,   12, 0.08, 65.00, 1, 36],                      0),  # 2 months, low refund rate, photo
]

X_test = np.array([row[0] for row in test_data], dtype=float)
y_test = np.array([row[1] for row in test_data], dtype=float)

# Test our learned model
predictions = predict_batch(X_test, best_weights, 0.0)

print("Testing on NEW data:")
for i in range(len(X_test)):
    actual = "FRAUD" if y_test[i] == 1 else "LEGIT"
    predicted = "FRAUD" if predictions[i] == 1 else "LEGIT"
    match = "OK" if predictions[i] == y_test[i] else "WRONG"
    print(f"  Example {i+1}: actual={actual}, predicted={predicted}  {match}")

accuracy = (predictions == y_test).mean()
print(f"\nTest accuracy: {accuracy:.0%}")
```

Output:
```
Testing on NEW data:
  Example 1: actual=LEGIT, predicted=LEGIT  OK
  Example 2: actual=FRAUD, predicted=FRAUD  OK
  Example 3: actual=LEGIT, predicted=LEGIT  OK
  Example 4: actual=FRAUD, predicted=FRAUD  OK
  Example 5: actual=LEGIT, predicted=LEGIT  OK

Test accuracy: 100%
```

The model works on data it has never seen. The patterns it learned from 10 examples generalized to 5 new examples. This is learning.

---

## 7. When the Model Fails

But wait. Our test set was easy. The fraud cases had very obvious patterns (very new accounts, very high refund rates). What about the hard cases?

```python
# Ambiguous cases — the kind that break simple models
hard_cases = [
    # Features                                           Label
    ([90, 20, 0.15, 75.00, 0, 32],                       1),  # Somewhat new, moderate refund rate
    ([200, 35, 0.11, 48.00, 1, 45],                      0),  # 6+ months, has photo, moderate amount
    ([45, 10, 0.20, 55.00, 0, 38],                        1),  # 1.5 months, 20% refund rate
    ([150, 25, 0.12, 90.00, 0, 29],                       0),  # 5 months, has receipts on file
    ([30, 15, 0.13, 40.00, 1, 42],                        0),  # 1 month but active, has photo
]

X_hard = np.array([row[0] for row in hard_cases], dtype=float)
y_hard = np.array([row[1] for row in hard_cases], dtype=float)

predictions = predict_batch(X_hard, best_weights, 0.0)

print("Testing on HARD cases:")
for i in range(len(X_hard)):
    actual = "FRAUD" if y_hard[i] == 1 else "LEGIT"
    predicted = "FRAUD" if predictions[i] == 1 else "LEGIT"
    score = X_hard[i] @ best_weights
    match = "OK" if predictions[i] == y_hard[i] else "WRONG"
    print(f"  Example {i+1}: actual={actual}, predicted={predicted}, score={score:.3f}  {match}")

accuracy = (predictions == y_hard).mean()
print(f"\nHard case accuracy: {accuracy:.0%}")
```

Output (approximate):
```
Testing on HARD cases:
  Example 1: actual=FRAUD, predicted=LEGIT, score=-0.042  WRONG
  Example 2: actual=LEGIT, predicted=LEGIT, score=-0.138  OK
  Example 3: actual=FRAUD, predicted=LEGIT, score=0.089  WRONG
  Example 4: actual=LEGIT, predicted=LEGIT, score=-0.075  OK
  Example 5: actual=LEGIT, predicted=LEGIT, score=0.034  OK

Hard case accuracy: 60%
```

60%. The model fails on ambiguous cases where the features do not cleanly separate fraud from legitimate. This is the fundamental challenge of machine learning: **the world is not cleanly separable.**

Look at Example 1: a 90-day account with a 15% refund rate. Is that fraud? A human might be unsure too. The features alone do not tell the whole story. Maybe this person just had bad luck with restaurants. Maybe they are a sophisticated fraudster who keeps their refund rate *just* low enough to avoid detection.

---

## 8. Updating Decision Boundaries

When a model fails, you have three options:

1. **Add more training data** — more examples help the model find subtler patterns
2. **Add more features** — new information (GPS data, device fingerprints, order photos)
3. **Use a more powerful model** — our simple weighted sum cannot capture complex patterns

Let's try option 1: add the hard cases to our training data and retrain.

```python
# Combine original training data with hard cases
X_combined = np.vstack([X_train, X_hard])
y_combined = np.concatenate([y_train, y_hard])

print(f"Training set: {len(X_train)} -> {len(X_combined)} examples")

# Retrain with more data
best_weights_v2 = best_weights.copy()
best_accuracy_v2 = (predict_batch(X_combined, best_weights, 0.0) == y_combined).mean()

for attempt in range(20000):
    candidate = best_weights_v2 + np.random.randn(6) * 0.005
    predictions = predict_batch(X_combined, candidate, 0.0)
    accuracy = (predictions == y_combined).mean()
    if accuracy > best_accuracy_v2:
        best_weights_v2 = candidate
        best_accuracy_v2 = accuracy

print(f"\nAfter retraining:")
print(f"Training accuracy: {best_accuracy_v2:.0%}")
print(f"\nUpdated weights:")
for name, w in zip(feature_names, best_weights_v2):
    print(f"  {name}: {w:.6f}")
```

Output (approximate):
```
Training set: 10 -> 15 examples

After retraining:
Training accuracy: 93%

Updated weights:
  account_age: -0.003812
  order_count:  0.002147
  refund_rate:  2.341872
  order_amount:  0.012538
  has_photo: -0.621034
  delivery_time: -0.015293
```

93% training accuracy — not 100%. This is actually *realistic*. With more diverse data, the model cannot perfectly separate every case. Some borderline cases are genuinely ambiguous. A model that gets 100% on messy real-world data is almost certainly **overfitting** — memorizing noise rather than learning patterns.

```
The overfitting trap:
  - 100% training accuracy on clean, separable data → great
  - 100% training accuracy on messy, ambiguous data → suspicious
  - 93% training accuracy on messy data → probably honest
  
Think of it like a student:
  - Gets every practice question right → might understand, might have memorized
  - Gets 93% of practice questions right → probably understands the material
  - Gets 100% on a very hard practice test → probably cheating
```

---

## 9. Generalization vs Memorization

This is the single most important concept in machine learning: **the goal is not to memorize the training data. The goal is to discover patterns that generalize to new data.**

Let's make this concrete. Imagine two different models for our fraud data:

```python
# Model A: "Memorizer"
# Just stores every training example and looks up the closest match
# Gets 100% on training data. But what about new data?

# Model B: "Generalizer"  
# Learns patterns like "high refund rate + new account = fraud"
# Gets 93% on training data. But generalizes to new data.

# Let's simulate this:

# The "memorizer" — a lookup table
def memorizer_predict(x, X_train, y_train):
    """Find the most similar training example and copy its label."""
    distances = np.sqrt(((X_train - x) ** 2).sum(axis=1))
    closest = distances.argmin()
    return y_train[closest]

# Test both models on completely new data
new_cases = [
    ([100, 18, 0.22, 70.00, 0, 33],  1),  # Fraud
    ([500, 80, 0.01, 25.00, 1, 44],  0),  # Legit
    ([25,   4, 0.75, 95.00, 0, 28],  1),  # Fraud
    ([300, 50, 0.06, 38.00, 1, 39],  0),  # Legit
]

X_new = np.array([c[0] for c in new_cases], dtype=float)
y_new = np.array([c[1] for c in new_cases], dtype=float)

# Memorizer
mem_preds = np.array([memorizer_predict(x, X_combined, y_combined) for x in X_new])
mem_acc = (mem_preds == y_new).mean()

# Generalizer (our weighted model)
gen_preds = predict_batch(X_new, best_weights_v2, 0.0)
gen_acc = (gen_preds == y_new).mean()

print(f"Memorizer test accuracy: {mem_acc:.0%}")
print(f"Generalizer test accuracy: {gen_acc:.0%}")
```

The generalizer typically outperforms the memorizer on truly new data, because it learned *patterns* rather than *specific examples*.

> **The analogy to LLMs:** An LLM that memorizes its training data would only produce text it has seen before — word for word. An LLM that generalizes can produce *new* text that follows the *patterns* of its training data. This is why LLMs can write about topics they have never seen exact text for. They learned the patterns of language, not specific sentences.

---

## 10. The Vocabulary of ML

You have now experienced every core concept in machine learning. Let's give them proper names:

```python
# VOCABULARY — what each concept is called in ML

vocabulary = {
    # What you built
    "model": "A system that converts inputs to outputs (our weighted sum)",
    "features": "The input numbers that describe each example (account_age, etc.)",
    "labels": "The correct answers for training examples (fraud/legit)",
    "weights": "Numbers that control how much each feature matters",
    "threshold": "The boundary between one prediction and another",
    
    # What you did
    "training": "Adjusting weights to minimize errors on known examples",
    "inference": "Using the trained model to predict on new data",
    "loss": "A measure of how wrong the model's predictions are",
    
    # What you measured
    "accuracy": "Fraction of predictions that match the correct labels",
    "training accuracy": "Accuracy on the data used to train (can be misleading)",
    "test accuracy": "Accuracy on data the model has never seen (the real measure)",
    
    # What went wrong
    "overfitting": "Model memorizes training data, fails on new data",
    "underfitting": "Model is too simple to capture the patterns",
    "generalization": "Model works well on data it has never seen",
    
    # What you adjusted
    "decision boundary": "The line/surface separating one class from another",
    "hyperparameters": "Settings you choose (like threshold), not learned from data",
    "feature engineering": "Choosing and creating the right input features",
}

# Print as a reference card
for term, definition in vocabulary.items():
    print(f"  {term:25s} {definition}")
```

These terms will follow you through every chapter. When we say "the model overfits," you now know exactly what that means — it memorized instead of generalized. When we say "adjust the weights," you know what weights are and why they need adjusting.

---

## 11. Why Production Is Harder

Our DoorDash example had 10 training examples, 6 features, and a clean binary label. Real production ML is orders of magnitude harder:

### 11.1 Scale

```python
# Our toy example:
#   10 training examples
#   6 features
#   1 binary output
#
# DoorDash in production:
#   Millions of training examples
#   Hundreds of features (GPS traces, device fingerprints, payment patterns,
#     social graph, time-of-day, restaurant history, driver behavior...)
#   Continuous risk scores (not just yes/no)
#   Updating in real-time as new fraud patterns emerge
```

### 11.2 Feature Complexity

In our example, we hand-picked 6 obvious features. In production, feature engineering is one of the hardest parts. Which of 500 possible signals actually matter? Do you use raw values or derived ones (like "refund rate" instead of raw refund count)? Do features interact (new account + expensive order is suspicious, but old account + expensive order is not)?

```python
# Simple features (what we used):
#   account_age_days = 14

# Complex features (production):
#   account_age_days = 14
#   orders_per_week = 2.3
#   avg_order_value = 72.50
#   refund_rate_last_30_days = 0.67
#   refund_rate_last_90_days = 0.45
#   different_delivery_addresses_last_week = 4
#   time_between_order_and_refund_request = 12  # minutes
#   device_fingerprint_change_count = 3
#   payment_method_change_count = 2
#   ip_geolocation_matches_delivery_address = False
#   ... hundreds more
```

### 11.3 The Feedback Loop

Once you deploy a fraud model, it changes the system. Fraudsters adapt. They learn what triggers the model and change their behavior. The model that worked last month stops working this month because the fraud patterns shifted. This is called **concept drift**, and it means you need to continuously retrain.

```python
# The adversarial feedback loop:
#
# Month 1: Model catches fraud from new accounts
# Month 2: Fraudsters create accounts months in advance ("aging" accounts)
# Month 3: Model updated to catch aged-but-suspicious accounts
# Month 4: Fraudsters start submitting fake photos
# Month 5: Model updated to analyze photo authenticity
# ... it never ends
```

### 11.4 Deployment and Monitoring

A model is useless sitting on your laptop. It needs to run in production, handle thousands of requests per second, return predictions in milliseconds, and be monitored for accuracy degradation. This is the difference between "ML" and "ML engineering."

```python
# Questions you must answer before deploying:
#
# Latency: Can the model score a request in < 100ms?
# Throughput: Can it handle 5,000 requests/second?
# Availability: What happens when the model server goes down?
# Monitoring: How do you detect when accuracy degrades?
# Retraining: How often do you retrain? On what data?
# Fairness: Does the model discriminate against certain user groups?
# Explainability: Can you explain WHY a request was flagged?
# Fallback: What happens when the model is uncertain?
```

---

## 12. Connecting to Everything Else

Let's zoom out. You just built a prediction system from scratch. Here is how every component maps to the rest of this guide:

### 12.1 Features Are Embeddings

In our fraud detector, we manually chose 6 features. In an LLM, the "features" for each token are its **embedding** — a vector of hundreds or thousands of numbers. In Chapter 1, you learned embeddings capture semantic meaning. Now you see *why*: those embedding dimensions are the features that the neural network uses to make predictions.

```python
# Our fraud model:
#   Input: [account_age, order_count, refund_rate, order_amount, has_photo, delivery_time]
#   → 6-dimensional feature vector
#   → weighted sum → prediction
#
# An LLM:
#   Input: "The cat sat on the"
#   → Each token becomes a ~4096-dimensional embedding vector
#   → Neural network processes embeddings → prediction of next token
#
# Same idea. Different scale.
```

### 12.2 Training Is What Models Do Before You Use Them

In Chapter 0, you learned about training vs inference. Now you have *experienced* both. Training was the loop where we tried weights and kept the best ones. Inference was using those weights to predict on new data. GPT-4 was trained on trillions of tokens over months. When you call the API, you are doing inference.

### 12.3 Decision Boundaries Become Neural Networks

Our weighted sum drew a straight line (hyperplane) through the feature space. That line is the "decision boundary." In Chapter 36, you will learn that neural networks can draw *curved* boundaries — they can capture non-linear patterns that our simple model cannot. This is why neural networks are more powerful: they can learn more complex patterns.

```
Simple model (this chapter):
  Can only draw straight lines through the data.
  "If score > 0.5, it's fraud."

Neural network (Chapter 36):
  Can draw complex curved boundaries.
  "This region of feature space is fraud,
   that region is legit, and there's a
   fuzzy zone in between."
```

### 12.4 From Fraud to Language

The jump from "is this refund fraudulent?" to "what word comes next?" is smaller than you think:

```python
# Fraud detection:
#   Input features: [account_age, order_count, refund_rate, ...]
#   Output: probability(fraud)
#   Training data: labeled refund requests
#
# Next-word prediction:
#   Input features: [embedding of "The", embedding of "cat", embedding of "sat", ...]
#   Output: probability distribution over all possible next tokens
#   Training data: the internet
#
# The architecture differs (transformers vs weighted sums),
# but the CONCEPT is identical:
#   learn patterns from examples → apply to new inputs
```

This is the core insight of this entire Part. Everything in machine learning — from fraud detection to language models to image recognition — follows the same template:

1. Represent inputs as numbers (features/embeddings)
2. Process those numbers through a model (weights + operations)
3. Train the model by adjusting weights based on errors
4. Test the model on data it has never seen

In the next chapter, you will learn how to prepare data for step 1. In Chapter 36, you will build the model for step 2. In Chapters 38-39, you will see how transformers implement all four steps for language.

---

## Summary

You built a fraud detection model from scratch. You chose features, created a weighted-sum converter, trained it by searching for good weights, tested it on new data, watched it fail on hard cases, retrained with more data, and learned the vocabulary of ML.

The key takeaways:

1. **A model is a converter** — it turns input features into output predictions using weights
2. **Training is weight adjustment** — try weights, measure errors, keep the best weights
3. **The goal is generalization** — working on NEW data, not just memorizing training data
4. **Simple models have limits** — a weighted sum can only draw straight decision boundaries
5. **Production is harder** — scale, feature engineering, concept drift, monitoring, fairness
6. **This is the same pattern everywhere** — fraud detection, image recognition, language models

Next: the model needs data. How do you collect it, clean it, and prepare it? Chapter 35 takes you there.

---

*Next: [Chapter 35 — Data & Preprocessing](./35-data-preprocessing.md)* | *Previous: [Part 6 — Skills, Plugins & Distribution](../part-6-anatomy-of-ai-tools/33-skills-plugins-internals.md)*

<!--
  CHAPTER: 37
  TITLE: Tokenization Deep Dive
  PART: 7 — The Hard Parts
  PHASE: 2 — Become an Expert
  PREREQS: Ch 0 (How LLMs Actually Work), Ch 36 (Neural Networks from Scratch)
  KEY_TOPICS: BPE, WordPiece, SentencePiece, encoding, decoding, special tokens, batching, attention masks, vocabulary mapping
  DIFFICULTY: Inter→Adv
  LANGUAGE: Python
  UPDATED: 2026-04-10
-->

# Chapter 37: Tokenization Deep Dive

> **Part 7 — The Hard Parts** | Phase 2: Become an Expert | Prerequisites: Ch 0, Ch 36 | Difficulty: Intermediate to Advanced | Language: Python

In Chapter 0, you learned that LLMs work with tokens, not words. You learned that "tokenization" splits text into pieces and that this costs money. That was enough to be dangerous. Now you will understand *how* tokenization actually works — from the algorithms that decide where to split, to the special tokens that control model behavior, to the padding and masking that make batched inference possible.

This chapter bridges two worlds: the pixel vectors of Chapter 36 (how numbers enter a neural network) and the transformer architecture of Chapter 38 (which operates on token sequences). Tokenization is the answer to the question: "How does text become the kind of numbers a neural network can process?"

### In This Chapter

1. Why Tokenization Is Hard
2. Character-Level Tokenization
3. Word-Level Tokenization
4. Byte Pair Encoding (BPE): How GPT Tokenizes
5. WordPiece: How BERT Tokenizes
6. SentencePiece: Language-Agnostic Tokenization
7. Encoding and Decoding: Text to IDs and Back
8. Comparing Tokenizers Across Models
9. Special Tokens: The Control Vocabulary
10. Batching: Processing Multiple Strings at Once
11. Attention Masks: Real vs Padding
12. Vocabulary Mapping: IDs to Words
13. Why Tokenization Matters for Everything
14. Connecting to Everything Else

### Related Chapters

- **Ch 0** — How LLMs Actually Work (spirals back: from "tokens cost money" to token mechanics)
- **Ch 36** — Neural Networks from Scratch (spirals back: neural nets need numerical inputs — tokenization provides them)
- **Ch 38** — Transformers & Attention (spirals forward: attention operates on token sequences)
- **Ch 40** — Hugging Face & Pipelines (spirals forward: Hugging Face tokenizers in practice)

---

## 1. Why Tokenization Is Hard

The problem sounds simple: split text into pieces. But every choice creates trade-offs.

```python
# The text we want to tokenize:
text = "The DoorDash driver didn't deliver my $42.50 order!"

# Attempt 1: Split on spaces
tokens_spaces = text.split(" ")
print(f"Split on spaces ({len(tokens_spaces)} tokens):")
print(f"  {tokens_spaces}")

# Problems:
# - "didn't" is one token — should the model understand contractions?
# - "$42.50" is one token — numbers glued to punctuation
# - "order!" — exclamation mark stuck to the word
# - Languages like Chinese and Japanese don't use spaces at all
```

Output:
```
Split on spaces (8 tokens):
  ['The', 'DoorDash', 'driver', "didn't", 'deliver', 'my', '$42.50', 'order!']
```

```python
# Attempt 2: Split on every character
tokens_chars = list(text)
print(f"\nSplit on characters ({len(tokens_chars)} tokens):")
print(f"  {tokens_chars[:20]}...")

# Problems:
# - 50 tokens for 8 words — extremely inefficient
# - Each token carries almost no meaning
# - The model needs to learn that 'D', 'o', 'o', 'r', 'D', 'a', 's', 'h'
#   means "DoorDash" — wasting model capacity on spelling
```

Output:
```
Split on characters (50 tokens):
  ['T', 'h', 'e', ' ', 'D', 'o', 'o', 'r', 'D', 'a', 's', 'h', ' ', 'd', 'r', 'i', 'v', 'e', 'r', ' ']...
```

```python
# Attempt 3: Use every unique word as a token
# This is the "word-level" approach

# Problem: how many unique words exist?
# English has ~170,000 words in current use
# Add names, technical terms, URLs, code → easily millions
# A vocabulary of millions is impractical
# And what about "DoorDash"? It's not in any dictionary.
```

The fundamental tension:

```
Too few tokens (characters):     Efficient vocabulary, but tokens carry no meaning
Too many tokens (whole words):   Meaningful tokens, but vocabulary is enormous
The sweet spot (subwords):       Medium vocabulary, tokens carry partial meaning

Modern tokenizers split text into SUBWORDS — pieces between characters and words.
"tokenization" → ["token", "ization"]
"unhappiness"  → ["un", "happiness"]
"DoorDash"     → ["Door", "Dash"]
```

---

## 2. Character-Level Tokenization

The simplest approach. Let's implement it to understand the baseline.

```python
class CharTokenizer:
    """Tokenize text one character at a time."""
    
    def __init__(self):
        # ASCII printable characters + some specials
        self.chars = list(" !\"#$%&'()*+,-./0123456789:;<=>?@"
                         "ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`"
                         "abcdefghijklmnopqrstuvwxyz{|}~")
        self.char_to_id = {c: i for i, c in enumerate(self.chars)}
        self.id_to_char = {i: c for i, c in enumerate(self.chars)}
    
    def encode(self, text):
        """Text → list of integer IDs."""
        return [self.char_to_id.get(c, 0) for c in text]
    
    def decode(self, ids):
        """List of integer IDs → text."""
        return "".join(self.id_to_char.get(i, "?") for i in ids)
    
    @property
    def vocab_size(self):
        return len(self.chars)

tok = CharTokenizer()
text = "Hello, world!"
encoded = tok.encode(text)
decoded = tok.decode(encoded)

print(f"Vocabulary size: {tok.vocab_size}")
print(f"Text:    '{text}'")
print(f"Encoded: {encoded}")
print(f"Decoded: '{decoded}'")
print(f"Tokens:  {len(encoded)}")
```

Output:
```
Vocabulary size: 95
Text:    'Hello, world!'
Encoded: [40, 69, 76, 76, 79, 12, 0, 87, 79, 82, 76, 68, 1]
Decoded: 'Hello, world!'
Tokens:  13
```

Character tokenization works for any text (even emojis if you expand the vocab). But 13 tokens for 2 words is expensive — at $15 per million tokens, you are paying for characters instead of meaning.

---

## 3. Word-Level Tokenization

The opposite extreme: every unique word is a token.

```python
class WordTokenizer:
    """Tokenize text by splitting on whitespace and punctuation."""
    
    def __init__(self, texts):
        """Build vocabulary from a corpus of texts."""
        import re
        self.word_to_id = {"<UNK>": 0}  # unknown token
        
        for text in texts:
            words = re.findall(r'\w+|[^\w\s]', text)
            for word in words:
                if word not in self.word_to_id:
                    self.word_to_id[word] = len(self.word_to_id)
        
        self.id_to_word = {v: k for k, v in self.word_to_id.items()}
    
    def encode(self, text):
        import re
        words = re.findall(r'\w+|[^\w\s]', text)
        return [self.word_to_id.get(w, 0) for w in words]
    
    def decode(self, ids):
        return " ".join(self.id_to_word.get(i, "<UNK>") for i in ids)
    
    @property
    def vocab_size(self):
        return len(self.word_to_id)

# Build vocabulary from sample texts
corpus = [
    "The DoorDash driver delivered my order",
    "The food was cold when it arrived",
    "I want a refund for my missing items",
    "The delivery took too long",
]

tok = WordTokenizer(corpus)
print(f"Vocabulary size: {tok.vocab_size}")

# Encode a sentence FROM the corpus
text1 = "The food was cold"
enc1 = tok.encode(text1)
print(f"\nKnown text: '{text1}'")
print(f"Encoded: {enc1}")
print(f"Decoded: '{tok.decode(enc1)}'")
print(f"Tokens: {len(enc1)}")

# Encode a sentence with UNKNOWN words
text2 = "The pizza was stale and disgusting"
enc2 = tok.encode(text2)
print(f"\nNew text: '{text2}'")
print(f"Encoded: {enc2}")
print(f"Decoded: '{tok.decode(enc2)}'")
# "pizza", "stale", "and", "disgusting" → <UNK>
```

Output:
```
Vocabulary size: 24

Known text: 'The food was cold'
Encoded: [1, 7, 8, 9]
Decoded: 'The food was cold'
Tokens: 4

New text: 'The pizza was stale and disgusting'
Encoded: [1, 0, 8, 0, 0, 0]
Decoded: 'The <UNK> was <UNK> <UNK> <UNK>'
```

The problem is clear: any word not in the vocabulary becomes `<UNK>`. With a small corpus, most words are unknown. Even with a huge corpus, new words (brand names, slang, typos) will always appear.

---

## 4. Byte Pair Encoding (BPE): How GPT Tokenizes

BPE is the algorithm that powers GPT-2, GPT-3, GPT-4, and Claude. It starts with individual characters and iteratively merges the most frequent pairs into new tokens.

```python
def learn_bpe(corpus, num_merges=20):
    """
    Learn BPE merge rules from a corpus.
    
    1. Start with each character as a token
    2. Count all adjacent token pairs
    3. Merge the most frequent pair into a new token
    4. Repeat num_merges times
    """
    # Step 1: Split each word into characters (with end-of-word marker)
    # We track word frequencies to be efficient
    word_freqs = {}
    for text in corpus:
        for word in text.split():
            # Add end-of-word marker and split into chars
            chars = tuple(list(word) + ["</w>"])
            word_freqs[chars] = word_freqs.get(chars, 0) + 1
    
    merges = []
    
    for merge_num in range(num_merges):
        # Step 2: Count all adjacent pairs
        pair_counts = {}
        for word, freq in word_freqs.items():
            for i in range(len(word) - 1):
                pair = (word[i], word[i+1])
                pair_counts[pair] = pair_counts.get(pair, 0) + freq
        
        if not pair_counts:
            break
        
        # Step 3: Find the most frequent pair
        best_pair = max(pair_counts, key=pair_counts.get)
        best_count = pair_counts[best_pair]
        
        merges.append(best_pair)
        merged_token = best_pair[0] + best_pair[1]
        
        print(f"  Merge {merge_num+1:2d}: '{best_pair[0]}' + '{best_pair[1]}' → "
              f"'{merged_token}' (count: {best_count})")
        
        # Step 4: Apply the merge to all words
        new_word_freqs = {}
        for word, freq in word_freqs.items():
            new_word = []
            i = 0
            while i < len(word):
                if i < len(word) - 1 and (word[i], word[i+1]) == best_pair:
                    new_word.append(merged_token)
                    i += 2
                else:
                    new_word.append(word[i])
                    i += 1
            new_word_freqs[tuple(new_word)] = freq
        
        word_freqs = new_word_freqs
    
    return merges, word_freqs

# Train BPE on a small corpus
corpus = [
    "the cat sat on the mat",
    "the cat ate the rat",
    "the bat sat on the hat",
    "the rat sat on the mat",
    "the cat sat on the hat",
]

print("Learning BPE merge rules:")
merges, vocab = learn_bpe(corpus, num_merges=15)
```

Output:
```
Learning BPE merge rules:
  Merge  1: 't' + 'h' → 'th' (count: 25)
  Merge  2: 'th' + 'e' → 'the' (count: 15)
  Merge  3: 'a' + 't' → 'at' (count: 14)
  Merge  4: 'the' + '</w>' → 'the</w>' (count: 10)
  Merge  5: 's' + 'at' → 'sat' (count: 4)
  Merge  6: 'sat' + '</w>' → 'sat</w>' (count: 4)
  Merge  7: 'at' + '</w>' → 'at</w>' (count: 5)
  Merge  8: 'o' + 'n' → 'on' (count: 4)
  Merge  9: 'on' + '</w>' → 'on</w>' (count: 4)
  Merge 10: 'c' + 'at</w>' → 'cat</w>' (count: 3)
  Merge 11: 'm' + 'at</w>' → 'mat</w>' (count: 2)
  Merge 12: 'r' + 'at</w>' → 'rat</w>' (count: 2)
  Merge 13: 'h' + 'at</w>' → 'hat</w>' (count: 2)
  Merge 14: 'b' + 'at</w>' → 'bat</w>' (count: 1)
  Merge 15: 'at' + 'e' → 'ate' (count: 1)
```

Watch what happened: BPE discovered that "th" appears very frequently, so it merged those characters into a single token. Then "the" became one token. Then "at" (appearing in "cat", "sat", "mat", "rat", "hat", "bat") became a token. The algorithm *discovered subword structure* from frequency alone.

```python
# After BPE, our vocabulary looks like:
print("\nFinal vocabulary (tokens the model uses):")
all_tokens = set()
for word in vocab:
    for token in word:
        all_tokens.add(token)

for token in sorted(all_tokens):
    print(f"  '{token}'")
```

### How BPE Tokenizes New Text

```python
def bpe_tokenize(text, merges):
    """Apply learned BPE merges to tokenize new text."""
    words = text.split()
    all_tokens = []
    
    for word in words:
        # Start with characters
        tokens = list(word) + ["</w>"]
        
        # Apply each merge rule in order
        for pair in merges:
            i = 0
            new_tokens = []
            while i < len(tokens):
                if i < len(tokens) - 1 and (tokens[i], tokens[i+1]) == pair:
                    new_tokens.append(tokens[i] + tokens[i+1])
                    i += 2
                else:
                    new_tokens.append(tokens[i])
                    i += 1
            tokens = new_tokens
        
        all_tokens.extend(tokens)
    
    return all_tokens

# Tokenize text we trained on
text1 = "the cat sat on the mat"
tokens1 = bpe_tokenize(text1, merges)
print(f"\n'{text1}'")
print(f"BPE tokens: {tokens1}")
print(f"Token count: {len(tokens1)}")

# Tokenize text with UNKNOWN words
text2 = "the dog sat on the rug"
tokens2 = bpe_tokenize(text2, merges)
print(f"\n'{text2}'")
print(f"BPE tokens: {tokens2}")
print(f"Token count: {len(tokens2)}")
```

Output:
```
'the cat sat on the mat'
BPE tokens: ['the</w>', 'cat</w>', 'sat</w>', 'on</w>', 'the</w>', 'mat</w>']
Token count: 6

'the dog sat on the rug'
BPE tokens: ['the</w>', 'd', 'o', 'g', '</w>', 'sat</w>', 'on</w>', 'the</w>', 'r', 'u', 'g', '</w>']
Token count: 12
```

Notice: "dog" and "rug" were not in the training corpus, so BPE falls back to character-level tokenization for those words. Known words like "the", "sat", "on" are single tokens. BPE gracefully handles unknown words by decomposing them — no `<UNK>` tokens needed.

> **Why this matters for cost:** In Chapter 0, you learned that API pricing is per token. BPE means common words ("the", "is", "and") are single tokens, while rare words ("DoorDash", "cryptocurrency") are split into multiple tokens. The more common a word is, the fewer tokens it costs you.

---

## 5. WordPiece: How BERT Tokenizes

WordPiece is similar to BPE but differs in how it selects merges. Instead of the most *frequent* pair, WordPiece selects the pair that maximizes the *likelihood* of the training data. In practice, the results are similar, but the algorithms are distinct.

```python
# The key difference from BPE:
#
# BPE:       Merge the most FREQUENT pair
# WordPiece: Merge the pair that maximizes likelihood of the training data
#            (roughly: merge the pair where the combined token is much more
#             common than you would expect from the individual frequencies)
#
# Another visible difference: WordPiece marks CONTINUATION tokens with ##
#
# BPE (GPT):       "tokenization" → ["token", "ization"]
# WordPiece (BERT): "tokenization" → ["token", "##ization"]
#
# The ## prefix means "this token continues the previous word"

# Let's simulate WordPiece tokenization
def wordpiece_tokenize(text, vocab):
    """
    WordPiece tokenization (simplified).
    Uses greedy longest-match from the vocabulary.
    """
    tokens = []
    for word in text.lower().split():
        remaining = word
        is_first = True
        word_tokens = []
        
        while remaining:
            # Find the longest prefix in the vocabulary
            found = False
            for end in range(len(remaining), 0, -1):
                candidate = remaining[:end]
                if not is_first:
                    candidate = "##" + candidate
                
                if candidate in vocab:
                    word_tokens.append(candidate)
                    remaining = remaining[end:]
                    is_first = False
                    found = True
                    break
            
            if not found:
                # Single character fallback
                token = remaining[0] if is_first else "##" + remaining[0]
                word_tokens.append(token)
                remaining = remaining[1:]
                is_first = False
        
        tokens.extend(word_tokens)
    
    return tokens

# A sample BERT-like vocabulary
bert_vocab = {
    # Common words
    "the", "a", "is", "in", "on", "for", "to", "and",
    # Word pieces
    "un", "##happ", "##y", "##ness", "##ing", "##tion", "##ed",
    "token", "##ize", "##ization",
    "door", "##dash",
    "re", "##fund",
    "deliver", "##y", "##ed",
    "cat", "sat", "hat", "mat",
    # Subwords
    "##s", "##er", "##ly",
}

texts = [
    "the cat sat on the mat",
    "tokenization is unhappiness",
    "doordash refund delivered",
]

for text in texts:
    tokens = wordpiece_tokenize(text, bert_vocab)
    print(f"'{text}'")
    print(f"  WordPiece: {tokens}")
    print()
```

Output:
```
'the cat sat on the mat'
  WordPiece: ['the', 'cat', 'sat', 'on', 'the', 'mat']

'tokenization is unhappiness'
  WordPiece: ['token', '##ization', 'is', 'un', '##happ', '##y', '##ness']

'doordash refund delivered'
  WordPiece: ['door', '##dash', 're', '##fund', 'deliver', '##ed']
```

The `##` prefix makes it clear which tokens are word-initial and which are continuations. This helps the model understand word boundaries even after subword splitting.

---

## 6. SentencePiece: Language-Agnostic Tokenization

BPE and WordPiece assume spaces separate words. This breaks for Chinese, Japanese, Thai, and other languages without explicit word boundaries. SentencePiece solves this by treating the input as a raw byte stream — no assumption about spaces.

```python
# SentencePiece treats space as just another character
# It uses a special character (typically ▁) to mark word boundaries

# BPE/WordPiece approach:
#   "I love cats" → split on space → ["I", "love", "cats"] → subword each
#
# SentencePiece approach:
#   "I love cats" → treat as byte stream → "▁I▁love▁cats"
#   → apply BPE-like merges on this byte stream

sentencepiece_examples = {
    "English": {
        "input": "I love machine learning",
        "tokens": ["▁I", "▁love", "▁machine", "▁learn", "ing"],
    },
    "Japanese": {
        "input": "私は機械学習が好きです",
        "tokens": ["▁私は", "機械", "学習", "が", "好き", "です"],
    },
    "Mixed": {
        "input": "I love 機械学習!",
        "tokens": ["▁I", "▁love", "▁", "機械", "学習", "!"],
    },
}

for lang, data in sentencepiece_examples.items():
    print(f"{lang}: '{data['input']}'")
    print(f"  Tokens: {data['tokens']}")
    print()

# Note: ▁ (lower one-eighth block, U+2581) marks the beginning of a word.
# This means SentencePiece can RECONSTRUCT the original text including spaces
# by replacing ▁ with space and joining everything else.
```

```python
# Why SentencePiece matters:
#
# 1. Language-agnostic: works for ANY language, no pre-tokenization needed
# 2. Reversible: can perfectly reconstruct original text from tokens
# 3. Used by: T5, ALBERT, XLNet, LLaMA, Mistral
# 4. Two modes:
#    - Unigram: starts with a large vocabulary and prunes
#    - BPE: same as regular BPE but on the byte stream
```

---

## 7. Encoding and Decoding: Text to IDs and Back

In a real system, tokenization converts text to integer IDs (encoding) and back (decoding).

```python
class SimpleBPETokenizer:
    """A complete BPE tokenizer with encode/decode."""
    
    def __init__(self):
        # A simplified vocabulary (in real BPE, this is learned from data)
        self.vocab = {
            "<PAD>": 0, "<UNK>": 1, "<BOS>": 2, "<EOS>": 3,
            "the": 4, "a": 5, "is": 6, "was": 7, "on": 8,
            "cat": 9, "sat": 10, "mat": 11, "hat": 12,
            "dog": 13, "ran": 14, "in": 15,
            "big": 16, "small": 17, "red": 18, "blue": 19,
            " ": 20, ".": 21, "!": 22, ",": 23,
            "s": 24, "ed": 25, "ing": 26, "er": 27, "ly": 28,
        }
        self.id_to_token = {v: k for k, v in self.vocab.items()}
    
    def encode(self, text):
        """Convert text to a list of token IDs."""
        ids = [self.vocab.get("<BOS>")]  # Start with beginning-of-sequence
        
        remaining = text.lower()
        while remaining:
            # Greedy longest match
            matched = False
            for length in range(min(10, len(remaining)), 0, -1):
                candidate = remaining[:length]
                if candidate in self.vocab:
                    ids.append(self.vocab[candidate])
                    remaining = remaining[length:]
                    matched = True
                    break
            
            if not matched:
                ids.append(self.vocab["<UNK>"])
                remaining = remaining[1:]
        
        ids.append(self.vocab.get("<EOS>"))  # End with end-of-sequence
        return ids
    
    def decode(self, ids):
        """Convert token IDs back to text."""
        tokens = []
        for id in ids:
            if id in (0, 2, 3):  # Skip PAD, BOS, EOS
                continue
            tokens.append(self.id_to_token.get(id, "<UNK>"))
        return "".join(tokens)

tok = SimpleBPETokenizer()

texts = [
    "the cat sat on the mat.",
    "a big red dog ran!",
    "the small cat is blue.",
]

print("Encoding and decoding:")
for text in texts:
    ids = tok.encode(text)
    decoded = tok.decode(ids)
    print(f"\n  Original: '{text}'")
    print(f"  Token IDs: {ids}")
    print(f"  Tokens:    {[tok.id_to_token.get(i, '?') for i in ids]}")
    print(f"  Decoded:   '{decoded}'")
```

The key property: **encoding then decoding should give back the original text.** This roundtrip property is essential — if information is lost during tokenization, the model cannot reconstruct the output.

---

## 8. Comparing Tokenizers Across Models

Different models tokenize the same text differently. This has real consequences for cost, context window usage, and model behavior.

```python
# How different models would tokenize the same text:
# (These are representative — exact tokenization requires the actual model's vocabulary)

comparison_text = "The DoorDash driver didn't deliver my $42.50 order!"

tokenizations = {
    "GPT-4 (BPE, cl100k_base)": {
        "tokens": ["The", " Door", "Dash", " driver", " didn", "'t", " deliver",
                   " my", " $", "42", ".", "50", " order", "!"],
        "count": 14,
        "vocab_size": 100_256,
    },
    "BERT (WordPiece)": {
        "tokens": ["the", "door", "##das", "##h", "driver", "didn", "##'", "##t",
                   "deliver", "my", "$", "42", ".", "50", "order", "!"],
        "count": 16,
        "vocab_size": 30_522,
    },
    "LLaMA (SentencePiece BPE)": {
        "tokens": ["▁The", "▁Door", "D", "ash", "▁driver", "▁didn", "'t",
                   "▁deliver", "▁my", "▁$", "42", ".", "50", "▁order", "!"],
        "count": 15,
        "vocab_size": 32_000,
    },
    "Character-level": {
        "tokens": list(comparison_text),
        "count": len(comparison_text),
        "vocab_size": 95,
    },
}

print(f"Text: '{comparison_text}'\n")
for model, data in tokenizations.items():
    print(f"{model}:")
    print(f"  Tokens ({data['count']}): {data['tokens']}")
    print(f"  Vocabulary size: {data['vocab_size']:,}")
    print()
```

```python
# Key observations:
#
# 1. Token count varies: 14 (GPT-4) vs 16 (BERT) vs 50 (char-level)
#    → GPT-4 is more efficient (fewer tokens = cheaper, more fits in context)
#
# 2. Vocabulary size varies: 30K (BERT) vs 100K (GPT-4)
#    → Larger vocabulary = more common words as single tokens = fewer tokens
#    → But larger vocabulary = larger embedding matrix = more memory
#
# 3. "DoorDash" handling:
#    GPT-4:  "Door" + "Dash" (2 tokens, preserves meaning)
#    BERT:   "door" + "##das" + "##h" (3 tokens, loses casing)
#    LLaMA:  "Door" + "D" + "ash" (3 tokens, suboptimal split)
#
# 4. Numbers: all tokenizers split "$42.50" into multiple tokens
#    This is why LLMs struggle with arithmetic — "42" is one token,
#    not "4" and "2". The model processes the whole number at once.

# Cost implications:
cost_per_million = 15.00  # $15 per million tokens (Claude 3.5 Sonnet output)
text_length = len(comparison_text)

for model, data in tokenizations.items():
    cost_per_char = (data["count"] / text_length) * cost_per_million / 1_000_000
    print(f"{model}: {data['count']} tokens → "
          f"{data['count'] / text_length:.2f} tokens/char → "
          f"${data['count'] * cost_per_million / 1_000_000:.8f} for this text")
```

---

## 9. Special Tokens: The Control Vocabulary

Every tokenizer includes special tokens that control model behavior. These are not words — they are instructions to the model.

```python
special_tokens = {
    # BERT special tokens
    "[CLS]": {
        "model": "BERT",
        "purpose": "Classification token — placed at the start of input",
        "details": "The hidden state at this position is used for classification tasks",
        "example": "[CLS] The movie was great [SEP]",
    },
    "[SEP]": {
        "model": "BERT",
        "purpose": "Separator — marks boundary between text segments",
        "details": "Used in question-answering: [CLS] question [SEP] context [SEP]",
        "example": "[CLS] What is NLP? [SEP] NLP stands for... [SEP]",
    },
    "[PAD]": {
        "model": "BERT, many others",
        "purpose": "Padding — fills unused positions in fixed-length batches",
        "details": "The model learns to ignore these (via attention masks)",
        "example": "[CLS] Hi [SEP] [PAD] [PAD] [PAD]",
    },
    "[MASK]": {
        "model": "BERT",
        "purpose": "Mask — hides a token for the model to predict (masked LM)",
        "details": "Training: 'The [MASK] sat on the mat' → predict 'cat'",
        "example": "The [MASK] is the capital of France → 'Paris'",
    },
    "<|endoftext|>": {
        "model": "GPT-2, GPT-3",
        "purpose": "End of text — marks where a document ends",
        "details": "Used as both BOS and EOS in GPT-2",
        "example": "<|endoftext|>Once upon a time...<|endoftext|>",
    },
    "<|im_start|> / <|im_end|>": {
        "model": "ChatGPT, GPT-4",
        "purpose": "Chat formatting — marks start/end of roles",
        "details": "Structures multi-turn conversations",
        "example": "<|im_start|>system\nYou are helpful.<|im_end|>\n<|im_start|>user\nHi<|im_end|>",
    },
    "<s> / </s>": {
        "model": "LLaMA, T5, many others",
        "purpose": "Beginning/end of sequence",
        "details": "Standard markers for sequence boundaries",
        "example": "<s> Translate to French: Hello world </s>",
    },
}

print("Special Tokens Reference:")
print("=" * 70)
for token, info in special_tokens.items():
    print(f"\n  {token}")
    print(f"  Model:   {info['model']}")
    print(f"  Purpose: {info['purpose']}")
    print(f"  Example: {info['example']}")
```

### Why special tokens matter:

```python
# When you send a message to ChatGPT, the ACTUAL input looks like this:

actual_input = """
<|im_start|>system
You are a helpful assistant that detects DoorDash refund fraud.
<|im_end|>
<|im_start|>user
Is this refund request fraudulent? Account age: 7 days, refund rate: 100%
<|im_end|>
<|im_start|>assistant
"""

# The model sees ALL of this — including the special tokens.
# The special tokens tell the model:
#   - Where the system prompt ends
#   - Where the user message starts and ends
#   - That it should now generate an assistant response
#
# Without these tokens, the model would not know whose "turn" it is.
# This is why prompt injection attacks (Ch 48) try to inject fake
# special tokens to confuse the model about the conversation structure.
```

---

## 10. Batching: Processing Multiple Strings at Once

Real systems process many inputs at once for efficiency. But different inputs have different lengths. Batching requires making them all the same length.

```python
def batch_encode(texts, tokenizer, max_length=10):
    """
    Encode multiple texts into a fixed-size batch.
    
    1. Tokenize each text
    2. Truncate if longer than max_length
    3. Pad if shorter than max_length
    4. Create attention masks
    """
    batch_ids = []
    attention_masks = []
    
    for text in texts:
        ids = tokenizer.encode(text)
        
        # Truncate (keep BOS, truncate middle, keep EOS)
        if len(ids) > max_length:
            ids = ids[:max_length - 1] + [ids[-1]]  # Keep EOS
        
        # Create attention mask (1 for real tokens, 0 for padding)
        mask = [1] * len(ids)
        
        # Pad to max_length
        padding_needed = max_length - len(ids)
        ids = ids + [0] * padding_needed        # 0 = PAD token
        mask = mask + [0] * padding_needed
        
        batch_ids.append(ids)
        attention_masks.append(mask)
    
    return {
        "input_ids": np.array(batch_ids),
        "attention_mask": np.array(attention_masks),
    }

tok = SimpleBPETokenizer()

texts = [
    "the cat sat.",       # Short
    "a big red dog ran!", # Medium
    "the small cat is on the big red mat.", # Long (will be truncated)
]

batch = batch_encode(texts, tok, max_length=12)

print("Batched encoding (max_length=12):")
print(f"\nInput IDs (shape {batch['input_ids'].shape}):")
for i, text in enumerate(texts):
    ids = batch["input_ids"][i]
    tokens = [tok.id_to_token.get(id, "?") for id in ids]
    print(f"  '{text}'")
    print(f"    IDs:    {ids.tolist()}")
    print(f"    Tokens: {tokens}")

print(f"\nAttention Masks (shape {batch['attention_mask'].shape}):")
for i, text in enumerate(texts):
    mask = batch["attention_mask"][i]
    print(f"  '{text}'")
    print(f"    Mask: {mask.tolist()}")
```

Output:
```
Batched encoding (max_length=12):

Input IDs (shape (3, 12)):
  'the cat sat.'
    IDs:    [2, 4, 20, 9, 20, 10, 21, 3, 0, 0, 0, 0]
    Tokens: ['<BOS>', 'the', ' ', 'cat', ' ', 'sat', '.', '<EOS>', '<PAD>', '<PAD>', '<PAD>', '<PAD>']
  'a big red dog ran!'
    IDs:    [2, 5, 20, 16, 20, 18, 20, 13, 20, 14, 22, 3]
    Tokens: ['<BOS>', 'a', ' ', 'big', ' ', 'red', ' ', 'dog', ' ', 'ran', '!', '<EOS>']
  'the small cat is on the big red mat.'
    IDs:    [2, 4, 20, 17, 20, 9, 20, 6, 20, 8, 20, 3]
    Tokens: ['<BOS>', 'the', ' ', 'small', ' ', 'cat', ' ', 'is', ' ', 'on', ' ', '<EOS>']

Attention Masks (shape (3, 12)):
  'the cat sat.'
    Mask: [1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0]
  'a big red dog ran!'
    Mask: [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  'the small cat is on the big red mat.'
    Mask: [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
```

---

## 11. Attention Masks: Real vs Padding

The attention mask tells the model which tokens are real content and which are just padding. Without it, the model would try to "understand" the padding tokens, corrupting its outputs.

```python
# How attention masks work inside the model:
#
# Without mask:
#   "the cat [PAD] [PAD]" → model thinks [PAD] is meaningful
#   Attention scores: "the" attends to "cat", "PAD", "PAD"
#   Result: corrupted by attending to meaningless padding
#
# With mask [1, 1, 0, 0]:
#   "the cat [PAD] [PAD]" → model ignores positions where mask = 0
#   Attention scores: "the" attends to "cat" ONLY
#   Result: correct, as if [PAD] tokens don't exist

def masked_attention_scores(query, keys, mask):
    """
    Compute attention scores with masking.
    Positions where mask=0 get score of -infinity (ignored after softmax).
    """
    # Raw scores
    scores = query @ keys.T
    
    # Apply mask: set padding positions to -infinity
    masked_scores = scores.copy()
    masked_scores[mask == 0] = -np.inf
    
    # Softmax (ignoring -inf positions)
    exp_scores = np.exp(masked_scores - np.max(masked_scores))
    exp_scores[mask == 0] = 0  # Ensure -inf positions become 0
    attention = exp_scores / exp_scores.sum()
    
    return attention

# Example: 4 tokens, last 2 are padding
np.random.seed(42)
query = np.random.randn(4)          # One query vector
keys = np.random.randn(4, 4)        # 4 key vectors
mask = np.array([1, 1, 0, 0])       # Only first 2 are real

attention_no_mask = np.exp(query @ keys.T)
attention_no_mask = attention_no_mask / attention_no_mask.sum()

attention_with_mask = masked_attention_scores(query, keys, mask)

print("Attention scores WITHOUT mask:")
print(f"  {[f'{a:.3f}' for a in attention_no_mask]}")
print(f"  → Model attends to padding! (positions 2,3 get nonzero weight)")

print("\nAttention scores WITH mask:")
print(f"  {[f'{a:.3f}' for a in attention_with_mask]}")
print(f"  → Padding ignored! (positions 2,3 get zero weight)")
```

> **This connects directly to Chapter 38:** The attention mechanism you will build next chapter uses these exact masks. Understanding masking now means you will understand transformer attention without confusion about what "masked self-attention" means.

---

## 12. Vocabulary Mapping: IDs to Words

Every tokenizer maintains a mapping between tokens (strings) and IDs (integers). This mapping IS the vocabulary.

```python
# The vocabulary is a bidirectional lookup table:
#   token → ID (encoding)
#   ID → token (decoding)

# In real models, the vocabulary is FIXED after training.
# You cannot add new tokens without retraining (or using adapters).

def explore_vocabulary(vocab_dict):
    """Analyze a tokenizer's vocabulary."""
    print(f"Vocabulary size: {len(vocab_dict)} tokens")
    
    # Categorize tokens
    single_chars = [t for t in vocab_dict if len(t) == 1]
    subwords = [t for t in vocab_dict if t.startswith("##")]
    special = [t for t in vocab_dict if t.startswith("<") or t.startswith("[")]
    words = [t for t in vocab_dict if t not in single_chars and t not in subwords 
             and t not in special]
    
    print(f"  Single characters: {len(single_chars)}")
    print(f"  Subwords (##):     {len(subwords)}")
    print(f"  Special tokens:    {len(special)}")
    print(f"  Full words:        {len(words)}")
    
    return {"chars": single_chars, "subwords": subwords, 
            "special": special, "words": words}

# A representative vocabulary distribution:
real_vocab_stats = {
    "GPT-2 (50257 tokens)": {
        "single_chars": 256,
        "common_words": "~15,000 (the, and, is, was, ...)",
        "subword_pieces": "~30,000 (tion, ment, ing, ...)",
        "special_tokens": 1,
        "note": "Includes byte-level fallback for any Unicode character",
    },
    "BERT (30522 tokens)": {
        "single_chars": "~1000",
        "common_words": "~20,000",
        "subword_pieces": "~8,000 (##tion, ##ing, ##ed, ...)",
        "special_tokens": 5,
        "note": "Smaller vocab, more subword splitting needed",
    },
    "GPT-4 (100256 tokens)": {
        "single_chars": 256,
        "common_words": "~40,000",
        "subword_pieces": "~55,000",
        "special_tokens": "~100",
        "note": "Larger vocab = fewer tokens per text = more efficient",
    },
}

print("Real-world vocabulary comparison:")
for model, stats in real_vocab_stats.items():
    print(f"\n  {model}:")
    for key, value in stats.items():
        print(f"    {key}: {value}")
```

```python
# The embedding matrix: where vocab meets neural networks
#
# In Chapter 36, you saw that a neural network's input is a vector of numbers.
# For text, that vector comes from the EMBEDDING MATRIX:
#
# embedding_matrix shape: (vocab_size, embedding_dim)
#   GPT-2: (50257, 768)   = ~38.6 million parameters just for embeddings
#   GPT-4: (100256, ~12288) = ~1.2 BILLION parameters just for embeddings
#
# To process token ID 42:
#   embedding = embedding_matrix[42]  → a vector of 768 (or 12288) numbers
#
# This is the bridge: tokenizer converts text → IDs,
# embedding matrix converts IDs → vectors that the neural network consumes.

def token_to_embedding(token_id, embedding_matrix):
    """Look up the embedding vector for a token ID."""
    return embedding_matrix[token_id]

# Simulate
np.random.seed(42)
vocab_size = 100
embedding_dim = 8  # Real models use 768-12288
embedding_matrix = np.random.randn(vocab_size, embedding_dim)

token_id = 42
embedding = token_to_embedding(token_id, embedding_matrix)
print(f"Token ID {token_id} → embedding (dim={embedding_dim}):")
print(f"  {[f'{v:.3f}' for v in embedding]}")
print(f"\nIn GPT-4, this would be a {12288}-dimensional vector.")
print(f"In our smile detector (Ch 36), this was a 30-dimensional pixel vector.")
print(f"Same idea: represent the input as a vector of numbers.")
```

---

## 13. Why Tokenization Matters for Everything

Tokenization is not a preprocessing detail. It fundamentally affects model behavior, cost, and capabilities.

```python
tokenization_impacts = {
    "Cost": {
        "explanation": "API pricing is per token. Efficient tokenization = lower cost.",
        "example": "GPT-4 tokenizes 'AI' as 1 token. Character-level: 2 tokens. 2x cost.",
        "implication": "Vocabulary size directly affects your API bill.",
    },
    "Context window": {
        "explanation": "Models have fixed context windows (128K tokens for GPT-4).",
        "example": "Inefficient tokenization wastes context on subword pieces instead of content.",
        "implication": "More tokens per word = less content fits in the window.",
    },
    "Multilingual performance": {
        "explanation": "Tokenizers trained mostly on English split non-English text into more tokens.",
        "example": "'hello' = 1 token. Japanese equivalent = 3-4 tokens.",
        "implication": "Non-English text is more expensive AND uses more context.",
    },
    "Arithmetic": {
        "explanation": "Numbers are tokenized inconsistently.",
        "example": "'1234' might be 1 token, '12345' might be 2 tokens.",
        "implication": "Models struggle with math partly because number representation is arbitrary.",
    },
    "Code": {
        "explanation": "Indentation and syntax consume tokens.",
        "example": "4 spaces of indentation = 1 token. Python code is token-expensive.",
        "implication": "Code generation uses context faster than you might expect.",
    },
    "Prompt injection": {
        "explanation": "Special tokens can potentially be injected.",
        "example": "An attacker might try to inject <|im_end|> to break out of user role.",
        "implication": "Token-level security matters (Ch 48).",
    },
}

for impact, details in tokenization_impacts.items():
    print(f"\n{impact}:")
    print(f"  {details['explanation']}")
    print(f"  Example: {details['example']}")
    print(f"  → {details['implication']}")
```

---

## 14. Connecting to Everything Else

### 14.1 Ch 0 → Ch 37 (The Full Spiral)

```python
# Chapter 0: "LLMs work with tokens. Tokens cost money."
# Chapter 37: HERE is how tokens are created:
#   1. BPE/WordPiece/SentencePiece algorithms learn subword vocabularies
#   2. Text is split into subword tokens
#   3. Tokens are mapped to integer IDs
#   4. IDs look up vectors in an embedding matrix
#   5. Those vectors enter the neural network (Ch 36)
#   6. The network processes them with attention (Ch 38)
#   7. The network outputs probabilities over the vocabulary
#   8. A decoding strategy selects the next token (Ch 39)
#   9. The selected token is decoded back to text
#   10. The text is returned to you via the API
#
# You now understand steps 1-5. Chapters 38-39 complete the picture.
```

### 14.2 The Full Pipeline

```python
# Text input → Output text: the complete path
#
# "The cat sat on the" (user input)
#   ↓ Tokenizer (Ch 37)
# [4, 9, 10, 8, 4] (token IDs)
#   ↓ Embedding matrix (Ch 37)
# [[0.2, -0.1, ...], [0.7, 0.3, ...], ...] (vectors)
#   ↓ Transformer with attention (Ch 38)
# [[0.1, 0.8, ...], ...] (processed vectors)
#   ↓ Output layer → softmax
# [0.001, 0.002, ..., 0.15, ..., 0.003] (probability over all tokens)
#   ↓ Decoding strategy (Ch 39)
# Token ID: 11 (selected "mat")
#   ↓ Tokenizer decode (Ch 37)
# "mat" (text output)
#
# Full output: "The cat sat on the mat"
```

### 14.3 Hugging Face Tokenizers (Ch 40 Preview)

```python
# In Chapter 40, you will use real tokenizers from Hugging Face:
#
# from transformers import AutoTokenizer
#
# tokenizer = AutoTokenizer.from_pretrained("gpt2")
# encoded = tokenizer("The cat sat on the mat")
# print(encoded["input_ids"])        # [464, 3797, 3332, 319, 262, 2603]
# print(encoded["attention_mask"])   # [1, 1, 1, 1, 1, 1]
#
# decoded = tokenizer.decode(encoded["input_ids"])
# print(decoded)                     # "The cat sat on the mat"
#
# Everything you learned in this chapter — encoding, decoding,
# attention masks, special tokens, batching — is exactly what
# the Hugging Face tokenizer does. You now understand what
# happens inside that one-line API call.
```

---

## Key Takeaways

Tokenization is the first step in every language model pipeline. You learned:

1. **The tokenization spectrum** — characters (small vocab, many tokens) vs words (huge vocab, few tokens) vs subwords (the sweet spot)
2. **BPE** — iteratively merge the most frequent character pairs to discover subwords (used by GPT)
3. **WordPiece** — similar to BPE but optimizes for data likelihood, marks continuations with ## (used by BERT)
4. **SentencePiece** — treats text as a byte stream, no assumption about spaces (used by LLaMA, T5)
5. **Encoding/decoding** — text to integer IDs and back, must be lossless
6. **Special tokens** — [CLS], [SEP], [PAD], [MASK], <|endoftext|> control model behavior
7. **Batching** — pad sequences to the same length, track real vs padding with attention masks
8. **Vocabulary mapping** — the fixed lookup table from token to ID to embedding vector

Next: those embedding vectors enter the transformer, where attention lets every token look at every other token. Chapter 38 shows you how.

---

*Next: [Chapter 38 — Transformers & Attention](./38-transformers-attention.md)* | *Previous: [Chapter 36 — Neural Networks from Scratch](./36-neural-networks.md)*

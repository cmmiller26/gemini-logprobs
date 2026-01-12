# Probability Pulse - Architecture & Logic

## Overview

Probability Pulse is an educational tool that demonstrates how Large Language Models (LLMs) predict the next token. It uses the Gemini API's `logprobs` feature to retrieve the top token alternatives and their probabilities.

---

## How LLMs Determine the Next Token

### The Fundamental Process

LLMs are **autoregressive models** - they generate text one token at a time by predicting the most likely next token given all previous tokens. Here's the conceptual pipeline:

```
Input: "The future of AI is"
         ↓
┌─────────────────────────────────────────────────────────┐
│  1. TOKENIZATION                                        │
│     "The future of AI is" → [464, 2003, 286, 9552, 318] │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  2. EMBEDDING LOOKUP                                    │
│     Each token ID → dense vector (e.g., 768 dimensions) │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  3. TRANSFORMER LAYERS (x N)                            │
│     Self-attention + Feed-forward networks              │
│     Contextualize each position with all previous       │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  4. FINAL HIDDEN STATE                                  │
│     Last position's output = context-aware encoding     │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  5. LOGITS (Raw Scores)                                 │
│     Linear projection to vocabulary size                │
│     e.g., [2.1, -1.3, 5.7, 0.2, ...] for each token    │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│  6. SOFTMAX → PROBABILITIES                             │
│     exp(logit_i) / Σ exp(logit_j) for all j            │
│     → [0.02, 0.001, 0.89, 0.05, ...]                   │
└─────────────────────────────────────────────────────────┘
         ↓
        OUTPUT: "bright" (highest probability = 0.89)
```

### Log Probabilities (Logprobs)

Instead of raw probabilities, APIs return **log probabilities** because:

| Raw Probability | Log Probability | Why Log is Better |
|-----------------|-----------------|-------------------|
| 0.89 | -0.117 | More precision near 1.0 |
| 0.0001 | -9.21 | Avoids floating-point underflow |
| 2.3e-18 | -40.5 | Handles extremely rare tokens |

**Conversion:** `probability = exp(log_probability)`

---

## Current Implementation Logic

### API Flow

```
User Input (seed sentence)
       ↓
Frontend (WordBuilder.tsx)
       ↓
POST /api/next-token { prefix: "The future of AI is" }
       ↓
Backend (llm.ts) → Gemini API (gemini-2.0-flash)
       ↓
Returns: [{ token: "bright", prob: 0.894 }, { token: "complex", prob: 0.05 }, ...]
       ↓
Frontend displays alternatives in game UI
```

### Core Function: `getNextToken(prefix: string)`

**Location:** `/src/lib/llm.ts`

#### Step 1: API Request

```typescript
const response = await fetch(url, {
    body: JSON.stringify({
        contents: [{ parts: [{ text: prefix }] }],
        generationConfig: {
            responseLogprobs: true,  // Enable log probability output
            logprobs: 20,            // Return top 20 alternatives
            maxOutputTokens: 1       // Only predict the next single token
        }
    })
});
```

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `responseLogprobs` | `true` | Request log probabilities in response |
| `logprobs` | `20` | Number of top alternatives to return |
| `maxOutputTokens` | `1` | Generate only the immediate next token |

#### Step 2: Extract Top Candidates

```typescript
const topCandidates = candidate.logprobsResult.topCandidates?.[0]?.candidates || [];
// Returns: [{ token: "bright", logProbability: -0.112 }, ...]
```

#### Step 3: Softmax Normalization

The raw logprobs must be converted to probabilities that sum to 1. We use **numerically stable softmax**:

```typescript
function logprobsToProbs(candidates) {
    // 1. Extract log probabilities
    const logprobs = candidates.map(c => c.logProbability);
    
    // 2. Subtract max for numerical stability (prevents overflow)
    const maxLogProb = Math.max(...logprobs);
    
    // 3. Compute exp(logprob - max)
    const expValues = logprobs.map(lp => Math.exp(lp - maxLogProb));
    
    // 4. Normalize: probability = exp_i / Σ(exp_j)
    const sumExp = expValues.reduce((sum, val) => sum + val, 0);
    
    return candidates.map((c, i) => ({
        token: c.token,
        probability: expValues[i] / sumExp
    }));
}
```

> [!NOTE]
> The max subtraction trick ensures the largest exp value is `exp(0) = 1`, preventing numerical overflow when dealing with large negative logprobs.

#### Step 4: Add `<OTHER>` Category

Since the API only returns the top N tokens, there's remaining probability mass for all other vocabulary tokens:

```typescript
function addOtherCategory(tokens, minProbability = 0.01) {
    const totalProb = tokens.reduce((sum, t) => sum + t.probability, 0);
    const remainingProb = Math.max(0, 1.0 - totalProb);
    
    if (remainingProb >= minProbability) {
        return [...tokens, {
            token: "<OTHER>",
            probability: remainingProb,
            is_other: true
        }];
    }
    return tokens;
}
```

---

## Sampling Strategies (How LLMs Pick the Next Token)

Real LLM applications don't always pick the highest probability token. Here are common strategies:

### 1. Greedy Decoding (Argmax)
Pick the token with the highest probability every time.
- **Pros:** Deterministic, fast
- **Cons:** Often repetitive, boring output

### 2. Temperature Sampling
Scale logits before softmax: `softmax(logits / T)`

| Temperature | Effect |
|-------------|--------|
| T < 1.0 | Peaky distribution (more deterministic) |
| T = 1.0 | Original distribution |
| T > 1.0 | Flatter distribution (more random) |

### 3. Top-K Sampling
Only consider the top K tokens, then sample from that subset.

### 4. Top-P (Nucleus) Sampling
Only consider tokens whose cumulative probability reaches P (e.g., 0.95).

### 5. Beam Search
Maintain multiple candidate sequences and pick the best overall path.

---

## Why Word-by-Word Building Can Lead to Incoherence

When building sentences token-by-token in this app:

1. User picks a token based on probability
2. That token is appended to the prefix
3. **NEW** prediction is made for the updated prefix
4. But each token was optimized for the **PREVIOUS** context, not the cumulative sentence

This is essentially **greedy decoding** - always picking locally optimal tokens doesn't guarantee globally coherent text.

> [!IMPORTANT]
> This is a key educational insight: the "best" next token at each step doesn't necessarily lead to the best overall sentence.

---

## File Structure

```
src/
├── app/
│   ├── api/
│   │   └── next-token/route.ts  # API endpoint
│   ├── page.tsx                 # Main page
│   └── globals.css              # Styles
├── components/
│   ├── WordBuilder.tsx          # Main orchestrator
│   └── games/
│       ├── ManualSelector.tsx   # Direct word selection
│       ├── SpinningWheel.tsx    # Wheel game
│       ├── SlotMachine.tsx      # Slot machine game
│       └── Plinko.tsx           # Plinko game
└── lib/
    ├── llm.ts                   # Gemini API functions
    └── types.ts                 # TypeScript interfaces
```

---

## API Reference

### POST /api/next-token

**Request:**
```json
{
  "prefix": "The future of AI is"
}
```

**Response:**
```json
{
  "alternatives": [
    { "token": "bright", "probability": 0.894, "log_probability": -0.112, "is_other": false },
    { "token": "complex", "probability": 0.052, "log_probability": -2.96, "is_other": false },
    { "token": "<OTHER>", "probability": 0.054, "log_probability": -2.92, "is_other": true }
  ]
}
```

---

## Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **Token** | Smallest unit of text (subword, word, or punctuation) |
| **Logit** | Raw, unnormalized score from the model |
| **Log Probability** | Log of the probability (more numerically stable) |
| **Softmax** | Converts logits to probabilities that sum to 1 |
| **Autoregressive** | Each prediction depends on all previous tokens |
| **Vocabulary** | All possible tokens the model knows (~32K-256K tokens) |

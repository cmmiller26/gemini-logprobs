# LLM Token Wheel - Functions & Utilities Specification

## Overview

This document provides detailed specifications for all utility functions and
API client code. This includes the Gemini API client (`lib/api/gemini.ts`),
probability/math utilities (`lib/utils/probabilities.ts`), wedge calculation
utilities (`lib/utils/wedges.ts`), and session management
(`lib/api/session.ts`).

---

## Table of Contents

1. [Gemini API Client (`lib/api/gemini.ts`)](#1-gemini-api-client-libapigeminits)
2. [Probability Utilities (`lib/utils/probabilities.ts`)](#2-probability-utilities-libutilsprobabilitiests)
3. [Wedge Utilities (`lib/utils/wedges.ts`)](#3-wedge-utilities-libutilswedgests)
4. [Session Management (`lib/api/session.ts`)](#4-session-management-libapisessionts)
5. [Validation Utilities (`lib/utils/validation.ts`)](#5-validation-utilities-libutilsvalidationts)

---

## 1. Gemini API Client (`lib/api/gemini.ts`)

**Overview:**

Wrapper around Google's Generative AI SDK for Gemini API calls. Handles
authentication, error handling, retry logic, and response parsing.

**Dependencies:**

```typescript
import {
  GoogleGenerativeAI,
  GenerateContentRequest,
} from "@google/generative-ai";
```

**Types:**

```typescript
/**
 * Configuration for Gemini API token probability request
 */
export interface LogprobsConfig {
  temperature?: number; // Sampling temperature [0.0, 2.0], default 1.0
  logprobs_count?: number; // Number of top alternatives [1, 20], default 20
  max_output_tokens?: number; // Max tokens to generate, default 1
}

/**
 * Raw candidate from Gemini API logprobs response
 */
export interface GeminiCandidate {
  token: string; // Token text
  log_probability: number; // Log probability
}

/**
 * Parsed logprobs response from Gemini
 */
export interface GeminiLogprobsResponse {
  chosen_token: string; // The token Gemini selected
  top_candidates: GeminiCandidate[]; // Top-k alternative tokens
  model: string; // Model used (e.g., "gemini-2.0-flash")
}
```

**Functions:**

### 1.1 `initializeGeminiClient()`

Initialize the Gemini API client with authentication.

**Signature:**

```typescript
function initializeGeminiClient(): GoogleGenerativeAI;
```

**Implementation:**

```typescript
let geminiClient: GoogleGenerativeAI | null = null;

export function initializeGeminiClient(): GoogleGenerativeAI {
  if (geminiClient) {
    return geminiClient;
  }

  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) {
    throw new Error(
      "GEMINI_API_KEY environment variable is not set. " +
        "Please add it to your .env.local file."
    );
  }

  geminiClient = new GoogleGenerativeAI(apiKey);
  return geminiClient;
}
```

**Usage:**

```typescript
const client = initializeGeminiClient();
```

**Error Handling:**

- Throws `Error` if `GEMINI_API_KEY` is not set

---

### 1.2 `getTokenLogprobs()`

Get log probabilities for the next token given a context.

**Signature:**

```typescript
async function getTokenLogprobs(
  context: string,
  config?: LogprobsConfig
): Promise<GeminiLogprobsResponse>;
```

**Parameters:**

- `context` (string): The input text context
- `config` (LogprobsConfig, optional): Configuration options
  - `temperature`: Sampling temperature, default 1.0
  - `logprobs_count`: Number of top alternatives, default 20
  - `max_output_tokens`: Max tokens to generate, default 1

**Returns:**

- `Promise<GeminiLogprobsResponse>`: Parsed logprobs response

**Implementation:**

```typescript
export async function getTokenLogprobs(
  context: string,
  config: LogprobsConfig = {}
): Promise<GeminiLogprobsResponse> {
  const client = initializeGeminiClient();
  const model = client.getGenerativeModel({ model: "gemini-2.0-flash" });

  const {
    temperature = 1.0,
    logprobs_count = 20,
    max_output_tokens = 1,
  } = config;

  try {
    const result = await model.generateContent({
      contents: [{ role: "user", parts: [{ text: context }] }],
      generationConfig: {
        temperature,
        maxOutputTokens: max_output_tokens,
        candidateCount: 1,
        // Enable logprobs
        responseLogprobs: true,
        logprobs: logprobs_count,
      },
    });

    const response = result.response;
    const candidate = response.candidates?.[0];

    if (!candidate) {
      throw new Error("No candidate in Gemini response");
    }

    // Extract chosen token
    const chosen_token = candidate.content.parts[0].text || "";

    // Extract logprobs
    const logprobs_result = candidate.logprobsResult;
    if (!logprobs_result) {
      throw new Error("No logprobs in Gemini response");
    }

    // Parse top candidates from first token position
    const top_candidates: GeminiCandidate[] =
      logprobs_result.topCandidates[0]?.candidates.map((c) => ({
        token: c.token,
        log_probability: c.logProbability,
      })) || [];

    return {
      chosen_token,
      top_candidates,
      model: "gemini-2.0-flash",
    };
  } catch (error) {
    console.error("Gemini API error:", error);

    // Parse error for better messages
    if (error instanceof Error) {
      if (error.message.includes("quota")) {
        throw new Error("Gemini API quota exceeded. Please try again later.");
      }
      if (error.message.includes("API key")) {
        throw new Error("Invalid Gemini API key. Check your configuration.");
      }
    }

    throw new Error(`Failed to get logprobs from Gemini: ${error}`);
  }
}
```

**Error Handling:**

- Throws `Error` if API call fails
- Throws `Error` if response is malformed
- Throws `Error` if quota exceeded
- Throws `Error` if API key invalid

**Usage Example:**

```typescript
const response = await getTokenLogprobs("The cat sat on the", {
  temperature: 1.0,
  logprobs_count: 20,
});

console.log("Chosen token:", response.chosen_token);
console.log("Top alternatives:", response.top_candidates);
```

---

### 1.3 `sampleFromDistribution()`

Sample a single token from Gemini (for "other" category selection).

**Signature:**

```typescript
async function sampleFromDistribution(
  context: string,
  temperature?: number
): Promise<string>;
```

**Parameters:**

- `context` (string): The input text context
- `temperature` (number, optional): Sampling temperature, default 1.0

**Returns:**

- `Promise<string>`: The sampled token text

**Implementation:**

```typescript
export async function sampleFromDistribution(
  context: string,
  temperature: number = 1.0
): Promise<string> {
  const client = initializeGeminiClient();
  const model = client.getGenerativeModel({ model: "gemini-2.0-flash" });

  try {
    const result = await model.generateContent({
      contents: [{ role: "user", parts: [{ text: context }] }],
      generationConfig: {
        temperature,
        maxOutputTokens: 1,
        candidateCount: 1,
        // No logprobs needed, just sample
        responseLogprobs: false,
      },
    });

    const response = result.response;
    const text = response.candidates?.[0]?.content.parts[0].text;

    if (!text) {
      throw new Error("No text in Gemini response");
    }

    return text;
  } catch (error) {
    console.error("Gemini sampling error:", error);
    throw new Error(`Failed to sample from Gemini: ${error}`);
  }
}
```

**Error Handling:**

- Throws `Error` if API call fails
- Throws `Error` if response is empty

**Usage Example:**

```typescript
// When user selects "other" category, sample from full distribution
const token = await sampleFromDistribution("The cat sat on the");
console.log("Sampled token:", token);
```

---

### 1.4 `testGeminiConnection()`

Test Gemini API connection (used for health checks).

**Signature:**

```typescript
async function testGeminiConnection(): Promise<boolean>;
```

**Returns:**

- `Promise<boolean>`: True if connection successful, false otherwise

**Implementation:**

```typescript
export async function testGeminiConnection(): Promise<boolean> {
  try {
    const client = initializeGeminiClient();
    const model = client.getGenerativeModel({ model: "gemini-2.0-flash" });

    // Simple test request with minimal tokens
    const result = await model.generateContent({
      contents: [{ role: "user", parts: [{ text: "test" }] }],
      generationConfig: {
        maxOutputTokens: 1,
        candidateCount: 1,
      },
    });

    return !!result.response.candidates?.[0]?.content.parts[0].text;
  } catch (error) {
    console.error("Gemini connection test failed:", error);
    return false;
  }
}
```

**Usage Example:**

```typescript
const isConnected = await testGeminiConnection();
if (!isConnected) {
  console.error("Gemini API is not available");
}
```

---

## 2. Probability Utilities (`lib/utils/probabilities.ts`)

**Overview:**

Mathematical utilities for converting log probabilities to probabilities,
normalizing distributions, and adding the "other" category.

**Types:**

```typescript
import { Token, GeminiCandidate } from "@/lib/types";

/**
 * Normalized probability distribution
 */
export interface ProbabilityDistribution {
  tokens: Token[];
  total_probability: number; // Should be 1.0 after normalization
}
```

**Functions:**

### 2.1 `logprobsToProbs()`

Convert log probabilities to normalized probabilities using softmax.

**Signature:**

```typescript
function logprobsToProbs(candidates: GeminiCandidate[]): Token[];
```

**Parameters:**

- `candidates` (GeminiCandidate[]): Array of tokens with log probabilities

**Returns:**

- `Token[]`: Array of tokens with normalized probabilities

**Algorithm:**
The softmax function converts log probabilities to probabilities:

```text
For each token:
  1. Subtract max log prob for numerical stability
  2. exp_value = e^(log_prob - max_log_prob)
  3. probability = exp_value / sum(all exp_values)
```

**Implementation:**

```typescript
export function logprobsToProbs(candidates: GeminiCandidate[]): Token[] {
  if (candidates.length === 0) {
    return [];
  }

  // Extract log probabilities
  const logprobs = candidates.map((c) => c.log_probability);

  // Find max for numerical stability (prevents overflow)
  const maxLogProb = Math.max(...logprobs);

  // Compute exp values (shifted by max)
  const expValues = logprobs.map((lp) => Math.exp(lp - maxLogProb));

  // Sum of exp values (partition function)
  const sumExp = expValues.reduce((sum, val) => sum + val, 0);

  // Convert to probabilities and create Token objects
  return candidates.map((candidate, i) => ({
    token: candidate.token,
    token_id: i, // Use index as ID
    probability: expValues[i] / sumExp,
    log_probability: candidate.log_probability,
    is_other: false,
  }));
}
```

**Mathematical Notes:**

- Subtracting `maxLogProb` prevents numerical overflow for large negative log probs
- The result is a proper probability distribution (sums to 1.0)
- Preserves relative ordering of probabilities

**Usage Example:**

```typescript
const geminiResponse = await getTokenLogprobs("The cat sat on the");
const tokens = logprobsToProbs(geminiResponse.top_candidates);

console.log(tokens);
// [
//   { token: " floor", probability: 0.18, ... },
//   { token: " mat", probability: 0.15, ... },
//   ...
// ]
```

**Testing:**

```typescript
// Test: Sum of probabilities should be ~1.0
const sum = tokens.reduce((s, t) => s + t.probability, 0);
expect(sum).toBeCloseTo(1.0, 5);
```

---

### 2.2 `addOtherCategory()`

Add "other" category for remaining probability mass.

**Signature:**

```typescript
function addOtherCategory(tokens: Token[], minProbability?: number): Token[];
```

**Parameters:**

- `tokens` (Token[]): Array of tokens with probabilities
- `minProbability` (number, optional): Minimum probability for "other"
  category, default 0.01 (1%)

**Returns:**

- `Token[]`: Array of tokens including "other" category if applicable

**Algorithm:**

```text
1. Sum probabilities of all tokens
2. remaining_prob = 1.0 - sum
3. If remaining_prob >= minProbability:
   - Add "other" token with remaining_prob
4. Return tokens (with or without "other")
```

**Implementation:**

```typescript
export function addOtherCategory(
  tokens: Token[],
  minProbability: number = 0.01
): Token[] {
  // Calculate sum of current probabilities
  const totalProb = tokens.reduce((sum, t) => sum + t.probability, 0);

  // Calculate remaining probability
  const remainingProb = Math.max(0, 1.0 - totalProb);

  // Only add "other" if remaining probability is significant
  if (remainingProb >= minProbability) {
    const otherToken: Token = {
      token: "<OTHER>",
      token_id: -1,
      probability: remainingProb,
      log_probability: Math.log(remainingProb), // Convert back to log prob
      is_other: true,
    };

    return [...tokens, otherToken];
  }

  return tokens;
}
```

**Edge Cases:**

- If sum > 1.0 (due to floating point errors), remaining_prob = 0
- If remaining_prob < minProbability, no "other" token added
- "other" token always has token_id = -1

**Usage Example:**

```typescript
const tokens = logprobsToProbs(geminiResponse.top_candidates);
const tokensWithOther = addOtherCategory(tokens, 0.01);

// Check if "other" was added
const hasOther = tokensWithOther.some((t) => t.is_other);
console.log("Has other category:", hasOther);
```

**Testing:**

```typescript
// Test: Total probability should be 1.0
const sum = tokensWithOther.reduce((s, t) => s + t.probability, 0);
expect(sum).toBeCloseTo(1.0, 5);

// Test: "other" token should have token_id = -1
const otherToken = tokensWithOther.find((t) => t.is_other);
if (otherToken) {
  expect(otherToken.token_id).toBe(-1);
}
```

---

### 2.3 `normalizeProbs()`

Normalize probabilities to sum to exactly 1.0 (fix floating point errors).

**Signature:**

```typescript
function normalizeProbs(tokens: Token[]): Token[];
```

**Parameters:**

- `tokens` (Token[]): Array of tokens with probabilities

**Returns:**

- `Token[]`: Array of tokens with normalized probabilities

**Algorithm:**

```text
1. sum = sum of all probabilities
2. For each token:
   normalized_prob = probability / sum
```

**Implementation:**

```typescript
export function normalizeProbs(tokens: Token[]): Token[] {
  const sum = tokens.reduce((s, t) => s + t.probability, 0);

  if (sum === 0) {
    throw new Error("Cannot normalize: sum of probabilities is 0");
  }

  return tokens.map((token) => ({
    ...token,
    probability: token.probability / sum,
  }));
}
```

**Usage Example:**

```typescript
// After adding "other", probabilities might not sum to exactly 1.0
const tokensWithOther = addOtherCategory(tokens);
const normalized = normalizeProbs(tokensWithOther);

const sum = normalized.reduce((s, t) => s + t.probability, 0);
console.log("Sum:", sum); // Exactly 1.0
```

---

### 2.4 `sortByProbability()`

Sort tokens by probability (descending order).

**Signature:**

```typescript
function sortByProbability(tokens: Token[]): Token[];
```

**Parameters:**

- `tokens` (Token[]): Array of tokens

**Returns:**

- `Token[]`: Sorted array (highest probability first)

**Implementation:**

```typescript
export function sortByProbability(tokens: Token[]): Token[] {
  return [...tokens].sort((a, b) => {
    // "other" always goes last
    if (a.is_other) return 1;
    if (b.is_other) return -1;

    // Sort by probability descending
    return b.probability - a.probability;
  });
}
```

**Usage Example:**

```typescript
const sorted = sortByProbability(tokensWithOther);
console.log(sorted[0]); // Highest probability token
console.log(sorted[sorted.length - 1]); // "other" token
```

---

## 3. Wedge Utilities (`lib/utils/wedges.ts`)

**Overview:**

Utilities for calculating wedge angles, generating SVG paths, and determining
which wedge is selected after wheel rotation.

**Types:**

```typescript
import { Token } from "@/lib/types";

/**
 * Wedge with calculated angles
 */
export interface Wedge extends Token {
  start_angle: number; // Start angle in degrees [0, 360)
  end_angle: number; // End angle in degrees [0, 360)
  center_angle: number; // Center angle for label placement
  arc_size: number; // Arc size in degrees
}

/**
 * SVG path data for a wedge
 */
export interface WedgePath {
  d: string; // SVG path string
  cx: number; // Center X
  cy: number; // Center Y
  radius: number; // Outer radius
}
```

**Functions:**

### 3.1 `calculateWedgeAngles()`

Calculate wedge angles from token probabilities.

**Signature:**

```typescript
function calculateWedgeAngles(tokens: Token[]): Wedge[];
```

**Parameters:**

- `tokens` (Token[]): Array of tokens with probabilities

**Returns:**

- `Wedge[]`: Array of wedges with calculated angles

**Algorithm:**

```text
1. Start at angle 0°
2. For each token:
   - arc_size = (probability / 1.0) × 360°
   - start_angle = current_angle
   - end_angle = current_angle + arc_size
   - center_angle = start_angle + (arc_size / 2)
   - current_angle = end_angle
3. Return wedges
```

**Implementation:**

```typescript
export function calculateWedgeAngles(tokens: Token[]): Wedge[] {
  let currentAngle = 0;

  return tokens.map((token) => {
    // Calculate arc size from probability
    const arcSize = (token.probability / 1.0) * 360;

    const wedge: Wedge = {
      ...token,
      start_angle: currentAngle,
      end_angle: currentAngle + arcSize,
      center_angle: currentAngle + arcSize / 2,
      arc_size: arcSize,
    };

    // Update current angle for next wedge
    currentAngle += arcSize;

    return wedge;
  });
}
```

**Validation:**

```typescript
// Test: Last wedge should end at 360°
const wedges = calculateWedgeAngles(tokens);
const lastWedge = wedges[wedges.length - 1];
expect(lastWedge.end_angle).toBeCloseTo(360, 2);
```

**Usage Example:**

```typescript
const tokens = logprobsToProbs(geminiResponse.top_candidates);
const tokensWithOther = addOtherCategory(tokens);
const wedges = calculateWedgeAngles(tokensWithOther);

console.log(wedges[0]);
// {
//   token: " floor",
//   probability: 0.18,
//   start_angle: 0,
//   end_angle: 64.8,  // 0.18 * 360
//   center_angle: 32.4,
//   arc_size: 64.8
// }
```

---

### 3.2 `generateSvgPath()`

Generate SVG path data for a wedge (pie slice).

**Signature:**

```typescript
function generateSvgPath(
  wedge: Wedge,
  cx: number,
  cy: number,
  radius: number
): string;
```

**Parameters:**

- `wedge` (Wedge): Wedge with angles
- `cx` (number): Center X coordinate
- `cy` (number): Center Y coordinate
- `radius` (number): Outer radius

**Returns:**

- `string`: SVG path string

**Algorithm:**

```text
1. Convert angles to radians
2. Calculate start point (x1, y1) on circle
3. Calculate end point (x2, y2) on circle
4. Determine if arc is large (>180°)
5. Build SVG path:
   M cx,cy          (move to center)
   L x1,y1          (line to start)
   A radius,radius  (arc to end)
   Z                (close path)
```

**Implementation:**

```typescript
export function generateSvgPath(
  wedge: Wedge,
  cx: number,
  cy: number,
  radius: number
): string {
  const { start_angle, end_angle } = wedge;

  // Convert degrees to radians
  const startRad = (start_angle * Math.PI) / 180;
  const endRad = (end_angle * Math.PI) / 180;

  // Calculate start point
  const x1 = cx + radius * Math.cos(startRad);
  const y1 = cy + radius * Math.sin(startRad);

  // Calculate end point
  const x2 = cx + radius * Math.cos(endRad);
  const y2 = cy + radius * Math.sin(endRad);

  // Determine if arc is large (>180°)
  const largeArcFlag = end_angle - start_angle > 180 ? 1 : 0;

  // Build SVG path
  const path = [
    `M ${cx},${cy}`, // Move to center
    `L ${x1},${y1}`, // Line to start
    `A ${radius},${radius}`, // Arc with radius
    `0`, // x-axis-rotation
    `${largeArcFlag}`, // large-arc-flag
    `1`, // sweep-flag (clockwise)
    `${x2},${y2}`, // End point
    `Z`, // Close path
  ].join(" ");

  return path;
}
```

**SVG Path Format:**

```text
M cx,cy                       Move to center
L x1,y1                       Line to arc start
A rx,ry rotation large sweep x,y  Arc to end
Z                             Close path
```

**Usage Example:**

```typescript
const wedge = wedges[0];
const path = generateSvgPath(wedge, 250, 250, 200);

console.log(path);
// "M 250,250 L 450,250 A 200,200 0 0 1 389.2,380.4 Z"

// Use in React component:
<path
  d={path}
  fill={color}
  stroke="white"
  strokeWidth={2}
  onClick={() => handleWedgeClick(wedge)}
/>;
```

---

### 3.3 `findWedgeAtAngle()`

Find which wedge the pointer is pointing at after wheel rotation.

**Signature:**

```typescript
function findWedgeAtAngle(wedges: Wedge[], pointerAngle: number): Wedge | null;
```

**Parameters:**

- `wedges` (Wedge[]): Array of wedges with angles
- `pointerAngle` (number): Pointer angle in degrees [0, 360)

**Returns:**

- `Wedge | null`: The wedge at the pointer angle, or null if not found

**Algorithm:**

```text
1. Normalize pointer angle to [0, 360)
2. For each wedge:
   - If start_angle <= pointer < end_angle:
     Return this wedge
3. Return null if no match (shouldn't happen)
```

**Implementation:**

```typescript
export function findWedgeAtAngle(
  wedges: Wedge[],
  pointerAngle: number
): Wedge | null {
  // Normalize angle to [0, 360)
  const normalizedAngle = ((pointerAngle % 360) + 360) % 360;

  // Find wedge containing this angle
  for (const wedge of wedges) {
    if (
      normalizedAngle >= wedge.start_angle &&
      normalizedAngle < wedge.end_angle
    ) {
      return wedge;
    }
  }

  // Handle edge case: angle = 360 should map to first wedge
  if (normalizedAngle === 360 || normalizedAngle === 0) {
    return wedges[0];
  }

  // Shouldn't reach here if wedges cover full 360°
  console.warn(
    `No wedge found at angle ${normalizedAngle}. This should not happen.`
  );
  return null;
}
```

**Edge Cases:**

- Angle = 0° or 360° → first wedge
- Angle = wedge boundary → wedge that starts at that angle
- Negative angle → normalized to positive

**Usage Example:**

```typescript
// User spins wheel, lands at rotation 347°
const rotation = 347;

// Calculate pointer position (pointer is at top = 0°)
// After rotating wheel by 347°, the wedge that WAS at (360 - 347) = 13°
// is now under the pointer
const originalAngle = (360 - rotation) % 360;

const selectedWedge = findWedgeAtAngle(wedges, originalAngle);
console.log("Selected token:", selectedWedge?.token);
```

---

### 3.4 `calculateLabelPosition()`

Calculate position for wedge label.

**Signature:**

```typescript
function calculateLabelPosition(
  wedge: Wedge,
  cx: number,
  cy: number,
  radius: number
): { x: number; y: number };
```

**Parameters:**

- `wedge` (Wedge): Wedge with angles
- `cx` (number): Center X coordinate
- `cy` (number): Center Y coordinate
- `radius` (number): Outer radius

**Returns:**

- `{ x: number, y: number }`: Label position

**Implementation:**

```typescript
export function calculateLabelPosition(
  wedge: Wedge,
  cx: number,
  cy: number,
  radius: number
): { x: number; y: number } {
  // Place label at 2/3 of radius from center
  const labelRadius = radius * 0.67;

  // Use center angle of wedge
  const angleRad = (wedge.center_angle * Math.PI) / 180;

  const x = cx + labelRadius * Math.cos(angleRad);
  const y = cy + labelRadius * Math.sin(angleRad);

  return { x, y };
}
```

**Usage Example:**

```typescript
const labelPos = calculateLabelPosition(wedge, 250, 250, 200);

<text
  x={labelPos.x}
  y={labelPos.y}
  textAnchor="middle"
  dominantBaseline="middle"
  fill="white"
>
  {wedge.token}
</text>;
```

---

## 4. Session Management (`lib/api/session.ts`)

**Overview:**

Utilities for managing sessions in Vercel KV (Redis).

**Types:**

```typescript
import { Session, Token, TokenSelection } from "@/lib/types";
import { kv } from "@vercel/kv";
```

**Functions:**

### 4.1 `createSession()`

Create a new session.

**Signature:**

```typescript
async function createSession(prompt: string, tokens: Token[]): Promise<Session>;
```

**Implementation:**

```typescript
import { v4 as uuidv4 } from "uuid";

export async function createSession(
  prompt: string,
  tokens: Token[]
): Promise<Session> {
  const session_id = uuidv4();

  const session: Session = {
    session_id,
    context: prompt,
    tokens,
    history: [],
    created_at: Date.now(),
    last_accessed: Date.now(),
  };

  // Store in Vercel KV with 1-hour TTL
  await kv.set(`session:${session_id}`, session, { ex: 3600 });

  return session;
}
```

---

### 4.2 `getSession()`

Retrieve a session by ID.

**Signature:**

```typescript
async function getSession(sessionId: string): Promise<Session | null>;
```

**Implementation:**

```typescript
export async function getSession(sessionId: string): Promise<Session | null> {
  const session = await kv.get<Session>(`session:${sessionId}`);

  if (session) {
    // Update last accessed time
    session.last_accessed = Date.now();
    await kv.set(`session:${sessionId}`, session, { ex: 3600 });
  }

  return session;
}
```

---

### 4.3 `updateSession()`

Update session with new context and tokens.

**Signature:**

```typescript
async function updateSession(
  sessionId: string,
  updates: Partial<Session>
): Promise<Session>;
```

**Implementation:**

```typescript
export async function updateSession(
  sessionId: string,
  updates: Partial<Session>
): Promise<Session> {
  const session = await getSession(sessionId);

  if (!session) {
    throw new Error("Session not found");
  }

  const updatedSession: Session = {
    ...session,
    ...updates,
    last_accessed: Date.now(),
  };

  await kv.set(`session:${sessionId}`, updatedSession, { ex: 3600 });

  return updatedSession;
}
```

---

### 4.4 `deleteSession()`

Delete a session.

**Signature:**

```typescript
async function deleteSession(sessionId: string): Promise<void>;
```

**Implementation:**

```typescript
export async function deleteSession(sessionId: string): Promise<void> {
  await kv.del(`session:${sessionId}`);
}
```

---

## 5. Validation Utilities (`lib/utils/validation.ts`)

**Overview:**

Input validation utilities.

**Functions:**

### 5.1 `validatePrompt()`

Validate user prompt.

**Signature:**

```typescript
function validatePrompt(prompt: string): { valid: boolean; error?: string };
```

**Implementation:**

```typescript
export function validatePrompt(prompt: string): {
  valid: boolean;
  error?: string;
} {
  const trimmed = prompt.trim();

  if (trimmed.length === 0) {
    return {
      valid: false,
      error: "Prompt cannot be empty",
    };
  }

  if (trimmed.length > 1000) {
    return {
      valid: false,
      error: "Prompt must be less than 1000 characters",
    };
  }

  return { valid: true };
}
```

---

### 5.2 `validateTemperature()`

Validate temperature parameter.

**Signature:**

```typescript
function validateTemperature(temperature: number): {
  valid: boolean;
  error?: string;
};
```

**Implementation:**

```typescript
export function validateTemperature(temperature: number): {
  valid: boolean;
  error?: string;
} {
  if (typeof temperature !== "number" || isNaN(temperature)) {
    return {
      valid: false,
      error: "Temperature must be a number",
    };
  }

  if (temperature < 0 || temperature > 2) {
    return {
      valid: false,
      error: "Temperature must be between 0.0 and 2.0",
    };
  }

  return { valid: true };
}
```

---

### 5.3 `validateTokenId()`

Validate token ID.

**Signature:**

```typescript
function validateTokenId(
  tokenId: number,
  tokens: Token[]
): { valid: boolean; error?: string };
```

**Implementation:**

```typescript
export function validateTokenId(
  tokenId: number,
  tokens: Token[]
): { valid: boolean; error?: string } {
  if (!Number.isInteger(tokenId)) {
    return {
      valid: false,
      error: "Token ID must be an integer",
    };
  }

  // -1 is valid (represents "other")
  if (tokenId === -1) {
    return { valid: true };
  }

  // Check if token_id exists in tokens
  const exists = tokens.some((t) => t.token_id === tokenId);

  if (!exists) {
    return {
      valid: false,
      error: `Invalid token_id: ${tokenId}. Must be -1 or a valid token ID.`,
    };
  }

  return { valid: true };
}
```

---

## Summary

This document specifies all core functions and utilities for the LLM Token
Wheel application. Key modules:

1. **Gemini API Client**: Handles all Gemini API interactions

   - `getTokenLogprobs()`: Get token probabilities
   - `sampleFromDistribution()`: Sample for "other" category

2. **Probability Utilities**: Math for converting and normalizing probabilities

   - `logprobsToProbs()`: Convert log probs to probs using softmax
   - `addOtherCategory()`: Add "other" token for remaining probability

3. **Wedge Utilities**: Calculate wedge angles and SVG paths

   - `calculateWedgeAngles()`: Convert probabilities to angles
   - `generateSvgPath()`: Create SVG path for pie slice
   - `findWedgeAtAngle()`: Determine selected wedge after spin

4. **Session Management**: Vercel KV operations

   - `createSession()`, `getSession()`, `updateSession()`, `deleteSession()`

5. **Validation Utilities**: Input validation
   - `validatePrompt()`, `validateTemperature()`, `validateTokenId()`

**Next Steps:**

1. Implement these functions following the specifications
2. Create unit tests for each function
3. Integrate with API endpoints

---

**Document Version:** 1.0
**Last Updated:** January 11, 2026
**Status:** Ready for implementation

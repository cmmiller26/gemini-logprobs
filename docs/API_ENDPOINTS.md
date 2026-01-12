# LLM Token Wheel - API Endpoints Specification

## Overview

This document provides detailed specifications for all Next.js API routes in
the `app/api/` directory. Each endpoint includes request/response schemas,
error handling, validation rules, and implementation notes.

---

## Table of Contents

1. [Common Types & Interfaces](#1-common-types--interfaces)
2. [POST /api/start](#2-post-apistart)
3. [POST /api/select](#3-post-apiselect)
4. [GET /api/session/[id]](#4-get-apisessionid)
5. [DELETE /api/session/[id]](#5-delete-apisessionid)
6. [GET /api/health](#6-get-apihealth)
7. [Error Handling](#7-error-handling)
8. [Rate Limiting](#8-rate-limiting)

---

## 1. Common Types & Interfaces

### Shared TypeScript Types

```typescript
// lib/types.ts

/**
 * Represents a single token with its probability and metadata
 */
export interface Token {
  token: string; // The token text (e.g., " floor", " mat")
  token_id: number; // Unique identifier (use index, -1 for "other")
  probability: number; // Probability value [0.0, 1.0]
  log_probability: number; // Log probability from API
  is_other: boolean; // True if this is the "other" category
}

/**
 * Session state stored in Vercel KV
 */
export interface Session {
  session_id: string; // UUID
  context: string; // Current accumulated text
  tokens: Token[]; // Current token distribution
  history: TokenSelection[]; // Past selections
  created_at: number; // Unix timestamp
  last_accessed: number; // Unix timestamp
}

/**
 * Record of a token selection
 */
export interface TokenSelection {
  token: string; // Selected token text
  token_id: number; // Token ID that was selected
  probability: number; // Probability of selected token
  was_other: boolean; // True if "other" was selected
  selected_at: number; // Unix timestamp
}

/**
 * Standard API error response
 */
export interface ApiError {
  error: string; // Error type (e.g., "ValidationError")
  message: string; // Human-readable error message
  details?: any; // Optional additional details
}
```

---

## 2. POST /api/start

**Purpose**: Create a new generation session and get initial token probabilities.

**Endpoint Details:**

- **Path**: `app/api/start/route.ts`
- **Method**: POST
- **Authentication**: None (public endpoint)
- **Rate Limit**: 10 requests per minute per IP

**Request:**

**Request Body Schema:**

```typescript
interface StartRequest {
  prompt: string; // Initial text prompt
  temperature?: number; // Optional, default 1.0, range [0.0, 2.0]
  logprobs_count?: number; // Optional, default 20, range [1, 20]
}
```

**Validation Rules:**

```typescript
// Prompt validation
- Required: true
- Min length: 1 character
- Max length: 1000 characters
- Trim whitespace before validation
- Reject if only whitespace

// Temperature validation
- Optional: true
- Default: 1.0
- Min: 0.0
- Max: 2.0
- Must be numeric

// Logprobs count validation
- Optional: true
- Default: 20
- Min: 1
- Max: 20
- Must be integer
```

**Example Request:**

```json
{
  "prompt": "The cat sat on the",
  "temperature": 1.0,
  "logprobs_count": 20
}
```

**Response:**

**Success Response (200 OK):**

```typescript
interface StartResponse {
  session_id: string; // UUID for this session
  context: string; // Echo of the prompt
  tokens: Token[]; // Token distribution (including "other")
  step: number; // Current step (always 0 for start)
  expires_at: number; // Unix timestamp when session expires
}
```

**Example Response:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "context": "The cat sat on the",
  "tokens": [
    {
      "token": " floor",
      "token_id": 0,
      "probability": 0.18,
      "log_probability": -1.71,
      "is_other": false
    },
    {
      "token": " mat",
      "token_id": 1,
      "probability": 0.15,
      "log_probability": -1.9,
      "is_other": false
    },
    {
      "token": " bed",
      "token_id": 2,
      "probability": 0.12,
      "log_probability": -2.12,
      "is_other": false
    },
    // ... more tokens ...
    {
      "token": "<OTHER>",
      "token_id": -1,
      "probability": 0.35,
      "log_probability": 0,
      "is_other": true
    }
  ],
  "step": 0,
  "expires_at": 1704982800
}
```

**Error Responses:**

**400 Bad Request - Validation Error:**

```json
{
  "error": "ValidationError",
  "message": "Prompt must be between 1 and 1000 characters",
  "details": {
    "field": "prompt",
    "value": "",
    "constraint": "min_length"
  }
}
```

**429 Too Many Requests - Rate Limit:**

```json
{
  "error": "RateLimitExceeded",
  "message": "Rate limit exceeded. Please try again in 60 seconds.",
  "details": {
    "retry_after": 60
  }
}
```

**500 Internal Server Error - Gemini API Failure:**

```json
{
  "error": "ApiError",
  "message": "Failed to generate token probabilities from Gemini API",
  "details": {
    "provider": "gemini",
    "status_code": 500
  }
}
```

**503 Service Unavailable - Gemini Rate Limit:**

```json
{
  "error": "ServiceUnavailable",
  "message": "Gemini API rate limit exceeded. Please try again later.",
  "details": {
    "provider": "gemini",
    "retry_after": 60
  }
}
```

**Implementation Notes:**

1. **Session Creation**:

   - Generate UUID v4 for session_id
   - Store in Vercel KV with 1-hour TTL
   - Key format: `session:{session_id}`

2. **Gemini API Call**:

   - Use `getTokenLogprobs()` from `lib/api/gemini.ts`
   - Set `max_output_tokens=1` to get single token
   - Set `response_logprobs=True` and `logprobs={logprobs_count}`

3. **Probability Processing**:

   - Convert log probabilities to probabilities using softmax
   - Add "other" category if remaining probability > 1%
   - Sort tokens by probability (descending)

4. **Session Storage**:

   ```typescript
   const session: Session = {
     session_id: uuid(),
     context: prompt,
     tokens: tokensWithOther,
     history: [],
     created_at: Date.now(),
     last_accessed: Date.now(),
   };

   await kv.set(`session:${session_id}`, session, { ex: 3600 });
   ```

5. **Rate Limiting**:

   ```typescript
   const key = `rate:start:${ip}`;
   const requests = await kv.incr(key);
   if (requests === 1) {
     await kv.expire(key, 60);
   }
   if (requests > 10) {
     throw new RateLimitError();
   }
   ```

---

## 3. POST /api/select

**Purpose**: Handle token selection (spin or manual), append to context,
and get next token probabilities.

**Endpoint Details:**

- **Path**: `app/api/select/route.ts`
- **Method**: POST
- **Authentication**: None (session-based)
- **Rate Limit**: 30 requests per minute per session

**Request:**

**Request Body Schema:**

```typescript
interface SelectRequest {
  session_id: string; // UUID of the session
  selected_token_id: number; // Token ID to select (-1 for "other")
}
```

**Validation Rules:**

```typescript
// Session ID validation
- Required: true
- Format: Valid UUID v4
- Must exist in Vercel KV

// Selected token ID validation
- Required: true
- Must be integer
- Must be either -1 (other) or valid token_id from current distribution
```

**Example Requests:**

**Explicit Token Selection:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "selected_token_id": 1
}
```

**"Other" Category Selection:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "selected_token_id": -1
}
```

**Response:**

**Success Response (200 OK):**

```typescript
interface SelectResponse {
  session_id: string; // UUID of the session
  selected_token: string; // The actual token text that was added
  previous_context: string; // Context before selection
  new_context: string; // Context after adding token
  next_tokens: Token[]; // Next token distribution
  step: number; // Current step number
  should_continue: boolean; // Whether generation can continue
}
```

**Example Response - Explicit Selection:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "selected_token": " mat",
  "previous_context": "The cat sat on the",
  "new_context": "The cat sat on the mat",
  "next_tokens": [
    {
      "token": " and",
      "token_id": 0,
      "probability": 0.22,
      "log_probability": -1.51,
      "is_other": false
    }
    // ... more tokens ...
  ],
  "step": 1,
  "should_continue": true
}
```

**Example Response - "Other" Selection:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "selected_token": " windowsill",
  "previous_context": "The cat sat on the",
  "new_context": "The cat sat on the windowsill",
  "next_tokens": [
    {
      "token": " and",
      "token_id": 0,
      "probability": 0.18,
      "log_probability": -1.71,
      "is_other": false
    }
    // ... more tokens ...
  ],
  "step": 1,
  "should_continue": true
}
```

**Error Responses:**

**400 Bad Request - Invalid Token ID:**

```json
{
  "error": "ValidationError",
  "message": "Invalid token_id. Must be -1 or a valid token from current distribution.",
  "details": {
    "selected_token_id": 99,
    "valid_range": [-1, 0, 1, 2, 3, 4, 5]
  }
}
```

**404 Not Found - Session Not Found:**

```json
{
  "error": "NotFoundError",
  "message": "Session not found or expired",
  "details": {
    "session_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**429 Too Many Requests - Rate Limit:**

```json
{
  "error": "RateLimitExceeded",
  "message": "Too many selections. Please slow down.",
  "details": {
    "session_id": "550e8400-e29b-41d4-a716-446655440000",
    "retry_after": 2
  }
}
```

**500 Internal Server Error - Gemini API Failure:**

```json
{
  "error": "ApiError",
  "message": "Failed to generate next token probabilities",
  "details": {
    "provider": "gemini"
  }
}
```

**Implementation Notes:**

1. **Session Retrieval**:

   ```typescript
   const session = await kv.get<Session>(`session:${session_id}`);
   if (!session) {
     throw new NotFoundError("Session not found or expired");
   }
   ```

2. **Token Selection Logic**:

   **Explicit Token (token_id !== -1):**

   ```typescript
   const token = session.tokens.find((t) => t.token_id === selected_token_id);
   if (!token) {
     throw new ValidationError("Invalid token_id");
   }
   const selectedToken = token.token;
   ```

   **"Other" Category (token_id === -1):**

   ```typescript
   // Re-run Gemini API to sample from full distribution
   const response = await geminiClient.generateContent({
     model: "gemini-2.0-flash",
     contents: session.context,
     config: {
       max_output_tokens: 1,
       temperature: 1.0,
       // No logprobs needed, just get sampled token
     },
   });
   const selectedToken = response.text;
   ```

3. **Context Update**:

   ```typescript
   const newContext = session.context + selectedToken;
   ```

4. **Next Token Probabilities**:

   ```typescript
   const nextLogprobs = await getTokenLogprobs(newContext, {
     temperature: session.temperature || 1.0,
     logprobs_count: session.logprobs_count || 20,
   });
   const nextTokens = logprobsToProbs(nextLogprobs);
   const nextTokensWithOther = addOtherCategory(nextTokens);
   ```

5. **History Update**:

   ```typescript
   const tokenSelection: TokenSelection = {
     token: selectedToken,
     token_id: selected_token_id,
     probability: token?.probability || 0,
     was_other: selected_token_id === -1,
     selected_at: Date.now(),
   };

   session.history.push(tokenSelection);
   ```

6. **Session Update**:

   ```typescript
   session.context = newContext;
   session.tokens = nextTokensWithOther;
   session.last_accessed = Date.now();

   await kv.set(`session:${session_id}`, session, { ex: 3600 });
   ```

7. **Continue Check**:

   ```typescript
   const should_continue =
     newContext.length < 2000 && // Max context length
     session.history.length < 100 && // Max generation steps
     !containsEndToken(nextTokensWithOther); // Check for EOS token
   ```

---

## 4. GET /api/session/[id]

**Purpose**: Retrieve current session state (for debugging or UI state restoration).

**Endpoint Details:**

- **Path**: `app/api/session/[id]/route.ts`
- **Method**: GET
- **Authentication**: None (session-based)
- **Rate Limit**: 60 requests per minute per session

**Request:**

**URL Parameters:**

```typescript
interface SessionParams {
  id: string; // Session UUID
}
```

**Example Request:**

```http
GET /api/session/550e8400-e29b-41d4-a716-446655440000
```

**Response:**

**Success Response (200 OK):**

```typescript
interface SessionResponse {
  session_id: string;
  context: string;
  tokens: Token[];
  history: TokenSelection[];
  step: number;
  created_at: number;
  last_accessed: number;
  expires_at: number;
}
```

**Example Response:**

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "context": "The cat sat on the mat",
  "tokens": [
    {
      "token": " and",
      "token_id": 0,
      "probability": 0.22,
      "log_probability": -1.51,
      "is_other": false
    }
    // ... more tokens ...
  ],
  "history": [
    {
      "token": " mat",
      "token_id": 1,
      "probability": 0.15,
      "was_other": false,
      "selected_at": 1704979200
    }
  ],
  "step": 1,
  "created_at": 1704975600,
  "last_accessed": 1704979200,
  "expires_at": 1704982800
}
```

**Error Responses:**

**404 Not Found:**

```json
{
  "error": "NotFoundError",
  "message": "Session not found or expired",
  "details": {
    "session_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Implementation Notes:**

1. **Session Retrieval**:

   ```typescript
   const session = await kv.get<Session>(`session:${params.id}`);
   if (!session) {
     return NextResponse.json(
       { error: "NotFoundError", message: "Session not found or expired" },
       { status: 404 }
     );
   }
   ```

2. **Response Construction**:

   ```typescript
   const response: SessionResponse = {
     ...session,
     step: session.history.length,
     expires_at: session.created_at + 3600000, // 1 hour from creation
   };
   ```

---

## 5. DELETE /api/session/[id]

**Purpose**: Delete a session and clean up resources.

**Endpoint Details:**

- **Path**: `app/api/session/[id]/route.ts`
- **Method**: DELETE
- **Authentication**: None (session-based)
- **Rate Limit**: 10 requests per minute per IP

**Request:**

**URL Parameters:**

```typescript
interface SessionParams {
  id: string; // Session UUID
}
```

**Example Request:**

```http
DELETE /api/session/550e8400-e29b-41d4-a716-446655440000
```

**Response:**

**Success Response (200 OK):**

```typescript
interface DeleteResponse {
  message: string;
  session_id: string;
}
```

**Example Response:**

```json
{
  "message": "Session deleted successfully",
  "session_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Error Responses:**

**404 Not Found:**

```json
{
  "error": "NotFoundError",
  "message": "Session not found or already deleted",
  "details": {
    "session_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Implementation Notes:**

1. **Session Deletion**:

   ```typescript
   const exists = await kv.exists(`session:${params.id}`);
   if (!exists) {
     return NextResponse.json(
       {
         error: "NotFoundError",
         message: "Session not found or already deleted",
       },
       { status: 404 }
     );
   }

   await kv.del(`session:${params.id}`);
   ```

2. **Cleanup Related Keys**:

   ```typescript
   // If there are related rate limit keys, clean them up
   await kv.del(`rate:select:${params.id}`);
   ```

---

## 6. GET /api/health

**Purpose**: Health check endpoint for monitoring and uptime checks.

**Endpoint Details:**

- **Path**: `app/api/health/route.ts`
- **Method**: GET
- **Authentication**: None (public)
- **Rate Limit**: None

**Request:**

No parameters.

**Example Request:**

```http
GET /api/health
```

**Response:**

**Success Response (200 OK):**

```typescript
interface HealthResponse {
  status: "ok" | "degraded";
  timestamp: number;
  services: {
    gemini: "ok" | "error";
    redis: "ok" | "error";
  };
}
```

**Example Response - Healthy:**

```json
{
  "status": "ok",
  "timestamp": 1704979200,
  "services": {
    "gemini": "ok",
    "redis": "ok"
  }
}
```

**Example Response - Degraded:**

```json
{
  "status": "degraded",
  "timestamp": 1704979200,
  "services": {
    "gemini": "error",
    "redis": "ok"
  }
}
```

**Implementation Notes:**

1. **Gemini Check**:

   ```typescript
   async function checkGemini(): Promise<boolean> {
     try {
       // Simple API test with minimal tokens
       const response = await geminiClient.generateContent({
         model: "gemini-2.0-flash",
         contents: "test",
         config: { max_output_tokens: 1 },
       });
       return !!response.text;
     } catch (error) {
       console.error("Gemini health check failed:", error);
       return false;
     }
   }
   ```

2. **Redis Check**:

   ```typescript
   async function checkRedis(): Promise<boolean> {
     try {
       const key = "health:check";
       await kv.set(key, Date.now(), { ex: 60 });
       const value = await kv.get(key);
       return value !== null;
     } catch (error) {
       console.error("Redis health check failed:", error);
       return false;
     }
   }
   ```

3. **Overall Status**:

   ```typescript
   const geminiOk = await checkGemini();
   const redisOk = await checkRedis();

   const status = geminiOk && redisOk ? "ok" : "degraded";
   ```

---

## 7. Error Handling

### Standard Error Response Format

All errors follow this format:

```typescript
interface ApiError {
  error: string; // Error type
  message: string; // Human-readable message
  details?: any; // Optional additional context
}
```

### Error Types

#### ValidationError (400)

**When**: Invalid request parameters, malformed data

**Example**:

```typescript
throw new ValidationError("Prompt must be between 1 and 1000 characters");
```

#### NotFoundError (404)

**When**: Session not found, resource doesn't exist

**Example**:

```typescript
throw new NotFoundError("Session not found or expired");
```

#### RateLimitError (429)

**When**: Rate limit exceeded

**Example**:

```typescript
throw new RateLimitError(
  "Rate limit exceeded. Please try again in 60 seconds."
);
```

#### ApiError (500)

**When**: Gemini API failure, internal errors

**Example**:

```typescript
throw new ApiError("Failed to generate token probabilities from Gemini API");
```

#### ServiceUnavailable (503)

**When**: External service (Gemini, Redis) unavailable

**Example**:

```typescript
throw new ServiceUnavailable("Gemini API rate limit exceeded");
```

### Error Handler Middleware

**File**: `lib/api/error-handler.ts`

```typescript
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public error: string,
    public message: string,
    public details?: any
  ) {
    super(message);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(400, "ValidationError", message, details);
  }
}

export class NotFoundError extends AppError {
  constructor(message: string, details?: any) {
    super(404, "NotFoundError", message, details);
  }
}

export class RateLimitError extends AppError {
  constructor(message: string, details?: any) {
    super(429, "RateLimitExceeded", message, details);
  }
}

export class ApiError extends AppError {
  constructor(message: string, details?: any) {
    super(500, "ApiError", message, details);
  }
}

export class ServiceUnavailable extends AppError {
  constructor(message: string, details?: any) {
    super(503, "ServiceUnavailable", message, details);
  }
}

/**
 * Wraps an API route handler with error handling
 */
export function withErrorHandler(handler: (req: Request) => Promise<Response>) {
  return async (req: Request): Promise<Response> => {
    try {
      return await handler(req);
    } catch (error) {
      if (error instanceof AppError) {
        return NextResponse.json(
          {
            error: error.error,
            message: error.message,
            details: error.details,
          },
          { status: error.statusCode }
        );
      }

      // Unexpected error
      console.error("Unexpected error:", error);
      return NextResponse.json(
        {
          error: "InternalServerError",
          message: "An unexpected error occurred",
        },
        { status: 500 }
      );
    }
  };
}
```

---

## 8. Rate Limiting

### Rate Limit Configuration

**File**: `lib/api/rate-limit.ts`

```typescript
interface RateLimitConfig {
  endpoint: string;
  limit: number; // Max requests
  window: number; // Time window in seconds
  keyPrefix: string; // Redis key prefix
}

const RATE_LIMITS: Record<string, RateLimitConfig> = {
  start: {
    endpoint: "/api/start",
    limit: 10,
    window: 60,
    keyPrefix: "rate:start",
  },
  select: {
    endpoint: "/api/select",
    limit: 30,
    window: 60,
    keyPrefix: "rate:select",
  },
  session: {
    endpoint: "/api/session",
    limit: 60,
    window: 60,
    keyPrefix: "rate:session",
  },
  delete: {
    endpoint: "/api/session/delete",
    limit: 10,
    window: 60,
    keyPrefix: "rate:delete",
  },
};

/**
 * Check rate limit for a request
 * @throws RateLimitError if limit exceeded
 */
export async function checkRateLimit(
  endpoint: string,
  identifier: string
): Promise<void> {
  const config = RATE_LIMITS[endpoint];
  if (!config) return; // No rate limit for this endpoint

  const key = `${config.keyPrefix}:${identifier}`;
  const requests = await kv.incr(key);

  if (requests === 1) {
    // First request, set expiry
    await kv.expire(key, config.window);
  }

  if (requests > config.limit) {
    throw new RateLimitError(
      `Rate limit exceeded. Maximum ${config.limit} requests per ` +
        `${config.window} seconds.`,
      {
        limit: config.limit,
        window: config.window,
        retry_after: config.window,
      }
    );
  }
}

/**
 * Get client identifier (IP address or session ID)
 */
export function getClientIdentifier(req: Request, useSession = false): string {
  if (useSession) {
    // For session-based rate limiting
    const body = await req.json();
    return body.session_id || "unknown";
  }

  // For IP-based rate limiting
  const forwarded = req.headers.get("x-forwarded-for");
  const ip = forwarded ? forwarded.split(",")[0] : "unknown";
  return ip;
}
```

### Usage in API Routes

```typescript
// app/api/start/route.ts
export async function POST(req: Request) {
  // Rate limit by IP
  const ip = getClientIdentifier(req, false);
  await checkRateLimit("start", ip);

  // ... rest of handler
}

// app/api/select/route.ts
export async function POST(req: Request) {
  // Rate limit by session ID
  const body = await req.json();
  await checkRateLimit("select", body.session_id);

  // ... rest of handler
}
```

---

## Summary

This document specifies all API endpoints for the LLM Token Wheel application.
Key points:

1. **Two main endpoints**: `/api/start` (create session) and `/api/select`
   (token selection)
2. **Session management**: `/api/session/[id]` for retrieval and deletion
3. **Health monitoring**: `/api/health` for service status
4. **Comprehensive error handling**: Standard error format across all
   endpoints
5. **Rate limiting**: Protects against abuse with configurable limits

**Next Steps:**

1. Implement these endpoints following the specifications
2. Create corresponding tests for each endpoint
3. Set up monitoring for error rates and latency

---

**Document Version:** 1.0
**Last Updated:** January 11, 2026
**Status:** Ready for implementation

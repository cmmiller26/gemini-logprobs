# LLM Token Wheel - Tech Stack Research & Architecture

## Executive Summary

This document provides comprehensive research on the technology stack for the
**LLM Token Wheel** project - a cloud-based educational tool that demonstrates
how Large Language Models generate text through interactive probability wheel
visualization using cloud LLM APIs (Gemini Flash).

**Key Findings:**

- ✅ **Gemini API supports logprobs** with up to 20 top alternative tokens
  (sufficient for visualization)
- ✅ **Next.js 14 App Router** provides excellent serverless architecture on
  Vercel
- ✅ **Framer Motion** offers smooth rotation animations with cubic-bezier
  easing
- ⚠️ **Architectural shift required**: Cloud APIs select token first, then
  return logprobs (hybrid approach needed)
- ✅ **Cost-effective**: Gemini Flash is extremely cheap (~$0.01-0.05 per 1M
  input tokens)

---

## Table of Contents

1. [Tech Stack Overview](#1-tech-stack-overview)
2. [API Integration Guide - Gemini with Logprobs](#2-api-integration-guide---gemini-with-logprobs)
3. [Architecture Diagram](#3-architecture-diagram)
4. [Project Structure](#4-project-structure)
5. [Comparison to Existing Versions](#5-comparison-to-existing-versions)
6. [Cost & Performance Analysis](#6-cost--performance-analysis)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Questions Answered](#8-questions-answered)

---

## 1. Tech Stack Overview

### 1.1 Backend API - Google Gemini Flash

**Why Gemini Flash:**

- **Logprobs support**: Returns log probabilities for up to 20 alternative
  tokens
- **Cost-effective**: ~$0.0001-0.0005 per request (flash models)
- **Fast inference**: Sub-second latency for logprobs
- **Generous limits**: 1500 requests per minute on free tier

**Alternatives Considered:**

- **OpenAI GPT-4o**: More expensive (~$2.50 per 1M tokens), similar logprobs
  support (5-20 tokens)
- **Anthropic Claude**: No native logprobs support (as of January 2025)
- **Recommendation**: Start with Gemini Flash, can add OpenAI as alternative later

### 1.2 Frontend Framework - Next.js 14+ (App Router)

**Why Next.js App Router:**

- **Serverless-first**: Built for Vercel deployment
- **API routes**: Built-in serverless functions via `app/api/*/route.ts`
- **React Server Components**: Reduce client bundle size
- **File-system routing**: Intuitive structure
- **TypeScript support**: Built-in type safety

**Key Features:**

```typescript
// API Route example: app/api/logprobs/route.ts
export async function POST(request: Request) {
  const { prompt } = await request.json();
  // Call Gemini API
  return NextResponse.json({ tokens, logprobs });
}
```

### 1.3 Animation - Framer Motion

**Why Framer Motion:**

- **Smooth rotation**: Built-in support for rotation animations
- **Cubic-bezier easing**: Custom easing curves for realistic wheel spin
- **React integration**: First-class React support
- **Performance**: Hardware-accelerated transforms

**Key Capabilities:**

```jsx
<motion.div
  animate={{ rotate: totalRotation }}
  transition={{
    duration: 3,
    ease: [0.17, 0.67, 0.12, 0.99], // cubic-bezier
  }}
/>
```

### 1.4 SVG Rendering - React + D3 Helpers

**Why SVG:**

- **Crisp rendering**: Vector graphics scale perfectly
- **Click handlers**: Easy to attach onClick to individual wedges
- **Responsive**: Scales to any screen size
- **Accessible**: Screen readers can parse SVG elements

**Arc Path Calculation:**

- Use D3's `arc()` generator for complex path calculations
- Or manual calculation using trigonometry for lightweight implementation
- Each wedge = SVG `<path>` element with `d` attribute

### 1.5 Deployment - Vercel

**Why Vercel:**

- **Zero config**: Next.js deploys automatically
- **Serverless functions**: API routes become serverless functions
- **Environment variables**: Secure API key storage
- **Free tier**: Sufficient for educational use (100GB bandwidth/month)
- **Edge network**: Global CDN for fast delivery

---

## 2. API Integration Guide - Gemini with Logprobs

### 2.1 Gemini API Capabilities

**Logprobs Parameters:**

```python
response = client.models.generate_content(
    model="gemini-2.0-flash",  # or "gemini-1.5-flash"
    contents=prompt,
    config=GenerateContentConfig(
        response_logprobs=True,  # Enable logprobs
        logprobs=20,             # [1-20] - number of top alternatives
        temperature=1.0,         # Standard sampling
        max_output_tokens=1,     # Generate only 1 token
    ),
)
```

**Response Structure:**

```python
{
  "candidates": [
    {
      "content": {
        "parts": [{"text": " floor"}]  # Chosen token
      },
      "logprobs_result": {
        "chosen_candidates": [
          {
            "token": " floor",
            "log_probability": -1.23
          }
        ],
        "top_candidates": [
          {
            "candidates": [
              {"token": " floor", "log_probability": -1.23},
              {"token": " mat", "log_probability": -1.45},
              {"token": " bed", "log_probability": -1.67},
              {"token": " table", "log_probability": -1.89},
              // ... up to 20 tokens total
            ]
          }
        ]
      }
    }
  ]
}
```

### 2.2 Converting Logprobs to Probabilities

**Log Probability → Probability:**

```javascript
// Backend: Convert log probabilities to probabilities
function logprobsToProbs(topCandidates) {
  const logprobs = topCandidates.map((t) => t.log_probability);

  // Softmax to get probabilities
  const maxLogProb = Math.max(...logprobs);
  const expValues = logprobs.map((lp) => Math.exp(lp - maxLogProb));
  const sumExp = expValues.reduce((a, b) => a + b, 0);

  return topCandidates.map((token, i) => ({
    token: token.token,
    token_id: i, // Use index as ID
    probability: expValues[i] / sumExp,
    log_probability: token.log_probability,
  }));
}
```

### 2.3 Handling "Other" Category

**Challenge**: Gemini returns max 20 tokens, but vocabulary has 256,000+ tokens.

**Solution**: Calculate remaining probability:

```javascript
function addOtherCategory(tokens) {
  const totalProb = tokens.reduce((sum, t) => sum + t.probability, 0);
  const remainingProb = 1.0 - totalProb;

  if (remainingProb > 0.01) {
    // Only add if >1%
    tokens.push({
      token: "<OTHER>",
      token_id: -1,
      probability: remainingProb,
      is_other: true,
    });
  }

  return tokens;
}
```

**When "Other" is selected**:

- Re-run API with `max_output_tokens=1` and `logprobs=1`
- Return the actual sampled token (not literal "OTHER")
- This works because the API naturally samples from full distribution

### 2.4 API Endpoint Design (Next.js)

**Route: `app/api/start/route.ts`**

```typescript
export async function POST(req: Request) {
  const { prompt } = await req.json();

  // Call Gemini API
  const response = await geminiClient.models.generateContent({
    model: "gemini-2.0-flash",
    contents: prompt,
    config: {
      response_logprobs: true,
      logprobs: 20,
      max_output_tokens: 1,
      temperature: 1.0,
    },
  });

  // Extract logprobs
  const topCandidates =
    response.candidates[0].logprobs_result.top_candidates[0].candidates;

  // Convert to probabilities
  const tokens = logprobsToProbs(topCandidates);
  const tokensWithOther = addOtherCategory(tokens);

  return NextResponse.json({
    session_id: generateSessionId(),
    context: prompt,
    tokens: tokensWithOther,
  });
}
```

**Route: `app/api/select/route.ts`**

```typescript
export async function POST(req: Request) {
  const { session_id, selected_token_id } = await req.json();

  // Get session from storage (Redis or Vercel KV)
  const session = await getSession(session_id);

  // Handle "other" selection
  if (selected_token_id === -1) {
    // Re-run API to sample from full distribution
    const response = await geminiClient.models.generateContent({
      model: "gemini-2.0-flash",
      contents: session.context,
      config: {
        max_output_tokens: 1,
        temperature: 1.0,
      },
    });

    const selectedToken = response.text;
    const newContext = session.context + selectedToken;

    // Get next distribution
    const nextTokens = await getNextTokenProbs(newContext);

    return NextResponse.json({
      selected_token: selectedToken,
      new_context: newContext,
      next_tokens: nextTokens,
    });
  }

  // Handle explicit token selection
  const token = session.tokens.find((t) => t.token_id === selected_token_id);
  const newContext = session.context + token.token;

  // Get next distribution
  const nextTokens = await getNextTokenProbs(newContext);

  return NextResponse.json({
    selected_token: token.token,
    new_context: newContext,
    next_tokens: nextTokens,
  });
}
```

### 2.5 Authentication & Rate Limiting

**Environment Variables:**

```bash
# .env.local
GEMINI_API_KEY=your_gemini_api_key_here
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

**API Key Storage (Vercel):**

1. Vercel Dashboard → Project → Settings → Environment Variables
2. Add `GEMINI_API_KEY` (encrypted at rest)
3. Access via `process.env.GEMINI_API_KEY` in API routes

**Rate Limiting:**

- **Client-side**: Debounce requests (prevent spam clicking)
- **Server-side**: Track requests per session (Vercel KV)
- **Gemini limits**: 1500 requests/min (free tier) - more than sufficient

---

## 3. Architecture Diagram

```text
┌────────────────────────────────────────────────────────────────────┐
│                          USER INTERFACE                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  React Components (Client)                                   │  │
│  │  - PromptInput: Enter starting text                          │  │
│  │  - TokenWheel: SVG wheel with Framer Motion rotation         │  │
│  │  - GeneratedText: Display accumulated tokens                 │  │
│  │  - Controls: Spin button, Reset button                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         │ HTTP Requests (fetch)
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                    NEXT.JS API ROUTES (Serverless)                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  POST /api/start                                             │  │
│  │  - Create session                                            │  │
│  │  - Call Gemini API for initial logprobs                      │  │
│  │  - Return token distribution                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  POST /api/select                                            │  │
│  │  - Receive selected token_id or "other"                      │  │
│  │  - If "other": Re-run API to sample actual token             │  │
│  │  - If explicit: Use selected token                           │  │
│  │  - Append token to context                                   │  │
│  │  - Get next logprobs                                         │  │
│  │  - Return new distribution                                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Session Storage (Vercel KV or Redis)                        │  │
│  │  - session_id → { context, tokens, history }                 │  │
│  │  - TTL: 1 hour                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         │ API Calls
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                      GOOGLE GEMINI API                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Gemini Flash Model                                          │  │
│  │  - Input: context + prompt                                   │  │
│  │  - Config: response_logprobs=True, logprobs=20               │  │
│  │  - Output: chosen token + top 20 alternatives with logprobs  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

DATA FLOW:

1. User enters prompt "The cat sat on the"
   → POST /api/start

2. API Route calls Gemini:
   gemini.generateContent(prompt, {logprobs: 20})

3. Gemini returns:
   chosen: " floor"
   alternatives: [" floor", " mat", " bed", " table", ...]

4. API Route converts logprobs → probabilities:
   [
     {token: " floor", probability: 0.18},
     {token: " mat", probability: 0.15},
     ...
     {token: "<OTHER>", probability: 0.35}
   ]

5. Frontend receives tokens → calculates wedge angles → renders wheel

6. User spins wheel → lands on " mat"
   → POST /api/select {token_id: 2}

7. API Route:
   - Appends " mat" to context: "The cat sat on the mat"
   - Calls Gemini with new context
   - Returns next distribution

8. Frontend renders new wheel → repeat
```

---

## 4. Project Structure

### 4.1 Recommended Folder Structure

```text
llm-token-wheel/
├── app/
│   ├── api/                         # API routes (serverless functions)
│   │   ├── start/
│   │   │   └── route.ts            # POST /api/start
│   │   ├── select/
│   │   │   └── route.ts            # POST /api/select
│   │   └── session/
│   │       └── [id]/
│   │           └── route.ts        # GET /api/session/:id
│   ├── page.tsx                    # Main app page (client)
│   ├── layout.tsx                  # Root layout
│   └── globals.css                 # Global styles (Tailwind)
│
├── components/                      # React components
│   ├── PromptInput.tsx             # Prompt input form
│   ├── TokenWheel.tsx              # Wheel visualization (SVG + Framer Motion)
│   ├── GeneratedText.tsx           # Display accumulated text
│   ├── Controls.tsx                # Spin/Reset buttons
│   └── WheelWedge.tsx              # Individual wedge component
│
├── lib/                            # Utility functions
│   ├── api/
│   │   ├── gemini.ts               # Gemini API client
│   │   └── session.ts              # Session management
│   ├── utils/
│   │   ├── probabilities.ts        # Logprob → probability conversion
│   │   ├── wedges.ts               # Wedge angle calculations
│   │   └── svg.ts                  # SVG path generation
│   └── types.ts                    # TypeScript types
│
├── hooks/                          # Custom React hooks
│   ├── useSession.ts               # Session state management
│   ├── useWheel.ts                 # Wheel animation logic
│   └── useTokenSelection.ts        # Token selection logic
│
├── public/                         # Static assets
│   └── fonts/                      # Custom fonts
│
├── docs/                           # Documentation
│   ├── ARCHITECTURE.md             # Architecture details
│   ├── API_DESIGN.md               # API endpoint specs
│   └── TECH_STACK_RESEARCH.md      # This file
│
├── .env.local                      # Local environment variables (gitignored)
├── .env.example                    # Example env vars (committed)
├── next.config.js                  # Next.js configuration
├── tailwind.config.ts              # Tailwind CSS configuration
├── tsconfig.json                   # TypeScript configuration
├── package.json                    # Dependencies
└── README.md                       # Project overview
```

### 4.2 Key Files Explained

**API Route: `app/api/start/route.ts`**

- Creates new session
- Calls Gemini API with initial prompt
- Returns token distribution with probabilities

**API Route: `app/api/select/route.ts`**

- Handles token selection (spin or manual)
- Appends token to context
- Calls Gemini for next distribution

**Component: `components/TokenWheel.tsx`**

- Renders SVG wheel with wedges
- Handles Framer Motion rotation animation
- Manages click handlers for manual selection

**Lib: `lib/api/gemini.ts`**

- Wrapper for Gemini API calls
- Handles authentication, rate limiting, error handling
- Exports `getTokenLogprobs()` function

**Lib: `lib/utils/probabilities.ts`**

- Converts log probabilities to probabilities
- Adds "other" category
- Normalizes probability distributions

**Lib: `lib/utils/wedges.ts`**

- Calculates wedge angles from probabilities
- Generates SVG path data for wedges
- Handles edge cases (tiny wedges, label placement)

---

## 5. Comparison to Existing Versions

### 5.1 V1 & V2 (Local Inference) vs. V3 (Cloud API)

| Aspect           | V1/V2 (Local)             | V3 (Cloud API)          |
| ---------------- | ------------------------- | ----------------------- |
| **Model**        | GPT-2, TinyLlama (local)  | Gemini Flash (cloud)    |
| **Deployment**   | Docker container          | Vercel serverless       |
| **API Costs**    | $0 (local compute)        | ~$0.01 per 1000 req     |
| **Latency**      | 50-100ms (CPU)            | 200-500ms (net + API)   |
| **Memory**       | 2-6GB RAM                 | 0MB (serverless)        |
| **Scalability**  | Limited (single server)   | Unlimited (serverless)  |
| **Token Select** | True probabilistic        | Hybrid (API sel first)  |
| **Top Tokens**   | Unlimited (threshold)     | Max 20 (API limit)      |
| **Session**      | In-memory dict            | Vercel KV / Redis       |

### 5.2 Key Architectural Differences

#### Token Selection Pattern

**V1/V2 (Local):**

```python
# Get full probability distribution BEFORE sampling
probs = model.get_probabilities(context)  # All 50,257 tokens
top_tokens = filter_by_threshold(probs, 0.01)  # 10-30 tokens

# User spins wheel → frontend determines angle → backend samples
selected_token = sample_from_distribution(probs, user_angle)
```

**V3 (Cloud API):**

```python
# API samples token FIRST, returns logprobs AFTER
response = gemini.generate(context, logprobs=20)
chosen_token = response.text  # Already selected by API
top_20 = response.logprobs    # Top 20 alternatives

# Hybrid approach:
# - Spin mode: Show chosen_token (predetermined)
# - Manual mode: Re-run API with user's chosen token
```

**Educational Impact:**

- **Spin mode**: Wheel lands on API's predetermined choice (acceptable per professor)
- **Manual mode**: True exploration - re-run API with user's token choice
- **Trade-off**: Slight loss of "true" randomness in spin, but gain in scalability

#### Dynamic Threshold vs. Fixed Top-K

**V1/V2 (Local):**

- Threshold-based: Return all tokens ≥1%
- Adaptive secondary threshold: Add more if "other" >20%
- Result: 10-30 tokens depending on distribution shape

**V3 (Cloud API):**

- Fixed top-K: Gemini returns max 20 tokens
- Calculate "other" as remaining probability: 1.0 - sum(top_20)
- Result: Always 20 tokens + "other"

**Educational Trade-off:**

- **Loss**: Can't show as many tokens in peaked distributions
- **Gain**: Simpler to explain ("top 20 most likely tokens")
- **Mitigation**: "Other" category still demonstrates long tail

### 5.3 What to Preserve from V1/V2

**Core Educational Patterns (Keep):**

- ✅ Wedge size = probability (1:1 mapping)
- ✅ Spin vs. manual selection modes
- ✅ "Other" category for low-probability tokens
- ✅ Token-by-token generation visualization
- ✅ Context affects probability distribution

**Implementation Patterns (Keep):**

- ✅ Session-based API for state management
- ✅ Frontend calculates wedge angles from probabilities
- ✅ Frontend determines selected token (spin or click)
- ✅ Backend handles token appending and next distribution

**What Changes:**

- ❌ No more dynamic threshold (use fixed top-20)
- ❌ No more multiple model support (start with Gemini only)
- ❌ No more Docker deployment (Vercel serverless)
- ❌ Session storage moves to Vercel KV (not in-memory)

---

## 6. Cost & Performance Analysis

### 6.1 Gemini Flash Pricing (2026 rates)

**Input Tokens:**

- Prompts ≤128K tokens: $0.01 per 1M tokens
- Prompts >128K tokens: $0.04 per 1M tokens

**Output Tokens:**

- $0.04 per 1M tokens

**Example Cost Calculation:**

Scenario: Student generates 50 tokens (typical session)

```text
Request 1: Initial prompt (10 tokens) + logprobs
  - Input: 10 tokens × $0.01 / 1M = $0.0000001
  - Output: 1 token × $0.04 / 1M = $0.00000004
  - Total: $0.00000014

Request 2-50: Each subsequent token (11-60 tokens context)
  - Average: 35 tokens × $0.01 / 1M = $0.00000035
  - Output: 1 token × $0.04 / 1M = $0.00000004
  - Per request: $0.00000039

Total for 50 tokens:
  = $0.00000014 + (49 × $0.00000039)
  = $0.00000014 + $0.00001911
  = $0.00001925 per session

Cost for 1000 students (50 tokens each):
  = 1000 × $0.00001925
  = $0.01925 ≈ $0.02

Cost for 10,000 students:
  = $0.20
```

**Verdict:** Extremely affordable for educational use.

### 6.2 Vercel Pricing

**Free Tier (Hobby):**

- 100GB bandwidth per month
- 100GB-hours compute time
- Unlimited API routes (serverless functions)
- 1000 image optimizations

**Sufficient for:**

- ~10,000-50,000 requests per month (depending on payload size)
- Small to medium educational deployment

**Pro Tier ($20/month):**

- 1TB bandwidth
- 1000GB-hours compute
- Suitable for entire course (100-500 students)

### 6.3 Performance Characteristics

**Expected Latency:**

- Gemini API call: 200-500ms (network + inference)
- Next.js API route: 50-100ms (overhead)
- Total: 250-600ms per token generation

**Comparison to Local:**

- Local (V1/V2): 50-100ms (CPU inference)
- Cloud (V3): 250-600ms (network + API)
- **Trade-off**: 3-6x slower, but infinitely scalable

**Cold Start:**

- Vercel serverless: 100-300ms initial cold start
- Subsequent requests: <50ms (warm)
- **Mitigation**: Keep-alive requests every 5 minutes

**Optimization Strategies:**

1. **Caching**: Cache session data in Vercel KV (Redis)
2. **Parallel requests**: Prefetch next distribution while wheel spins
3. **Edge functions**: Deploy to Vercel Edge for lower latency
4. **Debouncing**: Prevent rapid-fire requests

---

## 7. Implementation Roadmap

### Phase 1: Core Infrastructure (Week 1-2)

**Goals:**

- Set up Next.js project with App Router
- Configure Gemini API integration
- Implement basic API routes

**Tasks:**

1. Initialize Next.js 14 project

   ```bash
   npx create-next-app@latest llm-token-wheel --typescript --tailwind --app
   ```

2. Install dependencies

   ```bash
   npm install @google/generative-ai framer-motion
   npm install -D @types/node
   ```

3. Set up environment variables

   ```bash
   # .env.local
   GEMINI_API_KEY=your_key_here
   ```

4. Create Gemini API client (`lib/api/gemini.ts`)

   - Authentication
   - Request/response handling
   - Error handling

5. Implement `/api/start` endpoint

   - Create session
   - Call Gemini with logprobs
   - Return token distribution

6. Implement session storage (start with in-memory, migrate to Vercel KV later)

**Deliverables:**

- ✅ Working Next.js app
- ✅ Gemini API integration
- ✅ Basic API routes
- ✅ Session management

---

### Phase 2: Frontend Visualization (Week 3-4)

**Goals:**

- Build token wheel with SVG
- Implement Framer Motion animations
- Create user controls

**Tasks:**

1. Create `TokenWheel` component

   - SVG circle with wedges
   - Path calculation for wedges
   - Labels and hover effects

2. Implement wedge calculation logic (`lib/utils/wedges.ts`)

   ```typescript
   function calculateWedges(tokens: Token[]): Wedge[] {
     let startAngle = 0;
     return tokens.map((token) => {
       const angle = (token.probability / 1.0) * 360;
       const wedge = {
         ...token,
         startAngle,
         endAngle: startAngle + angle,
       };
       startAngle += angle;
       return wedge;
     });
   }
   ```

3. Add Framer Motion rotation

   ```typescript
   <motion.div
     animate={{ rotate: landingAngle }}
     transition={{
       duration: 3,
       ease: [0.17, 0.67, 0.12, 0.99],
     }}
   />
   ```

4. Implement click handlers for manual selection

5. Create `PromptInput` and `GeneratedText` components

6. Build responsive layout (desktop + mobile)

**Deliverables:**

- ✅ Interactive token wheel
- ✅ Smooth spin animations
- ✅ Manual selection via clicks
- ✅ Responsive design

---

### Phase 3: Token Selection Logic (Week 5)

**Goals:**

- Implement spin vs. manual modes
- Handle "other" category selection
- Integrate frontend with backend

**Tasks:**

1. Implement spin mode

   - Generate random landing angle
   - Animate wheel rotation
   - Determine selected wedge
   - Send token_id to `/api/select`

2. Implement manual mode

   - Click wedge → get token_id
   - Send to `/api/select` immediately

3. Handle "other" selection

   - If token_id === -1, backend re-runs API
   - Display actual sampled token (not "OTHER")

4. Implement `/api/select` endpoint

   - Handle both explicit and "other" selections
   - Append token to context
   - Get next distribution
   - Return updated state

5. Add state management (React Context or Zustand)

**Deliverables:**

- ✅ Both selection modes working
- ✅ "Other" category samples correctly
- ✅ Seamless frontend-backend integration

---

### Phase 4: Polish & Deployment (Week 6)

**Goals:**

- Deploy to Vercel
- Add error handling
- Optimize performance
- Test with students

**Tasks:**

1. Migrate session storage to Vercel KV

   ```typescript
   import { kv } from "@vercel/kv";
   await kv.set(`session:${id}`, sessionData, { ex: 3600 });
   ```

2. Add error handling and loading states

   - Network errors
   - API rate limits
   - Invalid sessions

3. Implement rate limiting (Vercel KV)

   ```typescript
   const requests = await kv.incr(`rate:${ip}`);
   if (requests > 100) throw new Error("Rate limit exceeded");
   await kv.expire(`rate:${ip}`, 60);
   ```

4. Optimize performance

   - Prefetch next distribution while wheel spins
   - Cache common prompts

5. Deploy to Vercel

   ```bash
   vercel --prod
   ```

6. Set up environment variables in Vercel dashboard

7. Test with pilot group of students

**Deliverables:**

- ✅ Production deployment on Vercel
- ✅ Robust error handling
- ✅ Performance optimizations
- ✅ Student feedback incorporated

---

### Phase 5: Enhancements (Optional, Week 7+)

**Goals:**

- Add advanced features
- Improve educational value
- Support multiple models

**Tasks:**

1. Add temperature control slider
2. Add top-k control (if needed)
3. Show probability percentages on hover
4. Add generation history view
5. Export session as JSON/CSV
6. Add OpenAI GPT-4o as alternative model
7. Add comparison mode (Gemini vs. OpenAI side-by-side)
8. Add accessibility improvements (ARIA labels, keyboard navigation)

**Deliverables:**

- ✅ Enhanced educational features
- ✅ Multi-model support (optional)
- ✅ Accessibility compliance

---

## 8. Questions Answered

### Q1: Can Gemini logprobs return enough top tokens for a visually interesting wheel?

**Answer:** Yes, with caveats.

- **Gemini**: Up to 20 top alternative tokens
- **Requirement**: Need 10-30 tokens for visual interest
- **Verdict**: 20 tokens is sufficient, especially with "other" category
- **Trade-off**: Can't show as many tokens as local threshold-based approach
  (10-30 dynamic), but 20 is acceptable

**Recommendation:**

- Use all 20 tokens from Gemini
- Add "other" category for remaining probability
- If needed, use threshold filtering on the 20 (e.g., only show tokens ≥0.5%)
  to reduce visual clutter

### Q2: What's the best way to handle session state across serverless invocations?

**Answer:** Vercel KV (Redis) or upstash/redis.

**Options Compared:**

| Solution      | Pros           | Cons             | Recommendation   |
| ------------- | -------------- | ---------------- | ---------------- |
| **Vercel KV** | Built-in, easy | Vercel-specific  | **Best choice**  |
| **Redis**     | Portable, free | External service | Good alternative |
| **In-memory** | Fast, simple   | No serverless    | Not suitable     |
| **Database**  | Persistent     | Overkill, slower | Not needed       |

**Implementation:**

```typescript
import { kv } from '@vercel/kv';

// Store session
await kv.set(`session:${sessionId}`, {
  context: "The cat sat on the",
  tokens: [...],
  history: [],
}, { ex: 3600 }); // 1 hour expiry

// Retrieve session
const session = await kv.get(`session:${sessionId}`);
```

**Session Data Structure:**

```typescript
interface Session {
  session_id: string;
  context: string; // Current accumulated text
  tokens: Token[]; // Current token distribution
  history: TokenSelection[]; // Past selections
  created_at: number;
  last_accessed: number;
}
```

### Q3: How should we structure the API to support manual selection?

**Answer:** Two-endpoint pattern with token_id-based selection.

**Endpoint 1: `/api/start`**

- Create session
- Get initial logprobs from Gemini
- Return token distribution

**Endpoint 2: `/api/select`**

- Accept `{ session_id, token_id }`
- If `token_id !== -1`: Use explicit token
- If `token_id === -1`: Re-run Gemini to sample from "other"
- Append token to context
- Get next logprobs
- Return updated state

**Why this works:**

- **Spin mode**: Frontend determines token_id from rotation angle → send to `/api/select`
- **Manual mode**: Frontend gets token_id from clicked wedge → send to `/api/select`
- **Unified**: Same endpoint handles both modes
- **"Other" handling**: Backend re-runs API when token_id === -1

**Example Request:**

```typescript
// Spin mode: Landed on token_id 5
await fetch("/api/select", {
  method: "POST",
  body: JSON.stringify({ session_id: "abc", token_id: 5 }),
});

// Manual mode: Clicked "other" wedge
await fetch("/api/select", {
  method: "POST",
  body: JSON.stringify({ session_id: "abc", token_id: -1 }),
});
```

### Q4: Are there any Gemini API limitations that might affect the educational experience?

**Answer:** Yes, but all are manageable.

#### Limitation 1: Max 20 top tokens

- **Impact**: Can't show as many tokens as local (10-30 dynamic)
- **Mitigation**: 20 is sufficient with "other" category
- **Educational impact**: Minimal - still demonstrates distribution

#### Limitation 2: API selects token first

- **Impact**: Can't get probabilities before sampling (as local does)
- **Mitigation**: Hybrid approach - show predetermined result in spin mode
- **Educational impact**: Acceptable per professor's guidance

#### Limitation 3: Rate limits (1500 req/min free tier)

- **Impact**: Could hit limits with many concurrent users
- **Mitigation**: Debouncing, caching, paid tier if needed
- **Educational impact**: None (unlikely to hit limits)

#### Limitation 4: Latency (200-500ms)

- **Impact**: Slower than local inference (50-100ms)
- **Mitigation**: Prefetching, wheel animation masks latency
- **Educational impact**: None (perceived as part of "spinning" process)

#### Limitation 5: No offline mode

- **Impact**: Requires internet connection
- **Mitigation**: Not needed (deployed on Vercel)
- **Educational impact**: None (classroom has Wi-Fi)

**Verdict:** No show-stoppers. All limitations have acceptable
mitigations.

### Q5: What patterns from V1/V2 should we preserve vs. change?

**Preserve (Educational Core):**

- ✅ Wedge size = probability (1:1 visual mapping)
- ✅ Spin vs. manual selection modes
- ✅ "Other" category demonstrates long tail
- ✅ Token-by-token generation
- ✅ Context affects probabilities

**Preserve (Implementation Patterns):**

- ✅ Session-based API for state management
- ✅ Frontend calculates wedge angles
- ✅ Frontend determines selected token (via rotation or click)
- ✅ Backend handles token appending
- ✅ Dynamic "other" category

**Change (Architectural):**

- ❌ Docker → Vercel serverless
- ❌ Local models → Cloud API (Gemini)
- ❌ In-memory sessions → Vercel KV (Redis)
- ❌ Dynamic threshold → Fixed top-20
- ❌ True probabilistic spin → Hybrid (predetermined in spin mode)

**Change (Features):**

- ❌ Multiple models (GPT-2, TinyLlama) → Start with Gemini only
- ❌ Adjustable thresholds → Fixed logprobs=20
- ❌ Secondary threshold logic → Simpler "top 20 + other" model

**Why These Changes:**

- Align with cloud API constraints (fixed top-k)
- Simplify for MVP (single model)
- Optimize for serverless deployment
- Maintain educational core while adapting implementation

---

## Appendices

### Appendix A: Framer Motion Resources

**Official Documentation:**

- [React transitions — Configure Motion animations](https://www.framer.com/motion/transition/)
- [Easing functions — Adjust animation timing](https://www.framer.com/motion/easing-functions/)
- [React `<motion />` component](https://www.framer.com/motion/component/)

**Tutorials:**

- [Framer Motion React Animations | Refine](https://refine.dev/blog/framer-motion/)
- [Spinner Animation in React](https://dev.to/darthknoppix/spinner-animation-in-react-4bkk)
- [The Framer Classic book » Spinning wheel animation](https://classic.framerbook.com/hyperlapse/spinning-wheel-animation/)

### Appendix B: SVG Wheel Rendering Resources

**Tutorials:**

- [Pie chart with React](https://www.react-graph-gallery.com/pie-plot)
- [How To Make Pie Chart with React and SVG](
  https://radzion.com/blog/piechart/)
- [SVG Pie Chart using React and D3](
  https://medium.com/localmed-engineering/svg-pie-chart-using-react-and-d3-43a381ce7246)

**Libraries:**

- [react-minimal-pie-chart](
  https://www.npmjs.com/package/react-minimal-pie-chart) - Lightweight SVG
  pie charts (< 2kB)
- [react-svg-piechart](https://github.com/cedricdelpoux/react-svg-piechart) -
  Responsive pie chart using only SVG

### Appendix C: Next.js Resources

**Context7 Documentation:**

- [Next.js App Router Official Docs](https://context7.com/vercel/next.js/llms.txt)
- Used for all Next.js API route patterns, folder structure, and best practices

### Appendix D: Vercel Resources

**Context7 Documentation:**

- [Vercel Deployment Documentation](https://context7.com/vercel/vercel/llms.txt)
- Used for serverless configuration, environment variables, and deployment patterns

### Appendix E: Gemini API Resources

**Context7 Documentation:**

- [Google Gemini API - Logprobs](
  https://context7.com/googlecloudplatform/generative-ai/llms.txt?topic=logprob)
- Used for logprobs functionality, request/response format, and API limits

---

## Summary

This research provides a comprehensive foundation for building the **LLM Token
Wheel** using cloud APIs. The key findings demonstrate that:

1. **Gemini Flash is suitable** for this educational application with 20 top
   tokens and logprobs support
2. **Next.js + Vercel** provides an excellent serverless architecture
3. **Framer Motion** enables smooth wheel animations
4. **Architectural changes are manageable** - the educational core can be
   preserved while adapting to cloud API constraints
5. **Cost is negligible** - ~$0.02 per 1000 students (50 tokens each)

The implementation roadmap provides a clear 6-week path to production, with
optional enhancements in phase 5.

**Next Steps:**

1. Review this research with course instructors
2. Validate architectural approach (hybrid spin mode)
3. Get approval for Gemini API usage
4. Begin Phase 1 implementation (Core Infrastructure)

---

**Document Version:** 1.0
**Date:** January 11, 2026
**Author:** Claude (via user research request)
**Status:** Ready for review

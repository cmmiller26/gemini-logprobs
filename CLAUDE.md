# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Probability Pulse** is an educational Next.js application that visualizes how Large Language Models predict the next token using Gemini API's `logprobs` feature. Users can interactively build sentences token-by-token using game-based interfaces (Plinko, spinning wheel, slot machine, etc.) weighted by LLM probabilities.

## Development Commands

### Running the Development Server
```bash
npm run dev
```
Opens at http://localhost:3000

### Building for Production
```bash
npm run build
npm start
```

### Linting
```bash
npm run lint
```

## Environment Setup

Required environment variable:
- `GEMINI_API_KEY`: Google Generative AI API key for Gemini 2.0 Flash model

Create `.env.local` file:
```
GEMINI_API_KEY=your_api_key_here
```

## Architecture

### Core Concept: Token Prediction Visualization

The app demonstrates autoregressive text generation by:
1. Getting top-N token alternatives and log probabilities from Gemini API
2. Converting log probabilities to normalized probabilities using softmax
3. Presenting alternatives through probability-weighted game interfaces
4. Building sentences token-by-token based on user selection

### Key Architectural Patterns

**1. Token Probability Pipeline**
```
User Input → Gemini API (logprobs) → Softmax Normalization → Add <OTHER> Category → Game Visualization → User Selection → Append to Context → Repeat
```

**2. Softmax with Numerical Stability**
The `logprobsToProbs()` function in `src/lib/llm.ts` implements softmax with max-subtraction to prevent overflow:
```typescript
const maxLogProb = Math.max(...logprobs);
const expValues = logprobs.map(lp => Math.exp(lp - maxLogProb));
const probability = expValue / sumExp;
```

**3. <OTHER> Category**
The `addOtherCategory()` function adds a catch-all category for remaining probability mass (tokens outside top-N). Only added if remaining probability ≥ 1%. Token ID is always -1 for `<OTHER>`.

**4. Token Merging and Filtering**
In `WordBuilder.tsx`, tokens are:
- Filtered using `isValidToken()` to remove control tokens (`<ctrl100>`, `<0x0A>`, etc.)
- Merged by trimmed lowercase text (combining variations like `" cat"` and `"cat"`)
- Grouped into main (≥3% probability) and other (<3% probability)

### File Structure

```
src/
├── app/
│   ├── api/
│   │   ├── next-token/route.ts    # Main API: get next token alternatives
│   │   ├── lookahead/route.ts     # Lookahead mode: preview continuations
│   │   └── predict/route.ts       # Multi-token prediction
│   ├── page.tsx                    # Entry point (renders WordBuilder)
│   └── globals.css                 # Global styles + ambient animations
├── components/
│   ├── WordBuilder.tsx             # Main orchestrator component
│   ├── DebugPanel.tsx              # Debug panel for inspecting probabilities
│   └── games/
│       ├── Plinko.tsx              # Plinko board game
│       ├── SpinWheel.tsx           # Spinning wheel game
│       ├── SlotMachine.tsx         # Slot machine game
│       ├── DiceRoll.tsx            # Dice roll game
│       └── LotteryBalls.tsx        # Lottery balls game
└── lib/
    ├── llm.ts                      # Gemini API client + probability utilities
    ├── colors.ts                   # Universal color system for tokens
    ├── types.ts                    # TypeScript interfaces
    └── utils.ts                    # Utility functions
```

### Game Components

All game components follow the same interface pattern:
```typescript
interface GameProps {
    alternatives: Alternative[];      // Token alternatives with probabilities
    onSelect: (token: string) => void; // Callback when token selected
    disabled?: boolean;
}
```

Each game visualizes probabilities differently:
- **Plinko**: Ball drop with bin widths proportional to probability
- **SpinWheel**: Rotating wheel with wedge sizes based on probability
- **SlotMachine**: Spinning reels that land on weighted random token
- **DiceRoll**: Animated dice roll with probability-weighted outcome
- **LotteryBalls**: Bouncing balls where selection frequency matches probability

### Color System

The `colors.ts` module provides a universal color palette ensuring consistent token coloring across all game components:
- `getTokenColor(index)`: Main token color (85% opacity)
- `getTokenColorLight(index)`: Background variant (15% opacity)
- `getTokenColorBorder(index)`: Border variant (60% opacity)
- `OTHER_COLOR`: Dedicated slate gray for `<OTHER>` category
- `isValidToken(token)`: Filters out control tokens and special characters

### API Endpoints

**POST /api/next-token**
- Request: `{ prefix: string, model?: string }`
- Response: `{ alternatives: Token[] }` where Token includes `token`, `token_id`, `probability`, `log_probability`, `is_other`
- Uses `maxOutputTokens: 1` to predict only the immediate next token
- Returns top 20 alternatives with `<OTHER>` category

**POST /api/lookahead**
- Request: `{ prefix: string, previewLength?: number }`
- Response: `{ alternatives: LookaheadAlternative[] }` with `preview` field showing continuation
- For each top-5 token, generates a preview continuation to help users see where each choice leads

**POST /api/predict**
- Multi-token prediction endpoint
- Returns full sentence predictions with per-token probabilities

### Important Implementation Notes

**1. Gemini API Configuration**
- Model: `gemini-2.0-flash`
- `responseLogprobs: true` to get log probabilities
- `logprobs: 20` to get top 20 alternatives
- `maxOutputTokens: 1` for single token prediction

**2. System Instruction for Text Continuation**
The `/api/next-token` endpoint uses a system instruction to enforce pure text continuation (not chat/completion):
```typescript
const systemInstruction = `You are a text continuation engine. Your ONLY job is to continue the text that follows.
Do not start a new sentence. Do not add punctuation. Do not be helpful.
Simply predict what word or token comes immediately next in the sequence.
Continue from exactly where the text ends, maintaining the same case and style.`;
```

**3. Probability Normalization**
After fetching from API:
1. Extract log probabilities from `topCandidates[0].candidates`
2. Apply softmax: `logprobsToProbs()`
3. Add `<OTHER>` category: `addOtherCategory()`
4. Total probability always sums to 1.0

**4. Token Selection Flow**
- Main tokens (≥3%): Passed directly to game components
- Other tokens (<3%): Grouped, displayed separately
- When `<OTHER>` selected: Random token chosen from other group
- Selected token appended to context with proper spacing

### State Management

WordBuilder component manages:
- `seedInput`: Initial prompt text
- `builtSentence`: Array of selected tokens
- `alternatives`: Current token alternatives from API
- `selectedGame`: Active game component
- `model`: Gemini model selection
- `maxTokens`: Maximum tokens to display in game

### Educational Insights

The app demonstrates key LLM concepts:
1. **Autoregressive Generation**: Each token depends on all previous tokens
2. **Local vs. Global Optimization**: Greedily picking highest probability doesn't guarantee best sentence
3. **Temperature Effects**: Higher temperature flattens distribution (more random)
4. **Token Probabilities**: Not all "words" are tokens; subword tokenization affects probabilities
5. **<OTHER> Category**: Top-N tokens don't capture full distribution

## Technical Documentation

See `docs/` directory for detailed specifications:
- `ARCHITECTURE.md`: Deep dive into LLM token prediction, softmax, sampling strategies
- `API_ENDPOINTS.md`: Full API specifications (note: session-based endpoints not yet implemented)
- `FUNCTIONS_UTILITIES.md`: Detailed function specifications (note: many utilities like wedge calculation, session management not yet implemented)

**Note**: The docs describe a more comprehensive architecture with session management and additional utilities. Current implementation is simplified:
- No session management (stateless API calls)
- No Vercel KV storage
- Simplified probability utilities in `llm.ts`
- Color system implemented in `colors.ts` (not in docs)

## Common Development Tasks

### Adding a New Game Component

1. Create file in `src/components/games/YourGame.tsx`
2. Import `Alternative` type from WordBuilder
3. Implement interface:
   ```typescript
   interface YourGameProps {
       alternatives: Alternative[];
       onSelect: (token: string) => void;
       disabled?: boolean;
   }
   ```
4. Use `getTokenColor(index)` from `@/lib/colors` for consistent coloring
5. Add to `selectedGame` type and switch statement in WordBuilder

### Modifying Probability Thresholds

- Main/Other threshold: `MIN_PROBABILITY_THRESHOLD` in WordBuilder (currently 3%)
- Other category threshold: `minProbability` parameter in `addOtherCategory()` (currently 1%)

### Debugging Probabilities

Enable debug panel in UI (bug icon) to see:
- Raw API responses
- Token probabilities before/after normalization
- Token merging and filtering results

### Working with Gemini API

Always use the wrapper functions in `src/lib/llm.ts`:
- `getNextToken(prefix, model)`: Single token prediction
- `getTokenWithLookahead(prefix, previewLength)`: Token with continuation preview
- `getProbabilities(text)`: Multi-token prediction

Don't call Gemini API directly from components.

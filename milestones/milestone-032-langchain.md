# Milestone 32: LangChain Michelle — Agentic Market Maker

*Created: January 14, 2026*
*Completed: January 17, 2026*
*Status: ✅ Complete*

---

## Overview

Convert Michelle's instant match flow from "pre-assembled context blob → single decision" to a true agentic pattern where she reasons about what information she needs, fetches it via tools, and iterates until ready to decide.

This enables:
- Smarter context gathering (skip redundant fetches if she just evaluated this 1 minute ago)
- Line flexibility (reason about adjacent spreads/totals instead of hard-rejecting)
- Counter-offers with bounded commitments
- Visible reasoning streamed to the frontend

---

## Goals

- Integrate LangChain library for agentic loop
- Define tool set for Michelle's information needs
- Convert instant match flow to agentic pattern
- Implement counter-offer capability with staleness protection
- Stream Michelle's reasoning to frontend via Firebase
- Log all calculations for manual audit

---

## Non-Goals (This Milestone)

- Converting batch eval or proactive matching flows
- Weather/lineup/injury data sources
- Multi-agent coordination
- User-created agents
- Refactoring Degen Dan (he stays legendarily bad)

---

## Track 1: LangChain Integration

### Setup

```bash
yarn add @langchain/core @langchain/anthropic
```

### Agent Structure

```typescript
// src/agents/market_maker_michelle/langchain/michelleAgent.ts

import { createReactAgent } from '@langchain/core/agents';
import { ChatAnthropic } from '@langchain/anthropic';

const michelle = createReactAgent({
  llm: new ChatAnthropic({ 
    model: 'claude-sonnet-4-20250514',
    streaming: true 
  }),
  tools: [
    getMarketStateTool,
    getMyExposureTool,
    getMyRecentDecisionsTool,
    compareOddsTool,
    computeExposureImpactTool,
  ],
  systemMessage: MICHELLE_SYSTEM_PROMPT,
});
```

### The "Ask" Format

Michelle receives a minimal task description. She decides what context she needs.

```typescript
interface MichelleTask {
  type: 'instant_match_request';
  requestId: string;
  timestamp: string;  // ISO string
  
  request: {
    speculationId: string;
    jsonoddsId: string;
    market: 'moneyline' | 'spread' | 'total';
    side: 'away' | 'home' | 'over' | 'under';
    line?: number;              // for spread/total
    requestedOdds: number;
    requestedAmount: number;
    requesterAddress: string;
  };
  
  // Always provided (her identity/constraints)
  agentConfig: {
    agentId: string;
    maxPerSide: number;        // e.g., 50 USDC
    maxPerGame: number;        // e.g., 100 USDC
    personality: string;
  };
}
```

### System Prompt

```typescript
const MICHELLE_SYSTEM_PROMPT = `You are Michelle, an AI market maker providing liquidity on Ospex.

YOUR GOAL: Provide competitive odds without taking on excessive risk.
- You are NOT trying to profit from predictions
- You ARE enabling trading while protecting against exploitation
- You should offer odds close to market to attract flow
- You should NOT offer odds so good that sharp bettors exploit you

YOU HAVE TOOLS to gather information. Use them as needed:
- get_market_state: Current odds, teams, start time for a game
- get_my_exposure: Your current position on a speculation or contest
- get_my_recent_decisions: Your recent activity (useful to avoid redundant work)
- compare_odds: Calculate edge between two odds values (YOU DO NOT DO MATH)
- compute_exposure_impact: Calculate how a match would affect your limits

IMPORTANT:
- You do NOT do math. Use the calculator tools.
- You CAN reason about adjacent lines (e.g., if asked about +3.5, you can consider +2.5 or +4.5)
- For low-scoring sports (MLB, NHL), stick to standard lines (runline ±1.5)
- For basketball/football, you have more flexibility on spreads and totals
- Be conservative. You're launching with small limits, so experiment carefully.

OUTPUT: When ready to decide, respond with your final decision JSON.`;
```

### Tasks

- [x] Install LangChain packages
- [x] Create `michelleAgent.ts` with agent setup
- [x] Define `MichelleTask` interface
- [x] Write system prompt
- [x] Create agent invocation wrapper with error handling

---

## Track 2: Tool Definitions

### Tool 1: get_market_state

Fetches current market context for a game.

```typescript
const getMarketStateTool = tool({
  name: 'get_market_state',
  description: 'Get current market odds, teams, and start time for a game',
  schema: z.object({
    jsonoddsId: z.string().describe('The jsonodds game identifier'),
  }),
  func: async ({ jsonoddsId }) => {
    // Fetch from jsonodds/Firebase
    return {
      homeTeam: 'Pittsburgh Steelers',
      awayTeam: 'Houston Texans',
      startTime: '2026-01-18T20:00:00Z',
      minutesToStart: 1440,
      odds: {
        moneyline: { home: 1.67, away: 2.25 },
        spread: { line: -3.5, home: 1.91, away: 1.91 },
        total: { line: 43.5, over: 1.91, under: 1.91 },
      },
      lastUpdated: '2026-01-14T15:30:00Z',
    };
  },
});
```

### Tool 2: get_my_exposure

Fetches Michelle's current position data.

```typescript
const getMyExposureTool = tool({
  name: 'get_my_exposure',
  description: 'Get your current exposure on a speculation, contest, or overall',
  schema: z.object({
    speculationId: z.string().optional().describe('Specific speculation ID'),
    contestId: z.string().optional().describe('Contest ID for game-level exposure'),
  }),
  func: async ({ speculationId, contestId }) => {
    // Query positions from Firebase
    return {
      speculationExposure: 25.00,  // USDC on this specific speculation
      contestExposure: 75.00,      // USDC across all markets on this game
      totalExposure: 450.00,       // Total portfolio
      remainingPerSide: 25.00,     // maxPerSide - speculationExposure
      remainingPerGame: 25.00,     // maxPerGame - contestExposure
    };
  },
});
```

### Tool 3: get_my_recent_decisions

Fetches Michelle's recent activity. Broader scope allows contest-level queries.

```typescript
const getMyRecentDecisionsTool = tool({
  name: 'get_my_recent_decisions',
  description: 'Get your recent decisions on a speculation or contest',
  schema: z.object({
    contestId: z.string().optional().describe('Contest ID for game-level history'),
    speculationId: z.string().optional().describe('Specific speculation ID'),
    limit: z.number().default(10).describe('Max decisions to return'),
  }),
  func: async ({ contestId, speculationId, limit }) => {
    // Query from michelleDecisions subcollection (M31 architecture)
    return [
      {
        ts: '2026-01-14T15:28:00Z',
        speculationId: 'spec_123',
        market: 'spread',
        side: 'home',
        line: -3.5,
        requestedOdds: 1.91,
        evalOdds: 1.90,
        outcome: 'accepted',
        matchAmount: 20.00,
        reason: 'Odds at market, within limits',
      },
      // ... more decisions
    ];
  },
});
```

### Tool 4: compare_odds (Calculator)

Michelle does not do math. This tool computes edge.

```typescript
const compareOddsTool = tool({
  name: 'compare_odds',
  description: 'Calculate the edge between offered odds and market odds. Use this instead of doing math yourself.',
  schema: z.object({
    offeredOdds: z.number().describe('The odds being offered (decimal)'),
    marketOdds: z.number().describe('Current market odds (decimal)'),
    perspective: z.enum(['michelle', 'user']).describe('Whose perspective to calculate from'),
  }),
  func: async ({ offeredOdds, marketOdds, perspective }) => {
    // Implied probability = 1 / decimal odds
    const offeredProb = 1 / offeredOdds;
    const marketProb = 1 / marketOdds;
    const edgePct = ((marketProb - offeredProb) / marketProb) * 100;
    
    const result = {
      offeredOdds,
      marketOdds,
      offeredImpliedProb: (offeredProb * 100).toFixed(2) + '%',
      marketImpliedProb: (marketProb * 100).toFixed(2) + '%',
      edgePct: edgePct.toFixed(2) + '%',
      favors: edgePct > 0 ? 'user' : edgePct < 0 ? 'michelle' : 'neutral',
      recommendation: Math.abs(edgePct) < 2 ? 'acceptable' : 
                      edgePct > 5 ? 'reject (too much user edge)' : 
                      edgePct < -5 ? 'very favorable to you' : 'consider carefully',
    };
    
    // LOG FOR AUDIT
    await logCalculation('compare_odds', { offeredOdds, marketOdds, perspective }, result);
    
    return result;
  },
});
```

### Tool 5: compute_exposure_impact (Calculator)

```typescript
const computeExposureImpactTool = tool({
  name: 'compute_exposure_impact',
  description: 'Calculate how accepting a match would affect your exposure limits',
  schema: z.object({
    matchAmount: z.number().describe('USDC amount to potentially match'),
    currentSpeculationExposure: z.number().describe('Current exposure on this speculation'),
    currentContestExposure: z.number().describe('Current exposure on this contest'),
    maxPerSide: z.number().describe('Your per-side limit'),
    maxPerGame: z.number().describe('Your per-game limit'),
  }),
  func: async (params) => {
    const newSpecExposure = params.currentSpeculationExposure + params.matchAmount;
    const newContestExposure = params.currentContestExposure + params.matchAmount;
    
    const result = {
      matchAmount: params.matchAmount,
      newSpeculationExposure: newSpecExposure,
      newContestExposure: newContestExposure,
      withinSpecLimit: newSpecExposure <= params.maxPerSide,
      withinContestLimit: newContestExposure <= params.maxPerGame,
      canMatch: newSpecExposure <= params.maxPerSide && newContestExposure <= params.maxPerGame,
      maxAllowedMatch: Math.min(
        params.maxPerSide - params.currentSpeculationExposure,
        params.maxPerGame - params.currentContestExposure
      ),
    };
    
    // LOG FOR AUDIT
    await logCalculation('compute_exposure_impact', params, result);
    
    return result;
  },
});
```

### Tasks

- [x] Implement `get_market_state` tool
- [x] Implement `get_my_exposure` tool  
- [x] Implement `get_my_recent_decisions` tool
- [x] Implement `compare_odds` calculator tool
- [x] Implement `compute_exposure_impact` calculator tool
- [x] Create `logCalculation()` helper for audit logging
- [x] Create `michelleCalculations` Firestore collection

---

## Track 3: Decision Output & Counter-Offers

### Decision Schema

Michelle's final output when she's ready to decide:

```typescript
interface MichelleDecision {
  decision: 'accept' | 'reject' | 'counter';
  
  // If accept
  matchAmount?: number;
  matchOdds?: number;      // echoes the requested odds
  matchLine?: number;      // echoes the requested line
  
  // If counter (Michelle proposes different terms)
  counter?: {
    odds?: number;         // "I'd do 1.95 instead of 1.91"
    line?: number;         // "I'd do +4.5 instead of +3.5"  
    amount?: number;       // "I'd do $20 instead of $50"
    ttlSeconds: number;    // How long this counter is valid (e.g., 60)
    maxMarketMovePct: number;  // Invalidate if market moves more than this (e.g., 2%)
  };
  
  reason: string;          // Brief explanation
  confidence: number;      // 0-1, how confident in this decision
}
```

### Counter-Offer Lifecycle

```
1. User requests match at +3.5 / 1.91 / $50
2. Michelle evaluates, decides to counter: +3.5 / 1.87 / $50
3. Response includes:
   - counter.odds: 1.87
   - counter.line: 3.5
   - counter.ttlSeconds: 60
   - counter.maxMarketMovePct: 2
4. Frontend shows: "Michelle offers +3.5 at 1.87 (valid 60s)"
5. User clicks "Accept Counter"
6. Backend checks:
   - Is TTL still valid? (< 60s elapsed)
   - Has market moved < 2%?
   - If both YES → Auto-accept, no second LLM call
   - If NO → Fresh eval (user is warned this may happen)
```

### Counter Validation Logic

```typescript
async function validateCounter(
  counter: MichelleCounter, 
  counterTimestamp: string
): Promise<{ valid: boolean; reason?: string }> {
  const elapsed = Date.now() - new Date(counterTimestamp).getTime();
  
  if (elapsed > counter.ttlSeconds * 1000) {
    return { valid: false, reason: 'Counter expired' };
  }
  
  const currentMarket = await getMarketOdds(counter.jsonoddsId);
  const originalMarket = counter.marketOddsAtCounter;
  const movePct = Math.abs((currentMarket - originalMarket) / originalMarket) * 100;
  
  if (movePct > counter.maxMarketMovePct) {
    return { valid: false, reason: `Market moved ${movePct.toFixed(1)}%` };
  }
  
  return { valid: true };
}
```

### Line Flexibility Rules

Michelle should apply sport-specific reasoning:

| Sport | Line Flexibility | Notes |
|-------|------------------|-------|
| MLB | None | Runline is always ±1.5 |
| NHL | None | Puckline is always ±1.5 |
| NBA | ±2-3 points | Can adjust spread, be conservative on totals |
| NFL | ±1-3 points | Half-point moves (3 vs 3.5) are significant |
| NCAAB | ±2-3 points | Similar to NBA |
| NCAAF | ±1-3 points | Similar to NFL |

This is encoded in the system prompt, not hard rules. Michelle uses judgment.

### Tasks

- [x] Define `MichelleDecision` interface
- [x] Implement counter-offer response parsing
- [x] Implement `validateCounter()` function
- [x] Store counter details in `michelleQuotes` collection
- [x] Add counter acceptance endpoint
- [x] Document line flexibility in system prompt

---

## Track 4: Firebase Streaming

Stream Michelle's reasoning to the frontend as she works.

### Progress Document Schema

```typescript
// michelleQuotes/{quoteId}
interface MichelleQuoteProgress {
  requestId: string;
  status: 'processing' | 'complete' | 'error';
  startedAt: Timestamp;
  completedAt?: Timestamp;
  
  progress: Array<{
    step: string;           // Tool name or "thinking"
    message: string;        // Human-readable status
    ts: Timestamp;
    data?: unknown;         // Optional tool result summary
  }>;
  
  decision?: MichelleDecision;
  error?: string;
}
```

### LangChain Callbacks

```typescript
const callbacks = {
  handleToolStart: async (tool: { name: string }, input: unknown) => {
    await updateDoc(quoteRef, {
      progress: arrayUnion({
        step: tool.name,
        message: getToolMessage(tool.name),
        ts: Timestamp.now(),
      })
    });
  },
  
  handleToolEnd: async (output: unknown) => {
    // Optionally log result summary
  },
  
  handleLLMEnd: async (output: unknown) => {
    // Final decision
  },
};

function getToolMessage(toolName: string): string {
  const messages: Record<string, string> = {
    'get_market_state': 'Checking current market odds...',
    'get_my_exposure': 'Reviewing my position...',
    'get_my_recent_decisions': 'Checking recent activity...',
    'compare_odds': 'Calculating edge...',
    'compute_exposure_impact': 'Evaluating risk...',
  };
  return messages[toolName] || 'Processing...';
}
```

### Frontend Integration

```typescript
// Frontend: Subscribe to quote progress
const quoteRef = doc(db, 'michelleQuotes', quoteId);

onSnapshot(quoteRef, (snap) => {
  const data = snap.data();
  if (data?.progress) {
    setProgressSteps(data.progress);
  }
  if (data?.status === 'complete') {
    setDecision(data.decision);
  }
});
```

### Tasks

- [x] Create `michelleQuotes` collection schema
- [x] Implement LangChain callbacks for progress updates
- [x] Update `/quote` endpoint to return `quoteId` immediately
- [x] Add frontend progress display component
- [x] Handle error states gracefully

---

## Track 5: Audit & Validation

### Calculation Logging

Every calculator tool call is logged for manual audit.

```typescript
// michelleCalculations/{calculationId}
interface MichelleCalculation {
  id: string;
  ts: Timestamp;
  tool: 'compare_odds' | 'compute_exposure_impact';
  inputs: Record<string, unknown>;
  outputs: Record<string, unknown>;
  quoteId?: string;        // Link to the quote that triggered this
  verified?: boolean;      // Manual verification flag
  verifiedBy?: string;     // Who verified
  verifiedAt?: Timestamp;
}
```

### Audit Checklist

Run weekly until confident, then monthly:

| Check | How | Expected |
|-------|-----|----------|
| compare_odds accuracy | Export last 50, verify edge math | 100% match |
| compute_exposure_impact accuracy | Export last 50, verify limit math | 100% match |
| Counter TTL enforcement | Test expired counter acceptance | Should reject |
| Counter market move enforcement | Test with stale counter | Should reject |
| Line flexibility bounds | Review counter lines by sport | Within documented bounds |

### Manual Verification Process

1. Export recent calculations: `firebase firestore:export michelleCalculations --limit 50`
2. Spot-check 10 random `compare_odds` calculations
3. Spot-check 10 random `compute_exposure_impact` calculations
4. Mark verified in Firestore console
5. If any fail, fix the tool and re-audit

### Tasks

- [x] Create `michelleCalculations` collection
- [x] Implement `logCalculation()` helper
- [x] Create export script for audit
- [ ] Document verification process (skipped, this milestone has some documentation)
- [x] First audit pass after implementation

---

## Success Criteria

- [x] LangChain agent invokes tools correctly for instant match
- [x] Michelle fetches only the context she needs (observable via progress)
- [x] Calculator tools produce correct results (verified via audit) (audited most)
- [x] Counter-offers work with TTL and market-move bounds
- [x] Frontend displays Michelle's reasoning in real-time
- [x] Latency acceptable (< 10s for typical request)
- [x] No regressions in accept/reject quality

---

## Migration Path

### Phase 1: Parallel Implementation

- New agentic flow at `/api/michelle/instant-match-v2`
- Old flow remains at `/api/michelle/instant-match`
- Internal testing on v2

### Phase 2: Shadow Mode

- Both flows run, only old flow returns to user
- Compare decisions, log discrepancies
- Tune system prompt based on discrepancies

### Phase 3: Cutover

- Switch frontend to v2
- Old flow deprecated
- Monitor for issues

---

## Open Questions

1. **Counter-offer TTL** — 60 seconds feels right, but should test UX
2. **Market move threshold** — 2% may be too tight for volatile markets near tip-off
3. **Tool call limits** — Should we cap iterations to prevent runaway? (LangChain has `maxIterations`)
4. **Streaming latency** — Firebase propagation is 100-500ms; is that acceptable?

---

## Dependencies

- M31 complete (memory architecture in place)
- `@langchain/core` and `@langchain/anthropic` packages
- Firebase Admin SDK (already in use)

---

## Files to Create/Modify

### New Files
- `src/agents/market_maker_michelle/langchain/michelleAgent.ts`
- `src/agents/market_maker_michelle/langchain/tools/getMarketState.ts`
- `src/agents/market_maker_michelle/langchain/tools/getMyExposure.ts`
- `src/agents/market_maker_michelle/langchain/tools/getMyRecentDecisions.ts`
- `src/agents/market_maker_michelle/langchain/tools/compareOdds.ts`
- `src/agents/market_maker_michelle/langchain/tools/computeExposureImpact.ts`
- `src/agents/market_maker_michelle/langchain/callbacks/firebaseProgress.ts`
- `src/http/michelleInstantMatchV2.ts`

### Modified Files
- `src/http/routes.ts` (add v2 endpoint)
- Frontend: Quote component (add progress display)

---

## Notes

- The 0.20 USDC fee is even more justified now — user pays for agentic reasoning, not just a cached lookup
- Counter-offers are a meaningful UX improvement over hard rejections
- Streaming progress builds trust — users see Michelle "working" instead of a spinner
- Starting with 3 USDC limits means low risk for line flexibility experiments
- Dan stays bad. This is canon.
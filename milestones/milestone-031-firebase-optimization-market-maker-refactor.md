# Milestone 31: Firebase Optimization & Michelle Memory Refactor

*Created: January 13, 2026*
*Completed: January 14, 2026*
*Status: ✅ Complete*

---

## Overview

Address three related issues discovered during M30 testing:

1. **Read inefficiency** – 196K Firestore reads/day with a single user is unsustainable at scale
2. **Memory architecture** – Michelle's memory stored as single nested document (unqueryable, approaching size limits)
3. **Stale evaluations** – Michelle rejects instant match requests because her eval is hours old, but doesn't re-evaluate

---

## Goals

- Identify and fix the source of excessive Firestore reads
- Establish monthly monitoring routine for Firebase costs/usage
- Migrate Michelle's memory to queryable subcollection pattern
- Fresh evaluation on every instant match request
- Michelle's decisions informed by her recent history on that speculation

---

## Track 0: Firebase Read Optimization

**Problem:** 196K reads/day for a single user. At scale, this pattern would cost ~$36K/month for 10K users.

**Target:** <5K reads/day for a single active user.

### Investigation Areas

1. **Frontend listeners** – Check for collection-level `onSnapshot` listeners that should be document-level or filtered queries
2. **Agent server batch jobs** – Michelle's evaluation cycles may be reading entire collections
3. **Redundant queries** – Components re-fetching data that's already available
4. **Missing pagination** – Loading full collections when only recent items needed

### Tasks

- [ ] Audit frontend for `onSnapshot` and `getDocs` calls on large collections (ai)
- [ ] Audit agent server for broad collection reads during batch eval (ai)
- [ ] Identify top 3-5 query patterns causing most reads (ai)
- [x] Implement fixes (filtered queries, pagination, caching where appropriate) (ai)
- [x] Verify reads drop to target range (pending)

### Monitoring Checklist (Monthly)

| Metric | Where to Check | Target |
|--------|----------------|--------|
| Daily reads | Firestore Usage tab | <5K per active user |
| Daily writes | Firestore Usage tab | <5K per active user |
| Storage size | Metrics Explorer → `data_and_index_storage_bytes` | <500 MB |
| Total spend | Billing → Reports | <$50/month |
| Function invocations | Cloud Functions metrics | Correlate with reads |

*Billing alert already configured at $50/month with 50%, 90%, 100% thresholds.*

---

## Track 1: Memory Architecture Migration

**Problem:** Michelle's memory is a single document with a nested array of thousands of entries. Can't view in Firebase console, can't query individual decisions, can't prune without rewriting entire document, approaching 1MB Firestore limit.

### Current Structure

```
agentMemory/
  market_maker_michelle__amoy/
    {
      decisions: [
        { ts, speculationId, outcome, reason, ... },
        { ts, speculationId, outcome, reason, ... },
        ... thousands more ...
      ]
    }
```

### Target Structure

```
agentMemory/
  market_maker_michelle__amoy/
    decisions/  (subcollection)
      {decisionId1}/
        { ts, speculationId, contestId, ... }
      {decisionId2}/
        { ts, speculationId, contestId, ... }
```

### Decision Document Schema

```typescript
interface MichelleDecision {
  // Identity
  id: string;                    // auto-generated
  ts: Timestamp;                 // when decision was made
  agentId: string;               // 'market_maker_michelle__amoy'
  
  // Speculation context
  speculationId: string;
  contestId: string;
  jsonoddsId: string;
  market: 'moneyline' | 'spread' | 'total';
  side: 'away' | 'home' | 'over' | 'under';
  theNumber?: number;            // for spread/total
  
  // Request details
  requestType: 'instant_match' | 'batch_eval';
  requestedOdds?: number;        // for instant_match
  requestedAmount?: number;      // for instant_match
  requesterAddress?: string;     // for instant_match
  
  // Market context at decision time
  marketOdds: number;            // from jsonodds/data source
  evalOdds: number;              // what Michelle decided to offer
  marketMovePct?: number;        // change since last eval
  timeToStart: number;           // minutes until game
  
  // Michelle's exposure (at decision time)
  currentExposure?: {
    speculationExposure: number;
    contestExposure: number;
    totalExposure: number;
  };
  
  // Outcome
  outcome: 'offer_updated' | 'matched' | 'rejected' | 'error';
  reason?: string;               // Michelle's explanation
  
  // For matched outcomes
  matchedAmount?: number;
  positionId?: string;
}
```

### Tasks

- [ ] Create new subcollection write path (altered this to a flatter schema)
- [x] Update Michelle's decision logging to use new schema
- [x] Create query helper: `getRecentDecisions(speculationId, limit)`
- [x] Verify old nested format no longer written to
- [x] Do NOT migrate old data (let it age out)
- [ ] Optional: Add pruning job (delete decisions older than 90 days) (skipped, for now)

### Files to Touch

- `ospex-agent-server/src/agents/market_maker_michelle/` (memory writes)
- Possibly new `src/utils/agentMemory.ts` for shared helpers

---

## Track 2: Fresh Eval on Instant Match

**Problem:** User sees "Agent 1.67 (5h ago)" in UI, requests match at 1.67, Michelle rejects because market has moved to 1.91 and her eval is stale. User is confused – they requested exactly what the UI showed.

**Solution:** On every instant match request, Michelle does a fresh evaluation. Her memory informs but doesn't constrain the new eval.

### Current Flow

1. User clicks "Request Match"
2. Preflight check
3. User pays 0.20 USDC fee
4. Michelle checks her stored eval (possibly hours old)
5. Michelle rejects with "stale" message
6. User confused, eval unchanged

### Target Flow

1. User clicks "Request Match"
2. Preflight check
3. User pays 0.20 USDC fee
4. **Gather fresh context:**
   - Current market odds
   - Time until game start
   - Michelle's current exposure on this speculation/contest
   - Michelle's recent decisions on this speculation (from subcollection)
5. **Michelle does fresh eval** – stateless, all context in prompt
6. **Michelle decides** on the specific request (accept/reject)
7. **Persist decision** to subcollection
8. **Update `agentOffers`** with fresh eval odds + timestamp
9. **Return response** to user with explanation
10. **UI updates** to show fresh offer

### Context Passed to Michelle

```typescript
interface MichelleEvalContext {
  // The request
  request: {
    speculationId: string;
    requestedOdds: number;
    requestedAmount: number;
    requesterAddress: string;
  };
  
  // Current market state
  market: {
    currentOdds: number;         // from jsonodds
    timeToStartMinutes: number;
    contestInfo: string;         // "Houston Texans @ Pittsburgh Steelers"
    marketType: string;          // "moneyline" | "spread" | "total"
    side: string;                // "away" | "home" | "over" | "under"
  };
  
  // Michelle's current position
  exposure: {
    thisSpeculation: number;     // USDC exposed on this specific speculation
    thisContest: number;         // USDC exposed across all markets on this contest
    total: number;               // Total portfolio exposure
  };
  
  // Recent history on this speculation
  recentDecisions: Array<{
    ts: Timestamp;
    evalOdds: number;
    marketOdds: number;
    outcome: string;
    reason?: string;
  }>;  // last 5-10 decisions, from subcollection
}
```

### Michelle's Prompt Structure

```
You are Michelle, a market maker providing liquidity on Ospex.

Your goal: Provide competitive odds without taking on excessive risk.
You are NOT trying to profit. You are trying to not get wrecked while enabling trading.

CURRENT REQUEST:
- User wants to match at {requestedOdds} for {requestedAmount} USDC
- Market type: {marketType} on {side}

MARKET STATE:
- Current market odds: {currentOdds}
- Time until game: {timeToStartMinutes} minutes
- Contest: {contestInfo}

YOUR EXPOSURE:
- This speculation: {thisSpeculation} USDC
- This contest: {thisContest} USDC
- Total: {total} USDC

YOUR RECENT HISTORY ON THIS SPECULATION:
{recentDecisions formatted}

INSTRUCTIONS:
1. Determine what odds YOU would offer right now (your fresh eval)
2. Decide if the user's requested odds ({requestedOdds}) are acceptable
3. If rejecting, explain why briefly

Respond with JSON:
{
  "freshEvalOdds": <your current offer>,
  "decision": "accept" | "reject",
  "reason": "<brief explanation>",
  "matchAmount": <if accepting, how much to match, up to requestedAmount>
}
```

### Tasks

- [ ] Create `gatherMichelleContext()` function (skipped)
- [x] Query for recent decisions on this speculation (function exists for this)
- [x] Build prompt with full context
- [x] Parse Michelle's response
- [x] Update `agentOffers` with fresh eval
- [ ] Persist decision to subcollection (testing)
- [ ] Return response to calling endpoint (untested)
- [ ] Handle edge cases (no market data, game already started, etc.) (untested)

### Files to Touch

- `ospex-agent-server/src/http/michelleInstantMatch.ts`
- `ospex-agent-server/src/agents/market_maker_michelle/`
- New or updated eval function

---

## Success Criteria

- [ ] Firestore reads reduced to <5K/day for single user (monitoring)
- [x] Monthly monitoring checklist documented and actionable
- [x] Michelle's decisions stored in queryable format (flat)
- [x] Old nested memory format deprecated (no new writes)
- [ ] Instant match triggers fresh eval, not stale lookup (testing)
- [ ] UI reflects fresh offer after instant match request (testing)
- [ ] Michelle's response includes her reasoning (testing)

---

## Out of Scope (Future)

- Counter-offers ("I'd reject 1.97 but accept 1.94") – M32+
- Outcome tracking (did Michelle's decision win/lose?) – M32+
- Multi-agent coordination – Future
- User-created agents – Future
- LangChain integration – When needed for module system
- Extensible context modules (lineup, weather, sentiment) – Future

---

## Dependencies

- M30 complete (Order Book UI working)
- Track B flow functional

---

## Sequencing

1. **Track 0 first** – Fix reads before building new features that might add more
2. **Track 1 second** – Memory architecture in place before fresh eval writes to it
3. **Track 2 third** – Fresh eval logic, now writing to proper structure

---

## Notes

- The 0.20 USDC instant match fee becomes more meaningful with fresh evals – user is paying for a guaranteed-current evaluation, not a stale cached response
- Historical nested memory blob can be ignored/archived – not worth migrating
- Michelle as "stateless evaluator with context injection" is the right pattern and scales to future agents

---

## Open Questions

1. **Fresh eval latency** – How long does a single-speculation eval take? Target <3 seconds for UX.
2. **Rate limiting** – Should we prevent spam re-requests? (Probably not needed if user pays 0.20 USDC each time)
3. **Pruning retention** – 90 days reasonable for decision history? Or shorter?
# M25: Market Maker Michelle - Agent Broadcasts Willingness

*Created: December 3, 2025*
*Completed: December 10, 2025*  
*Status: ✅ Complete*

## Overview

This milestone introduces a new agent paradigm: instead of picking favorite games to bet on, the agent evaluates ALL upcoming games and broadcasts acceptable odds ranges for each. This creates the foundation for a two-sided marketplace where users can see agent willingness upfront and (in future milestones) request matches.

**Core insight**: Agents become always-on counterparties. They've already told you what they want. The user's job is to find where their desired odds align with an agent's acceptable range.

## Goals

1. Create new agent "Market Maker Michelle" with market-making personality
2. Expand prompt to evaluate full game slate (not just top picks)
3. Store structured offers in Firebase for UI consumption
4. Display agent willingness on the Agents page (read-only for now)

## Agent Config: Market Maker Michelle

```json
{
  "agentId": "market_maker_michelle",
  "name": "Market Maker Michelle",
  "personality": "Calculated market maker who evaluates every game systematically. Not emotional about picks - just looking for edge based on the numbers. Willing to take either side if the price is right. Prefers liquid markets and avoids extreme positions.",
  "wallet": {
    "address": "TO_BE_GENERATED",
    "privateKey": "${PRIVATE_KEY_MARKET_MAKER_MICHELLE}"
  },
  "strategy": {
    "riskLevel": "moderate",
    "minOdds": 1.80,
    "maxOdds": 2.30,
    "maxBetSize": 50,
    "minBetSize": 10,
    "preferredMarkets": ["moneyline", "spread", "total"],
    "maxExposurePerGame": 100,
    "maxExposurePerSide": 50,
    "edgeThresholdPercent": 3,
    "oddsImprovement": 3
  },
  "evaluation": {
    "evaluateAllGames": true,
    "marketsPerGame": ["moneyline", "spread", "total"],
    "noInterestThreshold": 2.5,
    "maxOddsDeviation": 8
  },
  "leaderboards": [
    {
      "id": "9",
      "bankroll": 500,
      "entryFee": 10
    }
  ]
}
```

### Key differences from Degen Dan:

| Aspect | Degen Dan | Market Maker Michelle |
|--------|-----------|----------------------|
| Selection | Top 3 favorites | All games |
| Markets | ML + Spread | ML + Spread + Total |
| Output | Picks with conviction | Odds ranges per side |
| Personality | Aggressive, trendy | Systematic, neutral |

---

## Prompt Structure

The prompt needs to output structured data for EVERY game, not just selected picks.

### System Prompt

```
You are Market Maker Michelle, an AI sports betting market maker for Ospex.

Your job is to evaluate EVERY game on the slate and determine acceptable odds ranges for each market. You are not picking winners - you are setting prices at which you'd be willing to take either side.

Core principles:
1. You must evaluate ALL games provided, not just ones you like
2. For each market (moneyline, spread, total), output your acceptable odds for BOTH sides
3. Your odds should be based on the market reference but with your edge requirement
4. If you have no interest in a market, output "no_interest" with a reason
5. Your combined odds on both sides should never create arbitrage (sum of implied probabilities < 100%)

Output format is strict JSON - no commentary outside the JSON structure.
```

### User Prompt Template

```
Current time: {timestamp}
Your bankroll: {bankroll} USDC
Your current exposure: {currentExposure} USDC

Here are today's games with market odds:

{games}

For each game, evaluate all three markets (moneyline, spread, total) and provide your acceptable odds ranges.

Remember:
- Your odds must be better than market for the user (that's why they use Ospex)
- But not so good that you're giving away edge
- Edge threshold: {edgeThresholdPercent}%
- Max odds deviation from market: {maxOddsDeviation}%

Output your evaluation as a JSON array.
```

### Expected Output Format

```json
{
  "evaluatedAt": "2025-12-02T21:00:00.000Z",
  "agentId": "market_maker_michelle",
  "games": [
    {
      "contestId": "abc123",
      "jsonoddsId": "xyz789",
      "awayTeam": "Los Angeles Lakers",
      "homeTeam": "Boston Celtics",
      "matchTime": "2025-12-03T00:00:00.000Z",
      "markets": {
        "moneyline": {
          "status": "active",
          "away": {
            "marketOdds": 2.10,
            "myOdds": 2.15,
            "maxExposure": 50,
            "confidence": 0.6
          },
          "home": {
            "marketOdds": 1.80,
            "myOdds": 1.85,
            "maxExposure": 50,
            "confidence": 0.6
          }
        },
        "spread": {
          "status": "active",
          "line": -4.5,
          "away": {
            "marketOdds": 1.91,
            "myOdds": 1.95,
            "maxExposure": 50,
            "confidence": 0.7
          },
          "home": {
            "marketOdds": 1.91,
            "myOdds": 1.95,
            "maxExposure": 50,
            "confidence": 0.7
          }
        },
        "total": {
          "status": "active",
          "line": 220.5,
          "over": {
            "marketOdds": 1.91,
            "myOdds": 1.95,
            "maxExposure": 40,
            "confidence": 0.5
          },
          "under": {
            "marketOdds": 1.91,
            "myOdds": 1.95,
            "maxExposure": 40,
            "confidence": 0.5
          }
        }
      },
      "notes": "High-profile matchup, expect sharp action. Comfortable on both sides at these prices."
    },
    {
      "contestId": "def456",
      "jsonoddsId": "uvw012",
      "awayTeam": "Charlotte Hornets",
      "homeTeam": "Detroit Pistons",
      "matchTime": "2025-12-03T00:00:00.000Z",
      "markets": {
        "moneyline": {
          "status": "no_interest",
          "reason": "Both teams too unpredictable, spreads too wide"
        },
        "spread": {
          "status": "active",
          "line": 2.5,
          "away": {
            "marketOdds": 1.91,
            "myOdds": 1.98,
            "maxExposure": 30,
            "confidence": 0.4
          },
          "home": {
            "marketOdds": 1.91,
            "myOdds": 1.98,
            "maxExposure": 30,
            "confidence": 0.4
          }
        },
        "total": {
          "status": "no_interest",
          "reason": "Total too volatile for this matchup"
        }
      },
      "notes": "Low-interest game, only comfortable with spread at wide prices."
    }
  ]
}
```

---

## Firebase Schema

### Collection: `agentOffers`

Document ID: `{agentId}_{contestJsonoddsId}_{market}`

Example: `market_maker_michelle_abc123_moneyline`

```javascript
{
  // Identifiers
  agentId: "market_maker_michelle",
  agentName: "Market Maker Michelle",
  agentWallet: "0x...",
  contestId: "abc123",           // on-chain contest ID (if exists)
  jsonoddsId: "xyz789",          // source ID for matching
  
  // Game info (denormalized for query efficiency)
  awayTeam: "Los Angeles Lakers",
  homeTeam: "Boston Celtics",
  matchTime: Timestamp,
  sport: "NBA",
  sportId: 4,
  
  // Market info
  market: "moneyline",           // "moneyline" | "spread" | "total"
  line: null,                    // null for ML, number for spread/total
  
  // Agent's offers
  status: "active",              // "active" | "no_interest" | "expired" | "matched"
  
  // For active offers:
  awayOffer: {                   // or "overOffer" for totals
    marketOdds: 2.10,
    agentOdds: 2.15,
    maxExposure: 50,
    currentExposure: 0,
    confidence: 0.6
  },
  homeOffer: {                   // or "underOffer" for totals
    marketOdds: 1.80,
    agentOdds: 1.85,
    maxExposure: 50,
    currentExposure: 0,
    confidence: 0.6
  },
  
  // For no_interest:
  noInterestReason: null,        // or "Both teams too unpredictable"
  
  // Metadata
  evaluatedAt: Timestamp,
  evaluatedLine: 0,              // this will be the number for spreads and totals
  expiresAt: Timestamp,          // matchTime
  notes: "High-profile matchup...",
  
  // Tracking
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

### Indexes needed:

```
agentOffers:
  - status ASC, matchTime ASC (get active offers, sorted by game time)
  - jsonoddsId ASC, market ASC (get all offers for a specific game)
  - agentId ASC, status ASC (get all active offers for an agent)
```

---

## Line Movement Handling

Michelle's offers are pinned to the line at evaluation time:

- Offer stores: `evaluatedLine: -3.5`
- If market moves to -5.5, offer shows as "stale" in UI
- User can still match at the evaluated line if they want it
- Re-evaluation triggers if line moves > 3 points (configurable)
- Future enhancement: line-agnostic offers with acceptable ranges

## Implementation Tasks

### Phase 1: Agent Setup (Days 1-2)

- [X] Create `/agents/market_maker_michelle/` directory
- [X] Create `config.json` per spec above
- [X] Create `memory.json` (empty initial state)
- [X] Generate new wallet, fund with MATIC + USDC on Amoy
- [X] Add `PRIVATE_KEY_MARKET_MAKER_MICHELLE` to environment
- [ ] Test basic agent boot with existing scheduler (Skipped)

### Phase 2: Prompt & Evaluation Logic (Days 3-5)

- [X] Create the following:
  - New prompt template for full-slate evaluation
  - JSON parsing with validation
  - Fallback handling for malformed responses
- [X] Implement the following:
  - Load all upcoming contests (not just 24h - maybe 48h?)
  - Build game context with market odds
  - Call new prompt function
  - Parse and validate response
- [X] Add structured output validation
  - Ensure odds are within acceptable ranges
  - Ensure no arbitrage (away + home implied prob < 100%)
  - Flag anomalies for logging
- [X] Agent outputs human-readable lines (e.g., -3.5, 230.5)
- [X] Storage layer converts to theNumber using existing line-conversion.ts
- [X] LLM never sees or outputs theNumber directly

### Phase 3: Firebase Storage (Days 5-6)

- [x] Create `agentOffers` collection
- [x] Write `saveAgentOffers(agentId, offers[])` function
  - Upsert logic (update if exists, create if not)
  - Handle status transitions (active → matched, etc.)
- [x] Write `getAgentOffersForGame(jsonoddsId)` function
- [x] Write `getActiveOffersByAgent(agentId)` function
- [x] Add cleanup job: expire offers after matchTime
- [x] Track `currentExposure` per side in offer documents
- [x] Add `maxExposurePerSide` to config (default: half of maxExposurePerGame)
- [x] On match: increment currentExposure, check against max before allowing

### Phase 4: Scheduler Integration (Day 7)

- [ ] Add Market Maker Michelle to scheduler → defer
- [ ] Run evaluation on same cadence as Dan (08:00/20:00) or more frequently → defer
- [ ] Ensure offers are refreshed as odds change → defer to v2
- [ ] Add re-evaluation logic when market odds drift > threshold → defer to v2

### Phase 5: UI Display (Days 8-10)

- [X] Agents page redesign:
  - Search/filter by game
  - Show game cards with agent willingness
  - Display: "Michelle offers Lakers ML @ 2.15 (market: 2.10)"
- [X] Color coding:
  - Green: agent offering better than market
  - Yellow: close to market
  - Gray: no interest
- [X] Click to expand: show all markets for a game
- [ ] Mobile responsive (Partial - will need specific mobile layout and modifications)

### Phase 6: Testing & Polish (Days 11-14)

- [X] Run Michelle for 2-3 days, monitor outputs (manually completed)
- [X] Validate offers make sense vs. market
- [X] Check for any arbitrage situations
- [X] Tune prompts based on output quality
- [X] Load test: how long does full-slate evaluation take? (around 2 minutes)

---

## Success Criteria

1. **Michelle evaluates all games**: Every upcoming contest has an offer (or explicit no_interest) in Firebase
2. **Offers are sensible**: Agent odds are 2-8% better than market, no arb
3. **UI displays offers**: Users can see what Michelle would accept
4. **System is stable**: Runs on cron without errors for 48+ hours
5. **Token costs are reasonable**: Full slate evaluation < $0.50/run

---

## Token Cost Estimate

Assuming ~30 games/day across NBA, NHL, NCAAB, NCAAF:

**Input tokens per run:**
- System prompt: ~500 tokens
- Game context (30 games × ~200 tokens): ~6,000 tokens
- Agent state: ~300 tokens
- Total input: ~7,000 tokens

**Output tokens per run:**
- 30 games × ~150 tokens: ~4,500 tokens

**Cost per run (Claude Sonnet 4):**
- Input: 7,000 × $0.003/1K = $0.021
- Output: 4,500 × $0.015/1K = $0.068
- **Total: ~$0.09 per evaluation run**

**Daily cost (2 runs/day):** ~$0.18
**Monthly cost:** ~$5.40

This is very manageable. Even at 4 runs/day, you're under $15/month.

---

## Future Considerations (Not This Milestone)

- **On-demand re-evaluation**: User pays small fee to request fresh agent opinion
- **Agent-to-agent awareness**: Prevent arbitrage when multiple agents are live
- **Third-party agents**: Allow external agents to register offers
- **Match execution**: Actually match user orders against agent offers

---

## Notes

- Keep Degen Dan running separately - don't disrupt what works
- Michelle is the "workhorse" market maker; Dan is the "personality" bettor
- Start with one agent to prove the system before adding more
- The no-vig model means our math is cleaner - 2.00/2.00 is break-even, not arb
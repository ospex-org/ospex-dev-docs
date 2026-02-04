# Agent Architecture Audit

**Generated:** 2026-02-03
**Scope:** Michelle (Market Maker) and Dan (Degen Bettor) agent flows
**Status:** AUDIT ONLY - No code changes

---

## Summary Table: All Agent Flows

| Flow | Trigger | Entry Point | LLM Integration | Tools Available | Data to LLM | Decision Logic | Firebase Writes |
|------|---------|-------------|-----------------|-----------------|-------------|----------------|-----------------|
| **Michelle Instant Match** | User HTTP request | `src/http/michelleInstantMatch.ts` | LangChain agent | 9 tools | Pre-computed context + tool fetches | LLM decides (with edge guidelines) | `agentQuotes`, `positionEvaluations` |
| **Michelle Matching Job** | Cron (`*/15 * * * *`) | `src/agents/market_maker_michelle/matching-job.ts` | Raw Anthropic SDK | None | Candidates + offers + exposure | LLM decides which to match | `positionEvaluations` |
| **Michelle Offer Creation** | Cron (0 0,8,16 UTC) | `src/agents/market_maker_michelle/evaluator.ts` | LangChain agent | 9 tools | Game data + pre-computed context | LLM sets ask/ceiling odds | `agentOffers` |
| **Dan Evaluation** | Cron (`1 8,20 * * *`) | `src/agent.ts` | Raw Anthropic SDK | None | Markets JSON (no injuries) | LLM picks, conviction filters | `memory.json`, `agent_interests` |
| **Dan Matching** | Cron (`*/5 * * * *`) | `src/match.ts` | None | N/A | N/A | Code logic (find best maker) | Position writes via contract |

---

## Flow 1: Michelle Instant Match (HTTP API)

### 1. Trigger
User requests an instant match quote via the frontend. HTTP POST to `/api/michelle/quote`.

### 2. Entry Point
**File:** `src/http/michelleInstantMatch.ts`
**Function:** `handleMichelleQuote()` (Express route handler)

### 3. LLM Integration
**Type:** LangChain Agent
**Implementation:** `src/agents/market_maker_michelle/langchain/michelleAgent.ts`
**Function:** `invokeMichelleAgent(task, { onProgress })`

The LangChain agent is created via `createMichelleAgent()` which:
1. Builds a system prompt with personality + limits
2. Creates `ChatAnthropic` LLM wrapper
3. Registers 9 tools
4. Returns an agent that loops: LLM → tool calls → tool results → LLM output

### 4. Tools Available
From `src/tools/index.ts`:

| Tool | Purpose | Data Source |
|------|---------|-------------|
| `getMarketStateTool` | Fetch current odds | Firebase contests |
| `getMyExposureTool` | Query current exposure | Firebase agentOffers |
| `getMyRecentDecisionsTool` | Recent quote activity | Firebase agentQuotes |
| `compareOddsTool` | Calculate edge % | Pure calculation |
| `computeExposureImpactTool` | Check exposure limits | Pure calculation |
| `getInjuryReportTool` | Team injury summary | Supabase |
| `getInjuryDetailsTool` | Player injury details | Supabase |
| `getRosterTool` | Team roster | Supabase |
| `getStandingsTool` | Team standings | Supabase |

### 5. Data Injected as Context (Not via Tools)
Pre-computed context passed in `task.preComputedContext`:
- `currentMarketOdds` - Current odds from Firebase
- `awayTeam`, `homeTeam` - Team names
- `awayTeamAbbrev`, `homeTeamAbbrev` - Team abbreviations
- `timeToStartMinutes` - Minutes until game starts
- `currentExposure` - Michelle's exposure on this speculation
- `maxMatchAmount` - Maximum she can match given limits

**Line:** `src/http/michelleInstantMatch.ts:139-148`

### 6. Data Loaded but NOT Used
**`loadSportsIntelCache()`** is called at line 1032 but ONLY used to get team abbreviations, NOT passed to the LLM:
```typescript
const sportsIntelCache = await loadSportsIntelCache();
awayTeamAbbrev = getTeamAbbreviation(sportsIntelCache, awayTeam);
homeTeamAbbrev = getTeamAbbreviation(sportsIntelCache, homeTeam);
```

The injury summaries in the cache are NOT passed to the agent. The agent must call `getInjuryReportTool` explicitly to get injury data.

### 7. Decision Logic
**Hybrid: LLM decides with edge guidelines**

System prompt (`michelleAgent.ts:132-200`) gives Michelle edge thresholds as *guidelines*, not hard rules:
- 0-3% user edge: Typically ACCEPT
- 3-8% user edge: Typically COUNTER
- >8% user edge: Typically REJECT

She can deviate when justified by:
- Significant injury news
- Exposure situation (balancing sides)
- Time to game start
- Specific game factors

Hard constraints (enforced by tools):
- `maxPerSide` - Maximum exposure per side
- `maxPerGame` - Maximum exposure per game

### 8. What Gets Logged to Firebase

**Collection: `agentQuotes`**
- Document ID: `{quoteId}`
- Fields:
  - `status`: 'processing' | 'complete' | 'error'
  - `startedAt`, `completedAt`: ISO timestamps
  - `progress`: Array of `ProgressStep` objects (step, message, ts, method, details)
  - `decision`: { type, reason, counterOdds }

**Collection: `positionEvaluations`**
- Document ID: Auto-generated
- Fields:
  - `agentId`, `positionId`, `speculationId`, `jsonoddsId`, `market`
  - `decision`: 'skip' | 'accept' | 'counter'
  - `phase`: 'enrich' | 'evaluation' | 'execution'
  - `reasonCode`: e.g., 'SPEC_MISSING', 'SCORER_UNKNOWN'
  - `reason`: Human-readable
  - `timestamp`, 90-day TTL

**Collection: `usedFeeTxHashes`**
- Prevents replay of fee transactions

### 9. What the Frontend Displays
**Component:** `MichelleEvaluationLog.tsx`
**Data Source:** Real-time Firebase subscription to `agentQuotes`

Progress steps shown:
- `started` - "Michelle is reviewing your request..."
- `checking_market` - "Checking current market odds..." (agent_tool) or "Providing market context..." (pre_computed)
- `reviewing_injuries` - "Reviewing injury reports..."
- `reviewing_injury_details` - "Checking player injury details..."
- `reviewing_roster` - "Reviewing team roster..."
- `reviewing_exposure` - "Reviewing my current positions..."
- `checking_history` - "Checking recent activity on this game..."
- `calculating_edge` - "Calculating the edge..."
- `complete` - "Evaluation complete"

Each step includes:
- Elapsed time since start
- Method indicator (pre_computed vs agent_tool)
- Expandable details (e.g., actual odds, injury names)

### 10. Gaps/Lies
**Gap 1:** Progress steps are only logged when the LLM *calls a tool*. If Michelle decides without calling a tool, no progress step is shown for that data.

**Gap 2:** The `method` field distinguishes `pre_computed` vs `agent_tool`, but pre_computed context (market odds, exposure, max amount) is NOT logged as progress steps. Only tool calls are logged.

**Gap 3:** If Michelle makes a quick decision without calling injury tools, the frontend shows no injury analysis even though she *could have* checked them.

---

## Flow 2: Michelle Matching Job (Scheduled)

### 1. Trigger
Cron job: `*/15 * * * *` (every 15 minutes)

### 2. Entry Point
**File:** `src/agents/market_maker_michelle/matching-job.ts`
**Function:** `runMichelleMatchingOnce()`

### 3. LLM Integration
**Type:** Raw Anthropic SDK (NOT LangChain)
**Implementation:** Direct `client.messages.create()` call

```typescript
// matching-job.ts (inside function, around line 650+)
const msg = await client.messages.create({
  model: config.anthropicModel ?? 'claude-sonnet-4-5-20250929',
  max_tokens: 8000,
  temperature: 0,
  messages,
});
```

### 4. Tools Available
**None.** This flow uses a simple prompt → response pattern, not an agentic tool-calling flow.

### 5. Data Injected as Context (Not via Tools)
The prompt receives a fully pre-assembled context:
- All enriched candidates (position ID, speculation, market, maker, odds, amount)
- Michelle's existing offers (`agentOffers` docs) with ask/ceiling odds
- Current exposure by speculation and by (contest, market)
- Directional exposure limits
- Contest start times

Built by `buildMatchingPrompt()` in `src/agents/market_maker_michelle/matching-prompt.ts`

### 6. Data Loaded but NOT Used
**Injuries/standings are NOT loaded** for the matching job. The matching job only evaluates positions against Michelle's existing offers - it doesn't re-evaluate games.

### 7. Decision Logic
**LLM decides which candidates to match**

The LLM receives:
1. List of candidates (unmatched positions)
2. Michelle's existing offers with ask/ceiling for each market
3. Current exposure levels

It outputs which positions to match and for what amount, respecting:
- Michelle's ceiling odds (won't take worse than ceiling)
- Directional exposure limits
- Per-game exposure limits

### 8. What Gets Logged to Firebase

**Collection: `positionEvaluations`**

During the enrichment phase, positions are logged with reason codes:
- `SPEC_MISSING` - Speculation not found
- `SCORER_UNKNOWN` - Unknown market type
- `JSONODDS_MISSING` - No game ID
- `CONTEST_MISSING` - Game data not available
- `ALREADY_STARTED` - Game already started

After LLM decision:
- Matched positions logged with `decision: 'accept'`
- Passed positions logged with `decision: 'skip'` and reason

### 9. What the Frontend Displays
**Component:** `PositionEvaluationHistory.tsx` and `PositionEvaluationHistoryCard.tsx`

Shows historical evaluations grouped by agent and reason:
- Agent name (Michelle)
- Phase badge (Data Validation, Pre-screening, AI Analysis, Execution)
- Decision badge (Matched/Passed/Skipped)
- Reason text
- Expandable LLM reasoning for AI Analysis phase

### 10. Gaps/Lies
**Gap 1:** The frontend shows "AI Analysis" phase, but the matching job uses a simple prompt, not an agentic flow with tools. There's no tool-by-tool breakdown like the instant match flow.

**Gap 2:** If the LLM provides reasoning, it's stored. But there's no real-time progress - users only see results after the job completes.

**Gap 3:** The matching job doesn't check injuries/standings. Positions might be matched even if significant injury news broke since Michelle's last offer evaluation.

---

## Flow 3: Michelle Offer Creation/Update

### 1. Trigger
Cron job: `0 0,8,16 * * *` (midnight, 8am, 4pm UTC)

### 2. Entry Point
**File:** `src/agents/market_maker_michelle/evaluator.ts`
**Function:** `evaluateSlate()` (called from scheduler)

Also called from: `src/agents/market_maker_michelle/reevaluationScheduler.ts` for smart re-evaluation

### 3. LLM Integration
**Type:** LangChain Agent
**Implementation:** `src/agents/market_maker_michelle/langchain/gameEvaluator.ts`
**Function:** `evaluateGame()`

### 4. Tools Available
Same 9 tools as instant match flow.

### 5. Data Injected as Context (Not via Tools)
Pre-computed context passed to the game evaluator:
- Team names, sport
- Current market odds snapshot
- Game start time

### 6. Data Loaded but NOT Used
None identified - the evaluator uses the full LangChain agent with tool access.

### 7. Decision Logic
**LLM decides ask/ceiling odds per side**

Michelle evaluates each game and outputs:
- `askOdds` - The odds she'll offer (what she quotes to users)
- `askCeiling` - The worst odds she'll accept (for matching)
- `confidence` - How confident she is in her evaluation
- `reasoning` - Why she set these prices

### 8. What Gets Logged to Firebase

**Collection: `agentOffers`**
- Document ID: `{agentId}_{jsonoddsId}_{market}`
- Fields:
  - `agentId`, `jsonoddsId`, `market`
  - Side offers: `away`/`home`/`over`/`under` with `askOdds`, `askCeiling`, `confidence`
  - `reasoning`
  - `evaluatedAt`, `createdAt`, `updatedAt`

### 9. What the Frontend Displays
The `agentOffers` collection is read by the frontend to display Michelle's current offers on games. This appears in the game accordion as available liquidity.

### 10. Gaps/Lies
**Gap 1:** Offers are re-evaluated on a schedule (or when market moves >3%). Between evaluations, offers may be stale relative to breaking news.

---

## Flow 4: Degen Dan Evaluation

### 1. Trigger
Cron job: `1 8,20 * * *` (8:01am and 8:01pm UTC)

### 2. Entry Point
**File:** `src/agent.ts`
**Function:** `runOnce(agentId = 'degen_dan')`

### 3. LLM Integration
**Type:** Raw Anthropic SDK (NOT LangChain)
**Implementation:** `src/anthropic.ts`
**Function:** `suggestBetsWithRationale(markets, agentId)`

```typescript
const msg = await client.messages.create({
  model: config.anthropicModel ?? 'claude-sonnet-4-5-20250929',
  max_tokens: 8000,
  temperature: 0,
  ...(systemPrompt ? { system: systemPrompt } : {}),
  messages,
});
```

### 4. Tools Available
**None.** Dan uses a simple prompt → response pattern.

### 5. Data Injected as Context (Not via Tools)
From `src/agents/degen_dan/prompt.ts`:

**What IS passed:**
```typescript
markets: [
  {
    contestId, jsonoddsID, sport,
    awayTeam, homeTeam,
    startTime,
    moneyline: { away, home },
    spread: { awayTeamSpread, homeTeamSpread, awayOdds, homeOdds },
    total: { number, overOdds, underOdds }
  }
]
```

**What is NOT passed:**
- Injuries (parameter exists but never provided - see Gap 1)
- Standings
- Rosters
- Historical performance

### 6. Data Loaded but NOT Used
**CRITICAL GAP: Injuries are NOT passed to Dan's LLM**

`suggestBetsWithRationale()` accepts a third parameter `sportsIntel`:
```typescript
export async function suggestBetsWithRationale(
  markets: unknown[],
  agentId: string = 'degen_dan',
  sportsIntel?: Map<string, { injurySummary: string; hasSignificantInjuries: boolean }>
)
```

But at `agent.ts:448`, it's called with only 2 arguments:
```typescript
const { decisions, rationale } = await suggestBetsWithRationale(marketsToAnalyze, agentId);
```

**The infrastructure exists to pass injuries, but it's not wired up.**

### 7. Decision Logic
**Hybrid: LLM suggests, code filters by conviction**

1. LLM returns picks with `convictionScore` (0-10)
2. Code filters:
   - `convictionScore >= 8` → High conviction → Create position immediately
   - `convictionScore 4-7` → Medium conviction → Store as interest only
   - `convictionScore < 4` → Ignored

3. Code enforces:
   - Max active bets cap from config
   - Deduplication (if both ML and spread on same team)
   - Line validation (.5 for spreads/totals)

### 8. What Gets Logged to Firebase

**Local file: `agents/degen_dan/memory.json`**
- Decision history with contestId, market, pick, conviction, action, status
- Used to prevent re-analysis of same games

**Collection: `agent_interests`**
- Medium-conviction picks stored for later matching
- Fields: contestId, market, pick, line, amount, targetOdds, speculationScorer

**File: `logs/llm-response-{agentId}-{timestamp}.json`**
- Full LLM response for debugging

### 9. What the Frontend Displays
Dan doesn't have a user-facing instant match flow. His positions appear like any other user position. No special "agent evaluation" UI exists for Dan.

### 10. Gaps/Lies
**CRITICAL Gap 1:** Dan has NO access to injury data. He's making picks blind to injury news. The code structure supports passing injuries, but it's not connected.

**Gap 2:** Dan has NO access to standings, rosters, or any sports intelligence. He only sees odds.

**Gap 3:** Dan's prompt says "Here are today's available games" but he has no context about WHY the lines are what they are.

---

## Flow 5: Dan Matching Job

### 1. Trigger
Cron job: `*/5 * * * *` (every 5 minutes)

### 2. Entry Point
**File:** `src/match.ts`
**Function:** `runMatching(agentId)`

### 3. LLM Integration
**None.** Pure code logic.

### 4. Tools Available
N/A - no LLM involved.

### 5. Data Injected as Context
N/A - no LLM involved.

### 6. Data Loaded but NOT Used
N/A - matching is deterministic.

### 7. Decision Logic
**Pure code logic:**
1. Load Dan's interests from `agent_interests`
2. For each interest, find unmatched positions on the opposite side
3. Select best maker (highest unmatched amount)
4. Execute `completeUnmatchedPair()` on-chain
5. Decrement interest's remaining amount

### 8. What Gets Logged to Firebase
Position writes happen via smart contract events → Firebase sync.

### 9. What the Frontend Displays
Matched positions appear in position lists. No special matching UI.

### 10. Gaps/Lies
None - matching is transparent code logic.

---

## Localhost vs Production Differences

### Environment Variables
Both use the same codebase. Key differences via environment:
- `NETWORK`: 'polygon' (mainnet) vs 'amoy' (testnet)
- `MICHELLE_WALLET_ADDRESS`: Different wallets per environment
- Private keys resolved from `${ENV_VAR}` pattern

### Feature Flags
- `MICHELLE_MATCH_DRY_RUN`: When 'true', matching job logs but doesn't execute on-chain
- `MICHELLE_MIN_MATCH_USDC`: Minimum match size (default 1)
- `MICHELLE_MATCH_SCAN_DOCS`: Max positions to scan (default 500)
- `MICHELLE_MATCH_MAX_CANDIDATES`: Max candidates to evaluate (default 50)

### No Code Path Differences
The code paths are identical. All differences are configuration-driven.

---

## LangChain vs Raw Anthropic SDK

| Flow | Integration | Reason |
|------|-------------|--------|
| Michelle Instant Match | LangChain | Agentic - needs tool calling for on-demand data |
| Michelle Offer Evaluation | LangChain | Agentic - needs tools for game-specific analysis |
| Michelle Matching Job | Raw SDK | Simple - all context pre-assembled, no tools needed |
| Dan Evaluation | Raw SDK | Simple - just needs picks from market data |
| Dan Matching | None | No LLM - deterministic code |

### Why the Split?
**LangChain agents** are used when:
- The agent needs to gather information dynamically
- Different requests may need different data
- Tool calling provides better reasoning traces

**Raw Anthropic SDK** is used when:
- All context can be pre-assembled
- Single prompt → response is sufficient
- Lower latency is preferred (no tool loop)

---

## Sports Data Tracing

### Where Sports Data is Loaded

| Data Type | Loader | Location |
|-----------|--------|----------|
| Injuries | `getInjuryReportTool` | Supabase `injuries` table |
| Injury Details | `getInjuryDetailsTool` | Supabase `injuries` table |
| Rosters | `getRosterTool` | Supabase `rosters` table |
| Standings | `getStandingsTool` | Supabase teams/standings |
| Team Abbreviations | `loadSportsIntelCache()` | In-memory cache from Supabase |

### What Actually Reaches the LLM

| Agent | Flow | Injuries | Standings | Rosters |
|-------|------|----------|-----------|---------|
| Michelle | Instant Match | **Yes** (via tool) | **Yes** (via tool) | **Yes** (via tool) |
| Michelle | Matching Job | **No** | **No** | **No** |
| Michelle | Offer Evaluation | **Yes** (via tool) | **Yes** (via tool) | **Yes** (via tool) |
| Dan | Evaluation | **NO - WIRED BUT NOT CONNECTED** | **No** | **No** |

### Critical Finding: Dan is Blind

The `suggestBetsWithRationale()` function signature supports `sportsIntel` but:
1. `loadSportsIntelCache()` is never called in `agent.ts`
2. The third parameter is never passed at call site
3. Dan makes picks based ONLY on odds, with no injury/standings context

---

## Firebase Collections Summary

| Collection | Writers | Purpose |
|------------|---------|---------|
| `agentOffers` | Michelle evaluator | Active market-making offers |
| `agentQuotes` | Michelle instant match | Quote requests + real-time progress |
| `positionEvaluations` | Michelle matching job, instant match | Audit trail of agent decisions |
| `agent_interests` | Dan evaluation | Medium-conviction picks awaiting matching |
| `usedFeeTxHashes` | Michelle instant match | Fee tx replay prevention |
| `agentMeta` | Scheduler | Last claim scan timestamps |

### Legacy/Deprecated Collections
- `agentDecisions` - Old pattern, unclear if still used
- `agentCalculations` - Tool audit logs, partially migrated to `positionEvaluations`

---

## Frontend Progress Steps vs Reality

### Michelle Instant Match

| Step Shown | What Actually Happens |
|------------|----------------------|
| "Reviewing injury reports" | LLM called `getInjuryReportTool` |
| "Checking market odds" | LLM called `getMarketStateTool` |
| "Calculating the edge" | LLM called `compareOddsTool` |
| "Reviewing my current positions" | LLM called `getMyExposureTool` |

**Gap:** If Michelle decides quickly without calling a tool, the frontend shows no step for that analysis. Pre-computed context (market odds, exposure) has steps logged only if method is 'pre_computed', but these are rarely shown.

### Michelle Matching Job

| Step Shown | What Actually Happens |
|------------|----------------------|
| "Data Validation" | Position enrichment - checking spec/contest exists |
| "Pre-screening" | Filtering by start time, exposure limits |
| "AI Analysis" | Single LLM call with all candidates (NOT tool-by-tool) |
| "Execution" | On-chain match |

**Gap:** "AI Analysis" implies tool-based reasoning, but matching uses a simple prompt.

### Position Evaluation History

The `PositionEvaluationHistory` component shows **all** evaluations (instant match + matching job) in a unified view. This is accurate - it's pulling from `positionEvaluations` which both flows write to.

---

## Key Findings Summary

### Gaps/Lies Identified

1. **Dan has no injury data** - Infrastructure exists but not wired
2. **Matching job doesn't use sports data** - Matches against stale offers
3. **Progress steps only show tool calls** - Pre-computed context not visible
4. **"AI Analysis" label misleading** - Matching job is simple prompt, not agentic

### Architecture Decisions

1. **LangChain for instant match** - Good: enables on-demand data gathering
2. **Raw SDK for batch jobs** - Good: simpler, faster for pre-assembled context
3. **Tool context pattern** - Good: clean separation of concerns
4. **Real-time Firebase progress** - Good: great UX for instant match

### Technical Debt

1. **Dan injury integration** - Should wire up `sportsIntel` parameter
2. **Matching job re-evaluation** - Should check for significant market moves
3. **Progress logging consistency** - Pre-computed context should also log steps
4. **Collection naming** - `agentCalculations` partially deprecated, needs cleanup

---

*End of Audit*

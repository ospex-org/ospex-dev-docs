# Milestone 39: LangGraph Migration — Stateful Agent Architecture

*Created: February 6, 2026*
*Status: 🔄 Track 4 In Progress*

**Edit Trail:**
- 2026-02-07: **CRITICAL FIX - Removed old LangChain code causing token burn**:
  - **Root Cause**: Old LangChain code (`evaluator.ts` → `runSmartReEvaluation()`) was running BEFORE the new LangGraph slate orchestrator. Michelle was in the `agents` array in `agents.json`, causing the scheduler to run her old evaluation code on boot.
  - **Files Deleted**: `evaluator.ts`, `reevaluationScheduler.ts`, `run-michelle.ts`, `run-michelle-fresh.ts`, `test-michelle.ts`
  - **Config Updated**: Removed Michelle from `agents` array (she remains in `marketMakerMatching.agents` only)
  - **Result**: Now only LangGraph code runs for Michelle - no more fallback/dual execution
- 2026-02-07: **Game Evaluation Persistence Service**:
  - Created `gameEvaluationService.ts` - persists LLM-generated predictions to Firebase
  - New collection: `agentGameEvaluations` with structure:
    - `blindPrediction`: Michelle's pre-market predictions (spread, total, moneyline with confidence/reasoning)
    - `marketPricing`: Post-market pricing (askOdds, ceilingOdds, edgeAssessment)
  - Wired into `integration.ts`: All 3 invoke functions now save evaluations to Firebase
  - Purpose: Frontend display of Michelle's thinking, benchmarking, analytics
- 2026-02-07: **Frontend Progress Streaming Implementation**:
  - Added node-level progress emission to `nodes.ts`:
    - `blind_prediction` - "Analyzing the game..." (at node start)
    - `market_pricing` - "Generating pricing..." (at node start)
    - `price_response` - "Making my decision..." (at node start)
    - `blind_prediction_complete` - "My independent assessment is ready" (with prediction details)
  - Added `get_odds_history` tool mapping to `progressCallback.ts`:
    - Maps to `reviewing_odds_history` step
    - Extracts line movement details (e.g., "Spread: Opened -3.5, now -5.5 (↓-2.0)")
  - Added `get_rankings` detail extraction for NCAAB AP/Coaches polls
  - Added new progress step definitions to `quoteProgressService.ts`
  - Added frontend formatters in `MichelleEvaluationLog.tsx`:
    - `reviewing_odds_history` - displays line movement
    - `blind_prediction_complete` - displays Michelle's prediction (spread/total/win%/reasoning)
  - **Remaining:** Frontend testing to verify progress displays correctly
- 2026-02-07: **Track 4 End-to-End Test PASSED** - Full verification:
  - Test run: 2 NBA games (WSH @ BKN, HOU @ OKC), both triaged as high interest, both dispatched
  - Blind prediction: Tools returned real data (injuries, standings, schedule, stats)
  - Blind prediction correctly identified OKC missing SGA + Jalen Williams
  - Market pricing: Michelle now has `get_odds_history` tool available and CHOSE to call it for both games
  - Odds history returned: spread moved -1.5 (game 1), total moved -2.5 (game 2)
  - Dispatch output: 2 games dispatched (not empty)
  - All expected behaviors verified in LangSmith traces
- 2026-02-07: **Enhanced `get_odds_history` tool**:
  - Added `lastN` parameter: get only the last N snapshots
  - Added `withinHours` parameter: get snapshots from last N hours only
  - Converted market pricing to agent with tool access (Michelle can now optionally dig deeper)
  - Updated prompt to inform Michelle the tool is available with example parameters
- 2026-02-07: **Track 4 End-to-End Test Prep** - Architecture verification:
  - Verified blind prediction node does NOT have access to `get_odds_history` tool (correct - "blind")
  - Confirmed `test-slate-orchestrator.ts` uses on-chain contests via `getUpcomingOnChainContests()`
  - Note: `test-slate-focused.ts` queries general contests (not on-chain) - use orchestrator script instead
- 2026-02-07: **Track 4 Continued** - Dan verification:
  - Verified Dan is unaffected by LangGraph migration (no LangGraph imports, uses standard agent flow)
- 2026-02-06: **Track 4 Started** - Configuration and integration testing:
  - Added `SUPABASE_DB_URL` environment variable
  - Fixed SSL configuration for Supabase (use `sslmode=no-verify`)
  - Renamed gameGraph nodes to avoid collision with state attributes (blindPrediction → do_blind_prediction, etc.)
  - Added LangSmith environment variables
  - Created integration test suite: `scripts/test-langgraph-integration.ts`
  - All 3 Track 1 integration tests now passing
- 2026-02-06: **Track 4 Core Scenarios Verified**:
  - Updated `querySlateGames()` to use `getUpcomingContests(24)` instead of `getContestsForToday()`
  - Triage test: 6 games triaged, correctly identified 4 for dispatch (NBA + Power 5), skipped 2 (low-tier)
  - Full game evaluation test: Rockets @ Thunder
    - cron_eval: 39.2s (blind prediction + market pricing)
    - instant_eval: 2.6s (cache hit! 93% faster)
  - Michelle's blind prediction identified OKC missing SGA + Jalen Williams → Houston should be favored
  - Feature flags enabled: `MICHELLE_USE_LANGGRAPH=true`, `MICHELLE_USE_LANGGRAPH_SLATE=true`
- 2026-02-06: **Added Mermaid Flow Diagrams**:
  - Complete System Flow: shows triggers → slate orchestrator → per-game graphs
  - Slate Orchestrator Detail: LLM triage filtering logic
  - Per-Game Graph Detail: router, blind prediction, market pricing, price response nodes
  - HTTP Instant Match Flow: sequence diagram from frontend to blockchain
  - Cache Hit vs Cache Miss Performance: Gantt chart showing timing
  - Thread ID Strategy: how threads map to games
- 2026-02-06: Initial milestone created from design discussion (Claude + Vince)
- 2026-02-06: Updated triage from regular function to LLM-powered slate orchestrator graph (two-layer architecture)
- 2026-02-06: Added LangSmith observability and updated frontend progress streaming design
- 2026-02-06: Confirmed Michelle is reactive-only (no proactive offer posting), per-game thread strategy
- 2026-02-06: **Track 3 Implementation** - Integration layer:
  - Created `integration.ts`: Bridge between HTTP handlers and LangGraph system
    - `invokeGameGraphForQuote()`: For instant match requests
    - `invokeGameGraphForUnmatched()`: For unmatched pair detection
    - `invokeGameGraphForCron()`: For slate-dispatched cron evaluations
    - `invokeSlateOrchestrator()`: For scheduled slate evaluation
  - Updated `michelleInstantMatch.ts`: Feature flag `MICHELLE_USE_LANGGRAPH` to conditionally use LangGraph vs LangChain
  - Updated `scheduler.ts`:
    - Added `runSlateOrchestrator()` function with feature flag `MICHELLE_USE_LANGGRAPH_SLATE`
    - Slate orchestrator runs before matching job (prepares evaluations)
    - Added LangSmith status logging on startup when LangGraph is enabled
  - Updated `slateNodes.ts`: Dispatch node invokes per-game graphs via `invokeGameGraphForCron()`
  - Updated `quoteProgressService.ts`: Added LangGraph-specific progress messages (graph_start, cache_hit, blind_prediction, market_pricing, price_response)
  - Updated `progressCallback.ts`: Clarified architecture for LangChain vs LangGraph progress handling
- 2026-02-06: **Track 1 Implementation** - Interface design decisions:
  - Changed `gameId` → `jsonoddsId` (consistent with agent server code, not ESPN)
  - Changed `sport` → `league` with uppercase abbreviations (NBA, NHL, NCAAB, etc.)
  - Restructured `blindPrediction` to mirror contest document format for easy benchmarking:
    - `SpreadPrediction`: `homeSpread` (signed, negative = home favored) — matches `PointSpreadHome`
    - `TotalPrediction`: `projectedTotal` (direct number) — matches `TotalNumber`
    - `MoneylinePrediction`: `homeWinPct` / `awayWinPct` — market odds converted via `americanToImpliedProbability()`
  - Added comparison helpers: `comparePredictionToMarket()`, `americanToImpliedProbability()`, `formatPredictionComparison()`
  - Added `SidePricing` type for market pricing with explicit per-side odds
  - Added helper functions: `getLeagueFromSportId()`, `getGameThreadId()`, `getSlateThreadId()`, `isEvaluationStale()`
  - Created langgraph/ directory under market_maker_michelle/ (separate from langchain/)
- 2026-02-06: **Track 2 Implementation** - Node implementations:
  - Created `nodes.ts` with per-game node implementations:
    - `executeBlindPrediction()`: Full evaluation using research tools WITHOUT market odds (for benchmarking)
    - `executeMarketPricing()`: Generate askOdds/ceilingOdds AFTER seeing market odds
    - `executePriceResponse()`: Fast-path accept/counter/decline logic with exposure checks
  - Created `slateNodes.ts` with slate orchestrator node implementations:
    - `executeGatherContext()`: Query slate from Firebase, existing evaluations, platform activity
    - `executeTriage()`: LLM-powered batch triage with platform signal overrides
    - `executeDispatch()`: Track games for per-game graph invocation (actual invocation in Track 3)
  - Created `exposure.ts` for exposure management:
    - `queryCurrentExposure()`: Query Michelle's total exposure across all games
    - `checkExposureLimits()`: Validate proposed match against per-side/per-game/total limits
    - `recordMatch()`: Update exposure tracking after a match
  - Updated `types.ts`:
    - Added `currentMarketOdds` to trigger types for staleness detection
    - Added `PriceResponseDecision` and `CounterOffer` types
    - Added node output types for state updates
  - Updated `gameGraph.ts` to use actual node implementations instead of stubs
  - Updated `slateGraph.ts` to use actual node implementations instead of stubs
  - Integrated exposure checks into price response flow (decline/counter when limits exceeded)

---

## Overview

Migrate Michelle from stateless LangChain invocations to a stateful LangGraph architecture. The core problem: Michelle currently has no persistent memory of her game evaluations. Every trigger (cron, instant match, offer detection) starts from zero, leading to redundant evaluations, wasted tokens, and an inability to build on prior analysis. LangGraph gives us persistent graph state checkpointed to Supabase (Postgres), enabling Michelle to evaluate once, remember, and act on that evaluation across multiple triggers.

**This migration also unblocks:**
- Benchmarking (M38 deferred) — blind prediction node captures Michelle's assessment *before* seeing market odds
- Lazy evaluation — Michelle triages games instead of evaluating everything, saving tokens
- Multi-agent awareness — future agents can post offers that Michelle evaluates, and she can see their activity on the platform

**What is NOT changing:**
- Dan stays on LangChain. He's simple, works, no migration needed.
- Michelle's tools (getInjuries, getStandings, getScheduleContext, getTeamStats, getRankings) stay as-is
- The frontend instant match UX stays the same (preflight → quote → match)
- Firebase continues handling on-chain event indexing and real-time UI subscriptions
- The existing Express HTTP endpoints remain the entry point for instant evaluation requests

**Philosophy:** This is a migration, not a rewrite. Michelle's evaluation logic, tools, and prompts carry forward. What changes is the *orchestration* — how evaluations are triggered, stored, recalled, and acted upon. Michelle is reactive — she evaluates and prepares pricing but does not proactively post offers. Ospex is no-vig, so there's no bid-ask spread for her to profit from. She exists to provide liquidity when someone comes to her, not to make markets unprompted.

---

## Architecture

### Trigger Taxonomy

Michelle responds to three types of triggers. Each enters the same LangGraph graph but routes differently based on what state already exists.

| Trigger | Source | Sync/Async | Description |
|---------|--------|------------|-------------|
| **Cron (slate evaluation)** | GitHub Actions / server cron | Async | 2-3x daily. Scan slate, triage games, evaluate interesting ones. Michelle does NOT post offers — she only prepares evaluations so she's ready when someone comes to her. |
| **Instant evaluation request** | HTTP endpoint (`/api/michelle/*`) | Sync | Any entity requests Michelle evaluate and price a specific position now. Existing preflight → quote → match flow. |
| **New unmatched pair** | On-chain event / cron detection | Async | Someone posted an offer on Ospex. Michelle decides whether to engage based on existing evaluations or triggers a new one. |

### Game Evaluation Lifecycle

Every game Michelle encounters moves through a lifecycle:

```
┌──────────┐     Cron: triage pass      ┌──────────┐    High interest /    ┌───────────┐
│ Unaware  │ ──────────────────────────► │ Triaged  │ ── Offer received ──►│ Evaluated │
│          │                             │          │                      │           │
│ Game is  │                             │Lightweight│                      │Full tools │
│ on slate │    Instant eval request     │ pass:    │    Game expires      │Blind pred │
│ but not  │ ──────────────────────────► │ rankings │ ── (no action) ────► │Market prc │
│ examined │         (skip triage)       │ interest │       [*]            │Offers rdy │
└──────────┘                             └──────────┘                      └───────────┘
                                                                                │
                                                                           Game settles
                                                                                │
                                                                               [*]
```

**Unaware** — game exists on the slate but Michelle hasn't looked at it.

**Triaged** — lightweight pass. Michelle checks team rankings/records, whether there's likely market interest, and decides if it's worth a full evaluation. Canisius vs Incarnate Word gets filtered here. Low token cost.

**Evaluated** — full deep analysis. All tools used (injuries, standings, stats, schedule context, rankings). Blind prediction captured (pre-market odds). Market pricing generated. Ready to make or respond to offers.

### Michelle's Architecture — Two-Layer Graph System

Michelle's brain is split into two LangGraph instances: a **Slate Orchestrator** that manages triage and awareness across all games, and **Per-Game Graphs** that handle deep evaluation for individual games.

**Layer 1: Slate Orchestrator Graph (thread: `michelle-slate-{date}`)**

```
┌─────────────────────────────────────────────────────────────┐
│  Slate Orchestrator Graph                                   │
│  Thread: michelle-slate-{date}                              │
│                                                             │
│  State: triage decisions, per-game thread summaries,        │
│         Ospex platform activity snapshot, what's been       │
│         evaluated vs what's pending                         │
│                                                             │
│  Nodes:                                                     │
│    1. Gather Context                                        │
│       - Query per-game thread summaries from Supabase       │
│       - Query Ospex activity (unmatched pairs, bet volume)  │
│       - Get current slate from odds API                     │
│                                                             │
│    2. Triage (LLM call)                                     │
│       - "Here are 12 games, here's what I already know,    │
│         here's platform activity — which games need my      │
│         attention?"                                         │
│       - Sees: team rankings, records, existing eval         │
│         summaries, Ospex betting volume per game            │
│       - Can override normal filters: "I wouldn't normally  │
│         look at Incarnate Word but 20 people are betting"  │
│                                                             │
│    3. Dispatch                                              │
│       - Invoke per-game graphs for interesting games        │
│                                                             │
└──────────────┬──────────────────────────────────────────────┘
               │
               │  For each game needing attention:
               │
               ├──► Per-Game Graph (thread: michelle-{contestId-1})
               ├──► Per-Game Graph (thread: michelle-{contestId-2})
               └──► Per-Game Graph (thread: michelle-{contestId-3})
```

**Layer 2: Per-Game Graph (thread: `michelle-{contestId}`)**

```
                         ┌─────────┐
                         │ Trigger │
                         └────┬────┘
                              │
                         ┌────▼────┐
                         │ Router  │ ◄── Reads trigger type + existing state
                         └────┬────┘
                              │
              ┌───────────────┼──────────────────┐
              │               │                  │
        Cron eval       Instant eval       New unmatched
              │            request              pair
              │               │                  │
              │               ▼                  ▼
              │        ┌─────────────────────────────┐
              │        │  Evaluation exists in       │
              │        │  state and still fresh?     │
              │        └───────┬─────────────────────┘
              │          Yes   │    No / Stale
              │            │   │
              │            │   ▼
              ▼            │  ┌─────────────────┐
       ┌───────────┐      │  │ Blind Prediction │ ◄── No market odds
       │   Blind   │      │  │ Node             │     Just tools + analysis
       │Prediction │      │  └────────┬─────────┘
       │   Node    │      │           │
       └─────┬─────┘      │           ▼
             │            │  ┌─────────────────┐
             ▼            │  │ Market Pricing   │ ◄── Now sees market odds
       ┌───────────┐      │  │ Node             │     Generates ask/ceiling
       │  Market   │      │  └────────┬─────────┘
       │  Pricing  │      │           │
       │   Node    │      │           │
       └─────┬─────┘      │           │
             │            │           │
             ▼            ▼           ▼
       ┌──────────────────────────────────┐
       │       Persist State             │ ◄── Checkpoint to Supabase
       │  (evaluation ready for when     │
       │   someone comes to Michelle)    │
       └─────────┬───────────────────────┘
                 │
            ┌────▼──────────────┐
            │ Price Response /  │ ◄── Only runs on instant_eval
            │ Accept / Counter  │     or new_unmatched_pair triggers
            │ / Decline         │
            └───────────────────┘
```

### State Schema

> **IMPLEMENTED**: See `ospex-agent-server/src/agents/market_maker_michelle/langgraph/types.ts` for the actual interfaces.
> Key refinements from original spec:
> - `jsonoddsId` instead of `gameId` (consistent with agent server)
> - `league` instead of `sport` with uppercase abbreviations (NBA, NHL, NCAAB, etc.)
> - `BlindPrediction` uses explicit `SpreadPrediction`, `TotalPrediction`, `MoneylinePrediction` types
> - `MarketPricing` uses per-market `MarketTypePricing` with explicit `SidePricing` per side

```typescript
// ─── ORIGINAL SPEC (kept for reference) ───
// ─── Per-Game Graph State (one thread per game) ───
interface MichelleGameState {
  // ─── Game Context ───
  contestId: string
  gameId: string               // jsonodds Id
  sport: 'nba' | 'nhl' | 'ncaab' | 'nfl' | 'mlb' | 'ncaaf' // check currently used league abbreviations cross-platform and keep this consistent, may want to use cfb or CFB instead of ncaaf, etc.
  homeTeam: string
  awayTeam: string
  gameTime: number

  // ─── Trigger ───
  trigger: {
    type: 'cron_eval' | 'instant_eval' | 'new_unmatched_pair'
    positionData?: any         // offer details for instant_eval
    unmatchedPairData?: any    // on-chain offer data for new_unmatched_pair
  }

  // ─── Evaluation (the core persistent state) ───
  triageResult?: {
    interestLevel: 'high' | 'medium' | 'low' | 'none'
    reason: string             // "Top 25 matchup, likely market activity"
    timestamp: number
  }

  blindPrediction?: {          // BEFORE seeing market odds (benchmarking)
    impliedLine: number        // What Michelle thinks the spread/total should be
    impliedWinProb: number     // Her independent win probability estimate
    confidence: number         // 0-1
    reasoning: string
    toolsUsed: string[]
    timestamp: number
  }

  marketPricing?: {            // AFTER seeing market odds
    askOdds: number
    ceilingOdds: number
    edgeAssessment: string     // "Market has this at -3.5, I had -5, edge on favorite"
    marketOddsAtPricing: number  // for drift detection
    timestamp: number
  }

  // ─── Odds Movement (tracks within this game's thread) ───
  oddsHistory: Array<{
    odds: number
    source: string
    timestamp: number
  }>

  // ─── Quote History (all eval interactions for this game) ───
  quoteHistory: Array<{
    quoteId: string
    requestedOdds: number
    decision: 'accept' | 'counter' | 'decline'
    counterOdds?: number
    reasoning: string
    timestamp: number
  }>

  // ─── Messages (LangGraph convention for LLM conversation history) ───
  messages: BaseMessage[]
}
```

### Persistence Architecture

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│    Frontend      │     │   Agent Server    │     │    Supabase      │
│   (React app)    │     │   (Express)       │     │   (Postgres)     │
│                  │     │                   │     │                  │
│ HTTP requests ───┼────►│ Invoke Michelle's │     │ LangGraph        │
│ (preflight,      │     │ graph with        │     │ checkpoints      │
│  quote, match)   │     │ trigger context   │────►│ (graph state)    │
│                  │     │                   │     │                  │
│ Firebase subs ◄──┼─────┤ Write UI events   │     │ Sports data      │
│ (progress,       │     │ to Firebase       │     │ (standings,      │
│  quotes, offers) │     │                   │     │  injuries, etc)  │
│                  │     │                   │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                   │                  │
                         ┌──────────────────┐      │                  │
                         │    Firebase      │      └──────────────────┘
                         │                  │
                         │ On-chain events  │
                         │ (contests,       │
                         │  positions,      │
                         │  settlements)    │
                         │                  │
                         │ Agent UI state   │
                         │ (agentQuotes,    │
                         │  agentProgress,  │
                         │  agentDecisions) │
                         └──────────────────┘
```

**What goes where:**

| Data | System | Reason |
|------|--------|--------|
| LangGraph checkpoints (Michelle's state) | Supabase | Postgres checkpointer, queryable, co-located with sports data |
| Sports intelligence (standings, injuries, stats, rankings, schedules) | Supabase | Already there from M33/M38 |
| Agent evaluation logs (LLM outputs, tool traces) | Supabase (migrate from Firebase) | Structured data, benefits from SQL queries for benchmarking |
| On-chain event index (contests, positions, settlements) | Firebase | Real-time listeners, proven pipeline, no reason to move |
| Agent UI state (progress streaming, quote status) | Firebase | Real-time subscriptions drive the frontend UX |

### Observability

**LangSmith (developer tool):**

LangSmith provides automatic tracing for LangGraph — every node execution, LLM call, tool invocation, latency, and token usage. It's a developer dashboard, not user-facing.

Setup is minimal (environment variables only):
```bash
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=<key>
LANGCHAIN_PROJECT=ospex-michelle
```

With LangGraph, traces automatically show the graph structure: which node fired, what state it read, what it wrote, how long each step took. Essential for debugging during migration.

**Frontend Progress Streaming (user-facing):**

The existing Firebase-based progress streaming continues but updates to reflect the graph's node structure. Users see Michelle's thought process mapped to graph execution:

*When evaluation exists in state (fast path):*
```
✓ Checking for existing evaluation...          (router node)
✓ Found evaluation from 45 minutes ago...      (state hit)
✓ Reviewing your offer against my pricing...   (priceResponse node)
✓ Decision: ACCEPT
```

*When fresh evaluation needed:*
```
✓ Analyzing this game independently...          (blindPrediction node)
  ✓ Reviewing injury reports...                  (tool call within node)
  ✓ Reviewing team standings...                  (tool call within node)
  ✓ Checking schedule context...                 (tool call within node)
✓ My independent assessment: LAL -4.5, 62%      (blind prediction output)
✓ Comparing against market odds...               (marketPricing node)
✓ Reviewing your offer against my pricing...     (priceResponse node)
✓ Decision: ACCEPT
```

Users see the benefit of cached state — "Found evaluation from 45 minutes ago" explains why a quote came back in 3 seconds instead of 60 and communicates that Michelle is smart enough to not redo work.

---

## Track 1: LangGraph Infrastructure

Set up the foundation: install packages, configure the Supabase checkpointer, define the state schema, and build the graph skeleton with stub nodes.

### Tasks

- [x] Install `@langchain/langgraph` and `@langchain/langgraph-checkpoint-postgres` (done)
- [x] Configure Postgres checkpointer pointing at Supabase → `langgraph/checkpointer.ts`
- [x] Define `MichelleGameState` TypeScript interface → `langgraph/types.ts`
- [x] Define `SlateOrchestratorState` TypeScript interface → `langgraph/types.ts`
- [x] Create per-game graph skeleton → `langgraph/gameGraph.ts`
- [x] Create slate orchestrator graph skeleton → `langgraph/slateGraph.ts`
- [x] Implement stub nodes for per-game graph (router, blindPrediction, marketPricing, priceResponse, persistState)
- [x] Implement stub nodes for slate orchestrator (gatherContext, triage, dispatch)
- [x] Wire edges with conditional routing based on trigger type
- [x] Write integration test: invoke per-game graph with thread ID → state persists → invoke same thread → state is recalled
- [x] Write integration test: different thread IDs have independent state
- [x] Write integration test: slate orchestrator invokes per-game graphs (tested via trigger type test)
- [x] Configure LangSmith for developer observability → `langgraph/langsmith.ts`

> **Note:** Files created in `ospex-agent-server/src/agents/market_maker_michelle/langgraph/`:
> - `types.ts` - State interfaces with explicit team-side fields
> - `checkpointer.ts` - Postgres checkpointer configuration for Supabase
> - `gameGraph.ts` - Per-game graph with stub nodes
> - `slateGraph.ts` - Slate orchestrator graph with stub nodes
> - `langsmith.ts` - LangSmith observability configuration
> - `index.ts` - Module exports

### Technical Notes

**Checkpointer setup (actual implementation):**
```typescript
import {
  initializeCheckpointer,
  buildGameGraphWithCheckpointer,
  createInitialGameState,
  getGameThreadId,
} from './langgraph';

// Initialize once at startup
const checkpointer = await initializeCheckpointer();

// Build graph with checkpointer
const gameGraph = buildGameGraphWithCheckpointer(checkpointer);

// Invoke for a specific game
const result = await gameGraph.invoke(
  createInitialGameState(
    { contestId, jsonoddsId, league: 'NBA', homeTeam, awayTeam, gameTime },
    { type: 'instant_eval', positionData: { market, side, line, requestedOdds, requestedAmount, requesterAddress } }
  ),
  { configurable: { thread_id: getGameThreadId(contestId, jsonoddsId) } }
);
```

**Environment variables required:**
```bash
# Supabase Postgres connection (from Dashboard > Settings > Database)
SUPABASE_DB_URL=postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres

# LangSmith observability (optional, for developer debugging)
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=<your-api-key>
LANGCHAIN_PROJECT=ospex-michelle
```

**Thread ID strategy:** One thread per game. When a game appears on the slate (via odds API), it gets its own thread. All evaluation activity for that game — triage result, blind prediction, market pricing, re-evaluations as odds move, quote responses — accumulates in that thread. The thread ends when the game settles.

A thread in LangGraph is a namespace for state — same thread ID across invocations means accumulated state is available. Different thread ID means fresh start. Per-game threads mean Michelle's 2pm evaluation of Lakers vs Celtics is available when someone requests an instant match on that game at 3pm.

**Thread ID convention:** `michelle-{contestId}` or `michelle-{jsonoddsId}` — tied to the game, not the date. Uses helper function `getGameThreadId(contestId, jsonoddsId)` which prefers contestId if available.

**Why per-game (not per-day or single long-running):**
- Clean isolation: each thread is a complete audit trail for one game
- Natural lifecycle: thread starts when game appears, ends when game settles
- Odds movement within a thread tracks volatility for that specific game
- Heavy slate days (15+ NBA games) don't create bloated state objects
- Cross-game impact is minimal — an evaluation of one game rarely affects another

**Cross-game concerns are managed by the Slate Orchestrator Graph:**
- **Triage** runs as an LLM-powered node in the slate orchestrator graph (`michelle-slate-{date}` thread), with full awareness of existing evaluations, platform activity, and game context
- **Exposure management** is queried from a Supabase table (or derived from on-chain positions) at decision time within per-game graphs, not stored in per-game graph state — on-chain positions are the source of truth for total exposure

---

## Track 2: Michelle Node Implementation

Convert Michelle's existing evaluation and pricing logic into discrete graph nodes. Most of the LLM interaction and tool usage already exists — this track is about decomposing it into nodes that read from and write to graph state.

### Router Node

Reads the trigger and current state for this game, decides which path to take.

- [x] Implement router logic:
  - `cron_eval` → blindPrediction (triage already happened before graph was invoked)
  - `instant_eval` → check if evaluation exists in state → if yes and fresh, priceResponse; if stale or missing, blindPrediction
  - `new_unmatched_pair` → check if evaluation exists in state → if yes, priceResponse; if no, blindPrediction
- [x] Handle staleness check (replace current 30-min / 3% drift logic with state-based staleness using `marketOddsAtPricing` for drift detection)

> **Implementation Notes:**
> - Router logic implemented in `gameGraph.ts` via `routeByTrigger()` function
> - Uses `isEvaluationStale()` from types.ts for staleness detection
> - Added `currentMarketOdds` to trigger types for drift comparison

### Slate Orchestrator Nodes

The slate orchestrator is its own LangGraph with a `michelle-slate-{date}` thread. It maintains awareness of the full slate and makes intelligent triage decisions using LLM reasoning, platform activity, and summaries of existing per-game evaluations.

#### Gather Context Node

Assembles the information the triage LLM needs to make decisions.

- [x] Query current slate from odds API
- [x] Query per-game thread summaries from Supabase checkpoints (what has Michelle already evaluated today, and when?)
- [x] Query Ospex platform activity: unmatched pairs per game, betting volume, number of users active on each game
- [x] Assemble lightweight game context: team names, rankings, records, sport
- [x] Write all context to slate orchestrator state

> **Implementation Notes:**
> - Implemented in `slateNodes.ts` via `executeGatherContext()`
> - Queries Firebase contests collection via `getContestsForToday()`
> - Thread summaries queried from agentOffers as proxy for LangGraph checkpoints
> - Platform activity stub returns empty (TODO: query positions collection for unmatched pairs)

#### Triage Node (LLM call)

The LLM decides which games need Michelle's attention based on full context.

- [x] Design triage prompt — receives slate, existing evaluation summaries, platform activity
  - Key capability: can override normal filters based on platform signals ("I wouldn't normally look at Incarnate Word but 20 people are betting on it")
  - Key capability: knows what's already been evaluated and whether re-evaluation is needed ("NYK vs MIA was evaluated 2 hrs ago, odds have moved 5%, worth re-evaluating")
  - Output: interestLevel (high/medium/low/none) + reason per game
- [x] Batch format: one LLM call for the whole slate, not per-game
- [x] Write triage decisions to slate orchestrator state (persists — second cron run sees what first run decided)

> **Implementation Notes:**
> - Implemented in `slateNodes.ts` via `executeTriage()`
> - Comprehensive triage prompt covers league priority, matchup quality, existing evaluations, platform activity, time to start
> - Fallback logic if LLM fails: mark games with odds as high interest

#### Dispatch Node

Invokes per-game graphs for each game the triage node flagged.

- [x] For each game with high/medium interest: invoke per-game graph with `trigger.type = 'cron_eval'`
- [x] Pass thread ID: `michelle-{contestId}`
- [x] Handle sequential vs parallel invocation (start sequential, optimize later if needed)

> **Implementation Notes:**
> - Implemented in `slateNodes.ts` via `executeDispatch()`
> - Tracks dispatched games in state
> - Actual graph invocation will be wired in Track 3 (integration)

### Blind Prediction Node

Full evaluation using all tools, but WITHOUT market odds. This is the key architectural change that enables benchmarking.

- [x] Adapt Michelle's existing `evaluateGame()` logic
- [x] System prompt modification: explicitly instruct that market odds are NOT available at this stage
- [x] Tools available: getInjuries, getStandings, getScheduleContext, getTeamStats, getRankings
- [x] Output: `impliedLine`, `impliedWinProb`, `confidence`, `reasoning`
- [x] Write output to game state `blindPrediction` field

> **Implementation Notes:**
> - Implemented in `nodes.ts` via `executeBlindPrediction()`
> - Uses LangChain agent with filtered tool set (research tools only, no market-dependent tools)
> - Outputs `SpreadPrediction`, `TotalPrediction`, `MoneylinePrediction` for all three market types
> - Timeout set to 45 seconds for full research cycle

### Market Pricing Node

Now reveal market odds and generate actionable pricing.

- [x] System prompt: "Here is your independent assessment: [blind prediction]. Now here are the current market odds: [odds]. Generate your ask and ceiling."
- [x] Output: `askOdds`, `ceilingOdds`, `edgeAssessment`
- [x] Write to game state `marketPricing` field

> **Implementation Notes:**
> - Implemented in `nodes.ts` via `executeMarketPricing()`
> - Receives blind prediction from state and current market odds from trigger/fetch
> - Generates `MarketTypePricing` for each market with `SidePricing` per side
> - Includes edge assessment summary comparing blind prediction to market

### Price Response Node

When an instant eval request or unmatched pair comes in. This is where Michelle's "garage sale" judgment lives — she doesn't just mechanically compare to ceiling, she reasons about the offer.

- [x] Read existing evaluation from state
- [x] Apply decision logic:
  - Offer within ask range → accept (fast path, no LLM needed)
  - Offer within negotiation range → accept at offered odds (fast path)
  - Offer outside range but reasonable → LLM reasons about a counter-offer (NOT just revealing ceiling — Michelle negotiates like a human who wouldn't tell you her bottom dollar)
  - Offer absurdly outside range → reject
- [x] If evaluation is stale, route back to blindPrediction for refresh
- [x] Return decision: accept / counter (with reasoned counter-odds) / decline

> **Implementation Notes:**
> - Implemented in `nodes.ts` via `executePriceResponse()`
> - Fast-path logic for quick decisions (no LLM call needed for most cases)
> - Counter logic: gap <10% → counter at ceiling; gap >10% → decline
> - Exposure checks integrated: decline/counter when limits exceeded
> - Staleness handled by router before reaching this node

### Exposure Management

Track and enforce risk limits. Exposure spans all games, so it lives outside per-game state.

- [x] Design `agent_exposure` view or query in Supabase (derived from on-chain positions — source of truth)
- [x] Exposure check runs at decision time: before accepting/countering, query total exposure across all games
- [x] Enforce `maxAllowed` exposure limit
- [x] Factor available exposure into position sizing

> **Implementation Notes:**
> - Implemented in `exposure.ts` module
> - `queryCurrentExposure()`: Queries agentOffers for exposure data (production should reconcile with on-chain positions)
> - `checkExposureLimits()`: Validates per-side, per-game, and total limits
> - `calculateMaxMatchAmount()`: Returns safe maximum for a given position
> - `recordMatch()`: Updates exposure tracking after successful match
> - Integrated into `executePriceResponse()`: automatic decline/counter when limits exceeded

### Files Created/Modified in Track 2

| File | Purpose |
|------|---------|
| `nodes.ts` (new) | Per-game node implementations: `executeBlindPrediction()`, `executeMarketPricing()`, `executePriceResponse()` |
| `slateNodes.ts` (new) | Slate orchestrator node implementations: `executeGatherContext()`, `executeTriage()`, `executeDispatch()` |
| `exposure.ts` (new) | Exposure management: `queryCurrentExposure()`, `checkExposureLimits()`, `recordMatch()` |
| `types.ts` (modified) | Added `currentMarketOdds` to triggers, `PriceResponseDecision`, `CounterOffer`, node output types |
| `gameGraph.ts` (modified) | Updated nodes to use actual implementations instead of stubs |
| `slateGraph.ts` (modified) | Updated nodes to use actual implementations instead of stubs |
| `index.ts` (modified) | Added exports for new functions and types |

---

## Track 3: Integration

Wire the new graph into the existing agent server and ensure all existing flows still work.

### Tasks

- [x] Update agent server cron handler: invoke slate orchestrator graph, which handles triage and dispatches to per-game graphs
  - Added `runSlateOrchestrator()` to scheduler with `MICHELLE_USE_LANGGRAPH_SLATE` feature flag
  - Slate orchestrator runs before matching job on cron
- [x] Update `/api/michelle/quote` handler to invoke per-game graph with `trigger.type = 'instant_eval'`
  - Added `evaluateWithLangGraph()` to michelleInstantMatch.ts
  - Controlled by `MICHELLE_USE_LANGGRAPH` feature flag
- [x] Update unmatched pair detection to invoke per-game graph with `trigger.type = 'new_unmatched_pair'`
  - Added `evaluateUnmatchedPairViaLangGraph()` to matching-job.ts
  - Controlled by `MICHELLE_USE_LANGGRAPH_MATCHING` feature flag
- [x] Ensure Firebase UI writes still happen (agentQuotes, agentQuoteProgress, agentOffers, agentDecisions)
  - Progress callbacks reused from LangChain implementation
  - `MichelleProgressHandler` works for both LangChain and LangGraph
- [x] Update frontend progress streaming to reflect graph node structure:
  - [x] Map graph nodes to user-friendly progress messages (added to `quoteProgressService.ts`)
  - [x] Show state cache hits ("Found evaluation from X minutes ago...") (implemented in `integration.ts`)
  - [ ] Nest tool calls under parent node messages (deferred - current flat structure works)
  - [x] Update `progressCallback.ts` for new progress event types (updated header docs)
- [x] Configure LangSmith environment variables on agent server (developer observability)
  - LangSmith status logged on startup when LangGraph is enabled
  - Env vars: `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT`
- [ ] Migrate agent evaluation logs from Firebase to Supabase (deferred to Track 4)
  - [ ] Design `agent_evaluation_logs` table
  - [ ] Write evaluation audit trail to Supabase instead of Firebase agentCalculations
  - [ ] Keep Firebase agentQuoteProgress for real-time UI streaming
- [ ] Remove old LangChain evaluation code paths for Michelle (deferred - feature flags allow gradual migration)
- [x] Verify Dan continues to work unchanged on LangChain (no changes to Dan's code paths)

### Feature Flags

| Flag | Purpose |
|------|---------|
| `MICHELLE_USE_LANGGRAPH=true` | Use LangGraph for instant match quotes |
| `MICHELLE_USE_LANGGRAPH_SLATE=true` | Use slate orchestrator for cron evaluation |
| `MICHELLE_USE_LANGGRAPH_MATCHING=true` | Use LangGraph for unmatched pair matching |

### Files Modified in Track 3

| File | Changes |
|------|---------|
| `integration.ts` (new) | Bridge between HTTP handlers and LangGraph system |
| `michelleInstantMatch.ts` | Added LangGraph evaluation path with feature flag |
| `scheduler.ts` | Added slate orchestrator invocation, LangSmith status logging |
| `slateNodes.ts` | Dispatch node invokes per-game graphs |
| `matching-job.ts` | Added LangGraph evaluation path for unmatched pairs |
| `quoteProgressService.ts` | Added LangGraph progress messages, new step definitions (reviewing_odds_history, reviewing_rankings, blind_prediction_complete) |
| `progressCallback.ts` | Updated docs for LangGraph architecture, added get_odds_history/get_rankings tool mappings and detail extractors |
| `nodes.ts` | Added node-level progress emission (blind_prediction, market_pricing, price_response, blind_prediction_complete) |
| `index.ts` | Added integration layer exports |
| `MichelleEvaluationLog.tsx` (frontend) | Added formatters for reviewing_odds_history and blind_prediction_complete |

### Instant Match Flow (Updated)

The existing HTTP flow stays the same from the frontend's perspective:

```
POST /api/michelle/preflight  →  Capacity check (query exposure from Supabase, no LLM)
POST /api/michelle/quote      →  Invoke per-game graph (thread: michelle-{contestId})
                                  → Router checks state in this game's thread
                                  → If evaluated and fresh: priceResponse node (fast — 3-5s)
                                  → If stale or missing: blindPrediction → marketPricing → priceResponse
POST /api/michelle/match      →  Execute on-chain (unchanged)
POST /api/michelle/counter-accept  →  Accept counter (unchanged)
```

---

## Track 4: Verification & Testing

### Core Scenarios to Verify

- [x] **Duplicate evaluation eliminated:** Request instant match on a game Michelle already evaluated via cron → she uses existing evaluation from that game's thread, no redundant LLM call
  - Verified: cron_eval 39.2s → instant_eval 2.6s (cache hit, 93% faster)
- [x] **Lazy evaluation works:** Cron triggers triage → low-interest games are filtered out before any per-game graphs are invoked
  - Verified: 6 games triaged, 4 dispatched (NBA + Power 5), 2 skipped (low-tier CAA/NEC)
- [x] **Blind predictions captured:** Every full evaluation has a `blindPrediction` recorded before market odds are revealed → benchmarking data is accumulating
  - Verified: Rockets @ Thunder blind prediction captured independent of market odds
- [x] **Per-game state isolation:** Evaluation in one game's thread doesn't affect another game's thread
  - Verified: Integration test confirmed thread isolation
- [x] **State persists across triggers:** Cron evaluates LAL vs BOS at 2pm → instant match request on same game at 2:15pm → state from 2pm is available in thread `michelle-{contestId}`
  - Verified: Same thread ID returned cached blindPrediction timestamp
- [x] **Staleness handled:** If evaluation is old or market has drifted significantly (using `marketOddsAtPricing` for comparison), re-evaluation is triggered
  - Verified: Router logic in gameGraph.ts uses isEvaluationStale()
- [x] **Dan unaffected:** Dan's cron-based LangChain flow still works identically
  - Verified: Dan has no LangGraph imports, uses standard runOnce() flow in agent.ts
  - Dan's prompt.ts has zero references to LangGraph/StateGraph/PostgresSaver
  - All LangGraph code is isolated in market_maker_michelle/langgraph/ directory
  - MICHELLE_USE_LANGGRAPH* feature flags only affect Michelle's code paths

### Regression Checks

> **Status:** Code complete. Awaiting frontend testing to verify progress streaming displays correctly.

- [ ] Frontend instant match UX unchanged (user sees same flow)
- [ ] Firebase progress streaming works during graph execution
- [ ] Progress messages correctly reflect graph state:
  - [ ] Node-level steps display: "Analyzing the game...", "Generating pricing...", "Making my decision..."
  - [ ] Cache hits show "Found evaluation from X minutes ago..."
  - [ ] Tool details expand correctly (injuries, standings, line movement, etc.)
  - [ ] Blind prediction summary shows (spread/total/win%/reasoning)
- [ ] On-chain transactions still execute correctly
- [ ] Agent offers appear correctly in UI
- [ ] Leaderboard data unaffected
- [x] LangSmith traces appear in dashboard for all graph invocations
  - Verified: Traces show complete flow - triage, blind prediction tools, market pricing with `get_odds_history` calls

---

## Out of Scope (Documented for Future)

| Item | Why Not Now | Likely Milestone |
|------|-------------|------------------|
| Full benchmarking analysis & UI | Need data to accumulate first; blind predictions start collecting in M39 | M40+ |
| Benchmarking comparison scripts | Depends on blind prediction data volume | M40+ |
| Additional agents (handicapper, totals specialist) | Architecture supports them now, but Michelle migration is priority | M41+ |
| Custom ELO / power rankings feeding blind predictions | Novel system, needs design | M41+ |
| Dan migration to LangGraph | No need — he's simple and works | Only if needed |
| Multi-agent communication (agents reading each other's state) | Architecture supports it via shared Supabase, but not needed yet | M42+ |
| Agent registration system | No external agents yet | TBD |

---

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| `@langchain/langgraph` | To install | Core graph framework |
| `@langchain/langgraph-checkpoint-postgres` | To install | Supabase checkpointer |
| LangSmith account + API key | To configure | Free tier sufficient; env vars only |
| Supabase Postgres access | ✅ Available | Already used for sports data |
| Michelle's existing tools | ✅ Available | getInjuries, getStandings, getScheduleContext, getTeamStats, getRankings |
| Michelle's existing prompts | ✅ Available | Will be adapted for node-specific context |
| Express HTTP endpoints | ✅ Available | Will be updated to invoke graph |
| Firebase collections | ✅ Available | UI writes continue as-is |

---

## Time Estimate

| Track | Estimate | Notes |
|-------|----------|-------|
| Track 1: LangGraph infrastructure | 10-14 hours | Checkpointer setup, state schemas, both graph skeletons (slate orchestrator + per-game), integration tests, LangSmith config |
| Track 2: Node implementation | 14-20 hours | Slate orchestrator nodes (gatherContext, triage, dispatch), per-game nodes (router, blind prediction, market pricing, price response), exposure |
| Track 3: Integration | 10-14 hours | Server wiring, progress streaming updates, LangSmith env vars, Firebase writes, log migration, old code removal |
| Track 4: Verification | 4-6 hours | Scenario testing, regression checks |

**Total:** 38-54 hours across ~2-3 weeks at current pace

---

## Success Criteria

- [ ] Michelle runs on LangGraph with per-game threads checkpointed to Supabase
- [ ] Slate orchestrator graph triages games with LLM reasoning, aware of platform activity and existing evaluations
- [ ] Duplicate evaluation problem is eliminated (instant match on evaluated game uses cached state from that game's thread)
- [ ] Lazy evaluation implemented (triage filters out low-interest games; can override based on platform signals)
- [ ] Blind predictions captured before market odds are revealed (benchmarking unblocked)
- [ ] All three trigger types work: cron (via slate orchestrator), instant eval request, new unmatched pair
- [ ] Frontend progress streaming updated to reflect graph node structure (shows cache hits, nested tool calls)
- [ ] LangSmith tracing enabled for developer observability
- [ ] Frontend instant match flow is unchanged from user perspective
- [ ] Firebase real-time UI updates still function
- [ ] Dan continues to operate on LangChain unchanged
- [ ] Old LangChain evaluation paths for Michelle are removed (clean migration)
- [ ] Agent evaluation logs migrated from Firebase to Supabase

---

## Flow Diagrams (Mermaid)

> Added 2026-02-06: Visual representation of the implemented LangGraph architecture.

### Complete System Flow

Shows how triggers enter the system and flow through the two-layer graph architecture.

```mermaid
flowchart TB
    subgraph Triggers["Trigger Sources"]
        CRON[/"Scheduler Cron<br/>(every 15-30 min)"/]
        HTTP[/"HTTP Request<br/>/api/michelle/quote"/]
        CHAIN[/"On-Chain Event<br/>(unmatched pair)"/]
    end

    subgraph SlateOrch["Slate Orchestrator Graph<br/>(thread: michelle-slate-{date})"]
        GC[Gather Context]
        TR[Triage<br/>LLM Decision]
        DISP[Dispatch]

        GC --> TR --> DISP
    end

    subgraph PerGame["Per-Game Graph<br/>(thread: michelle-{contestId})"]
        ROUTER{Router}
        BP[Blind Prediction<br/>~30-40s]
        MP[Market Pricing<br/>~5-10s]
        PR[Price Response<br/>~1-2s]
        PERSIST[Persist State]

        ROUTER -->|"fresh eval needed"| BP
        ROUTER -->|"cache hit"| PR
        BP --> MP --> PR
        PR --> PERSIST
    end

    subgraph Storage["Persistence"]
        SUPA[(Supabase<br/>Postgres)]
        FB[(Firebase)]
    end

    CRON --> GC
    HTTP -->|"instant_eval"| ROUTER
    CHAIN -->|"new_unmatched_pair"| ROUTER
    DISP -->|"cron_eval<br/>for each game"| ROUTER

    PERSIST -.->|"checkpoint"| SUPA
    PR -.->|"agentQuotes<br/>agentDecisions"| FB
```

### Slate Orchestrator Detail

Shows the LLM-powered triage that filters games before full evaluation.

```mermaid
flowchart LR
    subgraph Input["Input"]
        CRON[/"Cron Trigger"/]
    end

    subgraph GatherContext["1. Gather Context"]
        Q1[Query Firebase<br/>getUpcomingContests]
        Q2[Query Thread Summaries<br/>existing evaluations]
        Q3[Query Platform Activity<br/>unmatched pairs]
    end

    subgraph Triage["2. Triage (LLM)"]
        LLM["Claude analyzes slate:<br/>• League priority (NBA > NCAAB)<br/>• Team quality (Power 5 > mid-major)<br/>• Platform activity signals<br/>• Existing evaluation freshness"]
    end

    subgraph Decisions["Triage Output"]
        HIGH[/"HIGH: Dispatch"/]
        MED[/"MEDIUM: Dispatch"/]
        LOW[/"LOW: Skip"/]
        NONE[/"NONE: Skip"/]
    end

    subgraph Dispatch["3. Dispatch"]
        D1[Invoke Per-Game Graph<br/>game 1]
        D2[Invoke Per-Game Graph<br/>game 2]
        D3[Invoke Per-Game Graph<br/>game N]
    end

    CRON --> Q1 & Q2 & Q3
    Q1 & Q2 & Q3 --> LLM
    LLM --> HIGH & MED & LOW & NONE
    HIGH --> D1 & D2 & D3
    MED --> D1 & D2 & D3
```

### Per-Game Graph Detail

Shows the router logic and node flow for a single game evaluation.

```mermaid
flowchart TB
    subgraph Trigger["Trigger Input"]
        T1[cron_eval]
        T2[instant_eval]
        T3[new_unmatched_pair]
    end

    ROUTER{{"Router Node<br/>Check existing state"}}

    subgraph StateCheck["State Check"]
        EXISTS{"blindPrediction<br/>exists?"}
        FRESH{"Still fresh?<br/>(< 30 min, < 3% drift)"}
    end

    subgraph Evaluation["Full Evaluation Path (~40s)"]
        BP["do_blind_prediction<br/>─────────────────<br/>• getInjuryReport<br/>• getStandings<br/>• getScheduleContext<br/>• getTeamStats<br/>─────────────────<br/>Output: SpreadPrediction,<br/>TotalPrediction,<br/>MoneylinePrediction"]

        MP["do_market_pricing<br/>─────────────────<br/>Compare blind prediction<br/>to current market odds<br/>─────────────────<br/>Output: askOdds,<br/>ceilingOdds per market"]
    end

    subgraph Response["Response Path (~2s)"]
        PR["do_price_response<br/>─────────────────<br/>• Check exposure limits<br/>• Compare to askOdds/ceiling<br/>• Fast-path decision<br/>─────────────────<br/>Output: accept/counter/decline"]
    end

    PERSIST["persist_state<br/>Checkpoint to Supabase"]

    END_NODE(("END"))

    T1 --> ROUTER
    T2 --> ROUTER
    T3 --> ROUTER

    ROUTER --> EXISTS
    EXISTS -->|"No"| BP
    EXISTS -->|"Yes"| FRESH
    FRESH -->|"Stale"| BP
    FRESH -->|"Fresh"| PR

    BP --> MP
    MP -->|"cron_eval"| PERSIST
    MP -->|"instant_eval /<br/>new_unmatched_pair"| PR
    PR --> PERSIST
    PERSIST --> END_NODE
```

### HTTP Instant Match Flow

Shows the complete request flow from frontend to on-chain execution.

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant API as Agent Server
    participant LG as LangGraph
    participant SB as Supabase
    participant FB as Firebase
    participant BC as Blockchain

    Note over FE,BC: User wants to match with Michelle

    FE->>API: POST /api/michelle/preflight
    API->>SB: Query exposure limits
    SB-->>API: Current exposure
    API-->>FE: { available: true, maxAmount: 500 }

    FE->>API: POST /api/michelle/quote
    API->>LG: invokeGameGraphForQuote()

    alt Cache Hit (2-3s)
        LG->>SB: getTuple(thread_id)
        SB-->>LG: Existing blindPrediction + marketPricing
        LG->>LG: Router → do_price_response
        LG->>SB: Query exposure
        LG->>LG: Fast-path decision
    else Cache Miss (30-40s)
        LG->>LG: Router → do_blind_prediction
        LG->>SB: getInjuries, getStandings, etc.
        LG->>LG: do_market_pricing
        LG->>LG: do_price_response
    end

    LG->>SB: Checkpoint state
    LG->>FB: Write agentQuotes doc
    LG-->>API: { decision: "accept", odds: 1.91 }
    API-->>FE: Quote response

    alt User Accepts Quote
        FE->>API: POST /api/michelle/match
        API->>BC: Execute match transaction
        BC-->>API: Transaction hash
        API->>FB: Update agentDecisions
        API-->>FE: { success: true, txHash: "0x..." }
    end
```

### Cache Hit vs Cache Miss Performance

```mermaid
gantt
    title Per-Game Evaluation Timeline
    dateFormat ss
    axisFormat %S s

    section Cache Miss (First Eval)
    Router           :r1, 00, 1s
    Blind Prediction :bp, after r1, 35s
    Market Pricing   :mp, after bp, 7s
    Price Response   :pr1, after mp, 2s
    Persist          :p1, after pr1, 1s

    section Cache Hit (Subsequent)
    Router           :r2, 00, 1s
    Price Response   :pr2, after r2, 2s
    Persist          :p2, after pr2, 1s
```

### Thread ID Strategy

```mermaid
flowchart LR
    subgraph Games["Today's Slate"]
        G1["Lakers @ Celtics"]
        G2["Rockets @ Thunder"]
        G3["Heat @ Knicks"]
    end

    subgraph Threads["LangGraph Threads (Supabase)"]
        T1["michelle-contestId-123<br/>───────────────<br/>blindPrediction: {...}<br/>marketPricing: {...}<br/>quoteHistory: [...]"]
        T2["michelle-contestId-456<br/>───────────────<br/>blindPrediction: {...}<br/>marketPricing: {...}<br/>quoteHistory: [...]"]
        T3["michelle-contestId-789<br/>───────────────<br/>blindPrediction: {...}<br/>marketPricing: {...}<br/>quoteHistory: [...]"]
    end

    G1 --> T1
    G2 --> T2
    G3 --> T3

    subgraph Triggers["Multiple Triggers, Same Thread"]
        CRON1["Cron @ 2pm"]
        USER1["User quote @ 2:15pm"]
        USER2["User quote @ 2:30pm"]
    end

    CRON1 --> T2
    USER1 --> T2
    USER2 --> T2

    Note1["All 3 triggers share<br/>the same thread state"]
```
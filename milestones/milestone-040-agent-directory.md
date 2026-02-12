# Milestone 40: Agent Showcase & March Madness Preparation

*Created: February 9, 2026*
*Status: 🔧 In Progress*
*Target Completion: March 14, 2026 (before Selection Sunday)*

**Edit Trail:**
- 2026-02-09: Initial milestone created from design discussion (Claude + Vince)
- 2026-02-09: Track 1 completed. Deviations: (1) Stored in Firebase `agentPerformanceMetrics` instead of Supabase - all agent data uses Firebase, Supabase is for sports reference data only. (2) Reused existing `marketFromScorerAddress()` from `matching-utils.ts` instead of creating new scorer-to-betType utility.
- 2026-02-09: Track 3 backend completed. Deviations: (1) Created separate `ospex-agent-server/src/http/evaluations.ts` for HTTP handlers rather than adding to `gameEvaluationService.ts` - follows existing pattern from `insights.ts` and `michelleInstantMatch.ts`. (2) Endpoints registered in ospex-agent-server (port 3000), not ospex-api-server - agent-server handles all agent-related logic, api-server is for infrastructure (secrets, pending state). Frontend tasks deferred to later phase.
- 2026-02-10: Stretch goal (Blind Eval Skip Logic) promoted and implemented. Blind predictions are now write-once per game thread. Re-evaluations (stale or drift) skip directly to market_pricing, preserving benchmarking integrity and saving ~30-40 seconds of LLM time per re-evaluation.
- 2026-02-11: Track 2 (Agent Directory Page) completed. Deviations: (1) Route is `/a` instead of `/agents` to match existing short route pattern (`/c/:id`, `/i/:id`, `/u/:address`). (2) Agent detail view deferred - clicking agent card navigates to existing `/u/:address` profile page which already shows positions and stats. (3) Recent activity section not added - profile page already shows this. (4) Used existing ProfileTabs component for sub-navigation instead of custom implementation.

---

## Overview

Build the frontend "agent showcase" layer and prepare the platform for its first real-world stress test: March Madness 2026. The core thesis: a user landing on Ospex during the tournament should immediately understand what agents are, see their track records, read their reasoning, and be able to interact with the market. This milestone combines four workstreams: a read-only agent directory with sortable benchmarks, surfacing LLM-generated reasoning on the frontend, UX fixes from real user testing, and a content ingestion pipeline for future pundit agents.

**Why now:**
- Selection Sunday is March 15. Games start March 20. This is a hard deadline.
- Michelle and Dan are autonomously betting against each other — the system works. Now it needs a frontend that shows that to the world.
- M39 (LangGraph migration) gave us blind predictions, persistent evaluation state, and structured LLM outputs. This milestone turns that infrastructure into user-facing value.
- Real user testing revealed critical UX gaps: unclear bet amounts, broken modals on smaller viewports, confusing position creation flow.

**What this is NOT:**
- Not "create your own agent" — the directory is read-only for now (showcase, not workshop)
- Not a full mobile optimization pass — we're fixing the specific viewport issues found in testing
- Not a revenue overhaul — we're doing a quick audit of existing payment gating, not building new payment infrastructure
- Not full pundit agent autonomy — we may be able to get one agent going as a content ingestion pipeline, with betting logic to follow

**Philosophy:** Ship what makes the site compelling to a stranger during March Madness. Every feature in this milestone should pass the test: "Does this help someone who just landed on Ospex understand what's happening and want to stick around?"

---

## Architecture

### Agent Performance Data Pipeline

The agent directory requires aggregated performance data that doesn't currently exist in a queryable form. Raw data exists across Firebase and on-chain — this milestone adds an aggregation layer.

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  On-Chain Data   │     │    Firebase       │     │    Firebase      │
│                  │     │                   │     │                  │
│ polygonPositions │     │ agentOffers       │     │ agentPerformance │
│ v2.3 (matched    │────►│ agentDecisions    │────►│ Metrics          │
│ amounts, odds)   │     │ positionEvals     │     │ (aggregated)     │
│                  │     │                   │     │                  │
│ polygonSpeculat- │     │ contests          │     │ Computed after   │
│ ionsv2.3        │     │ (sportId, teams)  │     │ scorer settles   │
│ (winSide,       │     │                   │     │                  │
│  status)        │     │                   │     │ [DEVIATION: Used │
└──────────────────┘     └──────────────────┘     │ Firebase, not    │
                                                  │ Supabase]        │
                                                  └──────────────────┘
                                                          │
                                                          ▼
                                                  ┌──────────────────┐
                                                  │    Frontend      │
                                                  │                  │
                                                  │ Agent Directory  │
                                                  │ (sortable by     │
                                                  │  sport, betType, │
                                                  │  record, ROI)    │
                                                  └──────────────────┘
```

**Aggregation service** (`performanceAggregationService.ts`): Runs on the existing scheduler, ideally after the scorer settles speculations. Queries settled positions for each agent, joins contest data for sport/league, maps speculationScorer address to bet type, computes metrics per `(agentId, sportId, betType)`, and upserts to an `agentPerformanceMetrics` collection.

**Metrics schema:**

```typescript
interface AgentPerformanceMetrics {
  agentId: string;
  sportId: number;       // 0=MLB, 1=NBA, 2=NHL, etc.
  betType: 'moneyline' | 'spread' | 'total';

  totalPositions: number;
  wins: number;
  losses: number;
  pushes: number;
  winRate: number;        // wins / (wins + losses)

  totalWagered: number;   // USDC
  totalReturned: number;  // USDC
  roi: number;            // (returned - wagered) / wagered

  periodStart: Timestamp;
  periodEnd: Timestamp;
  lastUpdated: Timestamp;
}
```

### LLM Reasoning Pipeline (Public vs Private Content)

Michelle's LangGraph nodes produce rich analytical content. Some of it should be public (the analysis), some must stay private (the pricing strategy). The filter is applied at the LangGraph level — content is tagged as it's generated, not filtered on read.

```
┌─────────────────────────────────────────────────────────────┐
│  LangGraph Per-Game Graph                                    │
│                                                              │
│  do_blind_prediction                                         │
│  ├── reasoning: string          → PUBLIC  ("Michelle's Take")│
│  ├── spread/total/ML predictions → PUBLIC  (the analysis)    │
│  └── confidence: number          → PUBLIC  (conviction level)│
│                                                              │
│  do_market_pricing                                           │
│  ├── edgeAssessment: string      → PUBLIC  (market view)     │
│  ├── askOdds: number             → PRIVATE (garage sale low) │
│  └── ceilingOdds: number         → PRIVATE (walk-away price) │
│                                                              │
│  do_price_response                                           │
│  ├── decision: string            → PUBLIC  (accept/decline)  │
│  ├── reasoning: string           → REDACTED (negotiation)    │
│  └── counterOdds: number         → PRIVATE                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ Firebase          │
              │ agentGameEvals    │
              │                   │
              │ publicContent: {  │
              │   reasoning,      │
              │   predictions,    │
              │   edgeAssessment  │
              │ }                 │
              │                   │
              │ privateContent: { │
              │   askOdds,        │
              │   ceilingOdds,    │
              │   counterLogic    │
              │ }                 │
              └──────────────────┘
```

The `gameEvaluationService.ts` (created in M39) already persists evaluations to Firebase. This milestone adds a `visibility` tag at write time so the frontend knows what to render.

### Content Ingestion Pipeline (Pundit Agents)

Generic web content ingestion tool, first use case: pundit NCAAB analysis. Designed to be reusable for any future pundit agent.

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Content Sources │     │  Ingestion Layer │     │  Agent Reasoning │
│                  │     │                  │     │                  │
│ ESPN articles    │     │ Firecrawl or     │     │ LLM extracts:    │
│ Bracket picks    │────►│ similar scraping │────►│ - Game picks     │
│ Video transcripts│     │ tool             │     │ - Confidence     │
│ (stretch)        │     │                  │     │ - Key reasoning  │
│                  │     │ Normalize to     │     │                  │
│                  │     │ structured JSON   │     │ Maps to Ospex    │
│                  │     │                  │     │ contest IDs      │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

**Phase 1 (this milestone):** Build the ingestion tool. Manually trigger it against pundit's published bracket/picks when they drop around Selection Sunday. LLM extracts structured picks. Store in Supabase.

**Phase 2 (future):** Agent autonomously scrapes on schedule, auto-creates positions on the orderbook.

---

## Track 1: Performance Aggregation Service

Build the data layer that powers the agent directory.

**Status: ✅ Complete (2026-02-09)**

### Tasks

- [x] Create `performanceAggregationService.ts` in agent server
  - Query settled speculations where agent has matched positions
  - Join contest data for sportId
  - Map speculationScorer address → bet type (moneyline/spread/total)
  - Compute metrics per (agentId, sportId, betType): wins, losses, pushes, winRate, ROI, totalWagered, totalReturned
  - Upsert to `agentPerformanceMetrics` collection in Firebase
- [x] Create scorer-to-betType mapping utility
  - ~~Map known speculationScorer contract addresses to human-readable bet types~~
  - ~~Keep mapping updatable as new scorer contracts are deployed~~
  - **Deviation:** Reused existing `marketFromScorerAddress()` from `matching-utils.ts` instead of creating new utility
- [x] Wire aggregation into scheduler
  - Run after scorer settles speculations (natural trigger point)
  - Incremental updates: only process newly settled speculations since last run
  - Log agent ID and metrics updated for debugging
  - Added 5-second delay to allow Firestore sync before aggregation
- [x] Create `getAgentPerformanceMetrics()` query function
  - Supports filtering by agentId, sportId, betType
  - Returns sorted results for directory consumption
  - Used by frontend API endpoint (to be added in Track 2)
  - Also added `getAgentOverallMetrics()` and `getAllPerformanceMetrics()`
- [x] Seed initial data
  - Created `scripts/seedPerformanceMetrics.ts` for historical data population
  - Ran against polygon mainnet: 19 positions processed, 9 metrics documents created

### Technical Notes

**Scorer address mapping:** The speculationScorer address on each speculation determines bet type. This is a finite set of known contract addresses that maps cleanly to moneyline/spread/total. Store the mapping in a config file — it changes only when new scorer contracts are deployed.

**90-day TTL on positionEvaluations:** Agent positions in positionEvaluations have a TTL. The aggregation service should process settled positions promptly. The aggregated metrics in `agentPerformanceMetrics` persist indefinitely — they're the historical record.

**Incremental vs full recompute:** Use a `lastProcessedTimestamp` to only process newly settled speculations. Include a manual "full recompute" script for corrections.

---

## Track 2: Agent Directory Page

Read-only showcase of all agents with sortable benchmark results.

**Status: ✅ Complete (2026-02-11)**

### Tasks

- [x] Create `/a` route and page component (deviation: used `/a` instead of `/agents` to match existing short route pattern)
- [x] Design agent card component
  - Agent name, avatar/icon, short description
  - Agent type badge (market maker, bettor, pundit)
  - Overall record and ROI
  - ~~Active/inactive status~~ (not needed - all registered agents are active)
- [x] Implement sortable performance table
  - Columns: Agent, Sport, Bet Type, Record (W-L-P), Win Rate, ROI, Total Wagered
  - Sort by any column (ascending/descending)
  - Filter by sport (NBA, NCAAB, NHL, etc.) and bet type (moneyline, spread, total)
  - Default sort: ROI descending (show the best performers first)
- [x] Create agent detail view (deviation: clicking agent card navigates to existing `/u/:address` profile page)
  - ~~Full performance breakdown by sport and bet type~~ (deferred - profile page shows overall stats)
  - ~~Recent activity (last N positions with outcomes)~~ (profile page already has this)
  - ~~Agent description and metadata (model info for transparency)~~ (profile page shows agent info)
- [x] Populate with live data from `agentPerformanceMetrics`
  - Michelle: market maker type, Sonnet 4 model, full tool access
  - Dan: bettor type, Sonnet 4.5, prompt-only, no tools, "vibes-based"
- [x] Add link to agent directory from main navigation (Agents tab now navigates to `/a`)
- [x] Sub-tabs: Directory (active), Create Agent (placeholder), My Agents (placeholder)

### Design Notes

**The Dan effect:** Dan's consistently bad performance is a feature, not a bug. The directory should make it trivially obvious that Dan loses money and Michelle makes money (or is at least more sophisticated). This is the pitch: "Even a simple AI can play, but the smart ones win."

**Agent type badges:** Market Maker (Michelle), Bettor (Dan), Pundit (coming soon). These communicate different roles in the ecosystem — Michelle provides liquidity, Dan takes positions, pundit translates expert opinion into bets.

**No "create agent" yet:** This page is a showcase. The "create your own agent" workflow is a future milestone. But the page design should leave room for a "Create Agent" CTA that we can activate later.

---

## Track 3: Surface Agent Reasoning

Pipe Michelle's analytical content to the frontend so users can see *why* the market maker is pricing things the way she is.

**Status: 🔧 Backend Complete, Frontend Pending**

### Tasks

- [x] Add `visibility` tagging to `gameEvaluationService.ts`
  - Tag blind prediction reasoning, predictions, confidence as `public`
  - Tag edge assessment (general market view) as `public`
  - Tag askOdds, ceilingOdds as `private`
  - Tag price response reasoning and counter logic as `private`
  - Store tagged content in `agentGameEvaluations` with `publicContent` / `privateContent` split
  - **Implementation:** Added `PublicContent` and `PrivateContent` interfaces, `saveGameEvaluation()` writes split structure, added public query functions: `getPublicGameEvaluation()`, `getPublicEvaluationByContest()`, `getPublicUpcomingEvaluations()`
- [x] Create API endpoints for public evaluations
  - `GET /api/agents/:agentId/evaluations/upcoming` - Upcoming game evaluations
  - `GET /api/agents/:agentId/evaluations/:jsonoddsId` - Evaluation by JSONOdds ID
  - `GET /api/agents/:agentId/evaluations?contestId=xxx` - Evaluation by contest ID
  - **Implementation:** `ospex-agent-server/src/http/evaluations.ts` + routes in `server.ts`
- [x] Create "Michelle's Take" component for market pages
  - Collapsible section on the speculation/market page
  - Shows: Michelle's reasoning, her predicted spread/total/ML, her confidence level, her general market view
  - Does NOT show: her specific pricing (askOdds, ceiling), her negotiation strategy, her edge size
  - Timestamp showing when the evaluation was done
  - Visual indicator if evaluation is stale (>2 hours old or significant line movement)
- [ ] Create agent insight component for Dan's posts (skipping)
  - Dan already posts insights — surface these with his branding
  - Different visual treatment than Michelle (Dan's are more casual/funny, Michelle's are analytical)
- [x] Handle "no evaluation yet" state gracefully
  - If Michelle hasn't evaluated a game, show "Michelle hasn't analyzed this game yet" rather than empty space
  - Optionally: "Request Michelle's analysis" CTA (triggers instant eval, costs 0.20 USDC — ties into instant match flow)

### Content Filtering Rules

| Field | Source Node | Visibility | Rationale |
|-------|-----------|------------|-----------|
| `blindPrediction.reasoning` | do_blind_prediction | **Public** | The analysis — injuries, matchups, trends |
| `blindPrediction.spread/total/ML` | do_blind_prediction | **Public** | Michelle's independent numbers |
| `blindPrediction.confidence` | do_blind_prediction | **Public** | Her conviction level |
| `marketPricing.edgeAssessment` | do_market_pricing | **Public** | "Market has this at X, I see it as Y" |
| `marketPricing.askOdds` | do_market_pricing | **Private** | Her starting offer — reveals strategy |
| `marketPricing.ceilingOdds` | do_market_pricing | **Private** | Her walk-away price — the "garage sale" floor |
| `priceResponse.decision` | do_price_response | **Public** | Accept/decline (already visible via offers) |
| `priceResponse.reasoning` | do_price_response | **Private** | Why she accepted/declined at that price |
| `priceResponse.counterOdds` | do_price_response | **Private** | Counter-offer specifics |

---

## Track 4: Modal UX Overhaul

Address critical usability issues from real user testing (iPhone 16E, general confusion about bet amounts and payouts).

### Tasks

- [x] Fix modal viewport issues on iPhone 16E (and similar smaller screens)
  - Test modal rendering on 390px width (iPhone SE/16E viewport)
  - Ensure modal content doesn't overflow or clip
  - Scrollable modal body if content exceeds viewport height
- [x] Redesign bet confirmation information hierarchy
  - **Most prominent:** "You're betting X USDC to win Y USDC" — this must be the single clearest element
  - **Secondary:** Odds, side, line
  - **Tertiary:** Market details, agent info
  - Users should never be confused about how much they bet or how much they stand to win
- [x] Improve "Create Position" → Modal transition
  - User feedback: the transition from the screen to the modal was confusing
  - Options: inline expansion instead of modal, step indicator, or clearer modal header explaining context
  - Evaluate whether modals are the right pattern or if inline forms would be clearer
- [x] Preserve the good stuff
  - Number line: tester liked it, keep it
  - Confetti: tester liked it, keep it
  - Don't over-engineer — fix the specific issues, don't redesign the whole flow

### Testing

- [x] Test on iPhone 16E viewport (390px width)
- [x] Test on other views where appropriate (Metamask, iPhone SE, Pixel 7, iPad Mini)
- [ ] Test the full flow: browse markets → create position → confirm → see result (modal improved on browse/create, will continue to test)
- [x] Verify bet amount and potential payout are immediately clear to a new user

---

## Track 5: Content Ingestion Pipeline (Pundit Agent)

Build a generic web content ingestion tool. First use case: A pundit's NCAAB tournament analysis.

### Tasks

- [ ] Evaluate and select scraping tool
  - Firecrawl (managed service, good for structured extraction)
  - Cheerio/Puppeteer (self-hosted, more control)
  - Decision criteria: reliability, cost, ease of extracting from ESPN's layout
- [ ] Build content ingestion service (`contentIngestionService.ts`)
  - Input: URL or list of URLs
  - Output: Raw text content, cleaned and structured
  - Handles: articles, bracket pages, potentially video transcript pages
  - Generic — not pundit-specific
- [ ] Build pundit extraction prompt
  - Input: Raw article/bracket content + game matchups
  - Output: Structured picks per game (team, confidence, key reasoning)
- [ ] Create pundit agent configuration
  - Content sources: ESPN bracket, ESPN articles with a specific pundit + "NCAA tournament"
  - Extraction schedule: manual trigger initially, automated in future
  - Maps extracted picks to Ospex contest IDs
- [ ] Store extracted picks in Supabase
  - `punditPicks` table: agentId, contestId, pick, confidence, reasoning, sourceUrl, extractedAt
  - Links to `agentPerformanceMetrics` once games settle
- [ ] Integration test: feed a sample pundit article → extract picks → verify structured output

### Timeline Notes

- Pundit typically publishes bracket around Selection Sunday (March 15)
- Pre-tournament analysis and power rankings start appearing in early March
- **Phase 1 target:** Have ingestion pipeline ready before Selection Sunday so we can process the bracket the moment it drops
- Video transcription is a stretch goal — focus on written content first

### Naming / Legal Considerations

- The agent is clearly a fan-created tool, not an endorsement
- Uses only publicly available content (same as any human reading ESPN)
- No paywalled content — only freely accessible ESPN articles and bracket publications
- If there is an objection, the agent name and content source can be swapped trivially — the pipeline is generic

---

## Track 6: Payment Gating Audit

Quick architectural review of existing payment infrastructure. Not a rebuild — a sanity check.

### Tasks

- [ ] Document current payment flow
  - How does the instant match USDC charge work? (on-chain vs off-chain?)
  - How does the insight posting charge work?
  - What endpoints are involved?
  - Where is the payment verified before the service is provided?
- [ ] Identify gating gaps
  - Can a user hit the instant match endpoint without paying?
  - Can a user post an insight without the USDC charge?
  - Are there timing/race conditions between payment and service delivery?
  - Is the USDC transfer verified on-chain before the action proceeds?
- [ ] Document findings and fix low-hanging fruit
  - If gaps exist that are trivial to fix, fix them in this milestone
  - If gaps require significant work, document them for a future milestone
- [ ] Review pricing model
  - 0.50 USDC for instant match — does this cover the LLM API cost per evaluation?
  - 0.25 USDC for insight posting — is this reasonable for the value provided?
  - Agents post insights for free — verify this path is clean and doesn't accidentally charge

### Cost Modeling

Quick estimate of per-user costs if the site scales:

| Action | Variable Cost | Revenue | Net |
|--------|--------------|---------|-----|
| Instant match | ~$0.05-0.15 LLM call + RPC | $0.50 USDC | Profitable |
| Insight posting | ~$0 (text storage) | $0.25 USDC | Profitable |
| Organic position creation | ~$0 (no LLM, minimal RPC) | Free | Cost-neutral |
| Agent evaluation (cron) | ~$0.05-0.15 per game per cycle | Free (platform cost) | Cost center |
| Firebase reads/writes | Usage-based | — | Monitor |

**The concern:** The cron-triggered agent evaluations (which are free to users) are the scaling cost risk. Michelle evaluating 15 NBA games + 40 NCAAB tournament games 3x daily adds up. Document the expected cost at various scale levels (100 users, 1K users, 10K users).

---

## Stretch Goal: Blind Eval Skip Logic for Re-Evaluated Games

When Michelle re-evaluates a game (e.g., odds shifted significantly), her "blind" prediction is no longer truly blind if prior evaluation context exists in the LangGraph thread state.

**Status: ✅ Complete (2026-02-10)**

### Tasks

- [x] Add `hasBlindPrediction` check at the router node level
  - If blindPrediction already exists in thread state for this game, skip `do_blind_prediction`
  - Route directly to `do_market_pricing` with the existing blind prediction
  - Log that blind eval was skipped: "Re-evaluation of {game} — using existing blind prediction from {timestamp}"
  - **Implementation:** Added `'market_pricing'` to `RouterDecision` type in `gameGraph.ts`, modified `routeByTrigger()` to check if blindPrediction is usable before routing to market_pricing
- [x] Ensure market pricing node handles re-evaluation gracefully
  - Receives existing blind prediction + new/current market odds
  - Generates updated pricing based on potentially different market conditions
  - Previous askOdds/ceilingOdds in state are overwritten with fresh values
  - **Implementation:** No changes needed — `marketPricingNode` already reads blindPrediction from state and uses trigger context for current odds
- [x] Update staleness logic
  - Current: stale evaluation triggers full re-run through blind prediction
  - New: stale evaluation triggers re-run starting at market pricing (blind prediction is a one-time-per-game artifact)
  - Exception: if the game context has changed dramatically (e.g., star player injured after initial eval), we may want a fresh blind eval — but this is a future enhancement
  - **Implementation:** Updated `routeByTrigger()` to route to `'market_pricing'` instead of `'blind_prediction'` when drift or staleness is detected but blindPrediction is usable

### Why This Matters

Blind predictions are the benchmarking foundation. If Michelle re-runs blind prediction with prior context in state, the second prediction is anchored to the first and not truly independent. Skipping it on re-evaluations preserves the integrity of the original blind assessment while still allowing market pricing to update as odds move.

**Savings:** Also avoids a redundant ~30-40 second LLM call with full tool usage on re-evaluations.

---

## Out of Scope (Documented for Future)

| Item | Why Not Now | Likely Milestone |
|------|-------------|------------------|
| "Create your own agent" UI | Directory is read-only first; agent creation needs config system (edge thresholds, model selection, tool access) | M41+ |
| Agent registration system | No external agents yet; current agents are managed internally | M41+ |
| Full mobile optimization pass | Fixing specific viewport issues only; broader mobile UX is later | M41+ |
| Video transcription for pundit agents | Written content first; video adds complexity (transcription API, cost, accuracy) | M41+ |
| Automated pundit content ingestion scheduling | Manual trigger for tournament; scheduled scraping is future | M41+ |
| Agent autonomous betting | Pipeline produces picks; translating to orderbook positions is next step | M41 |
| Live betting support | Fundamentally different product — expiry assumptions don't hold | TBD |
| Social features (following, notifications) | No users yet; premature | M42+ |

---

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| M39 LangGraph migration | ✅ Complete | Provides blind predictions, persistent evaluation state, `gameEvaluationService.ts` |
| `agentGameEvaluations` Firebase collection | ✅ Available | Created in M39; needs `visibility` tagging added |
| Settled speculations data | ✅ Available | `polygonSpeculationsv2.3` with winSide, speculationStatus |
| Scorer address mapping | ✅ Available | Reused existing `marketFromScorerAddress()` in `matching-utils.ts` |
| Firecrawl or scraping tool | To evaluate | For content ingestion pipeline |
| Pundit's published bracket | ~March 15 | External dependency — Selection Sunday |
| ESPN NCAAB content | Available now | Pre-tournament analysis exists; bracket drops later |
| Existing payment infrastructure | ✅ Functional | Needs documentation and audit, not rebuild |

---

## Time Estimate

| Track | Estimate | Notes |
|-------|----------|-------|
| Track 1: Performance aggregation service | 8-12 hours | Data pipeline, scheduler integration, seeding |
| Track 2: Agent directory page | 12-16 hours | New route, card components, sortable table, detail view |
| Track 3: Surface agent reasoning | 8-12 hours | Visibility tagging in LangGraph, "Michelle's Take" component, Dan insights |
| Track 4: Modal UX overhaul | 6-8 hours | Viewport fixes, information hierarchy redesign, transition improvement |
| Track 5: Content ingestion pipeline | 10-14 hours | Scraping tool evaluation, ingestion service, extraction prompt, pundit config |
| Track 6: Payment gating audit | 4-6 hours | Documentation, gap identification, low-hanging fixes, cost modeling |
| Stretch: Blind eval skip logic | 3-5 hours | Router modification, staleness logic update |

**Total:** 48-68 hours across ~5 weeks (Feb 9 – Mar 14)

At current pace (~2-3 hours/day on weekdays, larger blocks on weekends), this is achievable but tight. Tracks 1 and 5 are the critical path — the directory needs data (Track 1), and the pundit pipeline needs to be ready before Selection Sunday (Track 5).

**Suggested sequencing:**
1. Track 1 (aggregation service) — unblocks Track 2
2. Track 3 (reasoning pipeline) — modifies LangGraph nodes, smaller scope
3. Track 2 (directory page) — needs Track 1 data, biggest frontend task
4. Track 4 (modal UX) — independent, can be done in parallel
5. Track 5 (content ingestion) — can develop in parallel, but needs to be ready by Mar 15
6. Track 6 (payment audit) — independent, lowest priority, can slot in anywhere
7. Stretch goal — squeeze in if time allows

---

## Success Criteria

- [x] Agent directory page exists at `/a` with sortable performance data by sport and bet type
- [ ] Performance aggregation service runs on scheduler and produces accurate metrics
- [ ] "Michelle's Take" component displays her public reasoning on market pages
- [ ] Sensitive pricing data (askOdds, ceilingOdds, counter logic) is never exposed on the frontend
- [x] Modals render correctly on iPhone 16E viewport (390px)
- [x] Bet amount and potential payout are the most prominent elements in position confirmation
- [ ] Content ingestion pipeline can process a URL and extract structured picks
- [ ] Pundit agent configuration is ready to process his bracket when it drops (~March 15)
- [ ] Payment gating is documented with any low-hanging gaps fixed
- [ ] Cost model exists showing per-user costs at 100/1K/10K user scales
- [x] Dan's bad performance and Michelle's analysis are both visible and differentiated in the directory (agent cards show ROI with color coding, type badges differentiate roles)
- [x] (Stretch) Re-evaluated games skip blind prediction and go straight to market pricing
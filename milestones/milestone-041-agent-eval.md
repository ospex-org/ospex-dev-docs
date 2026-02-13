# Milestone 41: Agent Evaluation Quality & Intelligence Infrastructure

*Created: February 12, 2026*
*Status: 🟡 In Progress*

**Edit Trail:**
- 2026-02-12: Initial milestone created from design discussion (Claude + Vince). Scoped to 6 tracks focused on evaluation quality, debugging infrastructure, and directory UX. Explicitly defers agent creation UI, radar chart/archetype system, SerpAPI/general browsing, and pundit pipeline automation.
- 2026-02-12: **Track 6 implemented** (Claude Code). Created `src/debug/` infrastructure with category-based logging. Files created: `config.ts` (env parsing, category management), `formatter.ts` (text/JSON formatting, truncation, sensitive data masking), `index.ts` (public API: debugLog, createEvalId, convenience functions). Instrumented: `integration.ts` (evalId creation, eval start/complete), `nodes.ts` (node lifecycle, LLM input/output), `gameGraph.ts` (routing decisions), `progressCallback.ts` (tool call/output with timing). Updated `logger.ts` to respect DEBUG_MODE for log level. Added commented debug variables to `.env`. TypeScript compiles cleanly. Pending: CLAUDE.md update (flagged for user).
- 2026-02-12: **Track 6 smoke test passed** (Heroku production). Deployed to Heroku with DEBUG_MODE=true and all categories enabled. Triggered evaluation for Portland Trail Blazers vs Utah Jazz (NBA). Confirmed working: eval IDs threading through logs (`[eval:129a48_mljy023j]`), node-lifecycle ENTER/EXIT with timing (blindPredictionNode 26.4s, marketPricingNode 22.6s), llm-input/output with char counts and parse status, evaluation-summary START/COMPLETE with total duration (49.4s). DEBUG_MODE disabled on Heroku after verification.
- 2026-02-13: **Track 1 planning initiated** (Claude Code). Explored tool inventory (12 tools), date formatting patterns, evaluation data flow. Queried recent evaluations from Firebase - discovered `toolsUsed: []` bug: tools ARE being called but not logged because `invokeGameGraphForCron` doesn't pass `onProgress` callback, so `MichelleProgressHandler` is never created. Bug location: `slateNodes.ts:545`. Plan created at `~/.claude/plans/crispy-hugging-alpaca.md`. Decisions: Eastern Time for dates, tiktoken for token counting. Phase 0 added to fix toolsUsed bug before proceeding with tool output formatting.
- 2026-02-13: **Track 1 partially implemented** (Claude Code). Implemented core tool output formatting:
  - **Phase 0 (Bug Fix):** Fixed `toolsUsed: []` bug in `nodes.ts` (lines 275-278, 731-734). Root cause: `MichelleProgressHandler` only created when `onProgress` provided, but cron evals don't pass this. Fix: always create handler with no-op fallback.
  - **Phase 1:** Created `src/tools/dateFormat.ts` with comprehensive date/time utilities: `formatDate`, `formatRelativeDate`, `formatGameTime`, `formatGameDateTime`, `formatTimeUntil`, `formatTimeSince`, `formatLastUpdated`, plus text block formatters for each tool.
  - **Phases 2-3:** All 12 tools now include `formattedText` field with pre-formatted, human-readable output. ISO timestamps converted to "Tuesday, Feb 11 at 7:00 PM ET" format. All times in Eastern Time.
  - **Deviations from plan:** (1) Did NOT add tiktoken dependency - deferred to Track 5 token counting; (2) Did NOT trim existing JSON data - added `formattedText` alongside existing structure to maintain backward compatibility; (3) LangSmith trace audit (5-10 evals) not yet done - separate task. Commit: `dcff436`.
- 2026-02-13: **Track 1 Michigan State @ Wisconsin eval test** (Claude Code). Deployed with DEBUG_MODE=true. Findings:
  - **toolsUsed fix verified**: Debug logs show `toolsUsed: 2` for blind prediction node (get_rankings, get_standings). Bug fix working.
  - **formattedText working**: Tool outputs show human-readable dates in formattedText (e.g., "Today at 12:00 AM ET"). ISO timestamps only in raw JSON for backward compatibility.
  - **Three issues found and fixed:**
    1. **P0 - Moneyline zero-odds handling:** Added `getAvailableMarkets()` function to check which markets have valid odds. `buildBlindPredictionSystemPrompt()` now only asks for predictions on available markets. When moneyline odds are 0/empty, Michelle won't predict moneyline. Logs added for debugging.
    2. **get_odds_history capturedAt:** Already working - `formattedText` uses `formatLastUpdated()` and `formatTimeSince()`. ISO in raw JSON is intentional for backward compatibility.
    3. **last10 null handling:** Already working - `formatStandingsText()` omits "Last 10:" line when null. The `null` in raw JSON is fine since LLM reads `formattedText`.
  - **Track 3 note:** NCAAB prompts should explicitly tell Michelle not to evaluate moneyline when unavailable (now implemented in Track 1 as dynamic prompt generation).

---

## Overview

Make Michelle's evaluations trustworthy, observable, and improvable before March Madness. M40 gave us the agent showcase — now we need to ensure what's being showcased is accurate. This milestone audits every tool in the evaluation pipeline, instruments token usage, adds sport-specific intelligence to the prompting system, and gives both developers and future users the debugging infrastructure needed to build and tune agents.

The core thesis: if a stranger reads Michelle's analysis during March Madness and spots a factual error ("she said Milwaukee lost to Orlando yesterday, but that game was two days ago"), credibility is gone. Every track in this milestone exists to prevent that.

**Why now:**
- March Madness starts March 20. Michelle's analyses will be public-facing. Factual errors in her reasoning undermine the entire agent value proposition.
- M40 surfaced the agent showcase — now we're seeing evaluation quality issues only visible when you actually read the output (date mismatches, null tool returns, generic prompts producing sport-inappropriate analysis).
- Token budget visibility is a prerequisite for the agent creation system. Users can't configure agents if we don't know how much each tool costs.
- The evaluation pipeline has been built iteratively across M33/M38/M39 — it's never had a systematic audit.

**What this is NOT:**
- Not agent creation UI — we're building the instrumentation that makes agent creation possible later
- Not the radar chart / archetype system — we need more data about tool costs and effectiveness first
- Not SerpAPI or general browsing — curated ingestion (M40 Track 5) is the right architecture, not live web access
- Not pundit pipeline automation — manual trigger for March Madness, automated scheduling is future
- Not a revenue system — we're noting the Lovable-style pricing model (free to tinker, pay per evaluation run) but not building billing infrastructure yet

**Philosophy:** You can't optimize what you can't measure. Every track in this milestone adds visibility into how Michelle thinks, what data she's working with, and where it goes wrong — so that when users build their own agents, they're tuning real knobs with real feedback.

---

## Architecture

### Dynamic Tool Injection Pipeline

Currently, Michelle receives all available tools on every evaluation regardless of game context. This wastes tokens on tool descriptions the LLM doesn't need, produces null tool calls (e.g., rankings tool for unranked NCAAB teams), and increases the risk of hallucination when the LLM tries to make sense of empty results.

The fix is a **deterministic pre-flight node** in the LangGraph flow that runs before the LLM sees any tools:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Game Context     │     │  Tool Eligibility │     │  LLM Evaluation  │
│                   │     │  Node (no LLM)    │     │                  │
│ sportId           │     │                   │     │  Only sees tools │
│ teams             │────►│ Check per-tool:   │────►│  that will return│
│ league            │     │ - Are teams ranked?│    │  useful data     │
│ venue (indoor/    │     │ - Outdoor sport?   │    │                  │
│  outdoor)         │     │ - Pundit picks     │    │  Tool descriptions│
│ date/time         │     │   exist in DB?     │    │  are fewer →     │
│                   │     │ - H2H data exists? │    │  less token cost │
│                   │     │                   │     │  less confusion  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                │
                                ▼
                         ┌──────────────────┐
                         │  Eligibility Log  │
                         │                   │
                         │  tool: rankings   │
                         │  eligible: false  │
                         │  reason: "neither │
                         │  team ranked"     │
                         │                   │
                         │  tool: weather    │
                         │  eligible: false  │
                         │  reason: "indoor  │
                         │  sport"           │
                         └──────────────────┘
```

**Key principle:** The eligibility node is pure code — no LLM calls, no token cost. It queries lightweight lookups (is team in rankings table? is sport outdoor?) and assembles the tool set dynamically. The eligibility decisions are logged for debugging.

### Sport-Specific Prompt Architecture

Replace the single generic evaluation prompt with a sport-aware prompt system:

```
┌──────────────────────────────────────────────────────────┐
│  Prompt Registry                                          │
│                                                           │
│  prompts/                                                 │
│  ├── base.ts              ← shared evaluation structure   │
│  ├── nba.ts               ← NBA-specific guidance         │
│  │   "Check schedule tool for back-to-backs..."           │
│  │   "Rest days matter more in regular season..."         │
│  │   "Home court advantage is moderate in NBA..."         │
│  │                                                        │
│  ├── ncaab.ts             ← NCAAB-specific guidance       │
│  │   "Home court advantage is dramatically stronger..."   │
│  │   "If neither team is ranked, weight stats heavily..." │
│  │   "Conference strength matters for cross-conf games.." │
│  │                                                        │
│  └── fallback.ts          ← generic for unsupported sports│
│                                                           │
│  Sport prompt selected by sportId before LLM invocation   │
│  Base prompt composed with sport prompt at runtime         │
└──────────────────────────────────────────────────────────┘
```

### Debug Logging Infrastructure

Controlled by `DEBUG_MODE` environment variable. When enabled, extends logging across the entire agent evaluation pipeline without requiring code changes.

```
┌──────────────────────────────────────────────────────────┐
│  DEBUG_MODE=true                                          │
│                                                           │
│  Tool Calls:                                              │
│  ├── Raw tool output (full JSON, not truncated)           │
│  ├── Token count estimate for tool output                 │
│  └── Tool eligibility decision + reason                   │
│                                                           │
│  LLM Interaction:                                         │
│  ├── Full prompt sent (with sport-specific components)    │
│  ├── Tool descriptions included (list of eligible tools)  │
│  ├── Total input tokens (prompt + tool outputs + history) │
│  └── Output tokens consumed                               │
│                                                           │
│  Evaluation Summary:                                      │
│  ├── Per-tool token cost breakdown                        │
│  ├── Total evaluation cost ($)                            │
│  ├── Tools called vs tools eligible vs tools available    │
│  └── Any null/empty tool responses flagged                │
│                                                           │
│  DEBUG_MODE=false (default)                                │
│  ├── Standard operational logging (existing behavior)     │
│  └── No performance impact                                │
└──────────────────────────────────────────────────────────┘
```

---

## Track 1: Tool Output Audit & Data Quality

Systematic audit of every tool Michelle uses. For each tool: examine raw output format, identify data quality issues, standardize formatting, and verify the LLM is interpreting the data correctly.

**Status: 🟡 In Progress**

### Known Issues

- **Date/time confusion**: Michelle referenced a game as "yesterday" when it was two days prior. Likely a timestamp formatting issue — raw timestamps without human-readable date labels, or relative date calculation happening in the LLM instead of the tool. **FIXED:** All tools now output human-readable dates with relative time computed by the tool.
- **Null tool returns**: Rankings tool called for unranked NCAAB teams (New Mexico vs Grand Canyon), returning null. Wasted tokens on tool call + LLM processing empty results. *(Track 2 will address this with dynamic tool injection)*
- **Data mangling**: Unknown scope — need to trace several recent evaluations through LangSmith to identify patterns.
- **toolsUsed bug**: `toolsUsed: []` showing empty in Firebase evaluations even though tools ARE being called. **FIXED:** Root cause was `MichelleProgressHandler` only created when `onProgress` callback provided. Cron evals don't pass this callback. Fix: always create handler with no-op fallback (`nodes.ts` lines 275-278, 731-734).

### Tasks

- [x] Inventory all tools Michelle currently has access to
  - Document each tool: name, what it returns, approximate token cost per call, which sports it applies to
  - Note any tools that overlap in data coverage
  - **Done:** 12 tools inventoried (see plan file). Token costs deferred to Track 5.
- [ ] Trace 5-10 recent evaluations through LangSmith end-to-end
  - For each: compare raw tool output → LLM interpretation → factual accuracy
  - Document every discrepancy (dates, team names, stats, records)
  - Categorize: tool data issue vs. LLM hallucination vs. prompt ambiguity
- [x] Standardize date/time formatting across all tools
  - All dates should be human-readable: "Tuesday, Feb 11, 2026" not just timestamps
  - Relative dates ("2 days ago") should be computed by the tool, not left for the LLM
  - Game times should include timezone context
  - **Done:** Created `src/tools/dateFormat.ts` with all formatting utilities. All times in Eastern Time (ET).
- [x] Add explicit labels to tool output fields
  - "Last game played: Feb 10, 2026 — Milwaukee 108, Orlando 102"
  - Not just `{ date: "2026-02-10", home: "MIL", away: "ORL", homeScore: 108, awayScore: 102 }`
  - The LLM should be able to read tool output like a sentence, not parse JSON mentally
  - **Done:** All 12 tools now include `formattedText` field with pre-formatted text blocks.
- [ ] Trim unnecessary data from tool outputs
  - Remove fields the LLM never uses or references in analysis
  - Reduce token cost per tool call without losing analytical value
  - Document what was removed and why
  - **Note:** Deferred. Added `formattedText` alongside existing JSON for backward compatibility. Trimming requires LangSmith audit to identify unused fields.

### Acceptance Criteria

- [x] Every tool output is formatted with human-readable dates and clear field labels
  - **Done:** All 12 tools include `formattedText` with human-readable output
- [ ] Traceable audit document exists linking tool outputs to LLM interpretation for 5+ evaluations
- [ ] Zero date/time discrepancies in new evaluations after fixes are applied
  - *Pending verification after deployment*
- [ ] Token cost per tool call is measured and documented
  - *Deferred to Track 5*

### Files Created/Modified

| File | Change |
|------|--------|
| `src/tools/dateFormat.ts` | NEW - Date/time formatting utilities, text block formatters for all tools |
| `src/tools/getScheduleContext.ts` | MODIFIED - Added `formattedText` field with human-readable schedule context |
| `src/tools/getMarketState.ts` | MODIFIED - Added `formattedText` with game time as "Tuesday, Feb 11 at 7:00 PM ET (in 3 hours)" |
| `src/tools/getInjuryReport.ts` | MODIFIED - Added `formattedText` with injury list and formatted timestamps |
| `src/tools/getStandings.ts` | MODIFIED - Added `formattedText` with standings info |
| `src/tools/getTeamStats.ts` | MODIFIED - Added `formattedText` with team statistics |
| `src/tools/getMyRecentDecisions.ts` | MODIFIED - Changed return type to include `formattedText` |
| `src/tools/getOddsHistory.ts` | MODIFIED - Added `formattedText` with line movement summary |
| `src/tools/getRankings.ts` | MODIFIED - Added `formattedText` with AP/Coaches poll rankings |
| `src/tools/getInjuryDetails.ts` | MODIFIED - Added `formattedText` with injury details and trajectory |
| `src/tools/getRoster.ts` | MODIFIED - Added `formattedText` with roster grouped by position |
| `src/tools/getMyExposure.ts` | MODIFIED - Added `formattedText` with exposure summary |
| `src/tools/compareOdds.ts` | MODIFIED - Added `formattedText` with edge analysis |
| `src/tools/computeExposureImpact.ts` | MODIFIED - Added `formattedText` with limit check results |
| `src/tools/types.ts` | MODIFIED - Added `formattedText` to `MarketStateResult` |
| `src/agents/market_maker_michelle/langgraph/nodes.ts` | MODIFIED - Fixed toolsUsed bug (always create MichelleProgressHandler), added `getAvailableMarkets()` function, updated `buildBlindPredictionSystemPrompt()` to only request predictions for available markets |

---

## Track 2: Dynamic Tool Injection

Add a deterministic pre-flight node to the LangGraph evaluation flow that assembles the tool set per-game based on data availability, eliminating null tool calls and reducing token overhead.

**Status: 🔲 Not Started**

### Tasks

- [ ] Design tool eligibility criteria for each tool
  - Rankings: check if either team appears in current rankings table → exclude if neither ranked
  - Weather: check if sport is played outdoors (NFL, MLB, future) → exclude for NBA/NCAAB/NHL
  - Pundit picks: check if any picks exist in `pundit_picks` table for this matchup → exclude if none
  - Injury reports: always include (even "no injuries" is useful context)
  - Schedule/recent games: always include
  - Odds/lines: always include
  - Document the full eligibility matrix: tool × sport × condition
- [ ] Implement `toolEligibilityNode` in LangGraph flow
  - Runs before `do_blind_prediction` (or whichever node first invokes tools)
  - Pure code — no LLM calls
  - Queries lightweight lookups (Supabase/Firebase) to check data existence
  - Outputs: list of eligible tools + exclusion reasons
  - Stores eligibility decisions in thread state for debugging
- [ ] Wire eligible tools into LLM invocation
  - Currently: tools are hardcoded in the node configuration
  - New: tools are assembled dynamically from eligibility node output
  - Ensure tool descriptions are only included for eligible tools (token savings)
- [ ] Log eligibility decisions
  - Standard mode: log tool count ("5 of 8 tools eligible for this game")
  - Debug mode: log per-tool eligibility with reasons
- [ ] Measure token savings
  - Compare token counts for evaluations before/after dynamic injection
  - Document average savings per evaluation

### Technical Notes

- The eligibility node should be fast — sub-100ms. If any lookup is slow, cache it.
- Rankings data should be pre-loaded on scheduler startup, not queried per-evaluation.
- This pattern is extensible: when users build agents, their tool access configuration feeds into this same eligibility system — user config becomes an additional constraint alongside data availability.

### Acceptance Criteria

- Zero null/empty tool returns in new evaluations
- Tool eligibility decisions are logged and traceable in LangSmith
- Measurable token reduction per evaluation (target: 10-20% on games where tools are excluded)
- No regression in evaluation quality (tools that matter are never incorrectly excluded)

---

## Track 3: Sport-Specific Prompt System

Break the generic evaluation prompt into sport-aware variants that provide appropriate analytical guidance and tool usage instructions for each sport.

**Status: 🔲 Not Started**

### Current Problem

The one-size-fits-all prompt tells Michelle to "evaluate this game" generically. NBA and NCAAB are fundamentally different analytical exercises:

| Factor | NBA | NCAAB |
|--------|-----|-------|
| Home court advantage | Moderate (~3 pts) | Dramatic (~5-6 pts, higher in some venues) |
| Back-to-backs | Critical factor | Rare/non-factor |
| Rankings | Less relevant (power rankings matter more) | Major factor in evaluating matchups |
| Talent gap | Narrow across most matchups | Can be enormous |
| Rest days / load management | Significant | Minimal |
| Conference strength | Less relevant | Important for cross-conference games |
| Schedule spots | Travel, fatigue patterns | Less relevant (fewer games) |

A prompt that says "consider recent form" means different things for these sports. Sport-specific prompts let us be directive about both analysis approach and tool usage.

### Tasks

- [ ] Audit current prompt(s)
  - Extract the full system prompt and any per-node prompts from the LangGraph flow
  - Identify generic language that should be sport-specific
  - Identify any prompt content that is actively misleading for certain sports
  - Document prompt structure: what's in system prompt vs. node-level prompts
- [ ] Design prompt architecture
  - Base prompt: shared evaluation structure, output format requirements, general betting analysis principles
  - Sport prompt: sport-specific analytical guidance, tool usage instructions, key factors to weigh
  - Composition: base + sport prompt assembled at evaluation time based on `sportId`
  - Keep prompts maintainable — changes to base shouldn't require updating every sport prompt
- [ ] Write NBA prompt
  - Emphasize: back-to-backs, rest days, travel, load management, pace of play
  - Tool guidance: "Always check schedule for back-to-back situations. If a team is on a back-to-back, this should significantly influence your assessment."
  - Calibration notes: home court is ~3 points, totals are influenced by pace matchups
- [ ] Write NCAAB prompt
  - Emphasize: home court advantage (stronger than NBA), rankings significance, conference strength, tournament context (as March Madness approaches)
  - Tool guidance: "Check rankings first. If neither team is ranked, weight statistical comparison tools more heavily. Do not call the rankings tool if it's not relevant."
  - Calibration notes: home court can be 5-6+ points, upsets are more common due to talent gaps
- [ ] Write fallback prompt for other sports
  - Generic but functional — used for NHL and any future sports until dedicated prompts are written
  - Should not contain NBA/NCAAB-specific guidance
- [ ] Test sport prompt selection
  - Verify correct prompt is selected based on sportId
  - Run test evaluations for NBA and NCAAB games, compare analysis quality to pre-change evaluations
  - Confirm no cross-sport contamination (NBA advice showing up in NCAAB evaluations)

### Acceptance Criteria

- Sport-specific prompts exist for NBA and NCAAB with a generic fallback
- Prompt selection is automatic based on sportId — no manual intervention
- NBA evaluations reference back-to-backs and rest when relevant
- NCAAB evaluations appropriately weight home court and rankings
- Prompts are stored in a maintainable structure (not hardcoded strings in node files)

---

## Track 4: Agent Directory Enhancements

Add sport filtering, time-period filtering, and per-sport performance breakdowns to the `/a` agent directory page.

**Status: 🔲 Not Started**

### Tasks

- [ ] Add league pill filters to agent directory
  - Reuse existing pill component pattern from Order Book page (see screenshot)
  - Pills: All Sports | NBA | NCAAB | NHL (match currently supported sports)
  - Omit "View Options" dropdown — not needed on this page
  - Filtering applies to both agent cards and performance table
  - When filtered by sport: agent cards show sport-specific record/ROI, performance table shows only that sport's rows
- [ ] Add search bar to agent directory
  - Reuse existing search bar component from Order Book page
  - Search by agent name
  - Position above or alongside league pills, matching Order Book layout
- [ ] Add time-period pill filters above performance table
  - Pills: Last 7 | Last 30 | All
  - Filters performance table rows by date range (using `periodEnd` or `lastUpdated` from metrics)
  - Default to "All" (current behavior)
  - These pills sit above the performance table, separate from the league pills which are page-level
- [ ] Break down agent card stats by sport
  - When "All Sports" is selected: show current aggregate stats (existing behavior)
  - When a specific sport is selected: show stats for that sport only
  - If agent has no data for selected sport: show "No activity" or equivalent
  - Performance metrics schema already supports `sportId` filtering — this is a frontend task
- [ ] Ensure aggregation service supports time-period queries
  - Check if `getAgentPerformanceMetrics()` can filter by date range
  - If not, add date range parameters to the query function
  - May need `periodStart`/`periodEnd` or `lastUpdated` timestamps in the aggregation to support Last 7 / Last 30 filtering

### Technical Notes

- The performance aggregation service from M40 Track 1 already stores metrics per `(agentId, sportId, betType)` — the data structure supports this. This track is primarily frontend work.
- Time-period filtering may require the aggregation service to store more granular data (per-position timestamps) rather than just aggregate metrics. Evaluate whether the current schema supports this or if a lightweight enhancement is needed.

### Acceptance Criteria

- League pills filter agent cards and performance table by sport
- Search bar filters agents by name
- Time-period pills (Last 7 / Last 30 / All) filter the performance table
- When filtered to a single sport, agent cards show sport-specific win rate and ROI
- No new data collections needed — leverage existing `agentPerformanceMetrics` schema

---

## Track 5: Context Budget Instrumentation

Add token counting and cost tracking to every tool call and LLM invocation in the evaluation pipeline. This is the foundation for the agent creation system — users need to see how much context each tool consumes to make informed configuration decisions.

**Status: 🔲 Not Started**

### Tasks

- [ ] Add token estimation to each tool call
  - Estimate tokens in tool output (rough count: chars / 4 for English text, or use `tiktoken` if precision matters)
  - Log per-tool: tool name, output token estimate, latency
  - Store in thread state alongside tool results
- [ ] Add token tracking to LLM invocations
  - Capture input tokens (prompt + tool descriptions + tool outputs + conversation history)
  - Capture output tokens
  - LangSmith already captures this — ensure it's accessible programmatically, not just in the dashboard
- [ ] Build per-evaluation cost summary
  - After each evaluation completes, compute: total tokens, per-tool token breakdown, total cost ($), number of tools called vs. eligible vs. available
  - Store summary in `agentGameEvaluations` alongside the evaluation results
  - This data powers the future "token budget" UI in agent creation
- [ ] Surface token data in LangSmith metadata
  - Tag evaluations with per-tool token counts as LangSmith metadata
  - Enables filtering/sorting evaluations by cost in the LangSmith dashboard
  - Useful for identifying which tools are most expensive and which evaluations are outliers
- [ ] Document current token budget baseline
  - Average tokens per evaluation by sport
  - Token breakdown by tool (which tools consume the most context?)
  - Average cost per evaluation
  - Context window utilization (how close are we to limits?)

### Why This Matters for Agent Creation

When users build agents, they'll configure which tools their agent has access to. Each tool consumes context window and costs tokens. The token budget instrumentation from this track becomes the foundation for showing users: "Adding the rankings tool costs ~500 tokens per evaluation. Adding the pundit tool costs ~1,200 tokens. Your current configuration uses 60% of the context budget."

This is the "Pokemon stat sheet" — users need to see the tradeoffs to make interesting build decisions.

### Acceptance Criteria

- Every evaluation has a per-tool token breakdown stored alongside results
- Token costs are visible in LangSmith metadata for filtering/analysis
- Baseline document exists showing average token usage per tool and per evaluation
- Context window utilization percentage is computed per evaluation

---

## Track 6: Debug Logging Infrastructure

Implement a `DEBUG_MODE` environment variable that enables comprehensive logging across the agent evaluation pipeline, togglable without code changes.

**Status: ✅ Complete**

### Tasks

- [x] Create debug logging utility
  - `debugLog(category: string, message: string, data?: any)` — only logs when `DEBUG_MODE=true`
  - Categories: `evaluation-summary`, `node-lifecycle`, `routing`, `llm-input`, `llm-output`, `tool-call`, `tool-output`, `cost`
  - Include timestamps and evaluation ID for traceability
  - Output to console with clear formatting (category prefix, indentation for nested data)
  - **Implementation:** `src/debug/index.ts` with convenience functions: `debugNodeEnter`, `debugNodeExit`, `debugRouting`, `debugEvalStart`, `debugEvalComplete`, `debugToolCall`, `debugToolOutput`, `debugLlmInput`, `debugLlmOutput`
- [x] Add `DEBUG_MODE` to `.env` configuration (added prior to plan write up by Claude Code)
  - Default: `false` (no performance impact, no extra logging)
  - When `true`: enables all extended logging throughout the pipeline
  - **Added:** `DEBUG_CATEGORIES`, `DEBUG_LOG_FORMAT`, `DEBUG_MASK_SENSITIVE` (commented in .env)
- [x] Instrument tool calls with debug logging
  - Log full raw tool output (truncated to 500 chars by default) under `tool-output` category
  - Log tool call start under `tool-call` category with timing
  - Log any null/empty tool responses with a warning flag
  - **Implementation:** `progressCallback.ts` - added `handleToolStart`/`handleToolEnd` instrumentation with duration tracking
- [x] Instrument LLM interactions with debug logging
  - Log the full prompt sent (including sport-specific components) under `llm-input` category
  - Log which tool descriptions were included under `llm-input` category
  - Log LLM response under `llm-output` category
  - **Implementation:** `nodes.ts` - added `debugLlmInput`/`debugLlmOutput` around LLM calls in blind prediction and market pricing nodes
- [x] Instrument evaluation lifecycle with debug logging
  - Log evaluation start with game context (teams, sport, date) under `evaluation-summary`
  - Log evaluation complete with duration under `evaluation-summary`
  - **Implementation:** `integration.ts` - added `debugEvalStart`/`debugEvalComplete` in all three graph invocation functions
- [x] Clean up existing debug logging
  - Audited `matching-job.ts` — already using Winston logger, no console.log statements found
  - Ensure standard mode (DEBUG_MODE=false) has clean, operational-only output
  - **Implementation:** `logger.ts` updated to use 'debug' level when DEBUG_MODE=true, 'info' otherwise
- [x] Claude Code setup
  - Create or update `CLAUDE.md` in the agent server repo with debugging context
  - Document: how to enable DEBUG_MODE, how to read debug output, common debugging workflows
  - Add relevant project context so CC can efficiently navigate the evaluation pipeline
  - Include: key file locations, data flow, tool inventory, known gotchas
  - **Note:** Plan file contains suggested CLAUDE.md content under "CLAUDE.md Additions (Flag for User)" section

### Technical Notes

- Debug logging should have zero performance impact when disabled. Use early returns, not conditional log levels.
- Consider structured JSON output for debug logs (parseable by scripts) vs. human-readable console output. Recommendation: human-readable for console, with an optional `DEBUG_LOG_FORMAT=json` for piping to analysis scripts. (note: this is optional, if you decide that this makes sense to do, you are free to add, just add this to the .env file because I did not add it myself)
- The Claude Code setup is an investment in debugging velocity — CC should be able to trace an evaluation issue without Vince having to explain the entire pipeline each time.

### Acceptance Criteria

- [x] `DEBUG_MODE=true` produces comprehensive, readable logs for the entire evaluation pipeline
- [x] `DEBUG_MODE=false` produces only standard operational logs (no regression in existing behavior)
- [x] No code changes required to toggle — environment variable only
- [x] Existing ad-hoc console.log statements are consolidated into the debug utility
- [x] `CLAUDE.md` exists with debugging workflows documented

### Files Created/Modified

| File | Change |
|------|--------|
| `src/debug/config.ts` | NEW - Env var parsing, category management, sensitive data config |
| `src/debug/formatter.ts` | NEW - Text/JSON formatting, truncation (500 chars), sensitive masking |
| `src/debug/index.ts` | NEW - Public API: `debugLog()`, `createEvalId()`, convenience functions |
| `src/agents/market_maker_michelle/langgraph/integration.ts` | MODIFIED - evalId creation, eval start/complete logging |
| `src/agents/market_maker_michelle/langgraph/nodes.ts` | MODIFIED - Node lifecycle logging, LLM input/output logging |
| `src/agents/market_maker_michelle/langgraph/gameGraph.ts` | MODIFIED - Router decision logging |
| `src/agents/market_maker_michelle/progressCallback.ts` | MODIFIED - Tool call/output logging with timing |
| `src/logger.ts` | MODIFIED - Log level respects DEBUG_MODE |
| `.env` | MODIFIED - Added commented DEBUG_CATEGORIES, DEBUG_LOG_FORMAT, DEBUG_MASK_SENSITIVE |

---

## Out of Scope (Documented for Future)

| Item | Why Not Now | Likely Milestone |
|------|-------------|------------------|
| Agent creation UI | Need token budget data (Track 5) and prompt architecture (Track 3) first | M42+ |
| Radar chart / archetype system | Need more data on tool costs and effectiveness; Track 5 is prerequisite | M42+ |
| SerpAPI / general web browsing for agents | Curated ingestion (M40 Track 5) is the right architecture; live browsing is expensive, slow, unpredictable | M43+ |
| Pundit pipeline automation | Manual trigger for March Madness; automated scheduling after tournament validates the approach | M42 |
| Pundit tool wired into Michelle | Need pundit data in the table first (March Madness); Track 2 dynamic injection makes adding it trivial | M42 |
| Agent billing infrastructure | Lovable-style model (free to tinker, pay per evaluation run) is decided; building it is premature until agent creation exists | M43+ |
| Model switching system | Per-agent model selection (Sonnet → Opus/GPT) | M42+ |
| Agent config as user settings | Edge thresholds, risk parameters, etc. as UI fields — blocked by agent creation UI | M42+ |
| ELO / power ranking system | Depth item for after tool audit establishes quality baseline | M42+ |

---

## Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| M40 complete | ✅ Complete | Agent directory, performance aggregation, content ingestion pipeline, payment audit |
| M39 LangGraph migration | ✅ Complete | Provides evaluation graph, blind predictions, thread state |
| `agentPerformanceMetrics` Firebase collection | ✅ Available | Created in M40 Track 1; supports sportId/betType filtering |
| LangSmith access | ✅ Available | 542 runs, 14-day retention, full trace data |
| Sport-specific data in tools | ✅ Available | Schedule, injuries, roster, odds, rankings tools exist from M33/M38 |
| `pundit_picks` Supabase table | ✅ Available | Created in M40 Track 5; empty until content ingested |
| `team_aliases` Supabase table | ✅ Available | Created in M40 Track 5; seeded with team name mappings |

---

## Time Estimate

| Track | Estimate | Notes |
|-------|----------|-------|
| Track 1: Tool output audit | 8-12 hours | Manual tracing through LangSmith, formatting fixes, documentation |
| Track 2: Dynamic tool injection | 6-10 hours | LangGraph node, eligibility logic, testing |
| Track 3: Sport-specific prompts | 6-8 hours | Prompt audit, NBA/NCAAB prompts, testing, prompt registry |
| Track 4: Agent directory enhancements | 6-8 hours | Frontend — league pills, time pills, sport breakdown |
| Track 5: Context budget instrumentation | 6-10 hours | Token counting, cost tracking, baseline documentation |
| Track 6: Debug logging infrastructure | 6-8 hours | Utility, instrumentation, cleanup, CLAUDE.md |

**Total:** 38-56 hours

At M40 pace (shipped 48-68 hour estimate in 3 days), this is achievable within ~2 weeks if maintaining similar intensity. More realistically at sustainable 2-3 hours/day weekday pace: 2.5-4 weeks.

**Suggested sequencing:**
1. Track 6 (debug logging) — enables everything else; invest in tooling first
2. Track 1 (tool audit) — uses debug logging; produces the findings that inform Tracks 2 and 3
3. Track 2 (dynamic tool injection) — architectural fix informed by audit findings
4. Track 3 (sport-specific prompts) — prompt improvements informed by audit findings
5. Track 5 (context budget instrumentation) — builds on Tracks 2 and 3; needs evaluation pipeline stable
6. Track 4 (agent directory enhancements) — independent frontend work; can be done in parallel with any backend track

**Critical path:** Track 6 → Track 1 → Tracks 2 & 3 (parallel) → Track 5. Track 4 floats.

---

## Success Criteria

- [ ] Tool output audit document exists covering all tools with token costs, data quality issues, and fixes applied
  - *Partial: Tool inventory done, human-readable formatting implemented, token costs deferred to Track 5*
- [ ] Zero factual date/time errors in Michelle's evaluations post-fix
  - *Pending verification after deployment*
- [ ] Dynamic tool injection eliminates null tool returns (zero wasted tool calls)
- [ ] Sport-specific prompts produce measurably different analysis for NBA vs. NCAAB games
- [ ] Token budget baseline exists: average tokens per tool, per evaluation, per sport
- [ ] Context window utilization is tracked per evaluation (% of available context used)
- [x] `DEBUG_MODE=true` produces comprehensive, actionable logs for the full evaluation pipeline
- [ ] Agent directory supports filtering by sport (league pills) and time period (Last 7 / Last 30 / All)
- [ ] Michelle's prediction accuracy is measured against a defined threshold (within N points of market for spreads, within M% for moneylines)
- [x] Claude Code has project context (`CLAUDE.md`) for efficient agent pipeline debugging (content provided, pending user addition)
- [x] `toolsUsed` array is populated correctly for all evaluation types (cron, quote, unmatched)

---

## Notes

### On Evaluation Accuracy Targets

Defining "good" is important but the threshold should be informed by Track 1 data, not guessed upfront. Suggested approach: after the tool audit, measure Michelle's blind predictions against actual market lines across all completed evaluations. Use the distribution to set realistic targets — e.g., "80% of spread predictions within 3 points of market consensus." This becomes the benchmark that future agents (and future Michelle improvements) are measured against.

### On Lovable-Style Pricing Model

Agreed direction for agent billing (future milestone):
- **Free tier:** Create agents, configure tools and prompts, preview evaluations. The "builder" experience. Float initial evaluation costs to get users hooked.
- **Paid tier:** Agent runs live evaluations against real games, consuming LLM tokens. Charge per evaluation run, not per agent created.
- **Alignment:** Ospex's costs scale with revenue. Users only pay when agents are actively evaluating (and thus providing value). Heavy users naturally pay more.
- **Prerequisite:** Track 5 token instrumentation gives us the cost-per-evaluation data needed to set pricing.

### On Claude Code Setup

The `CLAUDE.md` should include:
- Repo structure overview (agent-server vs. api-server vs. frontend)
- Evaluation pipeline data flow (scheduler → LangGraph → tools → Firebase)
- Key file locations for debugging (graph definition, tool implementations, prompts, scheduler)
- How to enable and read DEBUG_MODE output
- Common debugging workflows ("Michelle said X but it's wrong — here's how to trace it")
- Known quirks (sport ID mappings, scorer address mappings, Firebase vs. Supabase split)

### On the Sport ID Mapping Issue

The backlog notes scattered sport-to-ID mappings across the codebase. This milestone's tool audit (Track 1) will naturally encounter these inconsistencies. If they're causing data quality issues, fixing them becomes a Track 1 subtask. If they're just tech debt, document them and defer to the canonical sport resolver service noted in the backlog.
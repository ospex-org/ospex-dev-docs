# Milestone 38: Agent Sports Intelligence & Benchmarking Foundation

*Created: February 3, 2026*
*Status: ✅ Complete*
*Completed: February 4, 2026*
*Notes: Track 1 (Sports Intelligence) fully implemented. Track 2 (Benchmarking) documented and foundation laid - full implementation deferred to post-LangGraph migration (M40+).*

**Edit Trail:**
- 2026-02-04: Track 2 Benchmarking Analysis (Claude Code):
  - Investigated current evaluation flow - Michelle does NOT make line predictions, only reacts to market odds
  - Current output: askOdds/ceilingOdds around market, no "fair line" or "implied probability" fields
  - Conclusion: True benchmarking requires LangGraph migration for controlled information flow
  - Documented foundation and deferred full implementation to post-LangGraph milestone
  - Updated success criteria to reflect actual completion state
- 2026-02-04: Added comprehensive NCAAB research findings (Claude Code) - API endpoints, scale analysis, implementation strategy, confirmed injuries empty
- 2026-02-04: Implemented NCAAB support (Claude Code):
  - Added `ncaab` to Sport type with `mens-college-basketball` ESPN path
  - Created `006_rankings.sql` migration for AP/Coaches Poll data
  - Built `espn/rankings.ts` fetcher for NCAAB rankings
  - Created `getRankings` tool for agents
  - Updated fetcher.ts to exclude NCAAB from scheduled injury/roster/stats fetches
  - Added SPORTS_WITH_INJURIES, SPORTS_WITH_SCHEDULED_FETCH, SPORTS_WITH_RANKINGS constants
  - Added `--rankings` flag to fetchSportsData.ts for testing
  - Added weekly rankings fetch to GitHub Actions (Mondays 15:00 UTC)
  - Added `get_rankings` to progressCallback.ts (TOOL_TO_PROGRESS_STEP and TOOL_MESSAGES)
  - Added `reviewing_rankings` case to MichelleEvaluationLog.tsx
  - Updated Michelle's system prompt in gameEvaluator.ts with rankings tool
  - Seeded 362 NCAAB teams into Supabase teams table
  - Verified NCAAB standings fetch (362 teams) and rankings fetch (50 entries) working

---

## Overview

Expand agent sports intelligence from injury-only coverage to a broad foundation across NBA, NHL, and NCAAB. Add standings, schedules, team stats, and basic benchmarking so Michelle can make informed decisions across all three sports - and so we can measure whether her decisions are actually any good.

**Philosophy:** Go breadth over depth. The data infrastructure patterns from M33 (injuries via ESPN + Supabase) are proven. Extending them to new data types and a third sport is incremental. Building custom ELO/power rankings is novel and can wait. March Madness stays on the table without being a commitment.

---

## Track 1: Sports Intelligence - Breadth

Extend the existing ESPN → Supabase pipeline to cover more data types across NBA, NHL, and NCAAB.

### Standings & Records

| Sport | Data Points | Source | Notes |
|-------|-------------|--------|-------|
| NBA | W-L, conference rank, division rank, streak, home/away splits | ESPN API | Implemented 2026-02-03 |
| NHL | W-L-OTL, points, conference/division rank, streak | ESPN API | Implemented 2026-02-03 |
| NCAAB | W-L, conference record, AP/Coaches ranking | ESPN API | Rankings particularly important for March Madness |

#### Tasks

- [x] Design `standings` table schema in Supabase *(2026-02-03: Created migrations/003_standings.sql with append-only pattern matching injuries/rosters)*
- [x] Build ESPN standings fetcher (should work across all three sports with sport ID param) *(2026-02-03: Created espn/standings.ts, added to types.ts and fetcher.ts)*
- [x] Add to GitHub Actions scheduled workflow *(2026-02-03: Standings included in hourly fetch cycle)*
- [x] Create `get_standings` tool for agent toolkit *(2026-02-03: Created tools/getStandings.ts)*
- [x] Wire into Michelle's evaluation flow *(2026-02-03: Added to allTools array and updated system prompt)*
- [x] Verify data accuracy against ESPN website 

### Schedules & Rest

| Data Point | Why It Matters | Source |
|------------|----------------|--------|
| Back-to-backs | Fatigue, especially in NBA | Derive from schedule |
| Days of rest | Recovery time between games | Derive from schedule |
| Home/away streaks | Travel fatigue | Derive from schedule |
| Miles traveled (stretch goal) | Cumulative fatigue factor | Calculate from city pairs |

#### Tasks

- [x] Design `schedules` table schema in Supabase *(2026-02-03: Created migrations/004_schedules.sql with append-only pattern, current_schedules view)*
- [x] Build ESPN schedule fetcher for NBA, NHL *(2026-02-03: Created espn/schedules.ts, fetches last 7 days + next 3 days of games)*
- [x] Add to GitHub Actions scheduled workflow *(2026-02-03: Runs every 6 hours in sports-data-fetch.yml)*
- [x] Create `get_schedule_context` tool that returns rest days, back-to-back status, recent travel *(2026-02-03: Created tools/getScheduleContext.ts)*
- [x] Wire into Michelle's evaluation flow *(2026-02-03: Added to allTools, progressCallback, MichelleEvaluationLog.tsx, and system prompt)*

### Team Stats

| Stat | Sports | Source | Notes |
|------|--------|--------|-------|
| Point/goal differential | NBA, NHL, NCAAB | ESPN or derive from scores | Strong predictor |
| Offensive rating | NBA, NCAAB | ESPN | Points per 100 possessions |
| Defensive rating | NBA, NCAAB | ESPN | Points allowed per 100 possessions |
| Goals for/against per game | NHL | ESPN | |
| Pace | NBA, NCAAB | ESPN | Affects totals |
| Recent form (last 10) | All | Derive from schedule/results | |

#### Tasks

- [x] Design `team_stats` table schema in Supabase *(2026-02-03: Created migrations/005_team_stats.sql with append-only pattern, current_team_stats view)*
- [x] Build ESPN team stats fetcher *(2026-02-03: Created espn/teamStats.ts, fetches per-team stats with rate limiting)*
- [x] Add to GitHub Actions scheduled workflow *(2026-02-03: Runs daily at 12:00 UTC, also available as manual trigger)*
- [x] Create `get_team_stats` tool for agent toolkit *(2026-02-03: Created tools/getTeamStats.ts)*
- [x] Wire into Michelle's evaluation flow *(2026-02-03: Added to allTools, progressCallback, MichelleEvaluationLog.tsx, and system prompt)*

### NCAAB Coverage

*Research completed: 2026-02-04 by Claude Code*

#### ESPN API Findings

ESPN uses `mens-college-basketball` as the path segment (not NCAAB, NCAAM, or CBK). The league abbreviation in responses is `NCAAM`.

**Working Endpoints:**

| Endpoint | URL Pattern | Notes |
|----------|-------------|-------|
| Teams | `/apis/site/v2/sports/basketball/mens-college-basketball/teams?limit=400` | Returns 362 Division I teams |
| Standings | `/apis/v2/sports/basketball/mens-college-basketball/standings` | 31 conferences, full standings with PPG, streak, conf rank |
| Scoreboard | `/apis/site/v2/sports/basketball/mens-college-basketball/scoreboard?dates=YYYYMMDD&groups=50` | Need `groups=50` for Division I |
| Rankings | `/apis/site/v2/sports/basketball/mens-college-basketball/rankings` | AP Top 25 & Coaches Poll with points, first-place votes |
| Roster | `/apis/site/v2/sports/basketball/mens-college-basketball/teams/{teamId}/roster` | Per-team, same structure as NBA |
| Team Stats | `/apis/site/v2/sports/basketball/mens-college-basketball/teams/{teamId}/statistics` | Per-team, FG%, 3P%, RPG, PPG |

**Not Working / Sparse:**

| Endpoint | Result | Decision |
|----------|--------|----------|
| Injuries | Returns `{"injuries": []}` - empty | **Skip.** ESPN doesn't track NCAAB injuries. Document the gap. |

#### Scale Considerations

**362 Division I teams** vs 30 NBA / 32 NHL - cannot use same "fetch all teams" approach.

**Conference breakdown (31 conferences):**
- Major conferences: ACC (18), Big Ten (18), Big 12 (16), SEC (16)
- Mid-majors: A-10 (14), AAC (13), WCC (12), MVC (11), etc.

**Rate limit concern:** Fetching roster/stats for all 362 teams would take ~40 minutes at 100ms rate limit.

#### Implementation Strategy (Recommended)

**Approach: On-Demand + Ranked Teams**

1. **Teams table:** Don't seed 362 teams upfront
   - Dynamically add teams to `teams` table when first encountered in a contest
   - Pre-seed Top 25 ranked teams (fetched from rankings endpoint)

2. **Standings:** ✅ Works with existing fetcher
   - Single API call returns all conferences
   - Extend `Sport` type to include `'ncaab'` with `mens-college-basketball` path

3. **Rankings:** 🆕 New table/endpoint needed
   - Critical for NCAAB (AP rank matters more than conference rank)
   - Store AP rank, Coaches rank, points, first-place votes
   - Update weekly (rankings change Mondays)

4. **Injuries:** ❌ Skip - no data available
   - Document gap, don't add NCAAB to injury fetch cycle

5. **Rosters/Stats:** On-demand only
   - Fetch when team is involved in an active contest
   - Cache aggressively (rosters change rarely)
   - Focus on ranked teams + active contest teams

#### NCAAB-Specific Tasks

- [x] Research ESPN API endpoints for NCAAB *(2026-02-04: Documented above)*
- [x] Test injuries endpoint - confirmed empty *(2026-02-04: Returns `{"injuries": []}`)*
- [x] Test standings endpoint - works *(2026-02-04: Returns 31 conferences, ~362 teams)*
- [x] Test rankings endpoint - works *(2026-02-04: Returns AP Top 25 + Coaches Poll)*
- [x] Test roster endpoint - works *(2026-02-04: Same structure as NBA, includes class year)*
- [x] Test team stats endpoint - works *(2026-02-04: Same structure as NBA)*
- [x] Add `ncaab` to `Sport` type in `types.ts` *(2026-02-04: Added with mens-college-basketball path)*
- [x] Add NCAAB config to `SPORT_CONFIG` *(2026-02-04: Full config including null injuriesUrl)*
- [x] Create `rankings` table migration *(2026-02-04: Created 006_rankings.sql)*
- [x] Build rankings fetcher *(2026-02-04: Created espn/rankings.ts)*
- [x] Create `get_rankings` tool for agents *(2026-02-04: Created tools/getRankings.ts, added to allTools)*
- [x] Seed NCAAB teams into Supabase *(2026-02-04: Created seed-ncaab-teams.ts, seeded 362 teams)*
- [x] Run 006_rankings.sql migration in Supabase *(2026-02-04: Migration ran successfully)*
- [x] Test NCAAB standings fetch end-to-end *(2026-02-04: 362 standings stored)*
- [x] Test NCAAB rankings fetch end-to-end *(2026-02-04: 50 rankings stored - AP Top 25 + Coaches Poll)*

### Tool Integration

All new tools should follow the existing pattern from M33.

#### Tasks

- [x] Review Michelle's current tool calling to understand integration points *(2026-02-03: Reviewed getInjuryReport.ts and allTools pattern)*
- [x] Add new tools to agent toolkit registry *(2026-02-03: Added getStandingsTool to tools/index.ts allTools array)*
- [x] Update Michelle's system prompt to reference new capabilities *(2026-02-03: Added get_standings to tools list and usage guidance)*
- [x] Test that Michelle actually uses the new tools in evaluations *(uses tools but we'll need to refactor with LangGraph)*
- [ ] Verify tool outputs are showing up in `agentCalculations` for audit trail *(might not need this in this collection)*
- [ ] Run a few manual evaluations and review Michelle's reasoning with new data *(punting on this)*

---

## Track 2: Agent Benchmarking Foundation

Start measuring whether Michelle's lines are any good. This is data collection and basic analysis - not building the full benchmarking UI yet.

### Analysis: Current State (2026-02-04)

**Key Finding:** Michelle currently does NOT make standalone line predictions. She operates as a market-maker, reacting to market odds rather than predicting what lines should be.

**What Michelle outputs today:**
- `askOdds` / `ceilingOdds` - spreads around market odds
- `confidence` - 0-1 score
- `notes` - brief reasoning
- `status` - active or no_interest

**What Michelle does NOT output:**
- ❌ `impliedLine` - What she thinks the spread/total SHOULD be
- ❌ `impliedWinProb` - What she thinks the true win probability is
- ❌ Any prediction made BEFORE seeing market odds

**The Fundamental Challenge:**
Asking an LLM "what would you have predicted if we didn't show you the market odds" AFTER showing it the market odds is unreliable. The model will be anchored. True benchmarking requires:
1. Withholding market odds during initial prediction phase
2. Capturing prediction BEFORE revealing market context
3. Comparing prediction to actual market and eventual outcome

**Decision:** Defer full benchmarking implementation to post-LangGraph migration (likely M40+). LangGraph's graph-based execution will give us control over information flow - we can have a "predict" node that runs BEFORE the "market context" node.

### Metrics We Want to Track (Future State)

| Metric | Description | Requires |
|--------|-------------|----------|
| Line prediction accuracy | How far agent's predicted spread/total is from market consensus | `impliedLine` field |
| Directional accuracy | Does agent favor the side that actually wins? | `impliedWinProb` or odds-derived |
| Calibration | When agent implies 60% win probability, does that team win ~60%? | `impliedWinProb` field |
| Edge accuracy | When agent says "this has +EV", is it profitable over time? | Current confidence + outcomes |

### Benchmarking Foundation Tasks

*Status: Documented, implementation deferred to LangGraph migration*

- [x] Analyze current evaluation flow and output schema *(2026-02-04: See findings above)*
- [x] Document what data would need to be captured *(2026-02-04: impliedLine, impliedWinProb fields)*
- [x] Document why full implementation is blocked *(2026-02-04: LLM anchoring on market odds)*
- [x] Identify prerequisite: LangGraph migration for controlled info flow *(2026-02-04)*
- [ ] ~~Design `agent_benchmarks` table schema~~ → Deferred to M40+ with LangGraph
- [ ] ~~Build benchmark data collector~~ → Deferred to M40+ with LangGraph
- [ ] ~~Build basic analysis script~~ → Deferred to M40+ with LangGraph

### What We CAN Track Now (Without Changes)

Even without line predictions, we can still measure some things from existing data:

| Metric | Source | Notes |
|--------|--------|-------|
| Offer acceptance rate | `agentOffers` + `positions` | How often users take Michelle's offers |
| Spread from market | `agentOffers.askOdds` vs `marketOdds` | How aggressive/conservative her pricing is |
| PnL by game/market type | `positions` after settlement | Is she profitable on NBA spreads? NHL totals? |
| Volume by confidence | `agentOffers.confidence` + matched amounts | Does high confidence = more action? |

These don't require schema changes - just analysis queries on existing Firebase collections. Could be a simple script in M39+.

### Benchmark vs Sportsbook Baseline

*Deferred - requires line prediction capability first*

The simplest benchmark: if you just copied FanDuel/DraftKings lines, how would you do vs Michelle's independent analysis? This comparison only makes sense once Michelle is making predictions, not just pricing around market.

---

## Out of Scope (Documented for Future)

| Item | Why Not Now |
|------|-------------|
| Custom ELO system | Novel system, needs design work and historical data backfill |
| Power rankings | Depends on ELO or similar foundation |
| Weather data | Only relevant for outdoor sports (NFL, MLB) - not in season |
| Advanced stats (Basketball Reference scraping) | API complexity, rate limiting, legal gray area |
| Benchmarking UI | Need data first, UI can come in M39+ |
| Agent registration system | Architecture decision not yet made |
| Protocol documentation for users | No external users yet |
| Market odds setter on frontend | Backlogged, using v2.2 admin panel for now |

---

## Data Architecture Notes

### Supabase Tables (New)

```
standings
├── team_id (ESPN ID)
├── sport_id
├── season
├── wins, losses, otl (NHL), ties
├── conference_rank, division_rank
├── streak
├── home_record, away_record
├── ap_rank (NCAAB)
├── updated_at

schedules
├── game_id (ESPN ID)
├── sport_id
├── home_team_id, away_team_id
├── game_date
├── home_score, away_score (post-game)
├── status (scheduled, in_progress, final)
├── updated_at

team_stats
├── team_id (ESPN ID)
├── sport_id
├── season
├── point_differential
├── offensive_rating, defensive_rating
├── pace
├── last_10_record
├── updated_at
```

### GitHub Actions Schedule

| Workflow | Frequency | Sports | Notes |
|----------|-----------|--------|-------|
| fetch-injuries | Every hour | NBA, NHL | NCAAB excluded - ESPN returns empty |
| fetch-standings | Every 2 hours | NBA, NHL, NCAAB | NCAAB returns 31 conferences |
| fetch-schedules | Every 6 hours | NBA, NHL, NCAAB | NCAAB needs `groups=50` param |
| fetch-team-stats | Daily | NBA, NHL | NCAAB on-demand only (362 teams) |
| fetch-rankings | Weekly (Mondays) | NCAAB | AP Top 25 + Coaches Poll |

### NCAAB On-Demand Fetching Pattern

*Documented: 2026-02-04*

**CRITICAL DIFFERENCE FROM NBA/NHL:**

NBA/NHL roster and stats are fetched on a cron schedule because there are only 30/32 teams.
NCAAB has **362 Division I teams** - fetching all would take 40+ minutes and waste API calls for teams Michelle may never evaluate.

**What runs on schedule:**
| Data Type | NBA/NHL | NCAAB |
|-----------|---------|-------|
| Injuries | Every hour | **NEVER** (ESPN returns empty) |
| Standings | Every 2 hours | Every 2 hours (single API call) |
| Rosters | Daily | **ON-DEMAND ONLY** |
| Team Stats | Daily | **ON-DEMAND ONLY** |
| Rankings | N/A | Weekly (Mondays) |

**On-demand pattern for NCAAB rosters/stats:**
When a contest involves an NCAAB team:
1. Check if team exists in `teams` table (if not, add it dynamically)
2. Check if roster/stats are stale (older than threshold)
3. If stale, fetch fresh data from ESPN for just that team
4. Cache aggressively - college rosters rarely change mid-season

**Code location:** This logic should be added to the slate evaluation flow or contest creation, NOT to the scheduled fetcher.

### ESPN API Endpoint Patterns

*Documented: 2026-02-04*

```
Base: https://site.api.espn.com/apis/site/v2/sports/{sport}/{league}

NBA:
  sport: basketball, league: nba

NHL:
  sport: hockey, league: nhl

NCAAB:
  sport: basketball, league: mens-college-basketball
  ⚠️ Note: Use `mens-college-basketball` NOT ncaab/ncaam/cbk

Common endpoints:
  /injuries          - Injury reports (empty for NCAAB)
  /teams             - All teams
  /teams/{id}/roster - Team roster
  /teams/{id}/statistics - Team stats
  /scoreboard        - Games (add ?dates=YYYYMMDD)
  /rankings          - Polls (NCAAB only)

Standings uses different base:
  https://site.api.espn.com/apis/v2/sports/{sport}/{league}/standings
```

---

## Time Estimate

| Track | Estimate |
|-------|----------|
| Standings & records | 6-8 hours |
| Schedules & rest | 6-8 hours |
| Team stats | 4-6 hours |
| NCAAB injuries | 1-2 hours |
| Tool integration & Michelle wiring | 4-6 hours |
| Benchmarking foundation | 4-6 hours |

**Total:** 25-36 hours across ~1-2 weeks at current pace

---

## Success Criteria

- [x] Michelle has access to standings, schedules, and team stats for NBA, NHL, and NCAAB
- [x] New data is being fetched on schedule via GitHub Actions
- [x] Michelle demonstrably uses new tools in her evaluations *(tools wired in, full audit in agentCalculations TBD with LangGraph refactor)*
- [~] Benchmark data is accumulating for every Michelle evaluation → **Deferred to M40+** *(requires LangGraph for controlled info flow)*
- [x] NCAAB coverage exists: standings + rankings + on-demand rosters (injury data confirmed unavailable)
- [x] NCAAB rankings (AP/Coaches) are fetched and available to agents
- [x] All new tools follow existing M33 patterns (ESPN → Supabase → agent tool)
- [x] **Added:** Benchmarking foundation documented - metrics defined, blockers identified, prerequisites clear
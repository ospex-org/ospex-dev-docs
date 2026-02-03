# Milestone 38: Agent Sports Intelligence & Benchmarking Foundation

*Created: February 3, 2026*
*Status: 🔵 In Progress*
*Notes: Mainnet is live and stable. First real-money contest scored, claimed, and settled successfully. Agents are running. Time to make them smarter.*

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

### NCAAB Injury Coverage

ESPN's NCAAB injury data is thin compared to NBA/NHL, but worth capturing what's available.

#### Tasks

- [ ] Extend existing injury fetcher to include NCAAB sport ID
- [ ] Test what ESPN actually returns for NCAAB injuries
- [ ] If sparse, document the gap and move on (don't over-engineer)

### Tool Integration

All new tools should follow the existing pattern from M33.

#### Tasks

- [x] Review Michelle's current tool calling to understand integration points *(2026-02-03: Reviewed getInjuryReport.ts and allTools pattern)*
- [x] Add new tools to agent toolkit registry *(2026-02-03: Added getStandingsTool to tools/index.ts allTools array)*
- [x] Update Michelle's system prompt to reference new capabilities *(2026-02-03: Added get_standings to tools list and usage guidance)*
- [ ] Test that Michelle actually uses the new tools in evaluations *(Pending: Need to test in prod)*
- [ ] Verify tool outputs are showing up in `agentCalculations` for audit trail *(Pending: Need to test in prod)*
- [ ] Run a few manual evaluations and review Michelle's reasoning with new data *(Pending: Need to test in prod)*

---

## Track 2: Agent Benchmarking Foundation

Start measuring whether Michelle's lines are any good. This is data collection and basic analysis - not building the full benchmarking UI yet.

### Line Accuracy Tracking

Compare Michelle's offered odds against market odds at the time of the offer.

| Metric | Description |
|--------|-------------|
| Line deviation | How far Michelle's spread/total is from market consensus |
| Odds accuracy | How close Michelle's decimal odds are to market |
| Directional accuracy | Does Michelle favor the side that actually wins? |
| Calibration | When Michelle implies 60% win probability, does that team win ~60% of the time? |

#### Tasks

- [ ] Design `agent_benchmarks` table schema (Supabase or Firebase - decide)
- [ ] Build benchmark data collector that captures Michelle's lines vs market at time of offer
- [ ] Run collector on every Michelle evaluation (low overhead, just logging)
- [ ] Build basic analysis script that computes accuracy metrics from collected data
- [ ] Let data accumulate for 2+ weeks before drawing conclusions
- [ ] Document what "good" looks like for each metric

### Benchmark vs Sportsbook Baseline

The simplest benchmark: if you just copied FanDuel/DraftKings lines, how would you do vs Michelle's independent analysis?

#### Tasks

- [ ] Capture market odds at game time for all contests with Michelle offers
- [ ] After game resolution, compare outcomes to both Michelle's and market lines
- [ ] Store comparison data for future analysis

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

| Workflow | Frequency | Sports |
|----------|-----------|--------|
| fetch-injuries (existing) | Every hour | NBA, NHL, → add NCAAB |
| fetch-standings (new) | Every 2 hours | NBA, NHL, NCAAB |
| fetch-schedules (new) | Every 6 hours | NBA, NHL, NCAAB |
| fetch-team-stats (new) | Every 6 hours | NBA, NHL, NCAAB |

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

- [ ] Michelle has access to standings, schedules, and team stats for NBA, NHL, and NCAAB
- [ ] New data is being fetched on schedule via GitHub Actions
- [ ] Michelle demonstrably uses new tools in her evaluations (visible in agentCalculations)
- [ ] Benchmark data is accumulating for every Michelle evaluation
- [ ] NCAAB coverage exists (even if injury data is thin)
- [ ] All new tools follow existing M33 patterns (ESPN → Supabase → agent tool)
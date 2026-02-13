# Milestone 33: Sports Intelligence Tools

*Created: January 18, 2025*
*Completed: January 19, 2025*
*Status: ✅ Complete*

---

## Overview

Add external sports data capabilities to Michelle (and future agents) via new LangChain tools. This enables agents to reason about injuries, lineups, and roster context when evaluating bets—moving beyond pure odds comparison toward actual sports understanding.

**Primary Goal**: Michelle can answer "Who's injured?" and "Who's starting?" for any NBA/NHL game.

**Secondary Goal**: Establish Supabase as the storage layer for sports reference data, creating a bounded migration experiment before the full Firebase→Supabase transition.

**Benchmark Goal**: Can an agent create a reasonable line without looking at sportsbooks? This milestone provides the data foundation to test that.

---

## Sports Priority

Based on current calendar (mid-January 2025):

| Sport | Priority | Rationale |
|-------|----------|-----------|
| **NBA** | 🥇 Primary | Full season, daily games, good data availability |
| **NHL** | 🥈 Secondary | Full season, daily games, extends same patterns |
| **NCAAB** | 🥉 Tertiary | March Madness prep (~8 weeks), but data quality concerns |
| **NFL** | ⏸️ Deferred | 3 games left, revisit in August |
| **NCAAF** | ⏸️ Deferred | 1 game left, revisit in July |
| **MLB** | ⏸️ Deferred | Season starts April, revisit in March |

---

## Data Sources

### Primary: ESPN Hidden API

ESPN maintains undocumented JSON endpoints that require no authentication. These are widely used but come with reliability risks.

**Endpoints we'll use:**

```
# NBA Injuries (all teams)
https://site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries

# NBA Injuries (specific teams)
https://site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries?team=LAL&team=BOS

# NHL Injuries
https://site.api.espn.com/apis/site/v2/sports/hockey/nhl/injuries

# NBA Team Roster
https://site.api.espn.com/apis/site/v2/sports/basketball/nba/teams/{TEAM_ID}/roster

# NBA Depth Chart (requires season)
https://sports.core.api.espn.com/v2/sports/basketball/nba/seasons/2025/teams/{TEAM_ID}/depthcharts
```

**Team ID mapping**: ESPN uses numeric team IDs. We'll maintain a static mapping file.

### Fallback: MySportsFeeds

Commercial API with 3-day free trial, then ~$35 CAD/month for NBA.

**Use cases:**
- ESPN endpoint breaks or changes schema
- ESPN rate limits us
- Need higher data quality for production

**Documentation**: https://www.mysportsfeeds.com/data-feeds/api/feed-types/player-injuries/

### Future: RSS Feeds for Breaking News

RotoWire RSS feeds can supplement structured data with breaking injury news. Deferred to future milestone.

---

## Architecture

### Storage: Supabase (New)

Sports reference data will live in Supabase, **not Firebase**. This is a deliberate bounded experiment:

1. Data is cleanly isolated from existing Firebase collections
2. Read patterns favor Postgres (cacheable, query-heavy, not real-time)
3. Proves out Supabase patterns before critical-path migration
4. Low stakes if something goes wrong

**Schema:**

```sql
-- Teams reference table
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sport TEXT NOT NULL,           -- 'nba', 'nhl', 'ncaab'
  espn_id TEXT NOT NULL,         -- ESPN's numeric ID
  abbrev TEXT NOT NULL,          -- 'LAL', 'BOS', etc.
  name TEXT NOT NULL,            -- 'Los Angeles Lakers'
  conference TEXT,
  division TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(sport, espn_id)
);

-- Injuries table (append-only for history, latest = current)
CREATE TABLE injuries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sport TEXT NOT NULL,
  team_id UUID REFERENCES teams(id),
  player_name TEXT NOT NULL,
  player_espn_id TEXT,
  position TEXT,
  injury_type TEXT,              -- 'Knee', 'Ankle', 'Illness', etc.
  status TEXT NOT NULL,          -- 'Out', 'Doubtful', 'Questionable', 'Day-To-Day'
  description TEXT,              -- Full injury description
  source TEXT NOT NULL,          -- 'espn', 'mysportsfeeds'
  fetched_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast lookups
CREATE INDEX idx_injuries_team_fetched ON injuries(team_id, fetched_at DESC);
CREATE INDEX idx_injuries_sport_fetched ON injuries(sport, fetched_at DESC);

-- Depth charts / rosters
CREATE TABLE rosters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sport TEXT NOT NULL,
  team_id UUID REFERENCES teams(id),
  player_name TEXT NOT NULL,
  player_espn_id TEXT,
  position TEXT,
  depth_order INT,               -- 1 = starter, 2 = backup, etc.
  jersey_number TEXT,
  source TEXT NOT NULL,
  fetched_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_rosters_team_fetched ON rosters(team_id, fetched_at DESC);

-- Schema change log (for ESPN API monitoring)
CREATE TABLE schema_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint TEXT NOT NULL,
  expected_schema JSONB NOT NULL,
  actual_schema JSONB NOT NULL,
  is_match BOOLEAN NOT NULL,
  diff_summary TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Scheduled Fetcher (Cloud Function)          │
│                     Runs every 30-60 minutes                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
     ┌─────────────────┐               ┌─────────────────┐
     │   ESPN API      │               │ MySportsFeeds   │
     │   (Primary)     │               │ (Fallback)      │
     └─────────────────┘               └─────────────────┘
              │                                   │
              └─────────────────┬─────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   Schema Validator  │
                    │   + Logger          │
                    └─────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │     Supabase        │
                    │   (injuries,        │
                    │    rosters,         │
                    │    schema_logs)     │
                    └─────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Michelle's Tool Layer                       │
│                                                                 │
│  get_injury_report(team) → queries Supabase injuries table      │
│  get_roster(team)        → queries Supabase rosters table       │
└─────────────────────────────────────────────────────────────────┘
```

### ESPN API Defensive Coding

**1. Schema Validation**

```typescript
// Define expected shape
const ESPN_INJURY_SCHEMA = z.object({
  items: z.array(z.object({
    athlete: z.object({
      displayName: z.string(),
      position: z.object({ abbreviation: z.string() }).optional(),
    }),
    type: z.object({ description: z.string() }).optional(),
    status: z.string(),
    details: z.object({ detail: z.string() }).optional(),
  })),
});

async function fetchWithSchemaValidation(url: string, schema: ZodSchema) {
  const response = await fetch(url);
  const data = await response.json();
  
  const result = schema.safeParse(data);
  
  if (!result.success) {
    // Log schema mismatch but don't crash
    await logSchemaChange(url, schema, data, result.error);
    logger.warn(`[ESPN] Schema mismatch for ${url}: ${result.error.message}`);
    // Return raw data anyway - let downstream handle gracefully
  }
  
  return data;
}
```

**2. Rate Limiting (Self-Imposed)**

```typescript
import Bottleneck from 'bottleneck';

const espnLimiter = new Bottleneck({
  minTime: 6000,        // 10 requests per minute max
  maxConcurrent: 1,     // Sequential requests only
  reservoir: 100,       // Max 100 requests per hour
  reservoirRefreshAmount: 100,
  reservoirRefreshInterval: 60 * 60 * 1000, // 1 hour
});

export const fetchESPN = espnLimiter.wrap(async (url: string) => {
  return fetch(url);
});
```

**3. Caching**

```typescript
// In-memory cache with TTL
const cache = new Map<string, { data: unknown; expires: number }>();

const CACHE_TTL = {
  injuries: 30 * 60 * 1000,    // 30 minutes
  roster: 60 * 60 * 1000,      // 1 hour
  depthChart: 60 * 60 * 1000,  // 1 hour
};

async function fetchWithCache(url: string, ttl: number) {
  const cached = cache.get(url);
  if (cached && cached.expires > Date.now()) {
    return cached.data;
  }
  
  const data = await fetchESPN(url);
  cache.set(url, { data, expires: Date.now() + ttl });
  return data;
}
```

**4. Circuit Breaker**

```typescript
import CircuitBreaker from 'opossum';

const espnBreaker = new CircuitBreaker(fetchESPN, {
  timeout: 10000,           // 10s timeout
  errorThresholdPercentage: 50,
  resetTimeout: 60000,      // Try again after 1 minute
});

espnBreaker.on('open', () => {
  logger.error('[ESPN] Circuit breaker OPEN - falling back to MySportsFeeds');
});

espnBreaker.fallback(async (url: string) => {
  // Convert ESPN URL to MySportsFeeds equivalent
  return fetchMySportsFeeds(url);
});
```

---

## Track 1: Supabase Setup

### Tasks

- [x] Create Supabase project
- [x] Run schema migrations (teams, injuries, rosters, schema_logs)
- [x] Add Supabase client to backend
- [x] Seed teams table with NBA and NHL teams + ESPN IDs
- [x] Create helper functions for common queries
- [x] Add environment variables for Supabase URL/key

<!-- [CLAUDE-ADDED 2026-01-18] Implementation details below -->

### Implementation Files (Added 2026-01-18)

| File | Purpose |
|------|---------|
| `ospex-agent-server/src/db/supabase/migrations/001_sports_data.sql` | Schema migration - run in Supabase Dashboard SQL Editor |
| `ospex-agent-server/src/db/supabase/client.ts` | Supabase client singleton with types |
| `ospex-agent-server/src/db/supabase/queries.ts` | Query helpers for injuries, rosters, teams |
| `ospex-agent-server/src/db/supabase/seeds/teams.ts` | NBA (30) and NHL (32) team seed data with ESPN IDs |
| `ospex-agent-server/src/db/supabase/index.ts` | Module exports |
| `ospex-agent-server/src/scripts/testSupabase.ts` | Connection test and verification script |

### Yarn Commands (Added 2026-01-18)

```bash
yarn supabase:test   # Test connection and seed teams
yarn supabase:seed   # Seed teams only
```

### Environment Variables (Added to .env)

```
SUPABASE_URL=https://sqzvnqcghplgagktstbw.supabase.co
SUPABASE_ANON_KEY=sb_publishable_...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
```

### ESPN API Findings (2026-01-18)

**Working Endpoints:**
- `site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries` - Returns teams with nested injury arrays
- `site.api.espn.com/apis/site/v2/sports/hockey/nhl/injuries` - Same structure for NHL
- `site.api.espn.com/apis/site/v2/sports/basketball/nba/teams/{ID}/roster` - Player roster with embedded injury status
- `site.api.espn.com/apis/site/v2/sports/{sport}/{league}/teams` - Team list with ESPN IDs

**Not Working:**
- `sports.core.api.espn.com/v2/sports/basketball/nba/seasons/{YEAR}/teams/{ID}/depthcharts` - Returns 404

**Key Finding:** ESPN uses `2026` for the current 2025-26 season.

### Schema Enhancements (vs. original spec)

Added tables:
- `fetch_logs` - Track successful/failed API calls for monitoring
- Enhanced `schema_logs` with `endpoint_type` and `sample_response` fields
- Added more indexes for query performance
- Added Row Level Security (RLS) policies

<!-- [/CLAUDE-ADDED] -->

### Team Seeding (Original Spec)

```typescript
// NBA teams with ESPN IDs
const NBA_TEAMS = [
  { sport: 'nba', espn_id: '1', abbrev: 'ATL', name: 'Atlanta Hawks' },
  { sport: 'nba', espn_id: '2', abbrev: 'BOS', name: 'Boston Celtics' },
  // ... all 30 teams
];

// NHL teams with ESPN IDs
const NHL_TEAMS = [
  { sport: 'nhl', espn_id: '1', abbrev: 'BOS', name: 'Boston Bruins' },
  // ... all 32 teams
];
```

---

## Track 2: Data Fetcher Service

### ESPN Integration

```typescript
// src/services/sportsData/espn/injuries.ts

interface ESPNInjury {
  athlete: {
    id: string;
    displayName: string;
    position?: { abbreviation: string };
  };
  type?: { description: string };
  status: string;
  details?: { detail: string };
}

export async function fetchNBAInjuries(): Promise<Injury[]> {
  const url = 'https://site.api.espn.com/apis/site/v2/sports/basketball/nba/injuries';
  
  const data = await fetchWithSchemaValidation(url, ESPN_INJURY_SCHEMA);
  
  // Transform ESPN format to our schema
  return data.items.map((item: ESPNInjury) => ({
    sport: 'nba',
    player_name: item.athlete.displayName,
    player_espn_id: item.athlete.id,
    position: item.athlete.position?.abbreviation,
    injury_type: item.type?.description,
    status: normalizeStatus(item.status),
    description: item.details?.detail,
    source: 'espn',
    fetched_at: new Date(),
  }));
}

function normalizeStatus(espnStatus: string): string {
  // ESPN uses various formats, normalize to our enum
  const normalized = espnStatus.toLowerCase();
  if (normalized.includes('out')) return 'Out';
  if (normalized.includes('doubtful')) return 'Doubtful';
  if (normalized.includes('questionable')) return 'Questionable';
  if (normalized.includes('day-to-day')) return 'Day-To-Day';
  if (normalized.includes('probable')) return 'Probable';
  return espnStatus; // Return as-is if unknown
}
```

### MySportsFeeds Integration

```typescript
// src/services/sportsData/mysportsfeeds/injuries.ts

const MSF_BASE = 'https://api.mysportsfeeds.com/v2.1/pull';

export async function fetchNBAInjuriesMSF(): Promise<Injury[]> {
  const url = `${MSF_BASE}/nba/current/player_injuries.json`;
  
  const response = await fetch(url, {
    headers: {
      'Authorization': `Basic ${Buffer.from(process.env.MSF_API_KEY + ':MYSPORTSFEEDS').toString('base64')}`,
    },
  });
  
  const data = await response.json();
  
  return data.players.map((p: any) => ({
    sport: 'nba',
    player_name: `${p.player.firstName} ${p.player.lastName}`,
    player_msf_id: p.player.id,
    position: p.player.primaryPosition,
    injury_type: p.injury?.description,
    status: p.injury?.playingProbability,
    description: p.injury?.notes,
    source: 'mysportsfeeds',
    fetched_at: new Date(),
  }));
}
```

### Scheduled Fetcher

```typescript
// src/functions/fetchSportsData.ts
// Cloud Function triggered every 30 minutes

export async function fetchSportsData() {
  const sports = ['nba', 'nhl'];
  
  for (const sport of sports) {
    try {
      // Try ESPN first
      const injuries = await fetchInjuries(sport, 'espn');
      await upsertInjuries(injuries);
      logger.info(`[SportsData] Fetched ${injuries.length} ${sport} injuries from ESPN`);
    } catch (error) {
      logger.error(`[SportsData] ESPN failed for ${sport}, trying MySportsFeeds`);
      
      try {
        const injuries = await fetchInjuries(sport, 'mysportsfeeds');
        await upsertInjuries(injuries);
        logger.info(`[SportsData] Fetched ${injuries.length} ${sport} injuries from MSF`);
      } catch (msfError) {
        logger.error(`[SportsData] Both sources failed for ${sport}`);
      }
    }
  }
}
```

### Tasks

- [x] Implement ESPN injury fetcher with schema validation
- [x] Implement ESPN roster/depth chart fetcher
- [x] Implement MySportsFeeds injury fetcher (stub - awaiting API access)
- [x] Implement MySportsFeeds roster fetcher (stub - awaiting API access)
- [x] Add Bottleneck rate limiting
- [ ] Add opossum circuit breaker (deferred - simple fallback pattern used instead)
- [x] Create fetcher service with CLI scripts
- [x] Add monitoring/alerting for fetch failures (via fetch_logs table)

<!-- [CLAUDE-ADDED 2026-01-18] Implementation details below -->

### Implementation Files (Added 2026-01-18)

| File | Purpose |
|------|---------|
| `ospex-agent-server/src/services/sportsData/espn/types.ts` | Zod schemas for ESPN API responses |
| `ospex-agent-server/src/services/sportsData/espn/injuries.ts` | ESPN injury fetcher with validation |
| `ospex-agent-server/src/services/sportsData/espn/rosters.ts` | ESPN roster fetcher (per-team) |
| `ospex-agent-server/src/services/sportsData/mysportsfeeds/index.ts` | MySportsFeeds stub (ready for API key) |
| `ospex-agent-server/src/services/sportsData/rateLimiter.ts` | Bottleneck rate limiters (ESPN: 10/min, MSF: 60/min) |
| `ospex-agent-server/src/services/sportsData/cache.ts` | In-memory TTL cache (injuries: 30min, rosters: 1hr) |
| `ospex-agent-server/src/services/sportsData/fetcher.ts` | Main orchestration with fallback logic |
| `ospex-agent-server/src/scripts/fetchSportsData.ts` | CLI script for manual fetching |

### Yarn Commands (Added 2026-01-18)

```bash
yarn sportsdata:fetch           # Fetch injuries only (quick, ~10s)
yarn sportsdata:fetch:full      # Fetch injuries + rosters (~10min)
yarn sportsdata:fetch:nba       # Fetch NBA injuries only
yarn sportsdata:fetch:nhl       # Fetch NHL injuries only
```

### First Test Run (2026-01-18)

```
Total Injuries: 211
  - NBA: 114 injuries
  - NHL: 97 injuries
Duration: 10.1 seconds
```

### Architecture Notes

**Rate Limiting:**
- ESPN: 10 requests/minute, max 100/hour (conservative for undocumented API)
- MySportsFeeds: 60 requests/minute, max 500/hour (more generous for paid API)

**Fallback Pattern:**
- Try ESPN first
- If ESPN fails, try MySportsFeeds (if configured)
- Log all fetch attempts to `fetch_logs` table for monitoring

**Caching:**
- Injuries cached 30 minutes
- Rosters cached 1 hour
- Cache cleared on full fetch

<!-- [/CLAUDE-ADDED] -->

---

## Track 3: Michelle's New Tools

<!-- [CLAUDE-UPDATED 2026-01-18] Expanded scope per user feedback -->

### Design Philosophy: Two-Pass Injury Lookup

**Problem Identified:** The original design used "latest record per player" which could show stale data. Example:
- Jan 18 2pm: Jamal Murray is "Day-To-Day" (ankle)
- Jan 18 5pm: Team announces Murray will play
- Jan 18 6pm fetch: Murray is NOT in ESPN's response (he's healthy)
- With "latest per player", Michelle still sees "Day-To-Day" from 2pm

**Solution:** If a player isn't in the latest fetch, they're not injured. The **absence of a record IS the signal**.

### Postgres Views for Current State

```sql
-- Migration 002: Current State Views (run after 001_sports_data.sql)

-- Returns only injuries from the most recent fetch for each team
CREATE VIEW current_injuries AS
SELECT i.*
FROM injuries i
INNER JOIN (
  SELECT team_id, MAX(fetched_at) as latest_fetch
  FROM injuries GROUP BY team_id
) latest ON i.team_id = latest.team_id
        AND i.fetched_at = latest.latest_fetch;

-- Same pattern for rosters
CREATE VIEW current_rosters AS
SELECT r.*
FROM rosters r
INNER JOIN (
  SELECT team_id, MAX(fetched_at) as latest_fetch
  FROM rosters GROUP BY team_id
) latest ON r.team_id = latest.team_id
        AND r.fetched_at = latest.latest_fetch;
```

### Tool 1: get_injury_report (Summary View)

**Purpose:** Quick overview of who's injured. Michelle checks this first.

**Returns:** Player name, position, status, injury type, lastUpdated timestamp

**Usage:** Check both teams before evaluating spread/total bets.

```typescript
// Example output
{
  team: "Los Angeles Lakers",
  injuries: [
    { player: "LeBron James", position: "F", status: "Questionable", injuryType: "Ankle" },
    { player: "Anthony Davis", position: "F/C", status: "Out", injuryType: "Knee" },
  ],
  lastUpdated: "2026-01-18T18:04:54Z",
  injuryCount: 2
}
```

### Tool 2: get_injury_details (Full Context)

**Purpose:** Deep dive on a specific player. Michelle calls this for key players whose absence "could move the line 1+ points."

**Returns:** Long-form injury description (the blurb), injury date, historical records for trajectory analysis.

**Parameters:**
- `playerName` - Full or partial name
- `sport` - 'nba' or 'nhl'
- `lastN` - Return last N records (default 1, max 10)
- `withinHours` - Only records within N hours (default 168 = 7 days)

```typescript
// Example output
{
  player: "Anthony Davis",
  records: [{
    status: "Out",
    injuryType: "Knee",
    shortComment: "Knee - left knee contusion",
    longComment: "Davis (knee) has been ruled out for Sunday's matchup against the Clippers. Davis is dealing with a left knee contusion sustained in Friday's game...",
    injuryDate: "2026-01-17",
    fetchedAt: "2026-01-18T18:04:54Z",
    team: "Los Angeles Lakers"
  }],
  recordCount: 1,
  trajectory: "stable"  // 'improving' | 'worsening' | 'stable' | 'unknown'
}
```

### Tool 3: get_roster (Current Snapshot)

**Purpose:** Understand who might replace injured players. Uses same "most recent fetch" pattern.

**Returns:** Player names, positions, jersey numbers, physical attributes.

### "Show Your Work" - Agent Context Snapshots

**Purpose:** Store the injuries, roster context, and reasoning that agents factored into decisions. Foundation for the future insights system.

**Firebase Collection:** `agentContextSnapshots`

```typescript
interface AgentContextSnapshot {
  id: string;
  quoteId?: string;        // Link to michelleQuotes if instant match
  evalId?: string;         // Link to evaluation if batch
  agentId: string;
  contestId: string;
  jsonoddsId: string;
  timestamp: Timestamp;

  injuriesConsidered: Array<{
    team: string;
    players: Array<{
      player: string;
      position?: string;
      status: string;
      injuryType?: string;
      impactAssessment?: 'high' | 'medium' | 'low' | 'none';
    }>;
  }>;

  rosterSnapshot?: Array<{
    team: string;
    playerNames: string[];
  }>;

  dataFreshness: {
    injuriesLastFetched?: string;  // ISO timestamp
    rostersLastFetched?: string;
  };

  contextSummary?: string;  // Agent's summary of how context affected decision
}
```

### Updated System Prompt Guidance

```typescript
const SPORTS_INTELLIGENCE_PROMPT = `
SPORTS CONTEXT TOOLS:
You have access to injury and roster information:
- get_injury_report(team, sport): Summary view - who's injured, their status
- get_injury_details(player, sport, lastN?, withinHours?): Full context for key players
- get_roster(team, sport): Current roster snapshot

TWO-PASS APPROACH:
1. First, call get_injury_report for both teams
2. Only call get_injury_details for players whose absence could move the line 1+ points
   - Don't drill into end-of-bench players
   - Star players Out/Doubtful = definitely investigate

USE THESE WHEN:
- Evaluating spread or total bets where player availability matters
- A key player injury could move the line 1+ points
- You want to understand why odds might differ from expectations

DON'T OVERUSE:
- Not every bet needs injury analysis
- Moneyline bets on heavy favorites may not need deep roster analysis
- If you just checked injuries for a team, don't check again for the same game

REASONING WITH INJURIES:
- "Out" = definitely not playing, factor heavily
- "Doubtful" = ~75% chance not playing, significant concern
- "Questionable" = ~50/50, minor factor
- "Day-To-Day" / "Probable" = likely playing, minimal concern

When injuries are significant, mention them in your decision reasoning.
`;
```

### Tasks

- [x] Create Postgres views migration (current_injuries, current_rosters)
- [x] Update Supabase queries to use "most recent fetch" pattern
- [x] Implement get_injury_report tool (summary)
- [x] Implement get_injury_details tool (full context + historical)
- [x] Implement get_roster tool
- [x] Create Firebase agentContextSnapshots collection and helpers
- [x] Update Michelle's system prompt with sports intelligence guidance
- [x] Wire tools into Michelle's LangChain agent
- [ ] Test with real NBA games

<!-- [CLAUDE-ADDED 2026-01-18] Implementation details below -->

### Implementation Files (Added 2026-01-18)

| File | Purpose |
|------|---------|
| `ospex-agent-server/src/db/supabase/migrations/002_current_state_views.sql` | Postgres views + helper functions |
| `ospex-agent-server/src/db/supabase/queries.ts` | New functions: `getCurrentInjuryReport`, `getInjuryDetails`, `getCurrentRosterReport` |
| `ospex-agent-server/src/agents/market_maker_michelle/langchain/tools/getInjuryReport.ts` | Summary injury tool |
| `ospex-agent-server/src/agents/market_maker_michelle/langchain/tools/getInjuryDetails.ts` | Full context injury tool with trajectory analysis |
| `ospex-agent-server/src/agents/market_maker_michelle/langchain/tools/getRoster.ts` | Current roster tool |
| `ospex-agent-server/src/firebase.ts` | Added `agentContextSnapshots` collection helpers |

### Key Design Decisions

1. **Two-pass injury lookup** prevents context bloat while allowing deep dives on important players
2. **Postgres views** make queries simple (`SELECT * FROM current_injuries WHERE team_id = ?`)
3. **Historical lookups** via `lastN` and `withinHours` parameters enable trajectory analysis
4. **Context snapshots** create an audit trail for agent reasoning (foundation for insights)

<!-- [/CLAUDE-ADDED] -->

---

## Track 4: Testing & Validation

### Line Creation Benchmark

The ultimate test: can Michelle create a reasonable line without looking at sportsbooks?

```typescript
// Test script: benchmark_line_creation.ts

async function benchmarkLineCreation(gameId: string) {
  // 1. Get the actual market line (but don't show Michelle)
  const actualLine = await getMarketLine(gameId);
  
  // 2. Ask Michelle to create a line using only:
  //    - Team records/standings
  //    - Injury reports
  //    - Home/away status
  //    - Historical matchups (future tool)
  
  const prompt = `
    Create a spread line for ${game.away} @ ${game.home}.
    You have access to injury reports and rosters.
    DO NOT look up current betting odds.
    Reason through the matchup and propose a spread.
  `;
  
  const michelleEstimate = await invokeMichelleForBenchmark(prompt);
  
  // 3. Compare
  return {
    game: `${game.away} @ ${game.home}`,
    actualSpread: actualLine.spread,
    michelleSpread: michelleEstimate.spread,
    difference: Math.abs(actualLine.spread - michelleEstimate.spread),
    reasoning: michelleEstimate.reasoning,
  };
}
```

### Manual Test Cases

| Test | Expected Behavior |
|------|-------------------|
| NBA game with star player Out | Michelle mentions injury in reasoning |
| NHL game, no significant injuries | Michelle doesn't over-research injuries |
| Request for team that doesn't exist | Graceful error, doesn't crash |
| ESPN API down | Falls back to MySportsFeeds or cached data |
| Schema change detected | Logs warning, still returns data if possible |

### Tasks

- [x] Create line creation benchmark script
- [x] Run benchmark on 10+ NBA games (6 games tested)
- [x] Document accuracy metrics (spreads off by an average of 4.75 points)
- [ ] Create test suite for tool error handling (see monitoring notes below)
- [ ] Test ESPN fallback to MySportsFeeds (awaiting access to mysportsfeeds)

### Error Monitoring (Current State)

**Available monitoring mechanisms:**

1. **`fetch_logs` table** - Query for failures:
   ```sql
   SELECT * FROM fetch_logs WHERE success = false ORDER BY created_at DESC LIMIT 20;
   ```

2. **`schema_logs` table** - Check for ESPN API schema changes:
   ```sql
   SELECT * FROM schema_logs WHERE is_match = false ORDER BY created_at DESC;
   ```

3. **Winston logs** - All errors logged via `logger.error()`, visible in Heroku logs:
   ```bash
   heroku logs --app ospex-agents --num 200 | grep -i error
   ```

**Not yet implemented:**
- Automated test suite for error conditions
- Health check endpoint
- Alerting/notifications for failures
- Monitoring dashboard

<!-- [CLAUDE-ADDED 2026-01-18] Implementation details below -->

### Implementation Files (Added 2026-01-18)

| File | Purpose |
|------|---------|
| `ospex-agent-server/src/scripts/benchmarkLineCreation.ts` | Line creation benchmark script |

### Yarn Commands (Added 2026-01-18)

```bash
yarn benchmark:line                          # Next 24 hours, NBA+NHL, up to 10 games
yarn benchmark:line --hours=48               # Next 48 hours
yarn benchmark:line --league=NBA             # NBA only
yarn benchmark:line --league=NHL             # NHL only
yarn benchmark:line --hours=72 --league=NBA --limit=5
yarn benchmark:line <jsonodds_id>            # Benchmark specific game by ID
```

### How It Works

1. Fetches upcoming contests from Firebase with spread/total lines
2. Retrieves injury reports for both teams from Supabase
3. Asks Claude (without showing actual odds) to estimate spread and total
4. Compares estimates to actual sportsbook lines
5. Reports accuracy metrics (within 3 points, within 5 points, etc.)

### Output Example

```
SPREAD ACCURACY:
  Games tested:      10
  Average diff:      3.2 points
  Within 3 points:   6/10 (60%)
  Within 5 points:   8/10 (80%)
```

<!-- [/CLAUDE-ADDED] -->

---

## Success Criteria

- [x] Supabase schema deployed and seeded with NBA/NHL teams
- [ ] ESPN fetcher running on schedule, populating injuries/rosters (just need to kick off github action)
- [x] Schema validation catching and logging any ESPN API changes (should be, in schema_logs table)
- [ ] MySportsFeeds fallback working when ESPN fails (skipped, no trial access)
- [x] Michelle can use get_injury_report and get_roster tools
- [x] Michelle mentions injuries in decision reasoning when relevant (in one-off test she did)
- [ ] Line creation benchmark shows Michelle within ±3 points of actual spread (stretch goal) (will continue to monitor this but we're not there yet)

---

## Non-Goals (This Milestone)

- Full Supabase migration of existing Firebase data
- College basketball support (data quality TBD)
- Historical stats / advanced metrics
- Weather data
- Real-time lineup changes (just pre-game)
- Insights storage (future milestone)

---

## Open Questions

1. **Supabase region** - Should match Firebase/Cloud Functions region for latency
2. **MySportsFeeds trial** - 3 days might not be enough; budget ~$35 CAD/month for NBA if needed
3. **Injury data freshness** - Is 30-minute refresh frequent enough? Probably yes for pre-game.
4. **College basketball** - Worth adding in February for March Madness prep, or skip this year?

---

## Dependencies

- M32 complete (LangChain agent architecture)
- Supabase account (free tier)
- MySportsFeeds API key (free trial, then paid)
- ESPN endpoints remain accessible (risk acknowledged)

---

## Files to Create

```
src/
├── services/
│   └── sportsData/
│       ├── espn/
│       │   ├── injuries.ts
│       │   ├── rosters.ts
│       │   └── schemaValidator.ts
│       ├── mysportsfeeds/
│       │   ├── injuries.ts
│       │   └── rosters.ts
│       ├── rateLimiter.ts
│       ├── circuitBreaker.ts
│       └── cache.ts
├── db/
│   └── supabase/
│       ├── client.ts
│       ├── migrations/
│       │   └── 001_sports_data.sql
│       └── seeds/
│           ├── nba_teams.ts
│           └── nhl_teams.ts
├── agents/
│   └── market_maker_michelle/
│       └── langchain/
│           └── tools/
│               ├── getInjuryReport.ts
│               └── getRoster.ts
└── functions/
    └── fetchSportsData.ts
```

---

## Notes

- This is our first Supabase code. Keep it simple, establish patterns.
- ESPN API could break any day. The fallback architecture isn't paranoia, it's prudence.
- Injury impact on lines varies wildly by sport and player. Michelle should learn this over time.
- The line creation benchmark is ambitious but gives us a measurable "smartness" metric.
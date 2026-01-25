# Firebase Collections Reference

*Generated: January 19, 2026*
*Last Updated: January 24, 2026*

This document provides a comprehensive reference for all Firebase Firestore collections used in the Ospex platform. It serves as the source of truth for understanding data structures, usage patterns, and migration planning.

---

## Table of Contents

1. [Collection Overview](#collection-overview)
2. [Agent Collections](#agent-collections)
3. [Michelle-Specific Collections](#michelle-specific-collections)
4. [On-Chain Synced Collections](#on-chain-synced-collections)
5. [Off-Chain Reference Collections](#off-chain-reference-collections)
6. [Archive Collections](#archive-collections)
7. [Orphaned/Unused Collections](#orphanedunused-collections)
8. [Consolidation Recommendations](#consolidation-recommendations)
9. [Migration Notes](#migration-notes)

---

## Collection Overview

| Collection | Documents | Purpose | Status |
|------------|-----------|---------|--------|
| `agentDecisions` | 42 | All agent decisions (Dan, Michelle) | Active |
| `agentDecisionsArchive` | Variable | Archived decisions (preserves LLM reasoning) | Active |
| `agentMemory` | 1 | Legacy nested agent memory | Deprecated |
| `agentMeta` | 1 | Agent metadata and flags | Active |
| `agentMatchHistory` | 0 | Empty, unused | Remove |
| `agentContextSnapshots` | 0 | Empty, interface exists | Keep (unused) |
| `agent_interests` | 66 | Pre-position conviction tracking | Active |
| `agentOffers` | 112 | Active market maker offers | Active |
| `agentOffers_archive` | 2,747 | Expired offers | Active |
| `michelleCalculations` | 56 | Tool audit log | Active |
| `michelleDecisions` | 1 | Michelle final decisions | Low Usage |
| `michelleQuotes` | 90 | Instant match quotes | Critical |
| `usedFeeTxHashes` | 65 | Fee replay prevention | Critical |
| `amoyContestsv2.3` | 242 | On-chain contests | Active |
| `amoySpeculationsv2.3` | 207 | On-chain speculations | Active |
| `amoyPositionsv2.3` | 397 | On-chain positions | Active |
| `contests` | 179 | Off-chain game data | Active |
| `speculations` | 1 | Off-chain spec tracking | Low Usage |
| `contests_archive` | 8,892 | Historical games | Active |
| `speculations_archive` | 87 | Historical specs | Active |

---

## Agent Collections

### `agentDecisions`

**Purpose:** Flat collection storing decisions from all agents. Used for auditing, insights, and "show your work" features.

**Document Count:** 42

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `agentId` | string | Agent identifier (e.g., "degen_dan", "market_maker_michelle") |
| `agentName` | string | Human-readable name |
| `agentWallet` | string | Agent's wallet address |
| `network` | string | Network (e.g., "amoy") |
| `requestType` | string | "instant_match" or batch evaluation |
| `decision` | string | "accept", "counter", "reject" |
| `reason` | string | Explanation of the decision |
| `market` | string | "moneyline", "spread", "total" |
| `pick` | string | "home", "away" |
| `line` | number/null | Line value for spread/total |
| `targetOdds` | number | Agent's target odds |
| `requestedOdds` | number | User's requested odds |
| `conviction` | number | Conviction score (1-10) |
| `maxWagerUSDC` | number | Maximum wager |
| `contestId` | string | Contest identifier |
| `jsonoddsID` | string | JSONOdds game ID |
| `createdAt` | string | ISO timestamp |
| `updatedAt` | Timestamp | Firestore timestamp |

**Written By:** `ospex-agent-server/src/services/memoryService.ts`

**Read By:** Agent tools for context, future Insights tab

---

### `agentMeta`

**Purpose:** Per-agent metadata including run timestamps, watched positions, and speculation flags.

**Document Count:** 1 (keyed by `{agentId}_{network}`)

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `agentId` | string | Agent identifier |
| `network` | string | Network name |
| `lastRun` | string | ISO timestamp of last run |
| `updatedAt` | Timestamp | Last update time |
| `watchedPositions` | array | Positions being tracked for claiming |
| `specFlags.{specId}` | object | Flags per speculation (claimed, etc.) |

**Written By:** `ospex-agent-server/src/services/memoryService.ts`

**Read By:** Agent scheduler, claim logic

---

### `agentMemory`

**Purpose:** Legacy collection with nested structure. Being phased out in favor of flat `agentDecisions` + `agentMeta`.

**Document Count:** 1

**Status:** Deprecated - migrate remaining reads to `agentMeta`

---

### `agent_interests`

**Purpose:** Stores agent betting interests before positions are created. Represents "conviction" in a market before committing capital.

**Document Count:** 66

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `agentWallet` | string | Agent's wallet |
| `agentName` | string | Agent name |
| `jsonoddsID` | string | Game identifier |
| `speculationScorer` | string | Scorer contract address |
| `theNumber` | string/null | Line value (null for moneyline) |
| `positionType` | string | "0" (Upper/Away/Over) or "1" (Lower/Home/Under) |
| `positionTypeString` | string | "Upper" or "Lower" |
| `targetOddsDecimal` | number | Target odds |
| `maxWagerUSDC` | number | Maximum position size |
| `convictionScore` | number | Conviction level (1-10) |
| `reasoning` | string | Why the agent likes this bet |
| `status` | string | "active" or "filled" |
| `createdAt` | Timestamp | Creation time |
| `expiresAt` | Timestamp | Expiration (typically game start) |

**Written By:** `ospex-agent-server/src/firebase.ts::addAgentInterest()`

**Read By:** Matching system, offer generation

---

### `agentOffers`

**Purpose:** Active market maker offers. These are the odds Michelle is willing to accept for each game/market.

**Document Count:** 112

**Document ID Format:** `{agentId}_{jsonoddsId}_{market}` (deterministic)

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `agentId` | string | "market_maker_michelle" |
| `agentName` | string | "Market Maker Michelle" |
| `agentWallet` | string | Michelle's wallet |
| `jsonoddsId` | string | Game identifier |
| `contestId` | string | Contest ID |
| `market` | string | "moneyline", "spread", "total" |
| `line` | number/null | Line value |
| `evaluatedLine` | number/null | Line at evaluation time |
| `sport` | string | Sport name (e.g., "NBA", "NCAAB") |
| `sportId` | number | Sport ID |
| `awayTeam` | string | Away team name |
| `homeTeam` | string | Home team name |
| `matchTime` | Timestamp | Game start time |
| `status` | string | "active", "invalid_odds" |
| `notes` | string | Michelle's market commentary |
| `homeOffer` | object | `{ odds, maxExposure, currentExposure }` |
| `awayOffer` | object | Same structure for away side |
| `overOffer` | object | For totals |
| `underOffer` | object | For totals |
| `noInterestReason` | string | Why no offer (if status is invalid) |
| `evaluatedAt` | Timestamp | Last evaluation time |
| `createdAt` | Timestamp | Creation time |
| `updatedAt` | Timestamp | Last update |
| `maxExposurePerGame` | number | Maximum risk per game |

**Written By:** `ospex-agent-server/src/services/offerService.ts`

**Read By:** Frontend (`ospex-lovable/src/lib/data/agentOffers.ts`), instant match flow

---

## Michelle-Specific Collections

### `michelleQuotes`

**Purpose:** Quote documents for the instant match flow. Each quote represents a user's request for Michelle to match their position.

**Document Count:** 90

**Document ID Format:** `{quoteId}__{network}`

**Critical For:** Michelle's evaluation log in the frontend stepper

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `quoteId` | string | UUID for this quote |
| `network` | string | Network name |
| `userWallet` | string | Requesting user's wallet |
| `jsonoddsId` | string | Game identifier |
| `market` | string | Market type |
| `line` | number/null | Line value |
| `side` | string | "away", "home" |
| `requestedOdds` | number | User's requested odds |
| `requestedAmount` | number | User's requested amount |
| `quotedAmount` | number | Amount Michelle will match |
| `feeTxHash` | string | Fee payment transaction hash |
| `status` | string | "pending", "complete", "counter_pending", "rejected" |
| `decision` | object | `{ decision, reason, matchAmount, counterOdds, confidence }` |
| `counter` | object | Counter-offer details if applicable |
| `progress` | array | Step-by-step evaluation progress (powers the UI) |
| `preflightSnapshot` | object | Market state at request time |
| `freshEvalOdds` | number | Michelle's fresh evaluation |
| `freshEvalAt` | string | When fresh eval was computed |
| `evalSnapshot` | object | Full evaluation context |
| `marketOddsAtQuote` | object | Market odds at quote time |
| `reason` | string | Decision explanation |
| `startedAt` | string | Evaluation start time |
| `completedAt` | string | Evaluation end time |
| `createdAt` | Timestamp | Creation time |
| `expiresAt` | Timestamp | Quote expiration |

**Written By:** `ospex-agent-server/src/services/michelleQuotesService.ts`, `quoteProgressService.ts`

**Read By:** Frontend evaluation log, instant match UI

---

### `michelleCalculations`

**Purpose:** Audit log of all mathematical calculations Michelle performs. Used for debugging and transparency.

**Document Count:** 56

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Calculation UUID |
| `quoteId` | string | Associated quote |
| `tool` | string | "compute_exposure", "compare_odds", "counter_odds" |
| `ts` | string | ISO timestamp |
| `inputs` | object | Tool input parameters |
| `outputs` | object | Calculation results |

**Written By:** `ospex-agent-server/src/services/calculationAuditService.ts`

**Read By:** Debugging tools, audit scripts

---

### `michelleDecisions`

**Purpose:** Michelle's instant match decisions. Appears to overlap with `agentDecisions`.

**Document Count:** 1 (very low usage)

**Key Fields:** Similar to `agentDecisions` but Michelle-specific

**Status:** Consider deprecating in favor of `agentDecisions`

---

### `usedFeeTxHashes`

**Purpose:** Prevents replay attacks by tracking claimed fee transaction hashes.

**Document Count:** 65

**Document ID Format:** `{feeTxHash}__{network}`

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `txHash` | string | The transaction hash |
| `network` | string | Network name |
| `usedAt` | Timestamp | When it was claimed |
| `quoteId` | string | Associated quote |
| `userWallet` | string | User who paid the fee |

**Written By:** `ospex-agent-server/src/services/michelleQuotesService.ts`

**Critical:** Do not delete - prevents double-claiming fees

---

## On-Chain Synced Collections

These collections mirror on-chain data, populated by cloud functions in `ospex-fdb`.

### `amoyContestsv2.3`

**Purpose:** On-chain contest data synced from blockchain events.

**Document Count:** 242

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `contestId` | string | On-chain contest ID |
| `jsonoddsID` | string | JSONOdds game ID |
| `rundownID` | string | Rundown API ID |
| `sportspageID` | number | Sportspage API ID |
| `Sport` | number | Sport enum |
| `AwayTeam` | string | Away team |
| `HomeTeam` | string | Home team |
| `MatchTime` | Timestamp | Game start |
| `status` | string | "Unverified", "Verified", "Scored" |
| `awayScore` | string | Away team score |
| `homeScore` | string | Home team score |
| `MoneyLineAway/Home` | string | American odds |
| `PointSpreadAway/Home` | string | Spread lines |
| `TotalNumber` | string | Total line |
| `contestCreator` | string | Creator wallet |
| `scoredAt` | Timestamp | When scored |

---

### `amoySpeculationsv2.3`

**Purpose:** On-chain speculation (market) data.

**Document Count:** 207

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `speculationId` | string | On-chain ID |
| `contestId` | string | Parent contest |
| `speculationCreator` | string | Creator wallet |
| `speculationScorer` | string | Scorer contract address |
| `theNumber` | string | Line value (scaled by 10) |
| `speculationStatus` | number | 0=open, 1=settled |
| `winSide` | number | Winning side (1=Away, 2=Home, 3=Over, 4=Under, 5=Push, etc.) |
| `createdAt` | Timestamp | Creation time |
| `settledAt` | Timestamp | Settlement time |

---

### `amoyPositionsv2.3`

**Purpose:** On-chain position data.

**Document Count:** 397

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `speculationId` | string | Parent speculation |
| `user` | string | Position owner wallet |
| `positionType` | string | "0" (Upper) or "1" (Lower) |
| `oddsPairId` | string | Odds pair identifier |
| `upperOdds` | string | Upper side odds (1e7 scaled) |
| `lowerOdds` | string | Lower side odds (1e7 scaled) |
| `matchedAmount` | string | Matched amount (wei6) |
| `unmatchedAmount` | string | Unmatched amount (wei6) |
| `unmatchedExpiry` | string | Unix timestamp for expiry |
| `claimed` | boolean | Whether winnings claimed |
| `createdAt` | Timestamp | Creation time |

---

## Off-Chain Reference Collections

### `contests`

**Purpose:** Off-chain game data from JSONOdds/Rundown/Sportspage APIs. Updated by `ospex-fdb/src/monitor.ts`.

**Document Count:** 179

**Key Fields:** Similar to `amoyContestsv2.3` plus:

| Field | Type | Description |
|-------|------|-------------|
| `isFinal` | boolean | Whether game has ended |
| `sportspageStatus` | string | "scheduled", "in progress", "final" |
| `eventStatus` | string | ESPN status code |
| `eventStatusDetail` | string | Human-readable status |

**Used For:** Determining when to score contests (checks `isFinal`)

---

### `speculations`

**Purpose:** Off-chain speculation tracking.

**Document Count:** 1 (very low usage)

**Status:** Appears mostly unused - investigate and potentially remove

---

## Archive Collections

### `agentDecisionsArchive`

**Purpose:** Archived agent decisions. Preserves LLM reasoning and decision history after games start.

**Document Count:** Variable (grows over time)

**Archived When:** Games start (via `cleanupExpiredMemoryDecisions()` in agent.ts)

**Key Fields:** Same as `agentDecisions` plus:

| Field | Type | Description |
|-------|------|-------------|
| `archivedAt` | Timestamp | When the decision was archived |
| `archiveReason` | string | Why it was archived (e.g., "cleanup_expired") |

**Written By:** `ospex-agent-server/src/services/memoryService.ts::replaceDecisions()`

**Retention:** Permanent (recommended: 365 days TTL)

**Important:** This collection preserves LLM reasoning that would otherwise be lost when active decisions are cleaned up.

---

### `agentOffers_archive`

**Purpose:** Expired market maker offers.

**Document Count:** 2,747

**Archived When:** Offers expire (past game start time)

**Fields:** Same as `agentOffers` plus `archivedAt`

---

### `contests_archive`

**Purpose:** Historical game data.

**Document Count:** 8,892

**Archived When:** Game is finished and no longer needed for active operations

---

### `speculations_archive`

**Purpose:** Historical speculation data.

**Document Count:** 87

---

## Orphaned/Unused Collections

### `agentMatchHistory`

**Document Count:** 0

**Status:** Empty - no writes found in codebase

**Recommendation:** Remove collection

---

### `agentContextSnapshots`

**Document Count:** 0

**Status:** Empty, but TypeScript interfaces exist in `firebase.ts`

**Purpose (intended):** Store "show your work" context (injuries, rosters) that agents considered

**Recommendation:** Keep for future use, implement writes

---

## Consolidation Recommendations

### Phase 1: Naming Consolidation (Low Risk)

| Current | Proposed | Rationale |
|---------|----------|-----------|
| `michelleCalculations` | `agentCalculations` | Add `agentId` field, works for any agent |
| `michelleDecisions` | Deprecate | Already captured in `agentDecisions` |
| `michelleQuotes` | `agentQuotes` | Add `agentId` field for multi-agent future |

### Phase 2: Cleanup (Safe to Execute)

1. **Delete `agentMatchHistory`** - Empty, no usage
2. **Migrate `agentMemory` reads to `agentMeta`** - Complete deprecation
3. **Audit `speculations` collection** - Only 1 doc, appears orphaned

### Phase 3: Multi-Agent Preparation

Add `agentId` field to:
- `michelleQuotes` → `agentQuotes`
- `michelleCalculations` → `agentCalculations`
- `usedFeeTxHashes` (already has wallet, may need agentId for agent-specific tracking)

### Document ID Strategy

Current deterministic IDs like `{agentId}_{jsonoddsId}_{market}` are good. Extend this pattern:
- Quotes: `{agentId}_{quoteId}_{network}`
- Calculations: `{agentId}_{calculationId}`

---

## Migration Notes

### Supabase Consideration

Collections that might migrate to Supabase:
- `agent_interests` - Analytical data, benefits from SQL queries
- `agentDecisions` - Historical data, good for analytics
- Archive collections - Large historical datasets

Collections to keep in Firestore:
- `agentOffers` - Needs real-time listeners for frontend
- `michelleQuotes` - Real-time progress updates
- `usedFeeTxHashes` - Security-critical, fast lookups
- On-chain synced collections - Event-driven updates

### Backwards Compatibility

When renaming collections:
1. Write to both old and new collection names during transition
2. Update all readers to check new name first, fall back to old
3. After all deployments, remove old collection writes
4. Archive old collection data before deletion

---

## Schema Discovery Script

A schema discovery script is available at:
```
ospex-agent-server/src/scripts/documentCollections.ts
```

Run with:
```bash
cd ospex-agent-server
yarn tsx src/scripts/documentCollections.ts
```

Outputs:
- `logs/collection_schemas_{timestamp}.json` - Full schema data
- `logs/collection_schemas_{timestamp}.md` - Markdown summary

---

*Document maintained by: Development Team*
*See also: milestone-034 for context on this documentation effort*

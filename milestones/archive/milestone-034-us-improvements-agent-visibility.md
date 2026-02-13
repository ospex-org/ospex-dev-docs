# Milestone 34: Instant Match UX & Agent Visibility Groundwork

*Created: January 19, 2025*
*Complete: January 19, 2025*
*Status: ✅ Complete*

---

## Overview

Two focused goals for a single work session:

1. **Celebrate the match** — Add confetti when instant match completes. The current "Position matched" text is anticlimactic; users need the dopamine hit.

2. **Agent visibility groundwork** — Document current Firebase collections and prepare the data layer for surfacing Michelle's reasoning on the frontend.

**Why these two?** Confetti is a quick UX win that makes testing more fun. The agent visibility work sets up future milestones (Insights tab, Settings tab, benchmarking) without committing to a specific UI yet.

---

## Track 1: Confetti on Instant Match

### Goal

When the stepper reaches "Matched" state, confetti rains down from the top of the viewport.

### Implementation

**Library:** `canvas-confetti` (lightweight, no dependencies, ~3KB gzipped)

```bash
npm install canvas-confetti
# or
yarn add canvas-confetti
```

**Trigger location:** The component that renders the stepper. When all steps are complete and status is "matched":

```typescript
import confetti from 'canvas-confetti';

// When match completes successfully
const celebrateMatch = () => {
  confetti({
    particleCount: 100,
    spread: 70,
    origin: { y: 0.3 },
    colors: ['#10b981', '#34d399', '#6ee7b7'], // Green theme to match success
  });
};

// Trigger when stepper reaches "Matched"
useEffect(() => {
  if (matchStatus === 'matched') {
    celebrateMatch();
  }
}, [matchStatus]);
```

**Design notes:**
- Keep it subtle — one burst, not continuous
- Green color palette to reinforce success
- Origin at y: 0.3 so it falls through the content area
- No confetti on counter-offers or declines, only on successful match

### Tasks

- [x] Install `canvas-confetti`
- [x] Add confetti trigger to instant match stepper component
- [x] Test on both Order Book page and Order Book Details page
- [x] Verify confetti doesn't fire on counter-offer (only final match)
- [x] Stretch: Team-specific confetti colors based on user's pick
- [x] Stretch: Tiny team logo SVGs in the confetti burst (delightful but scope creep) (8 teams have this)

### Files to Modify

| File | Change |
|------|--------|
| `package.json` | Add canvas-confetti dependency |
| `src/components/InstantMatch/MatchStepper.tsx` (or similar) | Add confetti trigger |

---

## Track 2: Agent Data Layer Documentation

### Goal

Document exactly what lives where in Firebase so we can make informed decisions about the Insights and Settings tabs.

### Current Collections (Agent-Related)

<!-- [Claude Code - Jan 19, 2026] Updated with actual findings from schema discovery -->

| Collection | Docs | Purpose | Status |
|------------|------|---------|--------|
| `agentDecisions` | 42 | Flat decisions from all agents (Dan, Michelle) | **Keep** |
| `agentMemory` | 1 | Legacy nested structure | **Deprecate** → migrate to `agentMeta` |
| `agentMeta` | 1 | Agent metadata, flags, watched positions | **Keep** |
| `agentMatchHistory` | 0 | Empty, no usage | **Remove** |
| `agentContextSnapshots` | 0 | Empty but interface exists | Keep for future |
| `agentOffers` | 112 | Active market maker offers | **Keep** (critical) |
| `agentOffers_archive` | 2,747 | Expired offers | **Keep** |
| `agent_interests` | 66 | Pre-position conviction tracking | **Keep** |
| `michelleCalculations` | 56 | Tool audit log (exposure, compare_odds) | **Keep** → rename to `agentCalculations` |
| `michelleDecisions` | 1 | Michelle final decisions (low usage) | **Deprecate** → use `agentDecisions` |
| `michelleQuotes` | 90 | Instant match quotes (powers eval log) | **Keep** → rename to `agentQuotes` |
| `usedFeeTxHashes` | 65 | Fee replay prevention | **Keep** (security critical) |

### Tasks

- [x] Query each collection and document its schema
- [x] Identify which data powers Michelle's evaluation log (the expandable section in instant match)
- [x] Decide: consolidate to `agent*` naming with agentId field?
- [x] Create a `FIREBASE_COLLECTIONS.md` doc in ospex-documentation

### Findings

<!-- [Claude Code - Jan 19, 2026] Added findings section -->

**Evaluation Log Data Source:**
The expandable "Michelle's evaluation log" is powered by the `michelleQuotes` collection, specifically the `progress` array field which contains step-by-step evaluation updates written by `quoteProgressService.ts`.

**Consolidation Decision:**
Yes, consolidate to `agent*` naming with `agentId` field. This prepares for multi-agent future. Phased approach:

1. **Phase 1:** Rename `michelleQuotes` → `agentQuotes`, `michelleCalculations` → `agentCalculations`
2. **Phase 2:** Delete `agentMatchHistory` (empty), deprecate `michelleDecisions` (use `agentDecisions`)
3. **Phase 3:** Add `agentId` field to all quote/calculation documents

**Orphaned Collections:**
- `agentMatchHistory` - 0 documents, safe to delete
- `speculations` (off-chain) - Only 1 document, appears orphaned

### Schema Discovery Script

Script created at: `ospex-agent-server/src/scripts/documentCollections.ts`

```bash
# Run the discovery script
cd ospex-agent-server
yarn tsx src/scripts/documentCollections.ts

# Outputs:
# - logs/collection_schemas_{timestamp}.json
# - logs/collection_schemas_{timestamp}.md
```

---

## Track 3: Insights vs Position History (Definitions)

### Insight (WHY)

The thesis behind a position. Why did this entity make this bet?

- **Agents:** Required. Every agent position should have an insight.
- **Humans:** Optional. Users can post insights but don't have to.
- **Storage:** `agentQuotes.insight` or new `insights` collection
- **UI Surface:** Profile page → Insights tab

### Position History (WHAT)

The audit trail of evaluations on a position.

- **Agents:** Every evaluation logged, even rejections
- **Humans:** No trail unless they take action (match/partial match)
- **Storage:** `agentCalculations` + new `positionEvaluations` collection?
- **UI Surface:** Position detail page → "Who looked at this?" section

### Current State

| Data | Exists? | Collection |
|------|---------|------------|
| Michelle's match reasoning | Yes | `agentQuotes.progress` |
| Michelle's rejection reasoning | Partial | `agentCalculations` has inputs/outputs but not full "why I passed" |
| Market Maker notes | Yes | `agentOffers.notes` |
| Human insights | No | Not implemented |

### Gap

We log Michelle's evaluations when she's *asked* (instant match flow), but not when she's *scanning* the order book and passing. The `[Michelle Matching] No enriched candidates` log doesn't store WHY she passed on each candidate.

### Decision: Yes, Log Agent Evaluations

<!-- [Claude Code - Jan 19, 2026] Track 3 Implementation -->

**Why log rejections?**
- Engagement: Users can see "Michelle looked at your position but passed because..."
- Debugging: Understand why positions aren't getting matched
- Analytics: Track patterns in agent behavior over time
- Trust: Transparency about agent decision-making

**Cost considerations:**
- Firebase writes: ~$0.18 per 100K writes → negligible at current scale
- LLM calls: Zero additional - we already have the LLM reasoning from matching-job
- Storage: 90-day TTL keeps collection bounded

### `positionEvaluations` Collection Schema

```typescript
interface PositionEvaluation {
  // === Identity ===
  id: string;                    // Auto-generated
  agentId: string;               // e.g., 'market_maker_michelle'
  positionId: string;            // The unmatched position being evaluated
  speculationId: string;

  // === Context ===
  jsonoddsId: string;
  market: 'moneyline' | 'spread' | 'total';
  side: 'away' | 'home' | 'over' | 'under';  // Which side the AGENT would take

  // === Position Details ===
  takerOdds?: number;            // Odds available to agent (may not exist for early skips)
  makerOdds?: number;
  amountUSDC?: number;           // Amount available to match

  // === Decision ===
  decision: 'match' | 'pass' | 'skip';
  phase: 'enrich' | 'prefilter' | 'llm' | 'execution';
  reasonCode: string;            // Machine-readable: 'EXPOSURE_LIMIT', 'NO_EVAL', 'LLM_PASS'
  reason: string;                // Human-readable explanation

  // === LLM Reasoning (only for LLM phase) ===
  llmReasoning?: string;         // Full reasoning from LLM response

  // === Timestamps ===
  evaluatedAt: Timestamp;
  expiresAt: Timestamp;          // TTL: 90 days
}
```

### Reason Codes

| Phase | Code | Human Message |
|-------|------|---------------|
| enrich | `SPEC_MISSING` | Position's speculation not found |
| enrich | `SCORER_UNKNOWN` | Unknown market type |
| enrich | `JSONODDS_MISSING` | Game ID not found |
| enrich | `CONTEST_MISSING` | Game data not available |
| enrich | `CONTEST_STARTED` | Game has already started |
| prefilter | `EXPOSURE_LIMIT` | Would exceed exposure limits |
| prefilter | `STAKE_TOO_SMALL` | Position too small to consider |
| prefilter | `NO_EVAL` | Haven't evaluated this game yet |
| prefilter | `LINE_MISMATCH` | Line doesn't match our evaluation |
| llm | `LLM_PASS` | Agent declined (see reasoning) |
| llm | `LLM_MATCH` | Agent accepted |
| execution | `EXEC_SUCCESS` | Match executed successfully |
| execution | `EXEC_FAILED` | Match failed (see reason) |

### Implementation Files

| File | Change |
|------|--------|
| `ospex-agent-server/src/services/positionEvaluationService.ts` | **NEW** - Service to log evaluations |
| `ospex-agent-server/src/agents/market_maker_michelle/matching-job.ts` | Add logging at each decision point |
| `ospex-lovable/src/components/agents/PositionEvaluationHistory.tsx` | **NEW** - Display eval history |

### Tasks

- [x] Design `positionEvaluations` schema
- [x] Create `positionEvaluationService.ts`
- [x] Add evaluation logging to matching-job enrichment phase
- [x] Add evaluation logging to matching-job pre-filter phase
- [x] Add evaluation logging to matching-job LLM phase
- [x] Create frontend component to display evaluation history (`PositionEvaluationHistory.tsx`)
- [ ] Wire component into position detail page (future milestone)

---

## Non-Goals (This Milestone)

- Full Insights tab implementation
- Settings tab implementation  
- Agent registration system
- Model switching UI
- Benchmark-as-a-service

---

## Success Criteria

- [x] Confetti fires on successful instant match (both pages)
- [x] Firebase collections documented with schemas
- [x] Decision made: consolidate michelle* → agent* or keep separate?
- [ ] (Stretch) Insights tab shows at least placeholder real data

<!-- [Claude Code - Jan 19, 2026] Track 1 and Track 2 completed -->

---

## Time Estimates

| Track | Estimate | Notes |
|-------|----------|-------|
| Confetti | 1-2 hours | Library install, find trigger point, test |
| Data documentation | 2-3 hours | Query collections, write doc, discuss consolidation |
| Insights mapping | 1-2 hours | Stretch goal if time permits |

**Total:** 4-7 hours, fits the work session.

---

## Notes

### On the Instant Match Flow

The current flow is actually really polished:
1. Number line visualization ✓
2. Create Position modal with clear pricing ✓
3. Stepper showing progress through evaluation ✓
4. Michelle's evaluation log (expandable) ✓
5. Counter-offer with accept/decline ✓

The only missing piece is the celebration at the end. Confetti fills that gap.

### On Firebase Collection Naming

Current state is a mix of:
- Generic (`agentDecisions`)
- Michelle-specific (`michelleCalculations`)

**Recommendation:** Consolidate to `agent*` pattern with an `agentId` field. This scales to multiple agents without creating new collections.

```typescript
// Before
michelleCalculations/doc123 = { inputs: {...}, outputs: {...} }

// After  
agentCalculations/doc123 = { agentId: '0xedf57...967', inputs: {...}, outputs: {...} }
```

### On the "Matched" State Detection

Need to find where the stepper state is managed. Look for:
- A state variable like `matchStatus`, `stepperState`, or similar
- The component that renders the green checkmarks
- The transition from "Position Created" to "Matched"

The confetti trigger hooks into this existing state management.
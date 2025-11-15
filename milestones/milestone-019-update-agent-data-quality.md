# Milestone 019: Agent Data Quality & Display
*Created: November 13, 2025*
*Completed: November 15, 2025*
*Status: ✅ Complete (note: anything pending, will correct/fix later)*

## Problem
Agent data writing to Firebase has inconsistencies that will cause issues when building transaction flows. Field naming is unclear, line numbers don't match smart contract conventions, and reasoning isn't being stored.

## What You're Building
Clean, consistent agent data schema that matches smart contract requirements and provides all information needed for UI display and future transaction flows.

## Success Criteria
- [X] `positionTypeString` correctly stores "Upper" or "Lower" (not team names)
- [X] Agent stores lines in memory as .5 increments only (matching smart contracts) (pending)
- [X] `theNumber` in agent_interests matches amoySpeculationsv2.3 format (whole number string) (pending)
- [X] `reasoning` field stored in agent_interests
- [X] Decision made on AwayTeam/HomeTeam field strategy
- [X] `contestId` field removed (defer to M20)
- [X] Both `createdAt` and `analyzedAt` timestamps stored
- [X] Memory format matches Firebase format
- [X] UI displays Dan's reasoning on agent cards
- [X] All fields verified through end-to-end test

## Implementation

### 1. Fix positionType and positionTypeString Fields

**Current (Wrong):**
```json
{
  "positionType": "Lower",
  "positionTypeString": "Phoenix Suns -4"  // ❌ Mixed data
}
```

**Target (Correct):**
```json
{
  "positionType": "1",           // should show number, same as amoyPositionsv2.3
  "positionTypeString": "Lower"  // ✅ Clean mapping
}
```

Note: will need to adjust TeamQuery.tsx on front end

### 2. Standardize Line Numbers (theNumber)

**Smart Contract Storage Format:**
- `theNumber` stored as **whole number string** (e.g., `"7"`, `"8"`, `"195"`)

**Agent Logic Updates:**

1. **When LLM returns a line:**
   - Agent should internally think, realize and analyze based on the fact that the smart contract numbers all effectively end in 0.5
   - IOW: agent should never have interest in a spread of -4.0, it needs to decide, does it want -3.5 or -4.5
   - Store in memory with `.5` precision

2. **Line Validation:**
   - Agent should ONLY have conviction on `.5` increment lines

### 3. Add Reasoning Field

**Current State:** Reasoning appears in logs but isn't stored.

**Agent Code Addition:**
```typescript
// In agent code, when storing to Firebase
const AgentInterest = {
  jsonoddsID: contest.jsonoddsID,
  agentName: "Degen Dan",
  awayTeam: contest.awayTeam,
  homeTeam: contest.homeTeam,
  positionType: "0" | "1",
  positionTypeString: positionType,  // "Upper" or "Lower"
  speculationScorer: scorerAddress,
  theNumber: String(), // will need to determine calc
  targetOddsDecimal: targetOdds,
  maxWagerUSDC: wagerAmount,
  convictionScore: conviction,
  status: "active",
  createdAt: Timestamp.now(),
  analyzedAt: memoryEntry.analyzedAt,  // From LLM analysis time
  reasoning: llmResponse.reasoning,  // ✅ ADD THIS
  expiresAt: Timestamp.fromMillis(gameTime)
};
```

**Frontend Display:** (suggested code)
```tsx
// In AgentInterestCard component
<div className="text-sm text-muted-foreground mb-4 line-clamp-3">
  {interest.reasoning || "No reasoning provided"}
</div>
```

### 4. Decision: AwayTeam/HomeTeam Fields

**Strategy: Store Redundantly (Option A)**

```json
{
  "jsonoddsID": "abc-123",
  "awayTeam": "Indiana Pacers",  // ✅ Store for fast access
  "homeTeam": "Phoenix Suns",
  "positionType": "Lower"
}
```

**Rationale:**
- Contests don't change after creation
- Agent interests are temporary (expire after game)
- Faster UI rendering (no extra lookups)
- Small storage cost, big performance gain

**Agent Implementation:**
```typescript
// When fetching contest data
const contest = await getContestById(jsonoddsID);

const interestDoc = {
  // ... other fields
  awayTeam: contest.AwayTeam,
  homeTeam: contest.HomeTeam
};
```

### 5. Remove contestId Field

**Current Problem:** Field is redundant with `jsonoddsID` and will be confusing when on-chain IDs are needed.

**Action:** 
- Remove `contestId` field entirely from agent_interests schema
- Use `jsonoddsID` for all lookups
- In M20, we'll add `chainContestId` when implementing transaction flow

**Agent Code:**
```typescript
// REMOVE this field
// contestId: contest.jsonoddsID,  ❌ Delete

// Keep only:
jsonoddsID: contest.jsonoddsID  // ✅ Single source of truth
```

### 6. Timestamp Strategy: createdAt vs analyzedAt

**Why We Need Both:**

**analyzedAt:** When LLM evaluated the game and formed opinion
- Could be minutes or hours before storing
- Represents when analysis happened
- Important for re-evaluation logic

**createdAt:** When interest was written to Firebase
- System timestamp
- Important for expiration and ordering
- Firestore-managed field

**Use Case Example:**
```
11/13/2025 03:43 AM - Agent analyzes game, forms conviction
                    - Stores in memory with analyzedAt timestamp
11/13/2025 03:45 AM - Agent writes to Firebase with createdAt
                    - Display: "Analyzed 7 hours ago"
11/13/2025 08:00 PM - Line moves significantly, Devin Booker ruled out
                    - Agent re-analyzes, NEW analyzedAt timestamp
                    - Updates or removes interest based on new analysis
```

**Implementation:**
```typescript
const interestDoc = {
  // ... other fields
  analyzedAt: memoryEntry.analyzedAt,  // From LLM analysis
  createdAt: Timestamp.now(),          // Firebase write time
};
```

**Frontend Display:**
```tsx
// Show analyzedAt for user context
<div className="text-xs text-muted-foreground">
  Analyzed {formatRelativeTime(interest.analyzedAt)}
</div>
```

### 7. Memory-to-Firebase Alignment

**Memory Format (agent's internal state):**
```json
{
  "jsonoddsID": "abc-123",
  "market": "spread",
  "pick": "home",
  "line": -4.0,
  "conviction": 6,
  "analyzedAt": "11/13/2025, 03:43 AM ET",
  "matchTime": "11/13/2025, 09:00 PM ET",
  "acceptableOddsRange": {
    "min": 1.865,
    "max": 2.061
  },
  "targetOdds": 1.963,
  "action": "stored_interest"
}
```

**Firebase Format (agent_interests doc):**
```json
{
  "jsonoddsID": "abc-123",
  "agentName": "Degen Dan",
  "awayTeam": "Indiana Pacers",
  "homeTeam": "Phoenix Suns",
  "positionType": "Lower",
  "positionTypeString": "Lower",
  "speculationScorer": "0xfbdd866d326edc6b5da7ebb533fe6fda4f73b758",
  "theNumber": "8",
  "targetOddsDecimal": 1.963,
  "maxWagerUSDC": 50,
  "convictionScore": 6,
  "reasoning": "Suns dominant at home, Pacers on back-to-back...",
  "status": "active",
  "analyzedAt": "2025-11-13T09:43:00Z",
  "createdAt": Timestamp,
  "expiresAt": Timestamp
}
```

Note on above: anything stored in Firebase should also be stored in memory of agent

**Position Type Mapping:**
```typescript
function getPositionType(pick: string, market: string): "Upper" | "Lower" {
  if (market === "moneyline" || market === "spread") {
    return pick === "away" ? "Upper" : "Lower";
  } else if (market === "total") {
    return pick === "over" ? "Upper" : "Lower";
  }
  throw new Error(`Invalid market/pick: ${market}/${pick}`);
}
```

### 8. Frontend Updates

**TeamQuery.tsx Changes:** (as needed)

### 9. Testing Checklist

**Agent Side:**

1. Run agent for upcoming games
2. Check Firebase console for stored interests:

**Field Validation:**
- [X] `positionTypeString`
- [X] `theNumber` = whole number string (e.g., "8", "7", "191")
- [X] `reasoning` field present with LLM explanation
- [X] `awayTeam` and `homeTeam` present
- [X] `contestId` field NOT present (removed)
- [X] `analyzedAt` timestamp present
- [X] `createdAt` timestamp present
- [X] All other fields match schema

**Frontend Side:**

1. Navigate to Agents page
2. Type team name (e.g., "Packers", "Lakers", "Warriors")
3. Verify card displays:
   - [X] Correct opposite-side team shown
   - [X] Line displays ending in 0.5
   - [X] Reasoning text visible and readable
   - [X] "Analyzed X hours ago" timestamp
   - [X] Odds and max bet display correctly
   - [X] League badge present

**Edge Cases:**
- [ ] Totals: No team-specific side (Over/Under displayed correctly) (pending)
- [ ] Moneyline: No line number (only team name shown) (pending)
- [X] Multiple interests: Grid layout working
- [X] Line conversion handled properly

**Memory-Firebase Alignment:**
1. Check agent memory logs for a stored interest
2. Find same interest in Firebase

### 10. Agent Code Files to Modify

**Primary Files:**
1. Main agent loop
   - Update Firebase write logic
   - Add reasoning field
   - Fix positionType fields
   - Convert line to theNumber format
   - Add both timestamps

2. Firebase utility
   - Update schema/types
   - Remove contestId references

3. Agent memory management
   - Ensure lines stored as .5 increments
   - Store analyzedAt from LLM response

**Frontend Files:**
1. `ospex-lovable/src/components/agents/TeamQuery.tsx`
   - Update positionType normalization
   - Add reasoning display
   - Add analyzedAt display

2. `ospex-lovable/src/lib/data/types.ts`
   - Update as needed

## Done When

Agent writes clean, consistent data to Firebase that:
1. ✅ Matches smart contract storage format
2. ✅ Uses clean position type strings
3. ✅ Includes reasoning
4. ✅ Provides all info needed for UI display
5. ✅ Is ready for M20 transaction implementation

**Final Test:** Run agent → query "Lakers" on frontend → see Dan's reasoning, correct line display, proper team mapping, analyzed timestamp, and all data renders accurately with no console errors.
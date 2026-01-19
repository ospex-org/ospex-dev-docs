# Milestone 28: Memory Migration + Mainnet Prep

*Created: December 31, 2025*
*Completed: January 3, 2026*
*Status: ✅ Complete*

---

## Overview

Migrate agent memory from ephemeral filesystem to Firestore, run a full leaderboard test cycle to shake out bugs, and prepare for Polygon mainnet deployment.

---

## Track 1: Memory → Firestore Migration

**Goal:** Agent state persists across deploys and dyno restarts.

### Current State

- Each agent has a `memory.json` file
- Contains: `decisions`, `specFlags`, `activeLeaderboard`, metadata
- Lost on every Heroku deploy/restart
- Requires manual intervention to reset stale state

### Target State

- Firestore collection: `agentMemory`
- Document per agent: `agentMemory/{agentId}`
- Same structure as current JSON, just stored persistently
- Can view/edit via Firebase console

### Tasks

- [x] Design Firestore schema for agent memory
- [x] Create `src/services/memoryService.ts` with:
  - `loadMemory(agentId): Promise<AgentMemory>`
  - `saveMemory(agentId, memory): Promise<void>`
  - `updateDecision(agentId, contestId, decision): Promise<void>` (added as `upsertDecision`)
  - `clearDecisions(agentId): Promise<void>` (skipped: We didn’t add a dedicated clearDecisions helper because the code paths that “clear/prune” decisions already use replaceDecisions(...) (which can be called with [] to clear).)
- [x] Update Dan's agent code to use memoryService instead of file reads/writes
- [ ] Update Michelle if she uses memory (check this) (skipped, Michelle doesn't use memory)
- [ ] Create migration script to seed Firestore from existing JSON (one-time) (skipped, started fresh, clean-slate)
- [ ] Test locally: verify memory persists across process restarts (skipped)
- [x] Deploy to Heroku, verify memory survives dyno restart
- [ ] Remove JSON memory files from repo (or keep as backup/template) (skipped)

### Schema Sketch

```typescript
// Firestore: agentMemory/{agentId}
{
  agentId: string,
  activeLeaderboard: string | null,
  decisions: {
    [oddsId: string]: {
      oddsId: string,
      oddsIdHome?: string,
      contestId: string,
      market: string,
      side: string,
      myOdds: number,
      status: 'pending' | 'created' | 'matched' | 'settled' | 'expired',
      createdAt: Timestamp,
      updatedAt: Timestamp,
      // ... other fields as needed
    }
  },
  specFlags: {
    [oddsId: string]: {
      spec: boolean,
      timestamp: Timestamp
    }
  },
  updatedAt: Timestamp
}
```

### Estimated Effort

3-5 days

---

## Track 2: Leaderboard Testing

**Goal:** Run a real 1-2 week leaderboard, compete with Dan, find and fix bugs.

### Setup

- [x] Create Leaderboard 10 on Amoy
- [x] Verify Dan is active and evaluating games
- [x] Verify Michelle is providing odds
- [x] Make some picks manually, verify they show up

### Things to Watch For

- Stale game issues (like the Dec 6 bug)
- Matching failures or unexpected matches
- UI bugs when viewing positions/leaderboard
- Settlement issues when games complete
- Any edge cases with odds, lines, or exposure

### Bug Tracking

Keep a running list below. Fix as you go, or batch if non-critical.

| Bug | Severity | Status | Notes |
|-----|----------|--------|-------|
| No agent ↔ market maker auto-fill | Low | Deferred | Design decision needed. Manual matching for now. Potential solution: background job X mins before game time fills unmatched agent positions via Michelle. (note: track 4 in this document should fix this) |
| Neutral site games cause API mismatch | Low | Known Limitation | CFP games, etc. Workaround: use 2-of-3 APIs manually. Rare enough to not engineer a fix now. |
| Group leaderboards | Low | Deferred | Need to be able to group leaderboards so that when a user wants to see only their leaderboard performance, it will not show just one (ie the current), but will show all official, will likely need a naming scheme for this to maintain consistency |
| Firebase pruning | Low | Deferred | Prune decisions for agents (stored in array field `decisions` in agentMemory collection, agentName__network document), suggestion to prune after 30 days or archive |
| Create process for monitoring agent and script wallets and oracle wallet | Low | Deferred | Need to ensure wallets stay topped up and might need some sort of admin panel for this |

### Success Criteria

- Complete a full leaderboard cycle (create → picks → games complete → settlement)
- No critical bugs blocking mainnet

---

## Track 3: Mainnet Prep

**Goal:** Ready to deploy to Polygon mainnet by mid-January.

### Tasks

- [ ] Review contract deployment checklist
- [ ] Verify frontend can switch networks cleanly
- [ ] Plan USDC funding for Michelle's wallet
- [ ] Decide on initial exposure limits for mainnet
- [ ] Any config changes needed for prod vs test

---

## Success Criteria

- [x] Agent memory survives Heroku restarts (verified)
- [ ] Completed at least one full leaderboard cycle on Amoy
- [ ] No known critical bugs
- [ ] Ready for mainnet deploy decision

---

## Track 4: Michelle User Matching (Track A - Post and Wait)

**Goal:** Users can create positions from the Agent page, and Michelle evaluates/matches them on a scheduled cadence.

### Architecture

**Track A: "Post and Wait" (free)**
- User creates position on-chain at their desired odds via Agent page
- Michelle runs a scheduled matching job (every 15 min)
- Job queries all unmatched positions
- For each position: Is this game evaluated? Are odds within acceptable range? Is there exposure room?
- If yes: Michelle creates matching position (up to her per-side limit)
- If no: Position stays open for other counterparties or user can cancel

### Current State

- Michelle evaluates games and generates offers ✅
- Agent page displays offers with odds line visualization ✅
- Users can create positions from Agent page ✅
- **Not implemented:** Scheduled job for Michelle to scan and match open positions

### Tasks

- [x] Verify user can create position from Agent page (Rangers game test)
- [x] Build Michelle matching prompt:
  - Input: open positions, current exposure, market odds, her original evaluations
  - Output: decisions array with action (match/pass), amount, reason
- [x] Create exposure tracking for Michelle in agentMemory:
  - Track exposure per game, per side
  - Update when matches occur
  - Reset when games settle
- [x] Create Michelle matching job:
  - Query unmatched positions (Firebase)
  - Query Michelle's current exposure (agentMemory)
  - Query current market odds
  - Build prompt with full landscape
  - Execute Michelle's decisions
  - Update exposure tracking
- [x] Schedule job to run every 15 minutes on Heroku
- [x] Add logging for Michelle's decisions (for debugging and future analysis)
- [x] Test end-to-end: User creates position → Michelle evaluates on next run → Match (or pass with reason)

### Success Criteria

- [x] User creates position from Agent page
- [x] Within 15 minutes, Michelle evaluates and matches (if within range)
- [x] Match visible on leaderboard/positions
- [x] If outside range, position remains open (no error, just unmatched)

### Notes on pre-filtering for Michelle

- Contest start time must be in the future (otherwise it dies in contestStarted)
- Michelle must have an evaluation/offer doc for that jsonoddsId + market (otherwise you’ll hit the no evaluation present pre-filter)
- For spread/total, line must match (theNumber) (otherwise you’ll hit the line mismatch pre-filter)
- Position must exist such that suggestedTakerStakeUSDC >= MICHELLE_MIN_MATCH_USDC (otherwise you’ll hit the stake pre-filter)
- Michelle must have capacity (maxMatchUSDC > 0) (otherwise you’ll hit the exposure pre-filter)

---

## Notes

*Add any observations, decisions, or context as the milestone progresses.*
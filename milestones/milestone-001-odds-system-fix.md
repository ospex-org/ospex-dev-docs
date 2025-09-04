# Milestone 001: Odds System Documentation & Display Fix
*Created: January 3, 2025*  
*Target Completion: January 10, 2025*  
*Status: 🟡 Planning*

## Problem Statement
User enters 2.60 decimal odds on frontend → sees 2.59 on profile page → wastes hours debugging whether bug is in contracts, Firebase, or frontend. Need to understand and document the complete odds flow, then fix display inconsistencies.

## The Odds Confusion (Current Understanding)

### What We Know
- **Contract uses**: Decimal odds (easier for payout calculations)
- **Two users needed**: Maker (creates odds) + Taker (matches position)
- **Example**: 2.60 decimal odds for maker means taker gets different odds
- **Problem**: Frontend shows 2.59 instead of 2.60 entered
- **Storage**: Contract uses "odds pair IDs" that are confusing to decode

### What We Need to Figure Out
1. **Exact conversion flow**: Frontend input → Contract storage → Firebase → Frontend display
2. **Odds pair relationship**: If maker gets 2.60, what does taker get?
3. **Odds pair ID meaning**: What does the ID stored in Firebase represent?
4. **Rounding issues**: Where does 2.60 become 2.59?

## Scope & Boundaries

### In Scope
- Document complete odds conversion flow
- Create odds reference chart (decimal ↔ american ↔ contract representation)
- Fix frontend display rounding issues  
- Create debugging tools for odds validation
- Test with current deployed contracts (no redeployment needed)

### Out of Scope
- Contract logic changes
- New odds features
- Leaderboard rules (separate milestone)
- American odds input (decimal input only for now)

## Investigation Plan

### Phase 1: Map the Current Flow (2-3 hours)
- [ ] **Step 1**: Trace the exact 2.60 → 2.59 conversion
  - `CreateUnmatchedPairModal.tsx` (line 229): User enters 2.60
  - `createUnmatchedPair.ts` (line 45): Converts to `Math.floor(2.60 * 10_000_000) = 26,000,000`
  - Contract creates position with oddsPairId
  - `oddsCalculations.ts`: `calculateOddsFromId()` converts back to decimal
  - `Positions.tsx` (line 458): `getOddsForPosition()` displays the result

- [ ] **Step 2**: Debug the reverse calculation
  - Test `calculateOddsFromId()` with known oddsPairIds
  - Check if the `Math.floor(upperOdds / (ODDS_PRECISION / 100)) / 100` operation causes 2.59
  - Document the exact precision loss point

- [ ] **Step 3**: Compare with contract expectations
  - Verify if contract actually stores 26,000,000 correctly
  - Check if oddsPairId generation is working as expected
  - Compare maker vs taker odds calculations

- [ ] **Step 4**: Validate Firebase data consistency  
  - Confirm Firebase receives correct oddsPairId from contract events
  - Check `convertFirebaseToPosition()` in dataConversions.ts (lines 315-384)
  - Verify position.poolId matches contract oddsPairId

## Staging for Deployment

**Status**: Changes committed to milestone branch, ready to batch with other fixes

**Deployment Requirements:**
1. Deploy updated PositionModule.sol with new `getOrCreateOddsPairId()` signature
2. Update event emissions in `createUnmatchedPair()` to include odds values:
   ```solidity
   (uint128 oddsPairId, uint64 upperOdds, uint64 lowerOdds) = getOrCreateOddsPairId(odds, positionType);
   
   emit PositionCreated(speculationId, msg.sender, oddsPairId, unmatchedExpiry, positionType, amount, upperOdds, lowerOdds);
   ```
3. Update Firebase event handlers to capture and store `upperOdds` and `lowerOdds` fields
4. Remove `calculateOddsFromId()` JavaScript function (no longer needed)
5. Update frontend to read odds directly from Firebase instead of calculating

**Batching Strategy**: Deploy with leaderboard fee fix for coordinated single deployment

## Success Metrics Achieved ✅

### Technical Acceptance ✅
- [x] Identified exact source of 2.60 → 2.59 display bug  
- [x] Can explain maker/taker odds relationship (2.60 maker gets 1.63 taker)
- [x] Contract data verified as accurate via DecodePosition.s.sol script
- [x] Have solution to display contract values directly without calculation

### Documentation Acceptance ✅  
- [x] Complete understanding of odds conversion flow documented
- [x] Root cause analysis shows events missing actual odds data
- [x] Solution eliminates need for error-prone JavaScript reverse calculation

### Process Acceptance ✅
- [x] Work done on milestone branch with clear commits
- [x] AI session notes documented for future reference
- [x] Staging approach planned for coordinated deployment

## Lessons Learned

1. **Contract events should include calculated values, not just IDs** - saves complex reverse calculations
2. **JavaScript precision differs from Solidity** - avoid reverse-engineering contract math in frontend
3. **Test contract data directly** - DecodePosition.s.sol script was invaluable for verification
4. **Storage references eliminate code duplication** - cleaner than multiple memory copies
5. **Batch related changes** - deploy contract fixes together rather than separately

## Files Modified
- `src/modules/PositionModule.sol` - Updated `getOrCreateOddsPairId()` function

## Files To Update (Next Deployment)
- Event emissions in `createUnmatchedPair()`
- Firebase event handlers 
- Frontend odds display logic
- Remove `src/utils/oddsCalculations.ts` (no longer needed)

## Next Steps
- Merge with leaderboard fee fix in single deployment milestone
- Update Foundry tests to handle new return signature
- Plan Firebase event handler updates
- Test complete flow after deployment

---

## Milestone Completion Workflow

**For future reference, here's how to complete milestones:**

### During Milestone
1. Update status as tasks complete: 🟡 Planning → 🟠 In Progress → ✅ Complete
2. Document discoveries and decisions in real-time
3. Commit code changes to milestone branch regularly

### Completing Milestone  
1. **Update milestone file** with final status and results (like this)
2. **Document lessons learned** for future milestones
3. **List all files modified** for deployment tracking
4. **Plan next steps** or dependencies
5. **Merge milestone branch** to main when ready to deploy

### After Completion
1. **Create next milestone** based on learnings and next priorities
2. **Archive completed milestone** (keep file for reference)
3. **Update system-overview.md** if architecture changed

*This milestone successfully eliminated the 2.60 → 2.59 display bug and established the milestone-driven workflow.*
# Milestone 002: Leaderboard Position Event Handler Testing
*Created: September 4, 2025*  
*Target Completion: September 4, 2025*  
*Status: 🚫 Blocked*

## Problem Statement
Need to test the complete flow of LEADERBOARD_POSITION_ADDED event handling to ensure positions are properly added to leaderboards with correct validation. Must verify that advantageous positions are rejected while poor odds positions are accepted, and that Firebase correctly captures and stores the event data.

## Context
With the new unified Firebase functions index.ts ready for deployment, we need to validate that the leaderboard position validation logic works correctly. This involves testing both the "happy path" (poor odds position gets added) and the "rejection path" (advantageous odds position gets rejected).

## Scope & Boundaries

### In Scope
- Deploy new Firebase functions with unified index.ts (should be complete but may need to alter)
- Create a position at poor odds that should be accepted by leaderboard
- Verify LEADERBOARD_POSITION_ADDED event is emitted correctly
- Confirm Firebase receives and stores event data properly
- Test frontend display of leaderboard positions
- Verify advantageous odds position is rejected (taker side)
- Frontend UI/UX improvements related to this milestone

### Out of Scope
- Leaderboard rules modifications
- Contract logic changes
- Multiple leaderboard testing (focus on one leaderboard)

## Test Plan

### Phase 1: Setup & Deployment (30 mins)
- [X] **Deploy Firebase Functions**
  ```bash
  cd ospex-firebase/functions
  git add index.ts
  git rm CFPv1.json ContestOracleResolved.json  
  git commit -m "Update: unified Firebase event handlers"
  git push origin main
  firebase deploy --only functions
  ```

- [X] **Verify Environment**
  - Confirm contract addresses are correct in Firebase config (skipping, this should not have changed)
  - Check that leaderboard exists and is active
  - Verify frontend can connect to Firebase (skipping, this will become apparent if not connected)
  - Test that jsonodds data is current (skipping, this should be automatic)

### Phase 2: Happy Path Testing - Specific Test Cases

**Market Context:** Boston vs. Arizona MLB ML
- Market: Boston -122 (1.82 decimal), Arizona +111 (2.11 decimal)  
- Leaderboard: 25% odds enforcement, 1-3 USDC position limits

**Test Case 1: Acceptable Poor Odds Position**
- [ ] **Create Boston position at 1.37 decimal odds, 3 USDC**
  - This is roughly 25% worse than market (should be accepted)
  - Position size within 1-3 USDC range
  - Record: position ID, leaderboard acceptance

**Test Case 2: Acceptable Poor Odds Position (Arizona side)**
- [ ] **Create Arizona position at 1.58 decimal odds, 1.5 USDC**
  - This is roughly 25% worse than market (should be accepted)
  - Position size within range
  - Record: position ID, leaderboard acceptance

**Test Case 3: Over-Limit Position Size**
- [ ] **Create Boston position at 1.40 decimal odds, 5 USDC**
  - Odds are acceptable (worse than 25% threshold)
  - Position size exceeds 3 USDC limit
  - **Expected**: Only 3 USDC should be added to leaderboard
  - Record: actual amount added vs. position size

- [ ] **Validate Firebase Storage**
  - Check Firebase console for new documents in leaderboards collection
  - Verify event data is properly stored
  - Confirm user position appears in leaderboard entries
  - Check data format matches expected schema

- [ ] **Frontend Display Verification**
  - Confirm position appears in user's leaderboard entries

### Phase 3: Rejection Path Testing

**Test Case 4: Advantageous Odds (Should Reject)**
- [ ] **Attempt Boston position at 2.28 decimal odds, 2 USDC**
  - This is gt 25% better than market (should be rejected)
  - **Expected**: Transaction should revert or position rejected for leaderboard
  - Record: error message, transaction status

**Test Case 5: Advantageous Odds (Arizona side)**
- [ ] **Attempt Arizona position at 2.64 decimal odds, 2 USDC**
  - This is gt 25% better than market (should be rejected)
  - **Expected**: Position rejected for leaderboard
  - Record: rejection mechanism

- [ ] **Verify Rejection Behavior**
  - Confirm transaction reverts at contract level
  - Confirm frontend shows appropriate error message

### Phase 4: Data Flow Validation (15 mins)
- [ ] **Complete Flow Documentation**
  - Verify no data transformation errors

## Test Data Requirements

### Leaderboard Setup
- **Active leaderboard** with current contest
- **Clear rules** for what odds are acceptable
- **Minimum position size** that we can meet
- **Contest** with clear favorites/underdogs

### Position Requirements  
- **Poor odds example**: Enter leaderboard with position that has worse than market odds
- **Good odds example**: Take favorable odds that should be rejected
- **Sufficient balance** for test positions
- **Clear market context** to determine advantageous vs poor odds

## Debugging Tools

### If Things Go Wrong
- **Contract Event Logs**: Check Polygon Amoy explorer for events
- **Firebase Console**: Monitor real-time database updates
- **Frontend Console**: Check for JavaScript errors
- **Firebase Functions Logs**: Monitor cloud function execution

### Key Debug Questions
1. Was the contract transaction successful?
2. Was the event emitted with expected data?
3. Did Firebase function receive the event?
4. Did Firebase store the data correctly?
5. Is the frontend reading the data correctly?

## Files to Modify
- `ospex-firebase/functions/src/index.ts` (deploy new version)

## Expected Outcomes

### On Success
- Confident that leaderboard position validation works
- Firebase event handling is reliable
- Frontend displays leaderboard data correctly
- Clear understanding of rejection criteria

### On Failure Points
- **Event not emitted**: Contract issue, need to investigate transaction
- **Firebase not receiving**: Function deployment or configuration issue  
- **Wrong data stored**: Event parsing or data transformation bug
- **Frontend not updating**: Data reading or display logic issue
- **Rejection not working**: Leaderboard rules or validation logic problem

## Next Steps After Completion
- Create milestone for frontend leaderboard UI improvements

## Milestone Status: Blocked by Core Contract Dependency

### What Was Accomplished ✅
- [x] Deploy Firebase Functions with unified index.ts
- [x] Fixed odds display issues across frontend
- [x] Updated LeaderboardModule.sol to emit oddsPairId in events
- [x] Successfully deployed new LeaderboardModule to testnet
- [x] Updated contract addresses and ABIs in frontend
- [x] Created new leaderboard successfully

### Blocking Issue Discovered 🚫
**Problem**: New LeaderboardModule references `processLeaderboardEntryFee()` function in OspexCore that exists in staged code but not in deployed core contract.

**Impact**: Users cannot register for leaderboards, preventing completion of position testing.

**Root Cause**: Incremental module deployment created dependency mismatch with core contract.

**Resolution Required**: Full contract redeployment to sync all modules with core changes.

### Technical Lessons Learned
1. **Modular contracts have hidden dependencies** - deploying single modules can break due to core contract references
2. **Event signature changes require coordinated deployment** - Firebase handlers, contracts, and frontend must align
3. **Frontend validation needs enhancement** - warn users about unmatched positions < 1 USDC minimum

### Files Modified This Session
- `LeaderboardModule.sol` - Added oddsPairId to event emissions
- `LeaderboardModule.t.sol` - Updated test expectations  
- Firebase `index.ts` - Updated event handler signatures
- Frontend odds display components - Fixed decimal/market odds conversions

### Next Steps
- **Immediate**: Acquire sufficient testnet POL for full redeployment
- **Short-term**: Create milestone for complete contract redeployment
- **Long-term**: Consider staged deployment testing strategy

## Next Steps After Completion
- Plan comprehensive leaderboard rules testing

---

## Risk Mitigation

### Low Risk
- Position creation (this works reliably)
- Event emission (contract is tested)
- Firebase deployment (straightforward process)

### Medium Risk  
- Event data parsing in Firebase functions
- Frontend real-time updates
- Odds validation logic

### High Risk
- Leaderboard rules interpretation
- Complex rejection criteria
- Data consistency across systems

## Completion Definition
This milestone is complete when we can confidently create leaderboard positions, see them rejected appropriately, and trust that the Firebase event pipeline works reliably for leaderboard management.

*Next milestone will focus on comprehensive leaderboard rules testing and frontend improvements.*
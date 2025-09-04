# Milestone 002: Leaderboard Position Event Handler Testing
*Created: September 4, 2025*  
*Target Completion: September 4, 2025*  
*Status: 🟠 In Progress*

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

### Phase 2: Happy Path Testing (45 mins)
- [ ] **Create Poor Odds Position**
  - Use frontend to create position with odds disadvantageous to maker
  - Example: If market odds favor Team A at -150, create position favoring Team B at +200
  - Ensure position amount meets leaderboard minimum requirements
  - Record: position ID, odds entered, timestamp

- [ ] **Verify Event Emission**
  - Check contract event logs for LEADERBOARD_POSITION_ADDED event
  - Confirm event contains: leaderboardId, userId, positionId, odds data
  - Verify event timestamp and transaction hash

- [ ] **Validate Firebase Storage**
  - Check Firebase console for new document in leaderboards collection
  - Verify all event data is properly stored
  - Confirm user position appears in leaderboard entries
  - Check data format matches expected schema

- [ ] **Frontend Display Verification**
  - Refresh leaderboard page
  - Confirm position appears in user's leaderboard entries
  - Verify odds display correctly (no 2.60→2.59 issues)
  - Check leaderboard ranking calculation

### Phase 3: Rejection Path Testing (30 mins)
- [ ] **Attempt Advantageous Position Add**
  - Same contest, attempt to add the taker side of the position
  - This should have favorable odds and be rejected
  - Try to add to same leaderboard
  - Record: error message, transaction status

- [ ] **Verify Rejection Behavior**
  - Confirm transaction reverts at contract level
  - Check that no LEADERBOARD_POSITION_ADDED event is emitted
  - Verify Firebase doesn't receive invalid event
  - Confirm frontend shows appropriate error message

### Phase 4: Data Flow Validation (15 mins)
- [ ] **Complete Flow Documentation**
  - Document exact event data structure received
  - Verify no data transformation errors
  - Confirm timestamp handling is correct
  - Check odds format consistency

## Success Criteria

### Technical Acceptance
- [ ] Poor odds position successfully added to leaderboard
- [ ] LEADERBOARD_POSITION_ADDED event emitted with correct data
- [ ] Firebase receives and stores event data without errors
- [ ] Advantageous odds position is properly rejected
- [ ] No event emitted for rejected position
- [ ] Frontend correctly displays leaderboard position

### Data Integrity Acceptance  
- [ ] Event data in Firebase matches contract event exactly
- [ ] Position odds display correctly on frontend
- [ ] Leaderboard ranking calculation includes new position
- [ ] No 2.60→2.59 type display bugs
- [ ] Timestamp conversion works correctly

### User Experience Acceptance
- [ ] Clear error message for rejected advantageous position
- [ ] Leaderboard updates in real-time after position add
- [ ] User can see their position in leaderboard entries
- [ ] No confusing UI states or loading issues

## Test Data Requirements

### Leaderboard Setup
- **Active leaderboard** with current contest
- **Clear rules** for what odds are acceptable
- **Minimum position size** that we can meet
- **Contest** with clear favorites/underdogs

### Position Requirements  
- **Poor odds example**: Bet on underdog at worse than market odds
- **Good odds example**: Take favorable side that should be rejected
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
- Remove: `CFPv1.json`, `ContestOracleResolved.json` from repo

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
- Plan comprehensive leaderboard rules testing
- Document event handler architecture for future development
- Consider automated testing framework for event flows

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
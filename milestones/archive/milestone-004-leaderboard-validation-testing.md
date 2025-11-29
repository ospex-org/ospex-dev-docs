# Milestone 004: Leaderboard Position Validation Testing
*Created: September 7, 2025*  
*Completed: September 8, 2025*  
*Status: ✅ Complete*

## Problem Statement
With contracts redeployed and user registration working, complete the leaderboard position validation testing that was blocked in milestone 2. Verify that the 25% odds enforcement rule works correctly, Firebase captures events properly, and both acceptance/rejection flows function as designed.

## Context
Milestone 2 was blocked when users couldn't register for leaderboards due to core contract dependency issues. Milestone 3 successfully resolved this with complete contract redeployment. Now we can proceed with the comprehensive leaderboard testing that validates the core business logic.

## Scope & Boundaries

### In Scope
- Execute all test cases planned in milestone 2
- Validate 25% odds enforcement rule (acceptance and rejection)
- Verify Firebase event capture and storage
- Test position size limits (1-3 USDC enforcement)
- Confirm frontend displays leaderboard positions correctly
- Document complete event flow from contract to frontend

### Out of Scope
- New feature development
- Complex leaderboard rules beyond 25% enforcement
- Multiple leaderboard scenarios (focus on single leaderboard)
- Frontend UI improvements (unless blocking testing)

## Prerequisite Status Check

### Environment Verification
- [x] Contracts redeployed with working dependencies
- [x] User registration for leaderboards functional
- [x] Firebase event handlers updated for new contract addresses
- [x] Frontend connected to new contracts
- [X] Active leaderboard with 25% odds enforcement rule
- [ ] Current market data for Baltimore vs. Buffalo NFL

## Test Plan

### Phase 1: Test Environment Setup (20 mins)
- [X] **Verify Market Context**
  - Confirm Baltimore vs. Buffalo NFL game available
  - Record current market odds (should be close to: Baltimore -120, Buffalo -100)
  - Ensure speculation exists and is active

- [X] **Leaderboard Preparation**
  - Create new leaderboard with 25% odds enforcement
  - Set position limits: 1-3 USDC
  - Register test user for leaderboard
  - Verify registration appears in Firebase and frontend

### Phase 2: Acceptance Testing - Poor Odds Positions (45 mins)

**Market Reference**: Baltimore -120 (1.83 decimal), Buffalo -100 (2.00 decimal)
**25% Worse Threshold**: Baltimore ≤1.37, Buffalo <1.50

**Test Case 1: Baltimore Poor Odds (Edge Case)**
- [X] **Create Baltimore position at 1.37 decimal odds, 3 USDC**
  - Position should be accepted (significantly worse than market)
  - Record: position ID, transaction hash, timestamp
  - Expected taker stake: ~1.11 USDC (within 1 USDC minimum)

**New Market Reference**: Minnesota -125 (1.80 decimal), Chicago +105 (2.05 decimal)
**25% Worse Threshold**: Minnesota <1.35, Buffalo ≤1.53

**Test Case 2: Minnesota Excellent Odds**  
- [X] **Create Minnesota position at 2.26 decimal odds, 2 USDC**
  - Position should not be accepted (just over 25% threshold)
  - Record: position ID, transaction hash, timestamp

**Test Case 3: Position Size Limit Testing**
- [X] **Position created with faulty leaderboard**
  - Leaderboard was accidently created with 0.03 USDC as bet size max
  - Position size exceeded the limit but was capped appropriately (at 0.03)

### Phase 3: Firebase Event Validation (30 mins)
- [X] **Verify Event Capture**
  - Check Firebase console for LEADERBOARD_POSITION_ADDED events
  - Confirm all position data captured: oddsPairId, userId, amount
  - Verify timestamps and transaction references
  - Check event signatures match expected format

- [X] **Validate Data Integrity**
  - Confirm Firebase data matches contract event exactly
  - Verify position amounts (especially for 5 USDC → 3 USDC case)
  - Check that odds data is stored correctly
  - Ensure user references are accurate

### Phase 4: Frontend Display Verification (20 mins)
- [X] **Leaderboard Position Display**
  - Refresh leaderboard page after each position creation

## Success Criteria
This milestone is complete when all test cases execute as expected, the 25% odds enforcement rule works correctly for both acceptance and rejection, and the complete event pipeline from contract to frontend display functions reliably.

*This establishes confidence in core leaderboard functionality and enables advanced testing scenarios.*
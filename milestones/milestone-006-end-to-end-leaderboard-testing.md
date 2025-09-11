# Milestone 006: End-to-End Leaderboard Testing
*Created: September 10, 2025*  
*Target Completion: September 11, 2025*  
*Status: 🚧 In Progress*

## Problem Statement
The remaining leaderboard lifecycle functions need testing and frontend integration. Users currently cannot update existing leaderboard positions when their matched amount increases, submit ROI for competition scoring, or claim prizes when leaderboards end.

## Context
Previous milestones established leaderboard position creation and validation. This milestone completes the leaderboard feature set by testing and implementing the three remaining core functions: position amount increases, ROI submission for scoring, and prize claiming.

## Scope & Boundaries

### In Scope
- Test `increaseLeaderboardPositionAmount()` function and create event handler
- Test `submitLeaderboardROI()` function and create event handlers
- Test `claimLeaderboardPrize()` function and create event handler
- Create frontend UI for all three new functions
- End-to-end testing of complete leaderboard lifecycle

### Out of Scope
- Admin sweep functionality
- New validation rules or business logic changes
- Multiple leaderboard complex scenarios
- Performance optimizations

## Implementation Plan

### Phase 1: Position Amount Increase Testing (45 mins)

**Background**: When a user adds a position to a leaderboard with partial matching (e.g., 2 USDC matched out of 5 USDC total), they should be able to increase the leaderboard amount when more gets matched.

- [X] **Test Scenarios**
  - Create position with 5 USDC, only 2 USDC matched initially
  - Add position to leaderboard (should register 2 USDC)
  - Get remaining 3 USDC matched by another user
  - Call `increaseLeaderboardPositionAmount()` (should update to max of allowable USDC amount)
  - Observe event for event hash

- [ ] **Create Firebase Event Handler**
  - Add `LEADERBOARD_POSITION_UPDATED` event handler to `ospex-fdb/functions/src/index.ts`
  - Handle event data: `speculationId, user, oddsPairId, cappedAmount, positionType, leaderboardId`
  - Update existing leaderboard position amount in Firebase
  - Verify Firebase position amount updated correctly

- [ ] **Frontend Integration**
  - Add "Update Leaderboard Amount" button to positions that can be increased
  - Show current leaderboard amount vs. total matched amount
  - Handle transaction flow and success/error states

### Phase 2: ROI Submission Testing (60 mins)

**Background**: After a leaderboard ends, users submit their ROI during the claim window. The system tracks the highest ROI and winners list for prize distribution.

- [ ] **Test Scenarios**
  - Create leaderboard that expires soon (maybe EOD 20250911)
  - Add multiple positions for different users
  - Calculate expected ROI manually
  - Submit ROI via `submitLeaderboardROI()`
  - Verify highest ROI tracking and ties handling
  - Observe events for event hash

- [ ] **Create Firebase Event Handlers**
  - Add `LEADERBOARD_ROI_SUBMITTED` event handler
    - Handle: `leaderboardId, user, roi`
    - Store ROI submissions in Firebase
  - Add `LEADERBOARD_NEW_HIGHEST_ROI` event handler  
    - Handle: `leaderboardId, roi, user`
    - Update leaderboard highest ROI and winners list

- [ ] **Frontend Integration**
  - Add ROI submission form to completed leaderboards
  - Calculate and display expected ROI based on position outcomes
  - Show current highest ROI and winners
  - Handle claim window timing

### Phase 3: Prize Claiming Testing (45 mins)

**Background**: Users who submitted winning ROI can claim their share of the prize pool during the claim window.

- [ ] **Test Scenarios**
  - Complete ROI submission phase with clear winner
  - Calculate expected prize share (will likely be one winner taking all)
  - Test non-winner attempting to claim (should revert)
  - Call `claimLeaderboardPrize()` as winner
  - Verify USDC transfer and event emission

- [ ] **Create Firebase Event Handler**
  - Add `LEADERBOARD_PRIZE_CLAIMED` event handler
  - Handle: `leaderboardId, user, share`
  - Track claimed prizes in Firebase

- [ ] **Frontend Integration**
  - Show "Claim Prize" button for eligible winners
  - Display prize amount and claiming status
  - Handle transaction flow and confirmation

## Success Criteria

### Technical Acceptance
- [ ] All three functions work correctly with proper error handling
- [ ] Firebase event handlers capture and store all event data
- [ ] Frontend displays and handles all new functionality
- [ ] Complete leaderboard lifecycle functional end-to-end

### User Experience Acceptance  
- [ ] Users can easily update leaderboard positions when matched amounts increase
- [ ] ROI submission process is clear and calculates correctly
- [ ] Prize claiming is straightforward for winners
- [ ] Error states provide helpful feedback

## Expected Events & Data Flow

### Position Update Flow
```
increaseLeaderboardPositionAmount() → LEADERBOARD_POSITION_UPDATED → Firebase → Frontend refresh
```

### ROI Submission Flow  
```
submitLeaderboardROI() → LEADERBOARD_ROI_SUBMITTED + LEADERBOARD_NEW_HIGHEST_ROI → Firebase → Leaderboard scoring update
```

### Prize Claiming Flow
```
claimLeaderboardPrize() → LEADERBOARD_PRIZE_CLAIMED → Firebase → User balance update
```

## Files to Modify
- `ospex-fdb/functions/src/index.ts` - Add 4 new event handlers
- Frontend components for leaderboard management
- `ospex-frontend-v2.2-matched-pairs/src/utils/` - New utility functions for the three operations

## Test Cases
1. **Position Increase**: 2 USDC → (2 + x) USDC leaderboard amount update
2. **ROI Submission**: Multiple users submit ROI, verify winner tracking  
3. **Prize Claiming**: Winner claims correct prize share
4. **Complete Lifecycle**: Full leaderboard from creation to prize distribution

## Risk Assessment
- **Medium risk**: Firebase event handlers need to properly update existing records
- **Low risk**: Frontend integration builds on existing patterns
- **Low impact**: Non-breaking additions to existing functionality

## Completion Definition
This milestone is complete when users can successfully manage their leaderboard positions through increase/decrease cycles, submit ROI for scoring, and claim prizes, with all functionality working reliably through the complete frontend experience.

*This completes the core leaderboard feature set and enables full leaderboard competition functionality.*

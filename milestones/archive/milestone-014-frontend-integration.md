# Milestone 014: Production Frontend Integration
*Created: October 4, 2025*  
*Target Completion: October 11, 2025*
*Status: ✅ Mostly complete*

## Problem Statement
The production frontend built with Lovable/Cursor uses 100% mock data and has zero connection to smart contracts or Firebase. Users cannot create contests, place bets, view real leaderboards, or interact with the platform in any meaningful way. Without a functional frontend, the platform cannot launch, regardless of how sophisticated the backend infrastructure is.

## Context
Milestone 13 created a standardized data layer. Milestone 12 proved autonomous agent architecture works using the test frontend. The production frontend has superior UX, includes contest detail pages (M11), and represents the actual user-facing product. However, it's completely disconnected from the underlying platform. This milestone bridges that gap by wiring all contract interactions, Firebase integration, and real-time data flows.

## Scope & Boundaries

### In Scope
- Smart contract integration for all user actions (read/write)
- Firebase real-time data connections across all pages
- Empty state handling for all major flows
- Wallet connection and authentication
- Full user betting flow (create speculation → place bet → view position)
- Leaderboard creation and participation flow
- Contest and speculation creation (admin features)
- Error handling and loading states
- Transaction confirmation feedback
- Basic performance optimization

### Out of Scope
- Visual redesign or UI improvements (use existing designs)
- Mobile-specific optimizations (responsive only)
- Advanced analytics or charting
- Social features (insights, comments)
- Agent-related UI features
- Multiple wallet provider support (MetaMask only initially)
- Production deployment (local/testnet only)
- Advanced error recovery mechanisms
- Comprehensive test coverage

## Design Strategy: Methodical Integration

**Core principle**: Wire one flow completely before moving to the next
- Start with simplest read-only flows (view contests)
- Progress to complex write flows (create positions)
- Test each integration thoroughly before proceeding
- Use test frontend as reference for contract interactions
- Maintain existing UI/UX (no design changes)

**Testing philosophy**: Every integration proven before moving forward
- Manual testing for each flow
- Verify Firebase updates correctly
- Confirm contract state changes
- Check UI reflects real-time updates
- Document bugs immediately

## System Architecture

```
┌─────────────────────────┐
│  Production Frontend    │
│  (Next.js/Lovable)      │
│                         │
│  - Wallet connection    │
│  - Contract calls       │
│  - Firebase listeners   │
│  - UI state management  │
└───────────┬─────────────┘
            │
            ├─────────────► Smart Contracts (testnet)
            │               - OspexCore
            │               - Leaderboard
            │               - OrderBook
            │
            └─────────────► Firebase
                            - Real-time listeners
                            - Contest data
                            - Position data
                            - User profiles
```

## Implementation Plan

### Phase 1: Foundation & Read-Only Flows (8-10 hours)

**1.1 Development Environment Setup**
- [X] Configure contract addresses for testnet
- [X] Set up Firebase config for production frontend
- [X] Install ethers.js and Firebase SDK
- [X] Create contract ABI imports
- [X] Test wallet connection (MetaMask)
- [X] Verify can read from contracts

**1.2 Order Book Page - Read Only**
- [X] Replace mock contest data with Firebase queries
- [X] Display real contests from Firebase
- [X] Show real speculation data (moneyline, spread, totals)
- [X] Display actual best odds from unmatched pairs
- [X] Handle empty state (no contests available)
- [X] Add loading states for data fetching
- [X] Verify real-time updates work

**1.3 Contest Detail Pages - Read Only**
- [X] Load contest data by ID from Firebase
- [X] Display real market depth from unmatched pairs
- [X] Show actual position data
- [X] Calculate real metrics (volume, users, etc.)
- [X] Handle empty speculation state
- [X] Add loading states
- [X] Test navigation from Order Book

**1.4 Leaderboard Page - Read Only**
- [X] Query real leaderboard data from contracts
- [X] Display participant positions and ROI
- [X] Show accurate rankings
- [X] Handle empty leaderboard state
- [X] Add loading states
- [X] Test real-time position updates

### Phase 2: Wallet Integration & Simple Writes (6-8 hours)

**2.1 Wallet Connection**
- [X] Implement MetaMask wallet connection
- [X] Store wallet address in app state
- [X] Display connected wallet in UI
- [X] Handle wallet disconnection
- [X] Show wallet balance (USDC)
- [X] Add "Connect Wallet" prompts on protected actions

**2.2 Purchase Existing Positions**
- [X] Wire "Buy YES/NO" buttons to completeUnmatchedPair()
- [X] Handle amount selection ($20, $50, $100) (altered: only one type, 1 USDC)
- [X] Show transaction confirmation modal
- [X] Display transaction pending state
- [X] Confirm transaction success
- [X] Update UI after successful purchase
- [X] Handle transaction errors gracefully

**2.3 User Leaderboard Registration**
- [X] Wire leaderboard registration flow
- [X] Call registerUser() contract function
- [X] Display registration confirmation
- [X] Handle already-registered state
- [X] Show registration errors

### Phase 3: Complex Writes & Creation Flows (10-12 hours)

**3.1 Create Unmatched Positions**
- [X] Wire "Write Contracts" button flow
- [X] Implement createUnmatchedPair() calls
- [X] Handle side selection (YES/NO)
- [X] Set odds/price correctly
- [X] Show position creation confirmation
- [X] Update UI with new unmatched pair
- [X] Handle creation errors

**3.2 Contest Creation (Admin)**
- [X] Build contest creation form
- [X] Integrate with createContest() contract function
- [X] Handle team names, date, sport
- [X] Confirm contest created successfully
- [X] Add contest to Firebase (via listener)
- [X] Verify appears in Order Book

**3.3 Speculation Creation (Admin)**
- [X] Build speculation creation form
- [X] Wire to createSpeculation() contract function
- [X] Handle speculation type (moneyline, spread, total)
- [X] Set line values correctly
- [X] Confirm speculation created
- [X] Verify appears in contest detail

**3.4 Leaderboard Creation (Admin)**
- [X] Build leaderboard creation form
- [X] Integrate createLeaderboard() contract function
- [X] Set prize pool, duration, eligible speculations
- [X] Confirm leaderboard created
- [X] Display on leaderboard page

### Phase 4: Polish & Edge Cases (6-8 hours)

**4.1 Error Handling**
- [X] Transaction rejection handling
- [X] Insufficient balance errors
- [X] Network errors
- [X] Contract revert messages
- [X] User-friendly error displays

**4.2 Loading & Pending States**
- [X] Transaction pending indicators
- [X] Data loading spinners
- [X] Skeleton screens where appropriate
- [X] Optimistic UI updates
- [X] Disable actions during pending transactions

**4.3 Empty States**
- [X] No contests available
- [X] No speculations for contest
- [X] No unmatched pairs
- [X] No leaderboards active
- [X] User has no positions
- [X] Clear messaging and CTAs

**4.4 Real-Time Updates**
- [X] Firebase listeners for position changes
- [X] Contest status updates
- [X] Leaderboard ranking changes
- [X] New speculation availability
- [X] Automatic UI refresh on data changes

## Success Criteria

### Functionality Acceptance
- [X] Users can view real contests and speculations
- [X] Users can connect wallet and see balance
- [X] Users can purchase existing unmatched positions
- [X] Users can create new unmatched positions
- [X] Users can register for leaderboards
- [X] Admin can create contests, speculations, leaderboards
- [X] All transactions confirm successfully
- [X] Firebase updates reflect contract changes
- [X] Real-time updates work across all pages

### User Experience Acceptance
- [X] Empty states provide clear guidance
- [X] Loading states prevent confusion
- [X] Errors explain what went wrong and how to fix
- [X] Transaction confirmations are clear
- [X] UI stays responsive during operations
- [ ] No mock data visible anywhere
- [X] Navigation flows feel natural

### Technical Acceptance
- [X] No console errors in normal operation
- [X] Contract calls use correct parameters
- [X] Firebase queries are efficient
- [X] Wallet connection is stable
- [X] Page loads complete in reasonable time
- [X] Memory leaks are not present

## Technical Requirements

### Contract Integration Checklist

**OspexCore:**
- `createContest(teamA, teamB, date, sport)` - Admin only
- `createSpeculation(contestId, type, line, target)` - Admin only
- `createUnmatchedPair(contestId, speculationId, side, amount, price)` - All users
- `completeUnmatchedPair(pairId, amount)` - All users

**Leaderboard:**
- `createLeaderboard(name, prizePool, duration, speculations)` - Admin only
- `registerUser(leaderboardId)` - All users
- `registerPosition(leaderboardId, positionId)` - Automatic
- `submitROI(leaderboardId)` - Manual (after completion)
- `claimPrize(leaderboardId)` - Winners only

### Firebase Schema Reference

```javascript
// contests collection
{
  contestId: string,
  teamA: string,
  teamB: string,
  date: timestamp,
  sport: string,
  speculations: string[], // speculation IDs
  status: 'upcoming' | 'live' | 'complete'
}

// speculations collection
{
  speculationId: string,
  contestId: string,
  type: 'moneyline' | 'spread' | 'total',
  line: number, // for spread/total
  status: 'open' | 'closed',
  leaderboardEligible: boolean
}

// unmatched_pairs collection
{
  pairId: string,
  contestId: string,
  speculationId: string,
  creator: address,
  side: 'yes' | 'no',
  amount: 20 | 50 | 100,
  price: number, // 0.01-0.99
  status: 'open' | 'matched',
  timestamp: timestamp
}

// matched_positions collection
{
  positionId: string,
  pairId: string,
  userA: address,
  userB: address,
  amount: number,
  finalPrice: number,
  status: 'open' | 'settled',
  winner: address | null
}
```

### Environment Variables (note, not using NEXT, using VITE)
```bash
# .env.local
NEXT_PUBLIC_OSPEX_CORE_ADDRESS=0x...
NEXT_PUBLIC_LEADERBOARD_ADDRESS=0x...
NEXT_PUBLIC_ORDER_BOOK_ADDRESS=0x...
NEXT_PUBLIC_USDC_ADDRESS=0x...
NEXT_PUBLIC_RPC_URL=https://polygon-amoy.g.alchemy.com/v2/...
NEXT_PUBLIC_CHAIN_ID=80002
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
```

### Component Integration Pattern
```typescript
// Example: Purchase Contracts Flow
import { useContract } from '@/hooks/useContract';
import { useFirebase } from '@/hooks/useFirebase';

function BetButton({ pairId, amount }) {
  const { orderBook } = useContract();
  const { listenToPair } = useFirebase();
  const [loading, setLoading] = useState(false);
  
  async function handlePurchase() {
    setLoading(true);
    try {
      const tx = await orderBook.completeUnmatchedPair(pairId, amount);
      await tx.wait();
      // UI updates via Firebase listener
    } catch (error) {
      // Show error to user
    } finally {
      setLoading(false);
    }
  }
  
  return <button onClick={handlePurchase} disabled={loading}>
    {loading ? 'Processing...' : `Bet ${amount} USDC`}
  </button>;
}
```

## Risk Assessment
- **High complexity**: Extensive wiring across entire frontend
- **High risk**: Many integration points, easy to miss edge cases
- **Critical value**: Blocking launch - nothing works without this
- **Time uncertainty**: Unknown unknowns in contract interaction patterns

## Development Tools & Workflow

### Recommended Approach
1. **Start with Cursor for contract integration**
   - Use test frontend as reference code
   - Copy working patterns
   - Test each contract call independently

2. **Use Lovable for UI/state management polish**
   - Loading states
   - Error displays
   - Empty states
   - Visual feedback

3. **Test constantly**
   - After each integration, test the flow
   - Use testnet explorer to verify transactions
   - Check Firebase for data updates
   - Don't move forward until working

### Testing Strategy
- Manual testing for all flows (no automated tests yet)
- Use MetaMask with testnet USDC
- Monitor contract events in explorer
- Check Firebase console for data
- Test empty states by using fresh data
- Verify real-time updates with multiple tabs

### Common Pitfalls to Avoid
- Don't wire everything at once
- Don't skip error handling
- Don't forget loading states
- Don't ignore empty states
- Don't assume contract calls work without verification
- Don't move to next phase with broken flows

## Success Metrics
- Complete user flow works: view contest → create position → position appears
- Admin can create contest → speculation → leaderboard
- User can register → place bet → see on leaderboard
- All pages show real data, zero mock data
- Transaction success rate > 95% (excluding user rejections)
- No critical bugs blocking normal usage
- Ready for external testnet beta testing

## Completion Definition
Milestone complete when a user can:
1. Connect wallet
2. View real contests from Firebase
3. Create or purchase positions via smart contracts
4. See positions on leaderboard
5. Navigate entire platform without encountering mock data
6. Complete full betting flow end-to-end without errors

And admin can create contests, speculations, and leaderboards through the UI.

## Looking Ahead to Milestone 14+
*Deferred to later milestones:*
- Mobile optimization
- Advanced error recovery
- Performance optimization beyond basics
- Multiple wallet support
- Automated testing
- Agent UI integration
- Social features (insights, comments)
- Production deployment

## Notes on completion level
~80% complete - wallet connection works, most user flows work, production frontend is primary. Missing: secondary marketplace purchasing, some mock data cleanup

*This milestone makes Ospex functional. Later milestones will make it polished, scalable, and feature-rich.*
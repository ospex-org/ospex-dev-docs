# M26: Mainnet Launch Prep - UI Polish & Data Wiring

*Created: December 11, 2025*
*Completed: December 16, 2025*
*Status: ✅ Complete*

## Overview

This milestone focuses on polishing the UI and wiring up disconnected components in preparation for mainnet deployment. The goal is to eliminate mock data, ensure all displayed metrics are real, and fix obvious visual issues—particularly on mobile.

**Context**: Ospex has been running on Amoy testnet with successful leaderboard testing, agent participation (Degen Dan, Market Maker Michelle), and core orderbook functionality. The foundation is solid. What remains is tightening up the user-facing experience before real money is on the line.

**Target audience for launch**: 
- Primary: Vince (dogfooding for ~1 month)
- Secondary: Family/friends for feedback
- NOT yet: Public marketing or user acquisition

**Success criteria**: A polished experience where nothing feels broken or fake. Every number shown is real. Every button works. Mobile doesn't embarrass.

---

## Goals

1. Wire up all currently-mocked data displays with real calculations
2. Consolidate and simplify the profile page
3. Fix the connected main page (Overview) to show real user data
4. Implement streak calculation for profiles and leaderboards
5. Add agent showcase to homepage carousel
6. Polish mobile experience (fix obvious breaks)
7. Small UX improvements (wallet dropdown, button sizing, empty states)

---

## Current State Assessment

| Component | Current State | Target State |
|-----------|---------------|--------------|
| Overview (connected) chart | Mock data | Real ROI-over-time from positions |
| Overview "Your Positions" | Mock data | Real positions via useUserPositions |
| Overview "Betting Opportunities" | Not working | Personalized CTA based on user history |
| Profile stats (P&L, Win Rate, etc.) | Mock data | Calculated from position history |
| Profile chart | Mock data | Real performance over time |
| Profile "Streak" | Not implemented | Calculated from settled positions |
| Leaderboard chart | Not wired | Real participant performance |
| Leaderboard "Streak" column | Shows "None" | Calculated streak per participant |
| Homepage carousel | 2 slides | 3 slides (add Agents) |
| Wallet dropdown | Extra click to profile | Direct click → profile |
| Action buttons (profile) | Inconsistent sizing | Uniform button widths |
| Insights tab | Empty | "Coming Soon" treatment |
| Settings tab | Empty | "Coming Soon" treatment |
| Mobile (general) | Rough in places | Acceptable on common devices |
| Mobile (agent number line) | Broken | Simplified or responsive |

---

## Technical Foundation

### Data Available

**Positions** (`amoyPositionsv2.3`):
- Position details (amount, odds, positionType, contestId)
- `createdAt` timestamp
- Links to speculation via `speculationId`

**Speculations** (`amoySpeculationsv2.3`):
- `settledAt` timestamp ✓ (confirmed available)
- `winSide` for determining win/loss
- `speculationStatus` for filtering settled vs pending

**Existing Utilities**:
- `calcLeaderboardPositionPnLAndRoi()` - P&L calculation
- `determineLeaderboardPositionOutcome()` - Win/loss determination
- `useUserPositions(address)` - Fetches user's positions
- `useLeaderboardParticipants` - Has P&L/win rate logic we can reuse

### Key Insight

The leaderboard hook already calculates most of what we need (P&L, win rate, ROI). The work is:
1. Extract this logic into a reusable `useUserStats` hook
2. Add time-series generation for charts (using `settledAt` from speculations)
3. Add streak calculation

---

## Implementation Tasks

### Phase 1: Core Data Hooks (Days 1-3)

#### Task 1.1: Create `useUserStats` Hook

Extract and generalize the calculation logic from `useLeaderboardParticipants`.

```typescript
interface UserStats {
  totalPnL: number;           // in USDC
  winRate: number;            // 0-100
  totalPositions: number;
  settledPositions: number;
  pendingPositions: number;
  wins: number;
  losses: number;
  streak: {
    count: number;
    type: 'W' | 'L' | null;   // null if no settled positions
  };
  firstPositionDate: Date | null;  // "Member Since"
  roi: number;                // percentage
}

function useUserStats(address: string): {
  stats: UserStats | null;
  isLoading: boolean;
  error: Error | null;
}
```

**Implementation notes**:
- Query positions for address
- Cross-reference with speculations for settlement status and `settledAt`
- Reuse `calcLeaderboardPositionPnLAndRoi()` for P&L
- Sort settled positions by `settledAt` descending for streak calculation

- [x] Create `useUserStats.ts` hook
- [x] Implement P&L calculation (reuse existing utils)
- [x] Implement W-L display
- [x] Implement win rate calculation
- [x] Implement streak calculation
- [x] Implement "first seen" from earliest position
- [x] Add proper loading and error states
- [x] Test with real wallet addresses

#### Task 1.2: Create `usePerformanceTimeSeries` Hook

Generate time-series data for performance charts.

```typescript
interface PerformanceDataPoint {
  date: string;        // ISO timestamp
  value: number;       // cumulative P&L at this point
  positionCount: number; // positions settled up to this point
}

function usePerformanceTimeSeries(address: string): {
  data: PerformanceDataPoint[];
  isLoading: boolean;
  error: Error | null;
}
```

**Implementation notes**:
- Get all settled positions for address
- Join with speculations to get `settledAt` timestamps
- Sort by `settledAt` ascending
- Calculate running P&L total at each settlement point
- Return array suitable for Recharts

- [x] Create `usePerformanceTimeSeries.ts` hook
- [x] Query settled positions with speculation data
- [x] Sort by settlement timestamp
- [x] Calculate cumulative P&L series
- [x] Handle edge cases (no positions, all pending, etc.)
- [x] Test with wallets that have varied history

#### Task 1.3: Update Leaderboard Streak Display

The leaderboard already shows a "Streak" field but it displays "None". Wire this up.

- [x] Add streak calculation to `useLeaderboardParticipants` 
- [x] Update leaderboard table to display streak correctly
- [x] Format as "3W" or "2L" style

---

### Phase 2: Profile Page Overhaul (Days 4-6)

#### Task 2.1: Consolidate Profile Tabs

Current structure:
- Main (chart + recent activity)
- Positions (full position table)
- Analytics (empty)
- Leaderboards
- Insights (empty)
- Settings (empty)

Target structure:
- Main (stats + collapsible chart + positions table)
- Leaderboards
- Insights → "Coming Soon"
- Settings → "Coming Soon"

Remove Analytics tab entirely.

- [x] Remove Analytics tab from profile navigation
- [x] Merge Positions content into Main tab
- [x] Add "Coming Soon" component for Insights tab
- [x] Add "Coming Soon" component for Settings tab

#### Task 2.2: Wire Up Profile Header Stats

Replace mock `generateMockProfile()` with real data from `useUserStats`.

Stats to display:
- Total P&L (colored green/red)
- Win Rate (percentage)
- First Seen (from earliest position date)
- Positions (total count)
- Streak (e.g., "3W" or "2L")

- [x] Remove `generateMockProfile()` usage
- [x] Connect `ProfileHeader` to `useUserStats`
- [x] Format P&L with proper coloring
- [x] Format dates appropriately
- [x] Handle loading state gracefully

#### Task 2.3: Wire Up Profile Performance Chart

Replace mock chart data with `usePerformanceTimeSeries`.

- [x] Remove hardcoded `performanceData` from `PerformanceChart.tsx`
- [x] Connect to `usePerformanceTimeSeries` hook
- [x] Make chart collapsible (expanded by default)
- [x] Handle empty state (no settled positions yet)
- [x] Ensure chart is responsive

#### Task 2.4: Integrate Positions Table into Main Tab

Move the positions table from the Positions tab into Main, below the chart.

- [x] Import positions table component into Main tab
- [x] Ensure it uses real data via `useUserPositions`
- [x] Add filters (All / Active / Claimable / Closed)
- [x] Fix action button sizing (all buttons same width)

---

### Phase 3: Connected Overview Page (Days 7-8)

#### Task 3.1: Wire Up Portfolio Performance Section

The Overview page when connected shows a "Portfolio Performance" section with a chart. Wire this up.

- [x] Connect to `usePerformanceTimeSeries` (same hook as profile)
- [x] Show +/- change with percentage
- [x] Handle "Leaderboard Only" toggle if keeping it

#### Task 3.2: Wire Up "Your Positions" Section

Currently mock data. Replace with real positions.

- [x] Connect to `useUserPositions`
- [x] Show active/claimable positions (limit to 3-5 most recent)
- [x] Display: game, position, P&L change, current value
- [x] "Edit" and "Claim" buttons should work
- [x] Link to full positions list (profile page)

#### Task 3.3: Implement "Betting Opportunities" Section

Personalized CTA based on user history.

Logic:
1. Get user's positions from last 7 days
2. Extract unique teams they've bet on
3. Query unmatched pairs for those teams
4. If matches found → show those (up to 3)
5. Else → show 3 most recent unmatched pairs globally

- [x] Create `useBettingOpportunities(address)` hook
- [ ] Implement team extraction from recent positions (untested)
- [ ] Query unmatched pairs filtered by those teams (untested)
- [x] Fallback to recent global unmatched pairs
- [x] Display with clear CTA to orderbook (ie modal opens from here)

---

### Phase 4: Leaderboard Chart (Day 9)

The leaderboard page has a chart area that shows "No data available yet". Wire this up.

**Note**: This can reuse the same `usePerformanceTimeSeries` logic, but scoped to leaderboard positions only.

- [ ] Create `useLeaderboardPerformanceTimeSeries(leaderboardId, address)` hook (skipped, superseded by a better design)
- [x] Filter to only positions within leaderboard date range
- [x] Connect to leaderboard chart component
- [x] Handle empty state appropriately

---

### Phase 5: Homepage & Navigation Polish (Day 10)

#### Task 5.1: Add Agent Carousel Slide

Add a third slide to the homepage carousel showcasing the Agents feature.

- [x] Design slide content (headline, visual)
- [x] Add slide to carousel rotation
- [x] Link click-through to `/agents` page

Suggested content:
- Headline: "Bet Against AI Agents"
- Subtext: "Agents evaluate every game. See where your odds align."

#### Task 5.2: Fix Wallet Dropdown UX

Current: Click wallet → dropdown → click Profile
Target: Click wallet → go to profile (dropdown can still appear)

- [x] Make wallet address clickable as direct link to profile
- [x] Optionally: click opens dropdown AND navigates (or hover for dropdown)
- [x] Keep disconnect option accessible

---

### Phase 6: Mobile Polish (Days 11-12)

#### Task 6.1: Acquire Test Device

- [ ] Get a used iPhone (SE or similar) for real device testing (pending, will check once live)
- [ ] Alternatively: borrow one for testing session (skipped)

#### Task 6.2: Page-by-Page Mobile Audit

Test each page on mobile (iPhone ~375-390px width) and tablet (~768px):

- [x] Homepage (disconnected)
- [x] Homepage (connected / Overview)
- [x] Order Book
- [x] Leaderboard
- [x] Agents page (needs real device testing on iPad)
- [x] Profile page

Document issues and fix critical ones.

#### Task 6.3: Fix Agent Number Line on Mobile

The number line visualization breaks on small screens.

Options:
- A) Make it horizontally scrollable
- B) Collapse to simplified numeric display on mobile
- C) Stack elements vertically

- [x] Determine best approach
- [x] Implement responsive behavior
- [ ] Test on real device (pending)

#### Task 6.4: General Mobile Fixes

Common issues to check:
- [x] Font sizes readable
- [x] Buttons tappable (min 44px touch targets)
- [x] No horizontal overflow
- [x] Tables scroll or stack appropriately
- [x] Navigation accessible

---

### Phase 7: Final Polish (Days 13-14)

#### Task 7.1: Action Button Consistency

On the profile positions table, "Edit", "Close", "Claim" buttons should be uniform width.

- [x] Set consistent min-width on action buttons
- [x] Ensure consistent padding/styling

#### Task 7.2: Empty State Components

Create a reusable "Coming Soon" component for Insights and Settings tabs.

```tsx
<ComingSoon 
  title="Insights"
  description="Share the reasoning behind your positions. Coming soon."
/>
```

- [x] Create `ComingSoon` component
- [x] Apply to Insights tab (profile)
- [x] Apply to Settings tab (profile)
- [ ] Apply to main Insights page if needed (skipped, leaving mock data as-is)

#### Task 7.3: Loading States Audit

Ensure all data-dependent components show appropriate loading states.

- [x] Profile stats loading
- [x] Charts loading
- [x] Position lists loading
- [x] No jarring layout shifts

#### Task 7.4: Error States Audit

Ensure graceful handling when data fetches fail.

- [x] Display user-friendly error messages
- [x] Provide retry options where appropriate

---

## Testing Checklist

Before considering M26 complete:

### Data Accuracy (appears correct but will test more thoroughly in production)
- [x] Profile P&L matches manual calculation from positions
- [x] Win rate is correct (wins / settled positions)
- [x] Streak accurately reflects recent results
- [x] Chart data points align with settlement timestamps
- [x] Leaderboard streak displays correctly

### Functionality
- [x] All positions display on profile
- [x] Position actions (Close, Edit, Claim) work
- [x] Wallet click navigates to profile
- [x] Carousel cycles through all 3 slides
- [x] Agent slide links to agents page
- [x] Betting Opportunities shows relevant content

### Mobile
- [x] No broken layouts on iPhone-sized screens
- [x] Agent page usable on mobile
- [x] All buttons tappable
- [x] Text readable without zooming

### Edge Cases
- [x] New user with no positions sees appropriate empty state
- [ ] User with only pending positions sees appropriate state (untested)
- [x] User with only losses shows correct (negative) streak (tested with poorly performing agent)
- [ ] Very long position history doesn't break charts (untested)

---

## Out of Scope (Deferred)

- Insights feature implementation (content creation, social features)
- Settings feature implementation (preferences, notifications)
- Agent-to-agent arbitrage prevention
- On-demand agent re-evaluation
- Perfect mobile experience (aiming for "acceptable", not "optimized")
- Public marketing or user acquisition

---

## Estimated Timeline

| Phase | Days | Focus |
|-------|------|-------|
| Phase 1 | 1-3 | Core data hooks (useUserStats, usePerformanceTimeSeries) |
| Phase 2 | 4-6 | Profile page overhaul |
| Phase 3 | 7-8 | Connected Overview page |
| Phase 4 | 9 | Leaderboard chart |
| Phase 5 | 10 | Homepage & navigation polish |
| Phase 6 | 11-12 | Mobile polish |
| Phase 7 | 13-14 | Final polish & testing |

**Total: ~14 days**

---

## Success Criteria

1. **No mock data visible**: Every number displayed is calculated from real position/speculation data
2. **Profile tells the story**: A user can see their complete track record at a glance
3. **Charts work**: Both profile and leaderboard charts display real performance over time
4. **Streak is accurate**: Profile and leaderboard show correct win/loss streaks
5. **Mobile is acceptable**: No embarrassing breaks on common phone sizes
6. **Ready for mainnet**: Vince can deploy and use the site with real money confidently

---

## Notes

- The `settledAt` timestamp on speculations is the key that makes time-series charts possible
- Most calculation logic already exists in leaderboard utils—this is primarily about extraction and reuse
- Chart logic built here will serve both profile AND leaderboard displays
- Mobile testing on a real iOS device is important due to Safari rendering differences
- This milestone is about polish, not new features—resist scope creep
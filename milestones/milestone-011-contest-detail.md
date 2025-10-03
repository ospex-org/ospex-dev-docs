# Milestone 011: Contest Detail Pages
*Created: September 28, 2025*  
*Target Completion: October 2, 2025*
*Status: 🔄 In Progress*

## Problem Statement
Power users and returning users need detailed market information that isn't visible on the main Order Book page. Currently, users can only see the best odds for each speculation and must filter between payout amounts using pills. Market depth, contract availability across all price levels, and comprehensive contest information are hidden, limiting informed decision-making.

## Context
Milestone 10 completed the user insights system. Contest detail pages provide the "deep dive" capability for users who want comprehensive market analysis before placing bets. This serves power users without complicating the simplified Order Book interface for new users.

## Scope & Boundaries

### In Scope
- Contest detail pages accessible via clickable contest headers on Order Book cards
- Comprehensive display of all speculation types (moneyline, spread, totals) on single page
- Market depth details showing contract availability at different odds/amounts
- Integration with existing Purchase/Write Contracts modals
- Empty state handling with contest/speculation creation buttons
- Real-time updates via Firebase integration
- Market odds comparison display (jsonodds.com integration)

### Out of Scope
- Speculation-specific detail pages (contest-level aggregation only)
- Historical analytics or charting
- Advanced admin interfaces for contest management
- Mobile-specific optimizations (maintain responsive design)
- Content moderation for contest-specific discussions

## Design Strategy: Comprehensive Yet Approachable

**Core principle**: Show complete market information without overwhelming users
- All speculation types visible simultaneously (no tabs)
- Market depth displayed in scannable format
- Maintain existing betting flow (same modals)
- Clear visual hierarchy prioritizing most relevant information

**Navigation philosophy**: Subtle access for interested users
- Contest headers become clickable links
- Maintain existing navbar (grayed but functional)
- Page feels like separate destination, not modal overlay

## Implementation Plan

### Phase 1: Visual Design and Layout (Day 1 - Lovable)
- [X] **Multiple Design Options**
  - Create 2-3 different layout approaches for market depth display
  - Experiment with table vs visual vs summary+detail combinations
  - Test different ways to organize multiple speculation types
  - Implement responsive design patterns

- [X] **Navigation Integration**
  - Make contest headers clickable on Order Book cards
  - Implement URL structure: `/c/[contestId]`
  - Handle navigation state (grayed navbar approach)

- [X] **Empty State Design**
  - Contest creation button for non-existent contests
  - Speculation creation interface for contests without markets
  - Clear messaging for different empty states

### Phase 2: Data Integration and Functionality (Day 2-3 - Cursor)
- [X] **Market Depth Implementation**
  - Display contract availability across all price levels and amounts
  - Show user counts and total amounts on each side
  - Aggregate data from Firebase for real-time updates
  - Handle different contract sizes (20, 50, 100 USDC) simultaneously

- [X] **Modal Integration**
  - Connect existing Purchase/Write Contracts modals to contest context
  - Ensure betting functionality works seamlessly from detail page
  - Maintain existing user flow and behavior patterns

- [ ] **Market Comparison Features** (note: this will need to be done later)
  - Integrate jsonodds.com market odds display
  - Calculate and show savings vs traditional sportsbooks
  - Display leaderboard eligibility indicators (subtle)

## Success Criteria

### Functionality Acceptance
- [X] Users can navigate to contest details via clickable headers
- [X] All speculation types (moneyline, spread, totals) display simultaneously
- [X] Market depth information shows contract availability across price levels
- [X] Purchase/Write Contracts modals function correctly in contest context
- [X] Contest/speculation creation works from empty states

### User Experience Acceptance
- [X] Page provides significantly more information than Order Book cards
- [X] Information hierarchy guides users to most relevant data first
- [X] Navigation feels natural and doesn't disrupt existing flow
- [X] Real-time updates reflect current market state
- [X] Page loads and performs well with multiple data sources

## Technical Requirements

### Data Display Priorities (First Iteration)
1. **Basic Contract Availability**: Show depth across all price levels
2. **Market Comparison**: Display market odds vs platform odds (not done, later)
3. **User Activity**: Number of unique users and total amounts per side
4. **Real-time Updates**: Firebase integration for live data (untested)

### URL and Navigation Structure
```
/c/[contestId] - Contest detail page
Order Book card headers → clickable links to contest pages
Navbar remains functional but visually de-emphasized
```

### Market Depth Data Model (example, actual model may differ)
```javascript
// Contest market depth structure
{
  contestId: string,
  speculations: [
    {
      speculationId: string,
      type: 'moneyline' | 'spread' | 'total',
      leaderboardEligible: boolean,
      marketOdds: { yes: number, no: number },
      contractLevels: [
        {
          amount: 20 | 50 | 100,
          pricePoints: [
            {
              price: number,
              availableContracts: number,
              uniqueUsers: number,
              side: 'yes' | 'no'
            }
          ]
        }
      ]
    }
  ]
}
```

### Component Architecture (suggested, not required)
- `ContestDetailPage` - Main page component
- `MarketDepthDisplay` - Table/visual showing contract availability
- `SpeculationSection` - Individual speculation with depth data
- `MarketComparison` - Odds comparison with external sources
- `ContestCreationButton` - Empty state contest creation
- Reuse existing: `PurchaseContractsModal`, `WriteContractsModal`

## Content and Information Architecture

### Page Sections (Top to Bottom)
1. **Contest Header**: Teams, date, sport, basic info
2. **Market Overview**: Summary stats, total activity
3. **Speculation Sections**: Moneyline, spread, totals (in that order)
4. **Market Depth Tables**: Contract availability by price/amount
5. **Activity Summary**: User counts, amounts per side

### Empty State Handling
- **No Contest**: "Create Contest" button with basic contest info
- **No Speculations**: "Add Speculations" section with type selection (ie create market)
- **No Activity**: Show available speculations with appropriate messaging

## Integration Notes

### Existing Systems
- Maintain Order Book card click behavior (headers only)
- Use existing modal system for all betting actions
- Leverage Firebase real-time capabilities for live updates
- Integrate jsonodds.com API for market comparison

### Data Sources
- Firebase: Contract data, user activity, position information
- Smart contracts: Contest and speculation definitions
- jsonodds.com: External market odds for comparison
- Local state: UI interactions, modal management

## Risk Assessment
- **Medium complexity**: Multiple data sources and real-time updates
- **Medium risk**: Performance with large amounts of market data
- **High value**: Provides competitive differentiation for power users
- **Scope creep risk**: Many possible features, must maintain focus

## Development Tools & Workflow

### Lovable Phase (Day 1)
- Create multiple layout options for market depth display
- Implement navigation and URL structure
- Design empty states and creation flows
- Focus on visual hierarchy and information organization

### Cursor Phase (Day 2-3)
- Integrate Firebase data sources
- Connect existing modal system
- Implement real-time updates
- Polish performance and error handling

### Testing Strategy
- Test with various contest states (empty, partial data, full activity)
- Verify modal integration works correctly
- Check performance with multiple real-time updates (used mock data for now)
- Validate navigation flow from Order Book

## Success Metrics
- Users can access detailed market information not available on Order Book
- Contest creation/speculation creation functions work for admin use
- Market depth display provides actionable information for decision-making
- Page performance remains acceptable with real-time data
- Betting functionality maintains existing user experience

## Completion Definition
Milestone complete when users can navigate from Order Book card headers to comprehensive contest detail pages showing market depth, contract availability, and external odds comparison, while maintaining full betting functionality through existing modal system.

*This milestone provides the "power user" interface for detailed market analysis while preserving the simplicity of the main Order Book for casual users.*
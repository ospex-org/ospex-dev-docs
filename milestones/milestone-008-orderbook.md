# Milestone 008: Orderbook UI Overhaul
*Created: September 22, 2025*  
*Target Completion: September 23, 2025*
*Status: 🟡 Planning*

## Problem Statement
Current orderbook UI exposes raw maker/taker complexity and is confusing for users. Need to create an approachable, card-based interface that hides technical details while showing available betting opportunities clearly.

## Context
Milestone 7 completed navigation and profile pages. The orderbook is the core interaction point where users place bets - if this is confusing, the entire platform fails. Must abstract away blockchain complexity into simple "available bets" interface.

## Scope & Boundaries

### In Scope
- Card-based betting interface for each game
- Smart aggregation of multiple makers at same odds
- Clear display of available betting limits
- Simple bet placement flow
- Game filtering and organization
- Profile links for individual makers
- "Create your own bet" functionality

### Out of Scope  
- Drag and drop interactions
- Advanced order routing optimization
- Real-time price updates
- Mobile responsiveness (desktop first)
- Multi-bet ticket functionality
- Data Integration and backend data transformation

## Design Decisions Made

### 1. Multi-Layer Organization Structure
```
▼ NFL - Sunday, Sept 22

Lakers vs Warriors - 8:00 PM
[Moneyline] [Spread] [Total] ← Tabs for bet types

Currently viewing: Moneyline
┌─────────── LAKERS WIN ───────────┐  ┌─────────── WARRIORS WIN ──────────┐
│ ┌─────────┐ ┌─────────┐ ┌─────────┐│  │ ┌─────────┐ ┌─────────┐ ┌─────────┐│
│ │ 2.10x   │ │ 1.92x   │ │ 1.85x   ││  │ │ 1.95x   │ │ 1.88x   │ │ 1.75x   ││
│ │ $100    │ │ $75     │ │ $200    ││  │ │ $150    │ │ $50     │ │ $300    ││
│ │ [Bet]   │ │ [Bet]   │ │ [Bet]   ││  │ │ [Bet]   │ │ [Bet]   │ │ [Bet]   ││
│ │0x1a2b..c│ │3 makers │ │0x9f8e..d││  │ │0x7f3a..e│ │2 makers │ │0x4d9c..b││
│ └─────────┘ └─────────┘ └─────────┘│  │ └─────────┘ └─────────┘ └─────────┘│
│ Don't see odds you want?           │  │ Don't see odds you want?           │
│ [Create Position]                  │  │ [Create Position]                  │
└───────────────────────────────────┘  └───────────────────────────────────┘
```

### 2. Bet Type Tab System
- **Moneyline**: Team A win vs Team B win
- **Spread**: Multiple spread lines (-3.5, -4.5, etc.) with slider/button selector
- **Total**: Over/under options with number selector
- Show warning icon on tabs where speculations don't exist

### 3. Line Selection for Spreads/Totals
```
Spread Tab:
Line: [-4.5] [-3.5] [+3.5] [+4.5] ← Button selector
Or: [Slider: ----●---- ] ← Visual line selector

┌─── LAKERS -3.5 ────┐  ┌─── WARRIORS +3.5 ────┐
│ [betting cards]    │  │ [betting cards]       │  
└────────────────────┘  └───────────────────────┘
```

### 4. Contest/Speculation Handling
- Only display games with existing contests and speculations
- "Create Position" only appears when speculation exists
- Missing bet types show: "No [spread] betting available yet. [Advanced Tools]"
- Advanced Tools link leads to separate contest/speculation creation interface

## Implementation Plan

### Phase 1: Core Card Components (Week 1)
- [ ] **BettingCard component**
  - Display odds, available amount, maker info
  - Handle single maker vs aggregated display
  - Consistent 4-line layout with maker info on bottom

- [ ] **GameSection component**  
  - Container for Lakers vs Warriors style layout
  - Both sides (team A win, team B win)
  - Expandable/collapsible game details

- [ ] **Game filtering system**
  - Pill filter component (reuse from profile tabs)
  - Filter games by sport league
  - Sort by date and away team alphabetically

### Phase 2: Betting Flow (Week 2)
- [ ] **Bet placement flow**
  - Click bet button → simple modal/form
  - Amount input with max limit validation  

- [ ] **Create new bet flow**
  - Modal for creating custom odds
  - Input: odds desired, amount to wager
  - Explanation of maker/taker relationship

## Technical Requirements

### Data Processing Logic
```javascript
// Transform raw positions into betting cards
const processGameData = (rawPositions) => {
  // Group by odds, aggregate amounts
  // Sort by odds (best first)
  // Format maker information
  // Return card-ready data
}
```

### Component Structure
- `OrderBookPage` - Main container with filters
- `GameSection` - Individual game with both sides
- `BettingCard` - Individual betting option
- `BetModal` - Bet placement interface
- `CreateBetModal` - New bet creation

## User Experience Flow

### Primary User Journey
1. User visits Order Book page
2. Filters by sport (optional)
3. Finds game of interest
4. Reviews available betting options (cards)
5. Clicks "Bet" on preferred odds
6. Enters amount, confirms transaction
7. Position appears in their profile

### Secondary Flows
- Create new bet when no suitable odds available  
- Click maker address to view their profile
- Expand/collapse game sections to browse options

## Success Criteria

### Usability Acceptance
- [ ] New user can understand available bets without explanation
- [ ] Betting process takes under 3 clicks from card to confirmation
- [ ] Clear distinction between user's positions and available bets
- [ ] No confusion about odds, amounts, or maker information

### Technical Acceptance
- [ ] Proper sorting of odds (best to worst)
- [ ] Clickable maker addresses link to profiles
- [ ] Responsive card layout on different screen sizes

## Mock Data Requirements
- Multiple games across different sports
- Various odds levels per game side
- Mix of single maker and aggregated positions
- Different bet sizes and available amounts

## Risk Assessment
- **Medium complexity**: Data transformation from raw orderbook to cards
- **Low risk**: UI components are straightforward  
- **High value**: Core user interaction - must be intuitive

## Files to Create/Modify (suggested, not required)
- `src/components/OrderBook/GameSection.tsx`
- `src/components/OrderBook/BettingCard.tsx` 
- `src/components/OrderBook/BetModal.tsx`
- `src/components/OrderBook/CreateBetModal.tsx`
- `src/pages/OrderBook.tsx` (major overhaul)

## Completion Definition
Milestone complete when orderbook displays games as card-based betting options, users can place bets through simple interface, and the technical complexity is completely hidden from user view.

*This milestone transforms the orderbook from a technical tool into a user-friendly betting interface.*
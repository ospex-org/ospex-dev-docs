# Milestone 009: Leaderboard UI Enhancement
*Created: September 25, 2025*  
*Target Completion: September 26, 2025*
*Status: ✅ Complete*

## Problem Statement
Current leaderboard display uses multiple navigation tabs which feels excessive for a simple rankings page. Need to consolidate into a single, clean view with performance visualization and sortable metrics table.

## Context
Milestone 8 completed the orderbook UI overhaul with card-based betting interface. The leaderboard is critical for user engagement and competitive dynamics, especially as the platform prepares for AI agent vs human competition. Must be intuitive and engaging without complex rules initially.

## Scope & Boundaries

### In Scope
- Performance chart showing top performers over time
- Consolidated metrics table (replacing separate nav tabs)
- Sortable columns for all key metrics
- Clean, scannable user ranking display
- Basic "Join Leaderboard" functionality
- User position highlighting
- Time period indication

### Out of Scope  
- Complex leaderboard rules (bankroll limits, percentage restrictions)
- Leaderboard creation interface (use existing admin page)
- Prize/reward systems
- Advanced filtering beyond sorting
- Real-time updates
- Historical leaderboard archives

## Design Strategy: Open Season Approach

**Simple entry requirements:**
- Any wallet can participate
- No bankroll declarations required
- Minimal restrictions to encourage participation
- Focus on getting users comfortable with concept

**Defer complexity to later milestones:**
- Basis point restrictions
- Deviation controls  
- Percentage betting limits
- "Pro League" features

## Implementation Plan

### Phase 1: Chart Integration (Day 1)
- [X] **Add Performance Chart**
  - Reuse chart component from Overview page
  - Show top 3-10 performers' progress over time
  - Color-coded lines for different users
  - Time period controls (7D, 30D, All Time)

- [X] **Chart Data Structure**
  - Track daily performance snapshots
  - Handle new users joining mid-period (not in mock data)
  - Smooth line rendering for active participants

### Phase 2: Table Consolidation (Day 1-2)
- [X] **Replace Nav Tabs with Single Table**
  - Remove separate ROI/P&L/Volume/Win Rate tabs
  - Create unified sortable table with all metrics
  - Default sort by ROI descending

- [X] **Table Columns:**
  - Rank (#1, #2, etc.)
  - User (address + verified badge if applicable)
  - Subtle indicator column (icon or badge) to distinguish human vs agent participants
  - ROI (primary metric, percentage format)
  - Total P&L (USDC format)
  - Volume (total USDC wagered)
  - Win Rate (percentage)
  - Active Bets (current open positions)
  - Streak (W/L streak with direction)

### Phase 3: User Experience Polish (Day 2)
- [X] **Current User Highlighting**
  - Highlight connected user's row
  - "Your Position" indicator

- [X] **Leaderboard Context**
  - Participant count and prize pool display
  - Simple "Enter Leaderboard" button

## Success Criteria

### Usability Acceptance
- [X] Single page shows all relevant leaderboard information
- [X] Users can quickly identify top performers and their own position
- [X] Sorting works intuitively for all metrics
- [X] Performance chart provides engaging visual story

### Visual Acceptance
- [X] Matches overall site styling and dark theme
- [X] Chart integrates seamlessly with table below
- [X] Table remains scannable with 10+ participants
- [X] Responsive design works on different screen sizes

## Technical Requirements

### Component Structure (suggested, not required)
- `LeaderboardChart` - Performance over time visualization
- `LeaderboardTable` - Sortable metrics table
- `JoinLeaderboardButton` - Entry point for new participants
- `LeaderboardHeader` - Time period and context info

## Mock Data Requirements
- 10-15 sample participants with varied performance
- Realistic win rates (45-75% range)
- Different position styles (conservative vs aggressive)
- Include connected user in middle of pack
- Performance history spanning 2-4 weeks

## Integration Notes

### Existing Systems
- Use current admin page for leaderboard creation/management
- Connect to existing position tracking
- Integrate with profile page links (clickable addresses)

### AI Agent Preparation
- Ensure agent positions display correctly
- Handle agent vs human identification
- Support for agent "personality" indicators

## Risk Assessment
- **Low complexity**: Mostly UI reorganization
- **Low risk**: Building on existing components
- **High value**: Core engagement feature for competitive dynamics

## Files to Create/Modify (suggested, not required)
- `src/components/Leaderboard/LeaderboardChart.tsx` (new)
- `src/components/Leaderboard/LeaderboardTable.tsx` (modify existing)
- `src/pages/Leaderboard.tsx` (major simplification)
- Remove separate tab components

## Success Metrics
- Single page loads showing all leaderboard data
- Chart displays top performers clearly
- Table sorting works for all columns
- Connected user's position is highlighted
- "Join Leaderboard" flow is functional

## Completion Definition
Milestone complete when leaderboard displays as single consolidated view with performance chart and sortable table, ready for human vs AI agent competition.

*This milestone establishes the leaderboard as an engaging competitive feature while maintaining simplicity for initial user adoption.*
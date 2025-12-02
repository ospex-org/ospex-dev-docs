# Milestone 24: Homepage Redesign (OverviewDisconnected)

*Created: November 29, 2025*
*Completed: December 2, 2025*  
*Status: ✅ Complete*

## Goal

Replace the current homepage (disconnected state) with a clear value proposition that answers "what is this?" immediately. Remove mock data, wire up real activity feed, and communicate the core message: **Verifiable Sports Betting**.

## Scope

### Hero Section (Above the Fold)

**Left side:**
- Headline: "Verifiable Sports Betting" (bold, large)
- Subheadline: "No sign-up, no fees, no limits."
- Three value props with green checkmarks:
  - **Verified Record** - Every position verified on-chain
  - **Zero Vig** - Keep 100% of your winnings
  - **No Limits** - Bet any amount, any time
- Primary CTA: "Connect Wallet" button

**Right side (carousel with subtle chevrons):**
- **Panel 1: Live Activity** - Recent matched bets (real data from Firebase), link to Order Book
- **Panel 2: Leaderboard** - Top 3 performers summary (address, ROI), link to Leaderboard tab
- **Panel 3: TBD** - Could be Agents, Insights, or Betting Opportunities - decide during implementation

### Below the Fold

- Remove entirely for now
- Keep it clean; the hero should do the heavy lifting

### Technical Requirements

- Keep existing navbar/tabs for consistency (note this is missing from the example but we want to place the content mentioned above in the space alloted so that the Overview page looks like the others on the site)
- Wire up real "recent activity" data (recently matched pairs from Firebase)
- Leaderboard summary pulls from same source as LeaderboardTable
- Mobile: Stack layout (hero left content on top, carousel below)
- Carousel transitions should be subtle (fade or slow slide, ~500ms) (also note the carousel will not auto-rotate, this needs to be done by clicking the left or right chevrons to make the carousel move in the appropriate direction)
- Remove all mock data from OverviewDisconnected (note - put the chart in a dedicated file and save it, because we want to use this exact styling later, so we don't want to lose it)

### Reference Implementation

Lovable generated a reference design for Option 2 (split hero layout). See:
- `src/pages/Homepage2.tsx`
- Lovable project: https://lovable.dev/projects/beaf1254-1095-4762-a865-1fc8af9d9cbc

## Out of Scope (Future Milestones)

- LeaderboardChart historical data (leave as "No data available" for now)
- Insights wiring
- Agent reasoning/explanation storage

## Acceptance Criteria

- [X] New hero section matches Option 2 layout (split: value props left, carousel right)
- [X] Three value props visible with checkmarks
- [X] Carousel on right with at least 2 panels (Live Activity + Leaderboard)
- [X] Live Activity shows real matched bets from Firebase
- [X] Leaderboard panel shows real top performers
- [X] No mock data anywhere on the page
- [X] Mobile responsive (stacked layout)
- [X] Navbar remains consistent with other tabs

## Notes

- Copy can be refined over time; focus on structure first
- The carousel should not auto-rotate (use chevrons for user control)
- Keep color usage minimal per existing design language
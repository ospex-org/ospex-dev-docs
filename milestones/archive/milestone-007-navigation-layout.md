# Milestone 007: Navigation & Layout Foundation
*Created: September 18, 2025*  
*Target Completion: September 19, 2025*
*Status: ✅ Complete*

## Problem Statement
Current Lovable frontend has attractive navigation structure but unclear user flow. Need to define what each page shows, how wallet connection affects content, and where profile functionality fits within the navigation paradigm.

## Context
Milestone 6 completed leaderboard functionality. Now need to establish the foundational navigation and layout structure before adding complex data integration. This will guide all future frontend development.

## Scope & Boundaries

### In Scope
- Define Overview page content for connected vs. non-connected states
- Establish profile page location and access pattern
- Plan main navigation structure and user flow
- Create wireframes/mockups for key page states
- Document navigation decisions for future milestones

### Out of Scope  
- Data integration with Firebase/backend
- Wallet connection implementation
- AI agents functionality (future milestone)
- Social features (posts, explanations)
- Complex responsive design

## Decision Points to Resolve

### 1. Overview Page Strategy
**Connected Wallet State:**
- User's portfolio performance chart
- Recent positions summary
- Active leaderboard participation
- Quick actions (place bet, join leaderboard)

**Non-Connected State:**  
- Platform overview/landing content
- Featured leaderboard with top performers
- Sample performance data to show capabilities
- Call-to-action to connect wallet

### 2. Profile Access Pattern
**Recommended Approach:** Contextual Overview + User Menu
- Overview page becomes personalized when connected
- Additional profile actions via dropdown from wallet button
- Dropdown options: "View Full Profile", "Settings", "Disconnect"

### 3. Main Navigation Structure
**Current:** Overview | Order Book | Leaderboard | Markets | Custom
**Proposed:** Overview | Order Book | Leaderboard | Markets | Agents

No additional profile navigation item needed.

## Implementation Plan

### Phase 1: Navigation Structure (2 hours)
- [X] **Update main navigation labels**
  - Change "Custom" to "Agents" 
  - Ensure consistent styling and hover states

- [X] **Design wallet connection states**
  - Connected: Show address
  - Not connected: "Connect Wallet" button
  - Dropdown menu design for connected state

### Phase 2: Overview Page States (3 hours)  
- [X] **Non-connected Overview mockup**
  - Platform introduction/value proposition
  - Featured leaderboard preview
  - Connect wallet call-to-action

- [X] **Connected Overview mockup**
  - Real user portfolio performance chart
  - Active positions summary (3-4 positions max)

### Phase 3: Profile Page Design (2 hours)
- [X] **Full Profile page mockup**
  - Complete position history
  - Detailed performance analytics  
  - Leaderboard history and achievements
  - Settings/preferences section

- [X] **Profile access flow**
  - Dropdown menu from connected wallet
  - "View Full Profile" link behavior
  - Back navigation to Overview

## Success Criteria

### User Experience Acceptance
- [X] Clear understanding of where each type of content lives
- [X] Intuitive navigation between connected/non-connected states  
- [X] Profile information easily accessible but not cluttering main nav
- [X] Consistent visual hierarchy and user flow

### Design Acceptance
- [X] Mockups for all Overview and Profile created
- [X] Navigation patterns documented and consistent
- [X] Visual style maintains cohesion across states
- [X] Mobile responsiveness considerations noted

## Deliverables
1. **Navigation specification document**
2. **Page state mockups** (Overview connected/non-connected, Profile)
3. **User flow diagram** showing navigation patterns
4. **Updated Lovable implementation** with navigation structure

## Files to Modify
- Main navigation component
- Overview page with conditional rendering
- Profile page component (new)
- Wallet connection component with dropdown

## Design Decisions to Consider
- Contextual Overview vs. dedicated Profile nav
- Wallet connection UX patterns
- Information hierarchy across different page states
- Mobile navigation considerations

## Risk Assessment
- **Low risk**: Pure design/layout work, no backend integration
- **Medium effort**: Requires design thinking and mockup creation
- **High value**: Establishes foundation for all future development

## Completion Definition
Milestone complete when navigation structure is clearly defined, page states are mocked up, and user flow is determined. This provides the blueprint for implementing data integration in subsequent milestones.

*This milestone establishes the foundational user experience before adding complexity.*
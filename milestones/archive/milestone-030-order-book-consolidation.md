# Milestone 30: Order Book Consolidation

*Created: January 7, 2026*
*Completed: January 12, 2026*
*Status: ✅ Complete*

---

## Overview

Consolidate the Agent Offers page functionality into the Order Book page, creating a unified betting interface where Michelle is just another source of liquidity. Replace the card-based UI with number line visualization, fix staleness issues, and polish the experience.

---

## Goals

- Single betting interface (Order Book page) with number line UI
- Michelle's offers visible alongside user liquidity
- Fresh odds data (fix staleness in both Michelle's decisions and UI display)
- Working View Options
- Progress stepper for Track B flow
- Mobile-ready

---

## Track 1: Staleness Fixes

**Goal:** Michelle makes decisions on current data, and the UI displays current data.

### Problem

1. **Michelle's eval staleness** — She evaluates on Thursday, market moves 5% by Sunday, she's still using Thursday's odds in her matching/quote decisions.

2. **Number line market marker staleness** — The "Mkt 1.91" label on the number line shows stale data.

Both come from the same source: Firebase `contests` collection, which refreshes every 30 minutes via `ospex-firebase`.

### Tasks

**Michelle prompt fix:**
- [x] In `matching-prompt.ts`, add current market odds to the candidate data
- [x] In `quote-prompt.ts` (Track B), same fix — pass current market odds
- [ ] Test: Create position, manually update Firebase odds, verify Michelle sees fresh data (skipped)

**Number line display fix:**
- [x] Identify where number line fetches market odds
- [x] Ensure it pulls from contest doc on render (not cached/stale)
- [x] Change market marker label from `Mkt 2.57` to `Mkt 2.57 (30m ago)` using contest `LastUpdated`

### Files to Touch

- `ospex-agent-server/src/agents/market_maker_michelle/matching-prompt.ts`
- `ospex-agent-server/src/agents/market_maker_michelle/quote-prompt.ts`
- `ospex-lovable/src/components/agents/SimpleOddsLine.tsx`
- `ospex-lovable/src/components/agents/AgentGameAccordion.tsx`

---

## Track 2: Order Book UI Migration

**Goal:** Order Book page uses number line UI instead of cards. Michelle is just liquidity on the line.

### Current State

- Order Book page: Card-based UI with Yes/No buttons, small number slider for spread/total
- Agent Offers page: Number line showing market, agent offer, user selection

### Target State

- Order Book page: Number line UI (lifted from Agent page)
- User flow: Select game → see number line → pick odds → Review Position → modal

### Number Selector (Stepper)

Since carousels have been cursed, use stepper buttons:

```
◀  Magic -7.5  ▶
```

**Rules:**
- Increment/decrement by 1.0
- Numbers always end in .5
- For spreads: skip -0.5 and +0.5 (that's moneyline)
- Range determined by config (see below)
- Totals can never go negative

**Config: Number Stepper Bounds**

Create config file (or add to existing config) with bounds per sport/market:

```typescript
// numberStepperConfig.ts
export const NUMBER_STEPPER_BOUNDS = {
  spread: {
    NFL: 3,   // ±3 from market = 7 numbers shown
    NBA: 3,
    MLB: 0,
    NHL: 0,
    CFB: 3,
    CBB: 3,
  },
  total: {
    NFL: 3,
    NBA: 5,   // ±5 from market = 11 numbers shown
    MLB: 2,
    NHL: 1,
    CFB: 3,
    CBB: 5,
  }
};

// Example: Market is Magic -7.5, sport is NBA
// Spread bounds = 3, so show: -4.5, -5.5, -6.5, -7.5, -8.5, -9.5, -10.5
// (7 numbers centered on -7.5)

// Example: Market total is 220.5, sport is NBA
// Total bounds = 5, so show: 215.5, 216.5, 217.5, 218.5, 219.5, 220.5, 221.5, 222.5, 223.5, 224.5, 225.5
// (11 numbers centered on 220.5)
```

This gives 12 configurable values (6 leagues × 2 market types). Can be adjusted later per sport if needed (e.g., NFL spreads might want tighter range than NBA).

**When number changes:**
- Number line re-renders with new odds
- Michelle's offer updates (or disappears if she hasn't evaluated that number)
- Market odds update

### Graceful Degradation

When Michelle hasn't evaluated the selected number:
- Her marker disappears from number line
- User can still create position (Track A only)

### Tasks

- [x] Create `numberStepperConfig.ts` with bounds per sport/market
- [x] Lift NumberLine component from Agent page
- [x] Create NumberStepper component (◀ Number ▶)
- [x] Wire stepper to update number line odds
- [x] Respect bounds config when stepping
- [x] Replace Order Book card UI with number line UI
- [ ] Handle "MM not available" state gracefully (pushed to track 3)
- [x] Update "View Full Market →" to go to detail page (clicking header does this)
- [ ] Test: All three market types (ML, spread, total) (in progress)
- [x] Test: Number stepping works correctly, respects bounds
- [ ] Test: Michelle offer appears/disappears based on eval availability (in progress)

### Detail Page

Simple swap — keep it functional, no major redesign (well, sort of major redesign):
- [x] Replace top section (speculation cards) with number line
- [x] Keep existing order depth visualization below
- [x] Ensure number line works the same as main Order Book page

### Files to Touch

- `ospex-lovable/src/lib/config/numberStepperConfig.ts` (new)
- `ospex-lovable/src/components/NumberStepper.tsx` (new)
- `ospex-lovable/src/components/orderbook/NumberLineBetPanel.tsx` (new)
- `ospex-lovable/src/components/agents/SimpleOddsLine.tsx` (extra markers: open positions dots)
- `ospex-lovable/src/lib/data/useUnmatchedOpenPositions.ts` (shared unmatched position subscription)
- `ospex-lovable/src/components/MarketDepthList.tsx` (now reuses shared unmatched position logic)
- `ospex-lovable/src/components/OrderBook.tsx`
- `ospex-lovable/src/components/OrderBookDetails.tsx`
- Card components can be left alone, we simply won't use them but should leave them as-is

---

## Track 3: Progress Indicator

**Goal:** Replace the text status area with a visual progress stepper for Track B flow.

### Current State

Text-based status: "Not available — contest not created on-chain"

### Target State

Visual stepper:

```
[●]━━━━━━[○]━━━━━━[○]━━━━━━[○]
Preflight  Fee     Michelle   Matched
           Paid    Evaluating
```

**States:**
1. **Idle** — No stepper visible, just "Ready" or similar
2. **Preflight** — First dot active, "Checking availability..."
3. **Fee** — Second dot active, "Waiting for fee approval..."
4. **Evaluating** — Third dot active, "Michelle is evaluating... usually takes 5-10 seconds"
5. **Matched** — All dots complete, "Matched"

**Error states:**
- Preflight fail: Stepper resets, error message below
- Fee rejected: Stepper resets, "Fee not approved"
- Michelle rejects: Third dot shows X, "Outside acceptable range" + reason
- Match fails: Fourth dot shows X, "Match failed — position open for other takers"

### Tasks

- [x] Create ProgressStepper component
- [x] Wire to Track B flow states
- [ ] Add timing estimate during "Evaluating" step (no specific counter added but may revisit later, it's pretty fast)
- [x] Handle all error states with appropriate messaging
- [x] Style to complement number line (not compete with it)
- [ ] Test: Full Track B flow shows correct progression (partial - never completed Track B but testing revealed issues with logic, storage)
- [x] Test: Error states display correctly

### Files to Touch

- `ospex-lovable/src/components/ProgressStepper.tsx` (new)
- `ospex-lovable/src/components/agents/*` (or wherever Track B state lives)

---

## Track 4: Polish

**Goal:** Fix broken things, improve mobile experience.

### View Options Dropdown

Current problems:
- Not wired to anything
- Clicking it breaks the page
- Looks terrible on mobile

**Options to support:**
- [x] Market odds (checked by default)
- [x] Market Maker offer (checked by default)
- [x] Existing user open positions (unchecked by default) (change this to checked by default after implementation) (note: combined this with agent open positions)
- [ ] Existing agent open positions (unchecked by default) (change this to checked by default after implementation) (N/A)
- [ ] Existing matched positions (unchecked by default) (skipped)
- [x] Market Maker notes (unchecked by default)

Tasks:
- [x] Fix the crash on click
- [x] Wire checkboxes to actually filter/show elements on number line
- [x] Mobile-friendly dropdown (need to determine where/how to display this)
- [x] Test each option toggles correctly

### Mobile Polish

- [x] Number line responsive on small screens
- [x] Number stepper touch-friendly (viewable)
- [x] Progress stepper readable on mobile (sort of)
- [ ] Modal (Create Position) works on mobile
- [x] View Options doesn't overlap other elements

### Stretch: Odds Format Toggle

- [ ] Add toggle somewhere: "American / Decimal" (skipped)
- [ ] Apply consistently across all odds displays (skipped)
- [ ] Persist preference (localStorage or user settings) (skipped)

*This is nice-to-have. If it delays the milestone, cut it.*

### Agents Page Transition

- [x] Update Agents page to "Create Your Own Agent" placeholder
- [x] Add "Coming Soon" messaging
- [x] Hide old Agent Offers functionality

---

## Migration Plan

### Phase 1: Staleness + Components (Days 1-2)
- Fix staleness issues (Track 1)
- Build NumberStepper component
- Build ProgressStepper component
- Fix View Options crash

### Phase 2: Integration (Days 3-4)
- Integrate number line into Order Book page
- Wire NumberStepper to odds updates
- Wire ProgressStepper to Track B flow
- Wire View Options to number line

### Phase 3: Detail Page + Polish (Days 5-6)
- Update Order Book detail page
- Mobile polish pass
- Testing all flows

### Phase 4: Cleanup (Day 7)
- Remove/deprecate old card components
- Final testing
- Documentation

---

## Rolling QA & Known Issues

| Bug | Severity | Status | Notes |
|-----|----------|--------|-------|
| View Options crashes page | High | To Fix | Track 4 |
| View Options not wired | Medium | To Fix | Track 4 |
| View Options mobile overlap | Medium | To Fix | Track 4 |
| Stale Michelle evals | Medium | To Fix | Track 1 |
| Stale number line market marker | Low | To Fix | Track 1 |
| Odds format inconsistent (American/Decimal) | Low | Stretch | Track 4 |
| Neutral site games API mismatch | Low | Known Limitation | Carried from M28 |
| Firebase pruning | Low | Deferred | Carried from M28 |
| Wallet monitoring | Low | Deferred | Carried from M28 |
| American / Decimal Toggle | Low | Deferred | Carried from M30 |

---

## Success Criteria

- [x] Order Book page uses number line UI
- [x] Number stepper allows changing spread/total numbers
- [x] Michelle's offer appears when she has an eval, gracefully absent when not
- [x] Track B flow shows progress stepper with clear state progression
- [x] View Options dropdown works and doesn't crash
- [x] Mobile experience is functional (not necessarily perfect)
- [ ] Michelle makes decisions on fresh market odds (punting on this)
- [x] Number line displays fresh market odds

---

## Out of Scope (Future)

- Module system (starting lineups) — M31+
- Agent commentary during evaluation — nice idea, not now
- Carousel number selector — maybe revisit if stepper feels too clunky
- Full mobile redesign — just fixing what's broken

---

## Notes

*Add observations, decisions, or context as the milestone progresses.*

---

## Resolved Questions

1. **Number stepper bounds** — Configurable by sport and market type. Defaults:
   - Spreads: ±3 from market (7 numbers shown)
   - Totals: ±5 from market (11 numbers shown, never negative)
   - See config spec for details

2. **Order Book detail page priority** — Functional, not broken. Swap in number line, keep existing depth cards. No major redesign.

3. **Agent page** — Keep nav link, repurpose page as "Create Your Own Agent" (coming soon placeholder). Full agent creation is future scope.
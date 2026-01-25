# Milestone 36: Insights Feature

*Created: January 23, 2026*
*Completed: January 25, 2026*
*Status: ✅ Complete*

---

## Overview

Surface LLM-generated reasoning so users can understand WHY agents (and other users) take positions. Agent insights are free to view; user insights cost a small fee to create.

**Philosophy:** We're already generating this content - don't waste it. Even if the quality is rough initially, having it visible creates engagement and accountability.

---

## Track 1: Agent Insights (Free to View)

Agents already generate reasoning in `agentDecisions.reasoning` and `positionEvaluations.llmReasoning`. Wire the existing stubbed UI to this real data.

### Tasks

- [ ] Create `insights` collection in Firebase with unified schema (see Data Model below) (queried agentDecisions directly instead of creating a new unified collection. The hooks transform data on-the-fly rather than ETL to a new collection)
- [x] Investigate Degen Dan's output - what does he store? Ensure his commentary flows into `insights` collection
- [ ] Create migration/transformer to populate `insights` from existing `agentDecisions` where `reasoning` exists (Created `transformers.ts` that convert on-the-fly, but no migration script to populate a new collection)
- [x] Create `useInsights()` hook - fetch from `insights` collection with filtering
- [x] Create `usePositionInsight()` hook - fetch insight by positionId
- [x] Wire `Insights.tsx` (global page) to real data, remove mock data
- [x] Wire `ProfileExplanationsTab.tsx` to real data filtered by wallet address
- [x] Remove "ComingSoon" badge from profile Insights tab (Profile.tsx line 81)
- [x] Add sanitization layer to strip private fields before returning to frontend

### Files to Modify

| File | Change |
|------|--------|
| `src/components/Insights.tsx` | Replace mock data with `useInsights()` hook |
| `src/components/profile/ProfileExplanationsTab.tsx` | Replace mock data with `useInsights({ address })` |
| `src/components/profile/Profile.tsx` | Remove ComingSoon badge (line 81) |
| `src/lib/data/insights/` | New directory for hooks |

---

## Track 2: User Insights (Paid to Create)

Users can optionally explain their positions. Costs 0.01 USDC (spam prevention, crypto-native feel). Plain text only, 500 char limit (already enforced by modal).

### Tasks

- [x] Wire `PositionInsightModal.tsx` to save to `insights` collection with `source: 'user'`
- [x] Implement 0.01 USDC fee using same pattern as instant match:
  - Frontend: USDC transfer to fee wallet
  - Backend: `verifyUsdcFeeTx()` on-chain verification
  - Backend: Atomic Firebase transaction (claim tx hash + create insight)
- [x] Add fee configuration to `.env` files (`INSIGHT_FEE_USDC=0.01`)
- [x] Wire `EditInsightModal.tsx` for editing (no additional fee for edits)
- [x] Add "Add Insight" button to position cards/detail page for position owner
- [x] Implement kill switch: `config/insights` Firestore doc with `userInsightsEnabled: boolean`
  - Backend checks this before accepting user insight submissions
  - Frontend checks this to hide/disable "Add Insight" button
  - Toggleable from Firebase console without redeploy

### Payment Flow

```
User clicks "Add Insight" on their position
         │
         ▼
┌─────────────────────────────────┐
│ 1. Show PositionInsightModal    │
│    - Position context displayed │
│    - 500 char text input        │
│    - "Post Insight (0.01 USDC)" │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ 2. Pay 0.01 USDC to fee wallet  │
│    via usdcEngine.ts            │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ 3. POST /api/insights           │
│    - feeTxHash                  │
│    - positionId                 │
│    - content                    │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ 4. Backend validation           │
│    - Verify fee on-chain        │
│    - Check tx hash not reused   │
│    - Verify user owns position  │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ 5. Atomic Firebase transaction  │
│    - Claim tx hash              │
│    - Create insight document    │
└─────────────────────────────────┘
```

### Files to Create/Modify

| File | Change |
|------|--------|
| `ospex-agent-server/src/http/insights.ts` | New endpoint for creating user insights |
| `ospex-lovable/src/lib/data/insights/useCreateInsight.ts` | Hook for payment + creation flow |
| `src/components/profile/PositionInsightModal.tsx` | Wire to real creation flow |

---

## Track 3: UI Integration

Add insight visibility to existing pages beyond the dedicated Insights tab.

### Position Detail Page

- [x] Display insight if one exists for this position
- [x] Show `positionEvaluations.llmReasoning` for agent-matched positions
- [x] "Add Insight" button if viewer is position owner and no insight exists

### Order Book Page

- [ ] Add insight count link: "3 insights" that links to filtered insights view (skipped)
- [ ] Count should reflect current market filter (spread, moneyline, total) (skipped)
- [x] Keep it unobtrusive - small link, not a prominent section (message-square-more icon after contest header)

### Contest Detail Page

- [ ] Add "Recent Insight" preview section (most recent insight for this contest) (skipped)
- [ ] Link to full insights filtered by this contest (skipped)
- [ ] Position below main content so it doesn't interfere with betting flow (skipped)
- [x] Use same styling as Order Book Page

### Tasks

- [x] Add insight display to `PositionDetailPage.tsx`
- [ ] Add insight count link to order book page (skipped)
- [ ] Add recent insight section to `ContestDetail.tsx` (skipped)
- [x] Create filtered insights view (by contest/market)

---

## Data Model

### `insights` Collection

Unified collection for both agent and user insights.

```typescript
interface Insight {
  id: string;                          // Auto-generated
  positionId: string;                  // Links to position
  address: string;                     // Wallet that created the insight
  source: 'agent' | 'user';            // Who created it
  agentId?: string;                    // If source='agent', which agent
  
  // Contest context
  jsonoddsId: string;
  league: 'NFL' | 'NBA' | 'MLB' | 'NHL' | 'CFB' | 'CBB';
  awayTeam: string;
  homeTeam: string;
  gameDate: Timestamp;
  
  // Position context
  market: 'moneyline' | 'spread' | 'total';
  side: 'away' | 'home' | 'over' | 'under';
  line?: number | null;
  odds: number;
  amount: number;                      // USDC
  
  // The insight itself
  content: string;                     // Max 500 chars
  conviction?: number;                 // 0-10, agents only
  
  // Metadata
  createdAt: Timestamp;
  updatedAt?: Timestamp;
  
  // Payment (user insights only)
  feeTxHash?: string;
}
```

### Private Fields (Never Return to Frontend)

These fields exist in source collections but should be stripped:

| Collection | Private Fields |
|------------|----------------|
| `agentOffers` | `ceilingOdds`, `currentExposure`, `maxExposure`, `confidence` |
| `agentDecisions` | `exposure.*` |
| `agentCalculations` | All fields |
| `positionEvaluations` | `takerOdds`, `makerOdds` |

**Implementation:** Create a `sanitizeInsight()` function that strips these fields before returning data. Apply at the data layer (hooks) so UI never sees private data.

### `config/insights` Document (Kill Switch)

```typescript
interface InsightsConfig {
  userInsightsEnabled: boolean;        // Toggle user insight creation
  // Future options:
  // maxInsightsPerDay?: number;
  // minPositionAgeMinutes?: number;
}
```

Toggleable from Firebase console at: `config/insights`

---

## Track 4: Future Credit System (Document Only)

**Not building this milestone** - just documenting the direction for future reference.

### Concept

Users buy credits upfront, spend them on various actions:

| Action | Credit Cost |
|--------|-------------|
| Create insight | 1 credit |
| Request instant match | 20 credits |
| Run agent benchmark | 10 credits |
| Ask agent "why this bet?" | 5 credits |

### Purchase Tiers

| Package | Price | Credits | Per-Credit |
|---------|-------|---------|------------|
| Starter | 0.50 USDC | 10 | 0.05 |
| Standard | 2.00 USDC | 50 | 0.04 |
| Pro | 5.00 USDC | 150 | 0.033 |

### Benefits

- Users don't feel "nickeled and dimed"
- Simpler mental model (I have X credits)
- Bulk discount rewards engagement
- Single payment flow to implement

### Implementation Notes (Future)

- `userCredits` collection tracking balance per wallet
- Credit purchase uses same fee verification pattern
- Credit spend is internal (no on-chain tx per action)
- Consider expiration? (credits expire after 90 days)

---

## Success Criteria

- [x] Global Insights page shows real agent insights (not mock data)
- [x] Profile Insights tab shows insights for that wallet
- [x] Users can create insights for their positions (with 0.01 USDC fee)
- [x] Position detail page shows insight if one exists
- [x] Order book shows insight count link (shows link to insights instead)
- [x] Contest detail shows recent insight preview (same as above)
- [x] Private fields (ceilingOdds, exposure, etc.) never exposed to frontend (did not notice this during testing)

---

## Non-Goals (This Milestone)

- Credit/subscription system (future milestone)
- Replies or interactions on insights
- Image uploads
- Rich text formatting
- Paywalled insights (users charging for their insights)
- Agent "ask why" feature

---

## Time Estimate

| Track | Estimate |
|-------|----------|
| Track 1: Agent insights | 3-4 hours |
| Track 2: User insights + payment | 3-4 hours |
| Track 3: UI integration | 2-3 hours |

**Total:** 8-11 hours across 2-3 sessions

---

## Open Questions

1. **Insight editing** - Can users edit their insight after posting? (Current assumption: yes, no additional fee)
2. **Insight deletion** - Can users delete? Should there be a cooldown?
3. **Agent insight generation** - Currently pulling from `agentDecisions.reasoning`. Should agents generate a separate, more polished "public insight" vs raw reasoning?

---

## Notes

### On Agent Insight Quality

The reasoning in `agentDecisions` is generated during evaluation - it's functional but may be dry or technical. Options:

1. **Use as-is** - Authentic but might be hard to read
2. **Post-process** - Run through LLM to make more readable (adds cost)
3. **Generate separately** - Have agents write a "public explanation" distinct from internal reasoning (adds cost)

Recommendation: Start with raw reasoning. See if users find it valuable. Polish later if needed.

### On Insight Discoverability

Insights are only useful if people see them. Potential surfaces:

- Global feed (implemented)
- Profile tab (implemented)
- Position detail (this milestone)
- Contest page (this milestone)
- Order book hover/preview (future)
- Push notifications for followed users (future)

### On Spam Prevention

0.01 USDC fee should prevent most spam. Additional measures if needed:
- Rate limit per wallet (max 10 insights/day)
- Report/flag mechanism
- Minimum position age before insight allowed
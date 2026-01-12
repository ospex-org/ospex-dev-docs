# Milestone 29: Michelle Instant Match (Track B)

*Created: January 2, 2026*
*Completed: January 6, 2026*
*Status: ✅ Complete*

---

## Overview

Implement "Instant Match" flow where users can pay a small fee (0.20 USDC) for an immediate quote from Michelle before creating a position. This provides certainty ("will I get matched?") in exchange for a fee that covers LLM costs.

Builds on Track A (Post and Wait) completed in M28.

---

## User Flow
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TRACK B: INSTANT MATCH                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   User       │     │  Pre-flight  │     │   Fee +      │     │   Position   │
│   Selects    │────▶│   Check      │────▶│   Quote      │────▶│   Creation   │
│   Odds       │     │   (free)     │     │   (0.20)     │     │   + Match    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                            │                    │                     │
                            ▼                    ▼                     ▼
                     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
                     │ FAIL:        │     │ REJECT:      │     │ SUCCESS:     │
                     │ "Not avail-  │     │ "Outside my  │     │ Position     │
                     │ able" + why  │     │ range" + why │     │ matched!     │
                     │ (no charge)  │     │ (fee kept)   │     │              │
                     └──────────────┘     └──────────────┘     └──────────────┘

DETAILED FLOW:
──────────────

1. User selects odds on number line, enters amount
   └─▶ NO position created yet

2. User clicks "Request Instant Quote (0.20 USDC)" (unsure on the UI here)

3. PRE-FLIGHT CHECK (free, no LLM call)
   ├─▶ Query Firebase: Does Michelle have eval for this game/market?
   ├─▶ Query Firebase: Is Michelle at max exposure on this side?
   ├─▶ Check: Has game started?
   │
   ├─▶ If ANY fail → Immediate rejection, no fee charged
   │   └─▶ UI shows: "Not available: [reason]"
   │
   └─▶ If ALL pass → Proceed to step 4

4. FEE COLLECTION
   └─▶ Transfer 0.20 USDC from user wallet to fee wallet

5. LLM QUOTE (Michelle evaluates)
   ├─▶ Fetch current market odds from Firebase
   ├─▶ Build prompt with: position details, current odds, exposure, eval
   ├─▶ Michelle returns judgment:
   │
   ├─▶ YES: "I'll match up to X USDC at these odds"
   │
   └─▶ NO: "Outside my acceptable range"
       └─▶ UI shows: Rejection reason, fee not refunded
       └─▶ User can adjust odds and try again (new fee)

6. POSITION CREATION
   └─▶ User creates position on-chain at quoted odds/amount

7. IMMEDIATE MATCH (webhook)
   ├─▶ Frontend calls match endpoint with quoteId after tx confirms
   ├─▶ Michelle validates quote still valid, not expired
   └─▶ Michelle creates matching position immediately

8. COMPLETION
   └─▶ User sees matched position in UI
```

---

## Technical Components

### 1. Fee Wallet

- Create new wallet for fee collection (separate from Michelle's market-making wallet)
- Cleaner accounting: market-making funds vs operational revenue
- Address to be configured in environment variables (FEE_WALLET_ADDRESS=0xdaC630aE52b868FF0A180458eFb9ac88e7425114)

### 2. Express Routes (agent-server)

Add web dyno capability to agent-server for HTTP endpoints.

**Procfile update:**
```
web: node dist/server.js
worker: node dist/scheduler.js
```

**New routes:**
```typescript
// POST /api/michelle/preflight
// Free check before fee collection
{
  request: {
    speculationId: string,
    market: 'moneyline' | 'spread' | 'total',
    side: 'away' | 'home' | 'over' | 'under',
    requestedOdds: number,
    requestedAmount: number,
    userWallet: string
  },
  response: {
    available: boolean,
    reason?: string,  // if not available
    currentExposure?: number,
    maxExposure?: number,
    evalExists?: boolean
  }
}

// POST /api/michelle/quote
// Called after fee is collected
{
  request: {
    speculationId: string,
    market: 'moneyline' | 'spread' | 'total',
    side: 'away' | 'home' | 'over' | 'under',
    requestedOdds: number,
    requestedAmount: number,
    userWallet: string,
    feeTxHash: string  // proof fee was paid
  },
  response: {
    accepted: boolean,
    quoteId?: string,        // UUID, only if accepted
    quotedAmount?: number,   // may be less than requested
    expiresAt?: string,      // ISO timestamp, 60 seconds from now
    reason: string           // explanation either way
  }
}

// POST /api/michelle/match
// Webhook called after position creation
{
  request: {
    quoteId: string,
    positionId: string,  // on-chain position ID
    userWallet: string
  },
  response: {
    matched: boolean,
    txHash?: string,
    error?: string
  }
}
```

### 3. Quote Storage (Firestore: `michelleQuotes`)
```typescript
{
  quoteId: string,           // UUID
  userWallet: string,
  speculationId: string,
  market: 'moneyline' | 'spread' | 'total',
  side: 'away' | 'home' | 'over' | 'under',
  requestedOdds: number,
  requestedAmount: number,
  quotedAmount: number,      // what Michelle agreed to
  feeTxHash: string,
  status: 'pending' | 'accepted' | 'expired' | 'matched' | 'rejected',
  reason: string,
  createdAt: Timestamp,
  expiresAt: Timestamp,      // createdAt + 60 seconds
  acceptedAt?: Timestamp,
  matchedAt?: Timestamp,
  matchTxHash?: string,
  
  // Snapshot for audit
  evalSnapshot: {
    confidence: number,
    agentOdds: number,
    marketOdds: number
  },
  marketOddsAtQuote: {
    away?: number,
    home?: number,
    over?: number,
    under?: number
  }
}
```

### 4. Quote Prompt Builder

New file: `ospex-agent-server/src/agents/market_maker_michelle/quote-prompt.ts`

Similar to matching-prompt but for single position evaluation:
- Include current market odds (fresh from Firebase)
- Include Michelle's original eval
- Include current exposure
- Ask for yes/no judgment with amount and reasoning

### 5. Immediate Match Handler

New file: `ospex-agent-server/src/agents/market_maker_michelle/instant-match.ts`

Called via webhook after position creation:
- Validate quote exists and not expired
- Validate user matches
- Check exposure one more time (race condition safety)
- Execute match on-chain
- Update quote status to 'matched'

### 6. Frontend Changes

**Agent page updates:**

## Proposed New Layout

┌─────────────────────────────────────────────────────────────────────────────┐
│ NFL  Seattle Seahawks vs San Francisco 49ers                 U 0 $ 0 USDC ^ |
|     Jan 5, 7:00 PM CST                                                      │
│      ┌─────────┐ ┌────────┐ ┌───────┐   ┌──────────┐ ┌───────────┐          │
│      │Moneyline│ │ Spread │ │ Total │   │ Seahawks │ │   49ers   │          │
│      └─────────┘ └────────┘ └───────┘   └──────────┘ └───────────┘          │
├─────────────────────────────────────────────────────────────────────────────┤
│ Seattle Seahawks ML                              [⚙ View Options ▾]        │
│                                                                             │
│  ═══════════●═══════════════════════════════════════════════════════════    │
│         Mkt 1.69    Agent 1.70                                              │
│                                                                             │
│  ┌────────────┐  ┌─────────────────┐              ┌──────────────────────┐  │
│  │ Odds: 1.72 │  │ Review Position │              │ Status: Ready        │  │
│  └────────────┘  └─────────────────┘              └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘

- Game header stays as-is except for the following changes:
  - Change date format from showing only time to showing date and time (as is on order book page: Jan 3, 2026, 11:00 AM CST)
  - Add user icon, count of users, dollar sign, amount, and USDC - exactly as-is on order book page, and right-align this but to the left of the existing chevron
- Show Moneyline, Spread, Total and Team selectors all on the same line
- Add View Options as a dropdown with checkboxes:
  - Market odds (checked by default)
  - Market Maker offer (checked by default)
  - Existing user open positions (unchecked by default)
  - Existing agent open positions (unchecked by default)
  - Existing matched positions (unchecked by default)
  - Market Maker notes (unchecked by default)
- Number line stays the same, View Options toggles what appears ON the line (other positions, agent markers, etc.)
- Add status area to last row, to right of existing odds and review position button

For Track B, the action row changes based on context:
- "Ready" (default)
- "Michelle is evaluating..." (after clicking request match in modal)
- "Quote accepted, creating position..." (no extra in-app confirmations)
- "Matched! ✓"
- "Match attempt failed — your position is open and may be matched later."

**Simplified Track B Flow:**

The modal that appears on this page when Review Position is clicked will have the option to select track A or track be:
Currently: Cancel, Request Match (2 buttons)
New: Cancel, Create Position (Free), Request Match (0.20 USDC) (3 buttons)\n+- Fee amount is configurable (frontend reads via env var so it can be changed easily)\n+- Once Track B starts, there are no further on-site confirmations; wallet prompts are the confirmations

User clicks "Request Match (0.20 USDC)"
         │
         ▼
┌─────────────────────────┐
│ Pre-flight check        │ (status area; modal closes immediately)
│ "Checking availability" │
└─────────────────────────┘
         │
         ├── FAIL → Show error, stop (no fee charged)
         │
         ▼ PASS
┌─────────────────────────┐
│ Wallet prompts fee tx   │ (0.20 USDC to fee wallet)
└─────────────────────────┘
         │
         ├── User rejects → Stop (no fee, no position)
         │
         ▼ User approves
┌─────────────────────────┐
│ LLM evaluates           │ (modal unloads, progress bar in status area: "Michelle is evaluating...")
└─────────────────────────┘
         │
         ├── REJECT → Show reason, stop (fee consumed, that's the deal)
         │
         ▼ ACCEPT
┌─────────────────────────┐
│ "Michelle will match    │
│ up to 50 USDC"          │
│                         │
│ Wallet prompts position │ (position creation tx)
└─────────────────────────┘
         │
         ├── User rejects → Quote expires eventually, fee consumed, no position
         │
         ▼ User approves
┌─────────────────────────┐
│ Position created        │
│ Calling Michelle...     │ (webhook fires)
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│ ✓ Matched!              │
│ Michelle matched 50 USDC│
└─────────────────────────┘

Any necessary updates can go in the status area

---

## Configuration

### Environment Variables (agent-server)
```
# Fee collection
MICHELLE_FEE_WALLET_ADDRESS=0x...
MICHELLE_FEE_WALLET_PRIVATE_KEY=${...}
MICHELLE_QUOTE_FEE_USDC=0.20
```

### Heroku
```bash
# Add web dyno
heroku ps:scale web=1 worker=1 -a ospex-agent-server
```

---

## Tasks

### Phase 1: Infrastructure

- [x] Create fee collection wallet
- [x] Add web dyno to agent-server (Procfile + Express setup)
- [x] Create Firebase location for Michelle bets (she uses memory now)
- [x] Implement pre-flight endpoint (`/api/michelle/preflight`)
- [ ] Test pre-flight with various scenarios (in range, out of range tested)

### Phase 2: Quote Flow

- [x] Implement quote prompt builder
- [x] Implement quote endpoint (`/api/michelle/quote`)
- [x] Fee verification (check feeTxHash is valid USDC transfer)
- [x] Quote storage
- [x] Test quote flow end-to-end (backend only)

### Phase 3: Match Execution

- [x] Implement instant match handler
- [x] Implement match webhook endpoint (`/api/michelle/match`)
- [x] Race condition handling (exposure filled between quote and match)
- [x] Quote status updates (pending → accepted → matched)
- [x] Test match execution

### Phase 4: Frontend Integration

- [x] Add "Request Instant Quote" button to modal
- [x] Pre-flight check UI (loading, error states)
- [x] Fee approval flow (USDC transfer)
- [x] Rejection messaging
- [x] Post-creation webhook integration
- [x] Match confirmation UI

### Phase 5: Polish & Edge Cases

- [ ] Expired quote handling (user took too long) (skipped)
- [x] Failed match handling (poor odds tested)
- [ ] Quote history in user's position view (did not implement)
- [x] Logging and monitoring for quote flow
- [x] Error messaging for all failure modes (not everything was tested but there is some error handling)

---

## Edge Cases & Error Handling

| Scenario | Handling |
|----------|----------|
| Pre-flight fails (no eval, max exposure, game started) | Immediate rejection, no fee, clear message |
| Fee tx fails/reverts | Don't proceed to quote, user retries |
| LLM rejects after fee | Fee kept, show reason, user can adjust and retry (new fee) |
| Position creation fails after accepting quote | Quote marked expired, no match attempted |
| Exposure filled between quote and match | Partial match or pass |
| Match tx fails | Log error, mark quote as failed, user has unmatched position (falls back to Track A) |

---

## Success Criteria

- [x] User can request instant quote and receive yes/no within seconds
- [x] Fee (0.20 USDC) collected before LLM call
- [x] Pre-flight prevents fee collection for deterministic rejections
- [x] Accepted quotes result in immediate match after position creation
- [x] All edge cases handled gracefully with clear user messaging

---

## Notes

*Add observations, decisions, or context as the milestone progresses.*

---

## Carried Forward from M28

### Rolling QA & Known Issues

| Bug | Severity | Status | Notes |
|-----|----------|--------|-------|
| No agent ↔ market maker auto-fill | N/A | Resolved | Track A + Track B solve this |
| Neutral site games cause API mismatch | Low | Known Limitation | CFP games, etc. Manual workaround. |
| Group leaderboards | Low | Deferred | Naming scheme for grouping needed |
| Firebase pruning | Low | Deferred | Prune agentMemory decisions after 30 days |
| Wallet monitoring | Low | Deferred | Admin panel for agent/oracle wallet balances |

### Mainnet Prep (from M28 Track 3)

- [ ] Review contract deployment checklist
- [ ] Verify frontend can switch networks cleanly
- [ ] Plan USDC funding for Michelle's wallet + fee wallet
- [ ] Decide on initial exposure limits for mainnet
- [ ] Config changes for prod vs test
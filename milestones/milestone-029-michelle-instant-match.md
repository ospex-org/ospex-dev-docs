# Milestone 29: Michelle Instant Match (Track B)

*Created: January 2, 2025*
*Target Completion: Mid-January 2025*
*Status: 🔴 Not Started*

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
   │   └─▶ Quote stored with 60-second TTL
   │   └─▶ UI shows: Accept button + countdown timer
   │
   └─▶ NO: "Outside my acceptable range"
       └─▶ UI shows: Rejection reason, fee not refunded
       └─▶ User can adjust odds and try again (new fee)

6. USER ACCEPTS QUOTE
   └─▶ Click "Accept & Create Position" within 60 seconds

7. POSITION CREATION
   └─▶ User creates position on-chain at quoted odds/amount

8. IMMEDIATE MATCH (webhook)
   ├─▶ Frontend calls match endpoint with quoteId after tx confirms
   ├─▶ Michelle validates quote still valid, not expired
   └─▶ Michelle creates matching position immediately

9. COMPLETION
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

- Add "Request Instant Quote (0.20 USDC)" button alongside current flow
- Pre-flight loading state while checking availability
- Quote response modal:
  - If accepted: Show amount, 60-second countdown, "Accept & Create Position" button
  - If rejected: Show reason, option to adjust odds
- Fee transaction approval flow (standard USDC transfer)
- Post-creation webhook call to trigger match

**Deterministic rejection UI:**

After pre-flight check fails, show clear messaging:
- "Market maker at capacity for this side"
- "This game has not been evaluated"
- "Game has already started"

No fee charged, no ambiguity.

---

## Configuration

### Environment Variables (agent-server)
```
# Fee collection
MICHELLE_FEE_WALLET_ADDRESS=0x...
MICHELLE_FEE_WALLET_PRIVATE_KEY=${...}
MICHELLE_QUOTE_FEE_USDC=0.20

# Quote settings
MICHELLE_QUOTE_TTL_SECONDS=60

# Existing (no change)
MICHELLE_WALLET_ADDRESS=0x...
MICHELLE_WALLET_PRIVATE_KEY=${...}
```

### Heroku
```bash
# Add web dyno
heroku ps:scale web=1 worker=1 -a ospex-agent-server
```

---

## Tasks

### Phase 1: Infrastructure

- [ ] Create fee collection wallet
- [ ] Add web dyno to agent-server (Procfile + Express setup)
- [ ] Create `michelleQuotes` Firestore collection
- [ ] Implement pre-flight endpoint (`/api/michelle/preflight`)
- [ ] Test pre-flight with various scenarios (no eval, max exposure, game started)

### Phase 2: Quote Flow

- [ ] Implement quote prompt builder
- [ ] Implement quote endpoint (`/api/michelle/quote`)
- [ ] Fee verification (check feeTxHash is valid USDC transfer)
- [ ] Quote storage and TTL handling
- [ ] Test quote flow end-to-end (backend only)

### Phase 3: Match Execution

- [ ] Implement instant match handler
- [ ] Implement match webhook endpoint (`/api/michelle/match`)
- [ ] Race condition handling (exposure filled between quote and match)
- [ ] Quote status updates (pending → accepted → matched)
- [ ] Test match execution

### Phase 4: Frontend Integration

- [ ] Add "Request Instant Quote" button to agent page
- [ ] Pre-flight check UI (loading, error states)
- [ ] Fee approval flow (USDC transfer)
- [ ] Quote response modal (countdown timer, accept button)
- [ ] Rejection messaging
- [ ] Post-creation webhook integration
- [ ] Match confirmation UI

### Phase 5: Polish & Edge Cases

- [ ] Expired quote handling (user took too long)
- [ ] Failed match handling (exposure filled, tx failed)
- [ ] Quote history in user's position view
- [ ] Logging and monitoring for quote flow
- [ ] Error messaging for all failure modes

---

## Edge Cases & Error Handling

| Scenario | Handling |
|----------|----------|
| Pre-flight fails (no eval, max exposure, game started) | Immediate rejection, no fee, clear message |
| Fee tx fails/reverts | Don't proceed to quote, user retries |
| LLM rejects after fee | Fee kept, show reason, user can adjust and retry (new fee) |
| Quote expires (60s) | Quote marked expired, user must request new quote (new fee) |
| Position creation fails after accepting quote | Quote marked expired, no match attempted |
| Exposure filled between quote and match | Partial match or pass, user keeps whatever was matched |
| Match tx fails | Log error, mark quote as failed, user has unmatched position (falls back to Track A) |

---

## Success Criteria

- [ ] User can request instant quote and receive yes/no within seconds
- [ ] Fee (0.20 USDC) collected before LLM call
- [ ] Pre-flight prevents fee collection for deterministic rejections
- [ ] Accepted quotes result in immediate match after position creation
- [ ] Quote TTL (60 seconds) enforced
- [ ] All edge cases handled gracefully with clear user messaging
- [ ] Full audit trail in `michelleQuotes` collection

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
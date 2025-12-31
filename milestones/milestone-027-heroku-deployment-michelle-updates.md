# Milestone 27: Heroku Deployment + Always-On Michelle

*Created: December 18, 2025*
*Completed: December 30, 2025*
*Status: ✅ Complete*


## Overview

Deploy the new Ospex frontend to ospex.org using Amoy testnet contracts, set up the agent infrastructure, and get Market Maker Michelle running on an automated schedule. Includes prompt simplification to reduce token costs.

## Goals

- Live site at ospex.org with new frontend
- Agent server deployed and running
- Michelle auto-evaluating slate every 8 hours
- Reduced Heroku costs through dyno cleanup
- Simplified Michelle prompt (no maths, judgment-only)

## Expected Timeline

3-4 days

## Monthly Impact

- Before: 6 dynos
- After: 4 dynos

---

## Phase 0: Cleanup Legacy Dynos

**Goal:** Remove unused dynos and reduce costs before adding new infrastructure.

### Tasks

- [x] Disable `node-test20230329` dyno
- [x] Disable `ospex-amoy` dyno
- [x] Disable `ospex-redesign` dyno
- [x] Disable `ospex-docs` dyno
- [x] Should just be `ospex` and `ospex-firebase`

### Notes

These dynos are legacy infrastructure that haven't been needed for months. Disabling before adding new dynos ensures we're not paying for overlap.

---

## Phase 1: GitHub Repository Setup

**Goal:** Get all code into GitHub repos that can connect to Heroku.

### Tasks

- [x] Create private GitHub repo: `ospex-frontend-v2.3`
- [x] Navigate to `ospex-lovable` folder locally
- [x] Initialize git if needed: `git init`
- [x] Add remote: `git remote add origin git@github.com:ospex-org/ospex-frontend-v3.git`
- [x] Push code: `git add . && git commit -m "Initial commit" && git push -u origin main`

- [x] Create private GitHub repo: `ospex-agents`
- [x] Navigate to `ospex-agent-server` folder locally
- [x] Initialize git: `git init`
- [x] Add remote: `git remote add origin git@github.com:ospex-org/ospex-agent-server.git`
- [x] Push code: `git add . && git commit -m "Initial commit" && git push -u origin main`

- [x] Create GitHub repo: `ospex-api` (can be public, secrets in .env)
- [x] Navigate to `ospex-api-server` folder locally
- [x] Initialize git: `git init`
- [x] Add remote: `git remote add origin git@github.com:ospex-org/ospex-api-server.git`
- [x] Push code: `git add . && git commit -m "Initial commit" && git push -u origin main`

### Notes

The Lovable GitHub connection can stay as-is - that's a separate repo for Lovable's build process. This new repo is specifically for Heroku deployment.

---

## Phase 2: Heroku Deployment

**Goal:** Get all three services running on Heroku.

### Tasks

**Frontend:**
- [x] Go to Heroku dashboard → `ospex` app
- [x] Disconnect current GitHub repo (ospex-front-end)
- [x] Connect to new repo: `ospex-frontend-v2.3`
- [x] Configure environment variables:
  - [ ] `REACT_APP_NETWORK=amoy` (or equivalent) (N/A)
  - [ ] Contract addresses for Amoy deployment (skipped, these are hardcoded in addresses file)
  - [x] Firebase config vars
  - [ ] API server URL (will be `https://ospex-api.herokuapp.com` or similar)
- [x] Deploy from main branch
- [x] Verify site loads at ospex.org

**API Server:**
- [x] Create new Heroku app: `ospex-api`
- [x] Connect to GitHub repo: `ospex-api-server`
- [x] Configure environment variables:
  - [x] Encryption secrets from .env
  - [x] Any API keys needed
- [x] Deploy from main branch
- [x] Test endpoint responds

**Agent Server:**
- [x] Create new Heroku app: `ospex-agents`
- [x] Connect to GitHub repo: `ospex-agent-server`
- [x] Configure environment variables:
  - [x] `ANTHROPIC_API_KEY`
  - [x] Firebase credentials
  - [x] Agent wallet private key (Michelle's wallet)
  - [x] Any other agent config
- [ ] Deploy from main branch (skipped, connected to github repo but not deployed yet)
- [ ] Verify dyno starts without errors (skipped, no reason to start this until it's in use)

### Verification

- [x] ospex.org loads in browser
- [x] Can connect wallet (MetaMask on Amoy)
- [ ] Games display with odds from Firebase (skipped, will test in later phases)
- [ ] Michelle's existing offers visible in Agent Offers tab (skipped, will test in later phases)

---

## Phase 3: No Maths Michelle

**Goal:** Simplify Michelle's prompt to reduce token usage and improve reliability. Move odds calculation from LLM to deterministic code.

### Current State

Michelle's prompt includes:
- Math examples for calculating overround
- Instructions for computing specific odds values
- Implied probability calculations

The evaluator already validates all this math anyway, so the LLM work is redundant.

### New Architecture

**Michelle outputs:**
```json
{
  "games": [
    {
      "contestId": "...",
      "markets": {
        "moneyline": {
          "status": "active",
          "edge_level": "standard",
          "notes": "Standard regular season game"
        },
        "spread": {
          "status": "active", 
          "edge_level": "tight",
          "notes": "High uncertainty, key player questionable"
        },
        "total": {
          "status": "no_interest",
          "reason": "Weather concerns, line likely to move"
        }
      }
    }
  ]
}
```

**Code calculates:**
- Target overround based on edge_level mapping
- Actual odds for both sides
- Exposure limits based on league/game profile

### Tasks

- [x] Create new prompt file: `prompt-v2.ts` (created `prompt.ts`, archived original file as `prompt-v1.ts`)
- [x] Define edge_level options: `tight` (101.5%), `standard` (101%), `generous` (100.5%)
- [x] Simplify system prompt - remove all math examples
- [x] Simplify user prompt - just ask for edge_level + notes per market
- [x] Update evaluator to handle new response format
- [ ] Add `calculateOddsFromEdgeLevel()` function (skipped: Michelle outputs confidence, we map that to an internal EdgeLevel, and then deterministic code computes the two-sided odds from the raw market odds)
- [x] Test locally with a few games
- [x] Compare token usage: old prompt vs new prompt (new prompt cost $0.15 for 102 offers)
- [x] Verify offers still pass validation
- [x] Push changes to agent repo

### Expected Token Savings

Current prompt is ~1500 tokens. New prompt should be ~500 tokens. At 5 games per batch, this saves ~5000 tokens per full slate evaluation.

---

## Phase 4: Michelle Cron Setup

**Goal:** Get Michelle running automatically every 8 hours.

### Tasks

- [x] Update scheduler for multi-agent use, using agent list from config

```typescript
// scheduler.ts - Modified approach
import { evaluateSlate } from './agents/market_maker_michelle/evaluator';
import { runOnce, runMatch } from './agent';
// ... etc

// Market makers: evaluation only, less frequent
cron.schedule('0 0,8,16 * * *', async () => {
  // Run all market maker evaluations
  await evaluateSlate(); // Michelle
  // await evaluateSlateBob(); // Future market maker
});

// Active traders: matching + on-chain, more frequent  
cron.schedule('*/5 * * * *', async () => {
  // Run all active trader logic
  await runMatch('degen_dan');
  // await runMatch('some_other_trader');
});

// agents.json
{
  "market_makers": ["market_maker_michelle"],
  "active_traders": ["degen_dan"]
}
```

### Notes

To create a new agent:
- `cp -r _template my_new_agent`
- Edit `config.json` (set ID, name, wallet, etc.)
- Edit `prompt.ts` (customize LLM prompts)
- Add to `agents.json`
- Set env var for wallet private key

- [x] Add Michelle to existing scheduler:
  - Import `evaluateSlate` from Michelle's evaluator
  - Add 8-hour cron for market maker evaluations
- [x] Keep existing Degen Dan matching schedule
- [ ] Test locally - trigger manual run (skipped)
- [x] Push to GitHub
- [x] Deploy to Heroku, single `ospex-agents` dyno (worker, changed from web)
- [x] Check Heroku logs to confirm scheduler initialized
- [x] Wait for first scheduled run
- [x] Verify new offers appear in Firebase
- [x] Verify offers display in frontend

### Verification

- [x] Heroku logs show "Michelle cron initialized"
- [x] After scheduled time, logs show evaluation ran
- [x] Firebase `agentOffers` collection has recent timestamps
- [x] Frontend Agent Offers tab shows fresh evaluations

---

## Phase 5: Final Verification & Documentation

**Goal:** Confirm everything works end-to-end and document the new architecture.

### End-to-End Test

- [x] Wait for Michelle cron to run at least once automatically
- [x] Open ospex.org in fresh browser
- [x] Connect wallet
- [x] Navigate to Agent Offers
- [x] Confirm Michelle's offers display with recent "evaluated X ago" timestamp
- [x] Select a game, view the odds slider
- [ ] (Optional) Create a test position to verify contract interaction works

### Architecture Documentation

- [x] Update README in each repo with:
  - What the service does
  - Environment variables required
  - How to run locally
  - Deployment notes

### Heroku Final State

| Dyno | Repo | Purpose | Cost |
|------|------|---------|------|
| `ospex` | ospex-frontend-v3 | Frontend | $7 |
| `ospex-firebase` | ospex-firebase | Odds monitoring | $7 |
| `ospex-agents` | ospex-agent-server | Michelle + future agents | $7 |
| `ospex-api` | ospex-api-server | Oracle encryption | $7 |

**Total: $28/month**

---

## Success Criteria

- [x] ospex.org serves new frontend on Amoy testnet
- [ ] Michelle's offers auto-refresh every 8 hours without manual intervention (should, we'll see)
- [x] Michelle uses simplified prompt (judgment only, no math)
- [x] Monthly Heroku cost reduced from $42 to $28
- [x] All repos in GitHub with proper structure

---

## Future Considerations (Not in Scope)

- On-demand Michelle re-evaluation (M28+)
- Payment flow for agent interactions (M28+)
- Degen Dan integration with Michelle (M28+)
- Additional agent templates (bowl game sentiment, etc.)
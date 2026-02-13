# Milestone 37: Contract Security Audit (AI) & Mainnet Readiness

*Created: January 27, 2026*
*Completed: February 1, 2028*
*Status: ✅ Complete*
*Notes: M37 scope-creeped into full deployment, ospex is now on Polygon, CL Functions is working, one test contest and four test speculations currently exist for M38*

---

## Overview

Prepare for mainnet deployment by (1) systematically stress-testing smart contracts for exploits, and (2) documenting everything required for the migration. Goal: eliminate unknowns so M38 deployment is mechanical execution, not discovery.

**Philosophy:** The contracts are going to hold real money. Once deployed, context-switching back to Solidity is painful and risky. Do the hard security work now. For the migration itself, the enemy is forgotten configuration and surprise dependencies - document everything before touching anything.

---

## Track 1: Contract Security Audit

Use Claude Code to systematically probe contracts for vulnerabilities. This isn't a formal audit.

### Attack Vectors to Test

| Category | Specific Tests |
|----------|----------------|
| **Reentrancy** | Can any external call be exploited to re-enter and drain funds? Test all functions that transfer ETH/tokens. |
| **Oracle Manipulation** | Can Chainlink oracle responses be manipulated or stale data exploited? What happens if oracle goes down? |
| **Rounding/Precision** | Can bet amounts or payouts be manipulated through rounding errors? Test edge cases with very small and very large amounts. |
| **Timing Attacks** | Can transactions be front-run profitably? Are there any time-dependent vulnerabilities around game resolution? |
| **Access Control** | Can non-owners call admin functions? Can users claim others' winnings? Can positions be modified after creation? |
| **Integer Overflow/Underflow** | Solidity 0.8+ has built-in protection, but verify no unchecked blocks create issues. |
| **Denial of Service** | Can anyone brick the contract or prevent legitimate users from withdrawing? |
| **Flash Loan Attacks** | Could flash loans be used to manipulate any contract state? |
| **Edge Cases** | What happens at exactly 0 USDC? At max uint256? When arrays are empty? When a game has no positions? |

### Methodology

1. **Static Analysis** - Claude Code reviews contract code looking for known vulnerability patterns
2. **Logic Review** - Walk through each public/external function and trace all state changes
3. **Exploit Attempts** - Write actual attack scripts that try to exploit identified weaknesses
4. **Edge Case Testing** - Deploy to local hardhat, run scenarios with extreme values

### Tasks

- [x] Inventory all deployed contracts and their addresses (testnet)
- [x] Document each contract's purpose and critical functions
- [x] Run static analysis pass on each contract
- [x] Create exploit test scripts for reentrancy scenarios
- [x] Create exploit test scripts for oracle manipulation
- [x] Test rounding with dust amounts (0.000001 USDC) and large amounts (1M USDC)
- [x] Verify access control on all admin functions
- [x] Test game resolution edge cases (ties, cancellations, oracle failures)
- [x] Document all findings (even if no vulnerabilities found)
- [x] Fix any identified issues
- [x] Re-test after fixes

### Files to Review

| Contract | Risk Level | Notes |
|----------|------------|-------|
| Core betting contract | Critical | Holds all user funds |
| USDC integration | Critical | Token transfers, approvals |
| Oracle integration | High | Game results, resolution |
| Position management | High | Creation, matching, claiming |
| Fee collection | Medium | Verify fees go to correct wallet |

### Success Criteria

- [x] All attack vectors tested with documented results
- [x] Zero critical or high-severity vulnerabilities remaining
- [x] Any medium/low issues documented with mitigation plan
- [x] Confidence level: "I would put my own money in this contract"

---

## Track 2: Mainnet Readiness Checklist

Document every single thing that needs to change for mainnet. No surprises on deploy day.

### Wallet Infrastructure

| Item | Testnet Value | Mainnet Value | Status |
|------|---------------|---------------|--------|
| Deployer wallet | N/A | 0xfd6C7Fc1F182de53AA636584f1c6B80d9D885886 | Done |
| Fee collection wallet | 0xdaC630aE52b868FF0A180458eFb9ac88e7425114 | 0xdaC630aE52b868FF0A180458eFb9ac88e7425114 | Done |
| Michelle's wallet | 0xB676D18cFb2D6F621b4B04adC8A6a676647a0679 | 0xB676D18cFb2D6F621b4B04adC8A6a676647a0679 | Done |
| Degen Dan's wallet | 0xeDf57Fc01028f4e5Ee9852eedebe5CD875130967 | 0xeDf57Fc01028f4e5Ee9852eedebe5CD875130967 | Done |
| Oracle subscription | Amoy | Polygon mainnet | Funded with LINK, pending consumer contract setup |

### Smart Contract Configuration

| Item | Change Required | Notes |
|------|-----------------|-------|
| Network ID | 80002 → 137 | Amoy to Polygon |
| USDC address | Testnet USDC → `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` | Polygon native USDC → `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |
| Chainlink oracle addresses | Testnet → Mainnet | Addresses listed below |
| Chainlink Functions Router | Testnet → `0xC22a79eBA640940ABB6dF0f7982cc119578E11De` | Mainnet → `0xdc2AAF042Aeff2E68B3e8E33F19e4B9fA7C73F10` |
| DON ID | Testnet → `fun-polygon-amoy-1` - `0x66756e2d706f6c79676f6e2d616d6f792d310000000000000000000000000000` | Mainnet → `fun-polygon-mainnet-1` - `0x66756e2d706f6c79676f6e2d6d61696e6e65742d310000000000000000000000` |
| Subscription ID | Testnet → 416 | Mainnet → 191 |
| LINK Token (ERC-677) | Testnet → `0x0Fd9e8d3aF1aaee056EB9e802c3A762a667b1904` | Mainnet → `0xb0897686c545045aFc77CF20eC7A532E3120E0F1` |
| Contract verification | Skip initially | Deploy unverified, verify later |

### Firebase Configuration

| Collection | Action | Notes |
|------------|--------|-------|
| All position collections | Create mainnet versions | `polygonPositions` vs `amoyPositions` - Functions file configured and deployed |
| Agent collections | No change needed | Already use `network` field in data/IDs - supports multi-network |
| Config documents | Review | Likely none - config is in env vars, not Firestore. Skip unless issues arise |
| Security rules | No change needed | Rules are auth-based, not network-specific |

**Decision needed:** Separate collections per network, or single collections with network field? Pros/cons:
- Separate: Cleaner isolation, can run testnet and mainnet simultaneously, but more collections to manage
- Combined: Simpler structure, but risk of testnet/mainnet data mixing
- Using Separate where appropriate

### Environment Variables

| Service | Variables to Update |
|---------|---------------------|
| Frontend (Heroku) | `VITE_NETWORK`, `VITE_CONTRACT_ADDRESS`, `VITE_USDC_ADDRESS` |
| Agent Server (Heroku) | Contract addresses, wallet private keys, network config |
| Indexer | Network endpoints, contract addresses |
| Any other services? | Document all |

### External Services

| Service | Testnet Setup | Mainnet Setup | Notes |
|---------|---------------|---------------|-------|
| Chainlink | Amoy subscription | Polygon subscription | Funded |
| Alchemy/Infura RPC | Amoy endpoint | Polygon endpoint | Check rate limits |
| Polygonscan | N/A | API key for verification | When ready to verify |

### Deployment Sequence

Document the exact order of operations:

```
1. Pre-deployment
   - [x] All wallets created and secured
   - [x] All env vars documented (all changed)
   - [x] Mainnet POL in deployer wallet
   - [x] Mainnet LINK for Chainlink subscription
   - [x] Mainnet USDC for agent wallets

2. Contract deployment
   - [x] Deploy core contract
   - [x] Deploy any auxiliary contracts
   - [x] Verify deployment successful
   - [x] Note all contract addresses (in shortcut)

3. Backend configuration
   - [x] Update Heroku env vars
   - [x] Restart services
   - [ ] Verify agents can read contracts (M38)

4. Frontend configuration
   - [x] Update hosting env vars
   - [x] Redeploy frontend
   - [x] Verify wallet connection works

5. Indexer configuration
   - [x] Point to mainnet contracts
   - [x] Verify events are being captured

6. Smoke testing
   - [x] Connect wallet on mainnet
   - [ ] Create small position (1 USDC) (M38)
   - [ ] Verify position appears in UI (M38)
   - [ ] Verify agents see position (M38)
   - [ ] Test instant match flow (M38)
   - [ ] Wait for game resolution (or manually trigger if possible) (M38)
   - [ ] Claim winnings (M38)
```

### Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Contract bug discovered post-deploy | Keep deployment minimal initially, conservative limits |
| Wallet compromise | Use hardware wallet for deployer, separate hot wallets for agents with limited funds |
| Oracle failure | Document manual resolution process |
| Frontend/backend mismatch | Deploy backend first, verify, then frontend |
| User accidentally uses wrong network | Clear UI indication of network, block if wrong |

### Conservative Limits for Initial Deployment

| Parameter | Initial Value | Rationale |
|-----------|---------------|-----------|
| Max bet size | 3 USDC | Limit exposure while validating |
| Michelle's bankroll | 50 USDC | Enough to provide liquidity, not enough to hurt if exploited |
| Dan's bankroll | 25 USDC | He loses anyway |
| Your test bankroll | 20 USDC | Enough to test all flows |

### Tasks

- [x] Create all mainnet wallet addresses (document securely)
- [x] Document all testnet configuration (baseline)
- [x] Document all required mainnet configuration changes
- [x] Decide on Firebase collection strategy
- [x] Create Chainlink subscription on Polygon mainnet
- [x] Document RPC endpoint requirements
- [x] Write deployment script (or document manual steps)
- [ ] Create rollback plan (what if something goes wrong?) (no rollback plan)
- [x] Estimate MATIC/LINK/USDC needed for deployment + initial operation

---

## Time Estimate

| Track | Estimate |
|-------|----------|
| Track 1: Contract security audit | 6-8 hours |
| Track 2: Mainnet readiness checklist | 3-4 hours |

**Total:** 9-12 hours across 2-3 sessions

---

## Open Questions

1. **Collection strategy** - Separate Firebase collections per network, or combined with network field?
2. **Agent wallet security** - Hardware wallet for agents, or acceptable to use hot wallets with limited funds?
3. **Indexer scope** - Does indexer need to support both networks simultaneously, or switch entirely?
4. **Verification timing** - At what point (traction level) do we verify contracts publicly?

---

## Success Criteria

- [x] All contract attack vectors tested with documented results
- [x] Zero critical vulnerabilities, any medium/low documented
- [ ] Complete inventory of all configuration changes required (skipped, just have what's in this doc)
- [ ] Deployment sequence documented step-by-step (skipped)
- [ ] Time estimate for M38 deployment is confident (not "idk maybe a weekend") (n/a)
- [ ] You could hand this document to another developer and they could execute the deployment (unlikely)

---

## Notes

### On "Unverified" Contracts

Deploying unverified is fine for now.

### On Security Audit Depth

This isn't a formal audit. A real audit from Trail of Bits or OpenZeppelin costs $50k+ and takes weeks. What we're doing is:
- Better than nothing
- Catches the obvious stuff
- Builds confidence through systematic review
- Documents our due diligence

### On the "Why Now" for Mainnet

Testnet served its purpose: you could break things without losing money. But now:
- The core betting loop works
- The agent infrastructure works
- MetaMask testnet issues are blocking real testing
- You need to validate with real stakes before building more features
- March Madness is a potential moment (even if soft launch)

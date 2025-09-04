# OSPEX v2 System Overview

## Project Summary
Ospex is a peer-to-peer sports betting orderbook using matched pairs for fulfillment. Core principles: no sign-up, no fees, no limits. Currently v2 in development with orderbook matching (vs v1 parimutuel pools), leaderboards with prizes, secondary marketplace, and agent integration.

**Status**: Testing phase - contracts deployed to Amoy testnet, frontend running locally, core functionality working but debugging data consistency issues.

---

## Architecture Overview

### Smart Contracts (Modular Design)
**Core Contract:**
- `OspexCore.sol` - Module registry and access control
- `OspexTypes.sol` - Shared data structures

**Business Logic Modules:**
- `ContestModule.sol` - Contest creation/management
- `SpeculationModule.sol` - Speculation (bet types) management  
- `PositionModule.sol` - User positions, matching, claiming
- `LeaderboardModule.sol` - Leaderboard creation/management
- `RulesModule.sol` - Leaderboard rules validation
- `SecondaryMarketModule.sol` - Position trading
- `TreasuryModule.sol` - Fee collection/routing
- `ContributionModule.sol` - Contribution handling
- `OracleModule.sol` - Oracle interactions

**Scoring Modules:**
- `MoneylineScorerModule.sol` - Win/loss scoring
- `SpreadScorerModule.sol` - Point spread scoring  
- `TotalScorerModule.sol` - Over/under scoring

### Frontend Stack
- **Framework**: React + Vite + Tailwind
- **Status**: Running locally, not deployed
- **Data Source**: Firebase (events + market data)
- **Deployment Plan**: Heroku + GitHub integration

### Backend Services
- **Firebase**: Event storage + market data from jsonodds.com
- **Google Cloud Functions**: Event handlers
- **Heroku Server**: Pulls jsonodds data every 30 min
- **API Server**: Encrypts secrets URL for oracle

### Oracle Integration
- **Provider**: Chainlink
- **Functions**: 
  - Contest validation (Rundown, Sportspage, Jsonodds APIs)
  - Contest scoring 
  - Market odds updates for leaderboard rules
- **Status**: Working reliably

### Agent Integration
- **MCP Server**: Built but untested since recent protocol changes
- **Purpose**: Agent participation + liquidity generation

---

## Current Deployment Status

### Smart Contracts
- **Network**: Polygon Amoy testnet
- **Status**: ⚠️ **NEEDS REDEPLOYMENT** - Core contract bug with leaderboard fee processing
- **Deployment Process**: Redeploy all contracts when core changes

### Frontend  
- **Environment**: Local development only
- **Testing**: Can create leaderboards and place bets
- **Focus**: Rules testing and validation

### Data Pipeline
- **Contract Events → Firebase**: ✅ Working
- **Jsonodds Monitoring**: ✅ Working (30min intervals)
- **Oracle Workflows**: ✅ Working (validation, scoring, odds updates)

---

## Known Issues & Pain Points

### Primary Development Blockers
1. **Data Consistency Issues**
   - Odds conversion discrepancies between contract/Firebase/frontend
   - Position perspective inversions (maker seeing taker view)
   - Timestamp conversion errors
   - Rounding issues (e.g., 2.60 odds displaying as 2.59)

2. **Root Cause Investigation Challenge**  
   - Hard to determine if bug is in: contract logic, Firebase data, or frontend calculations
   - No clear debugging workflow for data flow issues

3. **Contract Redeployment Friction**
   - Core contract changes require full redeployment
   - Breaks existing test data and workflows

### Secondary Issues
- LINK token approval nonce issues (occasional)
- MCP server sync with recent protocol changes
- Frontend UI/UX decisions still in flux

---

## Data Flow Map

```
1. Contest Creation → Chainlink Oracle (validation) → Contract Event → Firebase
2. Position Creation → Contract Event → Firebase → Frontend Display
3. Market Data → Jsonodds API → Heroku Server → Firebase → Leaderboard Rules
4. Position Matching → Contract Logic → Event → Firebase → Frontend Update
5. Scoring → Chainlink Oracle → Contract → Event → Firebase → Frontend
```

### Critical Data Conversion Points ⚠️
- **Odds Format**: Decimal ↔ Contract Format ↔ Display Format
- **Timestamps**: Block time ↔ Firebase ↔ Display time
- **Position Data**: Contract perspective ↔ User perspective

---

## Development Workflow Issues

### Current Testing Session Pattern (Problematic)
1. Create leaderboard via frontend
2. Notice UI inconsistency  
3. Debug: contract bug? Firebase data? Frontend calc?
4. Fix one issue, discover another
5. Get distracted by unrelated broken feature
6. Lose focus on original goal

### Needed: Systematic Debugging Approach
- Isolate data flow validation
- Component-level testing strategy  
- Clear issue categorization (contract vs data vs display)

---

## Next Priorities

### Immediate (This Week)
- [ ] Redeploy contracts with core bug fix
- [ ] Create data validation checklist for odds/position display
- [ ] Document debugging workflow for data consistency

### Short Term (Next 2 weeks)  
- [ ] Define MVP feature boundaries
- [ ] Set up systematic testing approach
- [ ] Create git workflow for incremental changes

### Long Term
- [ ] Deploy frontend to staging environment
- [ ] Audit preparation
- [ ] Agent integration testing

---

## Key Files & Resources

### Contract Deployment
- **Network**: Polygon Amoy testnet
- **Deployment Command**: [TO DOCUMENT]
- **Contract Addresses**: [TO UPDATE after redeployment]

### API Integrations
- **Jsonodds**: Market data (30min schedule)
- **Rundown/Sportspage**: Contest validation
- **Chainlink Oracle**: Contest validation, scoring, odds updates

### Development Environment
- **Frontend**: `yarn dev` (local)
- **Contracts**: Hardhat + Amoy testnet
- **Database**: Firebase (events + market data)

---

*Last Updated: [Sept. 3, 2025]*  
*Next Review: After contract redeployment*
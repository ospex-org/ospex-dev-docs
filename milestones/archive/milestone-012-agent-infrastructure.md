# Milestone 012: Autonomous Agent Infrastructure
*Created: October 3, 2025*  
*Target Completion: October 4, 2025*
*Status: ✅ Complete*

## Problem Statement
To validate end-to-end platform functionality and leaderboard competition mechanics, we need an autonomous agent that can analyze markets, make betting decisions, and execute transactions independently. Currently, all testing requires manual interaction, limiting our ability to stress-test the system and validate multi-user competitive dynamics.

## Context
Milestone 11 completed contest detail pages, giving users comprehensive market visibility. Before moving to production, we need to validate that the entire betting flow works autonomously: market analysis → decision-making → contract execution → leaderboard tracking. An autonomous agent provides this validation while also serving as the foundation for future bot-powered liquidity and competition features.

## Scope & Boundaries

### In Scope
- Standalone Node.js agent server with wallet and testnet USDC
- Firebase read access for market data retrieval
- Simple LLM integration (Claude API) for decision-making
- Basic betting logic using team popularity heuristic
- Direct smart contract interaction (purchase AND write contracts)
- Cron-based scheduling (1-2 runs per day)
- Firebase agent registry for identification
- Frontend visual indicator distinguishing agent from human users
- 2-day test leaderboard competition: you vs agent
- Manual contest/speculation creation workflow for agent targets

### Out of Scope
- Multiple competing agents (single agent only)
- Sophisticated multi-factor analysis or ML models
- Insight posting or social features
- Real-time agent monitoring dashboard
- Agent management UI in frontend
- Automated contest/speculation creation
- On-chain agent registry (Firebase only for M12)
- Deployment to production hosting (local development only)
- Portfolio optimization or bankroll management

## Design Strategy: Simplicity First

**Core principle**: Get the plumbing working with minimal complexity
- Single agent, simple decision logic, scheduled execution
- Manual intervention where automation would add complexity
- Focus on proving architecture, not intelligence
- Defer sophistication to Milestone 13

**Coordination workflow**: Agent decides → you create → agent executes
- Agent analyzes available markets and logs desired bets
- You manually create contests/speculations based on agent logs
- Agent executes transactions on next scheduled run
- You manually match agent positions when needed (not counted for leaderboard)

## System Architecture

```
┌─────────────────────────┐
│   Agent Server          │
│   (Node.js/local)       │
│                         │
│   - Own wallet          │
│   - Claude API client   │
│   - Cron scheduler      │
│   - Contract interface  │
└───────────┬─────────────┘
            │
       READ │ (market data, available positions)
            ▼
┌─────────────────────────┐
│      Firebase           │◄────────── Contract Events
│   (market state)        │            (existing listener)
│                         │
│   - contests            │
│   - speculations        │
│   - unmatched_pairs     │
│   - agents (registry)   │
└─────────────────────────┘
            │
            │ Agent does NOT write to Firebase
            │
            ▼
┌─────────────────────────┐
│   Smart Contracts       │◄────────── Agent executes:
│   (testnet)             │            - createUnmatchedPair()
│                         │            - completeUnmatchedPair()
│   - OspexCore           │            - registerUser() (for leaderboard participation)
│   - OrderBook           │
└─────────────────────────┘
            │
            │ Events emitted (PositionCreated, PositionMatched, etc.)
            │
            └──────────────────────────► Firebase
                                          (via existing listener)
```

## Implementation Plan

### Phase 1: Agent Server Foundation (4-6 hours)
- [X] **Project Setup**
  - Create new Node.js project in `/ospex-agent-server` directory
  - Install dependencies: ethers.js, @anthropic-ai/sdk, node-cron, firebase-admin
  - Set up TypeScript configuration
  - Create `.env` file for secrets (API keys, wallet private key, RPC URL)

- [X] **Wallet Configuration**
  - Generate new testnet wallet for agent
  - Fund wallet with 1000 testnet USDC

- [X] **Firebase Integration**
  - Create functions to read contests, speculations, unmatched_pairs
  - Test data retrieval and parsing

### Phase 2: Decision Logic & LLM Integration (4-6 hours)
- [X] **Market Analysis Module**
  - Fetch available contests from Firebase
  - Filter for upcoming games (next 24-48 hours)
  - Identify available speculations (moneyline, spread, totals)
  - Check for existing unmatched pairs and matched positions

- [X] **Claude API Integration**
  - Set up Anthropic SDK client
  - Create prompt template for betting decisions
  - Implement simple logic: "analyze these games and pick the more popular team"
  - Parse LLM response for bet decisions (game, side, amount)
  - Add logging for all API calls and responses

- [X] **Decision Validation**
  - Check agent has sufficient USDC balance
  - Verify target speculation exists
  - Log intended bets for manual review
  - Handle error cases gracefully

### Phase 3: Contract Execution (4-6 hours)
- [X] **Smart Contract Interface**
  - Import contract ABIs and addresses
  - Create ethers.js contract instances
  - Implement createUnmatchedPair() function call
  - Implement completeUnmatchedPair() function call
  - Implement registerUser() function call
  - Add transaction confirmation and error handling

- [X] **Betting Flow Logic**
  - Check if matching position exists (unmatched pair)
  - If match exists: call completeUnmatchedPair()
  - If no match: call createUnmatchedPair()
  - Log all transactions with txHash and status
  - Update local state after successful transactions

- [ ] **Scheduling Setup** (did not complete this, ran manually)
  - Configure node-cron for scheduled execution
  - Set initial schedule: twice daily (morning and evening)
  - Add manual trigger option for testing
  - Implement graceful shutdown handling

### Phase 4: Frontend Integration (2-3 hours) (skipped)
- [ ] **Agent Registry Display**
  - Query agents collection in Firebase
  - Create agent registry collection with agent wallet address
  - Add "bot" badge/indicator to agent positions
  - Style agent differently on leaderboard (subtle icon)
  - Show agent info on user profile page (if viewing agent)

- [ ] **Visual Indicators**
  - Small bot icon next to agent username on leaderboard
  - Agent positions marked in market depth displays
  - Consistent styling across all agent appearances

- [ ] **Testing & Validation**
  - Verify agent appears correctly in UI
  - Check leaderboard calculations include agent
  - Ensure agent positions display properly in contest details

## Success Criteria

### Functionality Acceptance
- [ ] Agent server runs on schedule without manual intervention (no)
- [X] Agent successfully reads market data from Firebase
- [X] Agent makes betting decisions using Claude API
- [X] Agent executes createUnmatchedPair() transactions successfully
- [X] Agent creates unmatched pairs when no match exists
- [X] Agent transactions appear in Firebase via contract events
- [X] Agent appears on leaderboard with correct positions
- [ ] Frontend distinguishes agent from human users visually (no)

### User Experience Acceptance
- [ ] You can compete against agent through normal UI
- [ ] Agent positions are clearly identifiable
- [ ] Leaderboard accurately tracks agent performance
- [X] 2-day test leaderboard completes successfully (partial - executed 2 positions w/ agent code)
- [X] Manual position matching workflow is manageable

### Technical Acceptance
- [X] Agent handles errors gracefully (no crashes)
- [X] All transactions confirmed on testnet
- [X] Logging provides clear audit trail
- [X] Firebase integration works reliably
- [X] Claude API calls stay within reasonable costs

## Technical Requirements

### Agent Server Stack
```
Node.js + TypeScript
- ethers.js (v6.x) - Smart contract interaction
- @anthropic-ai/sdk - Claude API client
- firebase-admin - Firebase read access
- node-cron - Scheduled execution (not used)
- dotenv - Environment variable management
- winston - Logging
```

### Firebase Agent Registry Schema (skipped)
```javascript
// Collection: agents
{
  agentId: "ospex-bot-001",
  walletAddress: "0x...",
  name: "Ospex Alpha",
  type: "autonomous",
  strategy: "simple-popularity",
  version: "1.0",
  active: true,
  createdAt: timestamp,
  lastRunAt: timestamp,
  totalBets: number,
  totalVolume: number
}
```

### Environment Variables
```bash
# .env file
PRIVATE_KEY=0x...                    # Agent wallet private key
AMOY_RPC_URL=https://testnet.rpc...       # Testnet RPC endpoint
ANTHROPIC_API_KEY=sk-ant-...         # Claude API key
FIREBASE=...                         # Firebase related
OSPEX_CORE_ADDRESS=0x...             # and others
```

### Claude API Prompt Template (Example)
```
You are a sports betting agent analyzing markets on Ospex.
Available games in next 48 hours:
[Game data from Firebase]

Rules:
- Pick 2-3 games to bet on
- Use simple heuristic: bet on more popular/favored team
- Bet size: $50 per position
- Return JSON: [{ game, speculation, side, amount }]

Analyze and return your bets.
```

### Contract Interaction Functions
```typescript
interface BetDecision {
  contestId: string;
  speculationId: string;
  side: 'yes' | 'no';
  amount: 20 | 50 | 100;
  odds: number;
}

// all code contained in ospex-agent-server project

```

## Manual Coordination Workflow

Since agent cannot create contests/speculations, follow this process:

### Day Before Test Leaderboard
1. Agent runs analysis, logs desired bets to console and file
2. You review agent log and identify missing contests/speculations
3. You manually create required contests and speculations in UI
4. Verify markets exist in Firebase

### During Test Leaderboard
1. Agent runs on schedule (9 AM, 9 PM) (skipped, manual run)
2. Agent finds matches and purchases, or creates unmatched pairs
3. You check for agent unmatched pairs needing matching
4. You manually match agent positions (not counted for your leaderboard score)
5. Monitor leaderboard to see competition results

### Post-Test Leaderboard
1. Review agent performance vs your performance
2. Analyze agent logs for decision quality
3. Identify improvements for Milestone 13
4. Document any bugs or issues

## Integration Notes

### Existing Systems
- Agent reads from Firebase (no writes)
- Agent uses existing smart contract interfaces
- Existing contract event listener populates Firebase
- Frontend queries agents collection for visual indicators
- Leaderboard calculations automatically include agent

### Data Flow
```
Agent → Claude API → Decision
Decision → Smart Contract → Transaction
Transaction → Event → Listener → Firebase
Firebase → Frontend → Display
```

### Manual Intervention Points
- Contest/speculation creation (you create based on agent logs)
- Position matching (you match agent unmatched pairs)
- Schedule adjustments (you modify cron schedule if needed) (skipped, manual)
- Balance management (you fund agent wallet if depleted)

## Risk Assessment
- **Medium complexity**: Multiple integrations (Firebase, Claude API, contracts)
- **Low risk**: Local execution, testnet only, manual fallbacks available
- **Medium value**: Validates end-to-end flow, foundation for future agent features
- **Coordination overhead**: Manual steps required due to simplified scope

## Development Tools & Workflow

### Initial Setup (Local)
1. Create `/ospex-agent-server` directory in project root
2. Initialize Node.js project with TypeScript
3. Set up environment variables and secrets
4. Generate and fund agent wallet
5. Test Firebase connection and data retrieval

### Development Process
1. Build modules incrementally (wallet → Firebase → LLM → contracts)
2. Test each module independently before integration
3. Use console logging extensively for debugging
4. Run manual executions
5. Monitor testnet explorer for transaction confirmation

### Testing Strategy
- Unit test individual modules (wallet, Firebase, LLM)
- Integration test full betting flow with manual trigger
- Run leaderboard test with real competition
- Document all issues and edge cases

## Success Metrics
- Agent completes 4 scheduled runs over 2 days without crashes (skipped)
- Agent successfully places at least 3 bets during test period (did 2 bets)
- At least 1 bet creates new position (createUnmatchedPair) (both were created, none matched)
- Agent appears on leaderboard with correct win/loss record
- You can identify agent positions in UI consistently
- Manual coordination workflow is manageable (< 30 min/day) (not sure about this)

## Completion Definition
Milestone complete when autonomous agent successfully competes against you on a test leaderboard, making betting decisions via Claude API, executing transactions on smart contracts, and appearing correctly in the frontend UI with proper visual identification.

## Looking Ahead to Milestone 13
*Deferred to M13 or later:*
- Sophisticated multi-factor analysis
- Insight posting and social features
- Real-time monitoring dashboard
- Multiple competing agents
- Agent management UI
- Deployment to Heroku
- On-chain agent registry

*This milestone proves the agent architecture works. M13 (or later) will make it intelligent and production-ready.*
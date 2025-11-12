# Milestone 016: Agent Bet Tiering Logic
*Created: November 10, 2025*  
*Updated: November 11, 2025*
*Status: ✅ Complete*

## Problem
Degen Dan (agent) currently creates all positions he likes. Need him to tier bets into high-conviction (create immediately) and medium-conviction (store as interest, wait for counterparty). Also need to avoid stale analysis and prevent information leakage about agent's acceptable odds range.

## Build
Logic in Dan's decision-making that:
1. Scores each potential bet with conviction level (0-10)
2. Creates positions for high-conviction (8-10)
3. Stores interests for medium-conviction (4-7) with target odds only
4. Stores private acceptable range in memory.json (not Firebase)
5. Re-analyzes games close to start time to avoid stale decisions

## Success Criteria
- [X] Agent evaluates games and assigns conviction scores (LLM returns 0-1 confidence)
- [X] High-conviction bets create positions using single-tx (speculation + pair)
- [X] Medium-conviction bets write to `agent_interests` with targetOddsDecimal only
- [X] Acceptable range stored in memory.json (private)
- [X] Can see both high and medium bets in one agent run
- [X] Agent logs show conviction tier and which games re-analyzed

### 1. Update memory.json Structure
Add analysis timestamp and private acceptable range:

```typescript
// In types.ts - add new type
export interface AgentDecisionMemory {
  jsonoddsID: string;
  contestId?: string;
  conviction: number;  // 0-10
  analyzedAt: string;  // ISO timestamp
  
  // Private - agent's acceptable range (NOT in Firebase)
  acceptableOddsRange: {
    min: number;
    max: number;
  };
  
  targetOdds: number;  // What was stored publicly
  action: 'created_position' | 'stored_interest' | 'skipped';
}

// In memory.json structure
{
  lastRun: string;
  decisions: AgentDecisionMemory[];
}
```

### 2. Re-analysis Logic
Check if game needs re-analysis based on time:

```typescript
// In agent.ts
function needsReanalysis(game: any, memory: any): boolean {
  const previous = memory.decisions?.find(d => d.jsonoddsID === game.jsonoddsID);
  
  // Never analyzed before
  if (!previous) return true;
  
  // Check if odds moved significantly (>5%)
  const currentOdds = getMarketOdds(game, previous.market, previous.pick);
  const previousOdds = previous.targetOdds;
  const oddsChange = Math.abs(currentOdds - previousOdds) / previousOdds;
  
  if (oddsChange > 0.05) {  // 5% threshold
    console.log(`Odds moved ${(oddsChange * 100).toFixed(1)}% on ${game.HomeTeam} vs ${game.AwayTeam} - re-analyzing`);
    return true;
  }
  
  // No significant changes - use cached decision
  console.log(`Using cached decision for ${game.HomeTeam} vs ${game.AwayTeam}`);
  return false;
}

async function runEvaluation() {
  const memory = loadMemory();
  const allGames = await getUpcomingGames();
  
  // Filter to games needing analysis (new or odds changed)
  const gamesToAnalyze = allGames.filter(g => needsReanalysis(g, memory));
  
  console.log(`Analyzing ${gamesToAnalyze.length} of ${allGames.length} games`);
  
  // Continue with evaluation...
}
```

### 3. Conviction-Based Routing
Convert LLM confidence (0-1) to conviction (0-10) and route:

```typescript
// In agent.ts
async function processPicks(picks: LlmPick[], games: any[]) {
  for (const pick of picks) {
    const game = games.find(g => g.jsonoddsID === pick.contestId);
    if (!game) continue;
    
    // Convert confidence 0-1 → conviction 0-10
    const conviction = Math.round((pick.confidence ?? 0.5) * 10);
    
    // Calculate odds range
    const marketOdds = getMarketOdds(game, pick.market, pick.pick);
    const minOdds = marketOdds * 0.95;
    const maxOdds = marketOdds * 1.05;
    const targetOdds = (minOdds + maxOdds) / 2;
    
    if (conviction >= 8) {
      // HIGH CONVICTION - create position on-chain
      console.log(`HIGH (${conviction}/10): ${pick.pick} on ${game.HomeTeam} vs ${game.AwayTeam}`);
      
      await createPositionWithSpeculation(game, pick, targetOdds);
      
      // Save to memory
      saveDecision({
        jsonoddsID: game.jsonoddsID,
        conviction,
        analyzedAt: new Date().toISOString(),
        acceptableOddsRange: { min: minOdds, max: maxOdds },
        targetOdds,
        action: 'created_position'
      });
    }
    else if (conviction >= 4) {
      // MEDIUM CONVICTION - store interest only
      console.log(`MEDIUM (${conviction}/10): ${pick.pick} on ${game.HomeTeam} vs ${game.AwayTeam}`);
      
      await addAgentInterest({
        agentWallet: config.wallet.address,
        agentName: config.name,
        contestId: game.contestId || '',
        jsonoddsID: game.jsonoddsID,
        speculationScorer: getScorerAddress(pick.market),
        theNumber: pick.line ? String(pick.line) : null,
        positionType: getPositionType(pick.pick),
        positionTypeString: formatPickString(pick),
        targetOddsDecimal: targetOdds,  // Only public odds
        maxWagerUSDC: pick.amount || 50,
        convictionScore: conviction
      });
      
      // Save to memory with private range
      saveDecision({
        jsonoddsID: game.jsonoddsID,
        conviction,
        analyzedAt: new Date().toISOString(),
        acceptableOddsRange: { min: minOdds, max: maxOdds },  // Secret
        targetOdds,
        action: 'stored_interest'
      });
    }
    else {
      // LOW CONVICTION - skip
      console.log(`LOW (${conviction}/10): Skipping ${game.HomeTeam} vs ${game.AwayTeam}`);
      saveDecision({
        jsonoddsID: game.jsonoddsID,
        conviction,
        analyzedAt: new Date().toISOString(),
        acceptableOddsRange: { min: 0, max: 0 },
        targetOdds: 0,
        action: 'skipped'
      });
    }
  }
}
```

### 4. Single-Transaction Position Creation
Use your new contract function for efficiency:

```typescript
// In contracts.ts
async function createPositionWithSpeculation(
  game: any, 
  pick: LlmPick, 
  targetOdds: number
) {
  // Check if speculation exists
  const speculationExists = await checkSpeculationExists(
    game.contestId, 
    pick.market
  );
  
  if (speculationExists) {
    // Normal flow - just create unmatched pair
    await createUnmatchedPair(/* ... */);
  } else {
    // Single transaction - create speculation + pair
    await createUnmatchedPairWithSpeculation(/* ... */);
  }
}
```

## Testing

### Manual Test Flow:
1. Run agent evaluation: `yarn agent:evaluate`
2. Check console output:
   - Should show HIGH/MEDIUM/LOW tags
   - Should show which games re-analyzed vs skipped
3. Check Firebase `agent_interests`:
   - Should have medium-conviction bets
   - Should only have `targetOddsDecimal` (not min/max)
4. Check memory.json:
   - Should have `acceptableOddsRange` for all decisions
   - Should have `analyzedAt` timestamps
5. Check on-chain:
   - Should have high-conviction positions created
6. Run agent again 1 hour later:
   - Should skip games with stable odds (cached decisions)
   - Should only re-analyze if odds moved >5%
7. Test odds change:
   - Manually update odds in Firebase for a game
   - Run agent again
   - Should re-analyze only that game

### Expected Output:
```
Analyzing 12 of 45 games
HIGH (9/10): away on Warriors vs Bucks
MEDIUM (6/10): home on Panthers vs Golden Knights
LOW (3/10): Skipping Rangers vs Oilers
```

## Notes
- Conviction thresholds (4-7, 8-10) can be adjusted if needed
- Re-analysis threshold (5% odds change) can be tuned based on testing
- temperature: 0 in Anthropic call ensures deterministic responses
- Private acceptable range never leaves memory.json
- Users only see target odds, can't game the agent

## Done When
- Agent runs and shows mix of HIGH/MEDIUM/LOW picks
- Firebase `agent_interests` has medium-conviction bets with target odds only
- memory.json has private acceptable ranges
- On-chain has high-conviction positions

# Milestone 015: Agent Interest Storage Schema
*Created: November 10, 2025*  
*Target Completion: November 11, 2025*
*Status: ✅ Complete*

## Problem
Agents need a place to store "medium-conviction" bets - positions they would take if counterparty volume exists, but won't initiate themselves.

## What You're Building
Firebase collection `agent_interests` with schema for storing agent betting interests and ability to query by game.

## Success Criteria
- [X] Collection exists in Firebase with correct structure
- [X] Can manually write an interest document
- [X] Can query interests by game (jsonoddsID)
- [X] Can read back what was written

## Schema

```javascript
// Collection: agent_interests
{
  id: string,                    // Auto-generated doc ID
  agentWallet: string,           // Agent's wallet address
  agentName: string,             // "Degen Dan"
  contestId?: string,            // Numeric on-chain ID (empty if not yet created)
  jsonoddsID: string,            // JsonOdds ID (always present)
  speculationScorer: string,     // Scorer contract address (determines market type)
  theNumber?: number,            // For spread/total only (null for moneyline)
  positionType: 'Upper' | 'Lower',  // Upper or Lower position
  positionTypeString: string,    // Human-readable (e.g., "Eagles", "over", "Warriors -3")
  
  // What the agent would accept
  minOddsDecimal: number,        // e.g., 1.85
  maxOddsDecimal: number,        // e.g., 2.10
  maxWagerUSDC: number,          // e.g., 100
  
  // Metadata
  convictionScore: number,       // 0-10, medium conviction ~4-7
  createdAt: Timestamp,
  expiresAt: Timestamp,          // Auto-expire old interests
  status: 'active' | 'expired' | 'filled'
}
```

## Implementation

### 1. Create Test Document (Manual)
Go to Firebase console and manually create one document in `agent_interests` collection with all fields populated.

**Example document:**
```javascript
{
  agentName: "Degen Dan",
  agentWallet: "0xeDf57Fc01028f4e5Ee9852eedebe5CD875130967",
  contestId: "",  // Empty if game not on-chain yet
  jsonoddsID: "069450da-621b-4fd9-991e-65a084648ec4",
  speculationScorer: "0x0c36eb2aaec0915b7e68420dbfa180f9dc9b8a9c",
  theNumber: 0,
  positionType: "Upper",
  positionTypeString: "Vegas Golden Knights",
  minOddsDecimal: 1.85,
  maxOddsDecimal: 2.1,
  maxWagerUSDC: 100,
  convictionScore: 5,
  createdAt: Timestamp.now(),
  expiresAt: Timestamp.fromMillis(Date.now() + 24 * 60 * 60 * 1000),
  status: "active"
}
```

### 2. Write Script to Add Interest
Create simple Node.js script (or add to existing agent code):

```javascript
async function storeAgentInterest(data) {
  const doc = {
    agentWallet: data.agentWallet,
    agentName: data.agentName,
    contestId: data.contestId || "",  // Empty string if not on-chain
    jsonoddsID: data.jsonoddsID,
    speculationScorer: data.speculationScorer,
    theNumber: data.theNumber ?? null,
    positionType: data.positionType,
    positionTypeString: data.positionTypeString,
    minOddsDecimal: data.minOddsDecimal,
    maxOddsDecimal: data.maxOddsDecimal,
    maxWagerUSDC: data.maxWagerUSDC,
    convictionScore: data.convictionScore,
    createdAt: Timestamp.now(),
    expiresAt: Timestamp.fromMillis(Date.now() + 24 * 60 * 60 * 1000),
    status: 'active'
  };
  
  await db.collection('agent_interests').add(doc);
}
```

### 3. Query by Game
```javascript
async function getInterestsForGame(jsonoddsID) {
  const snap = await db.collection('agent_interests')
    .where('jsonoddsID', '==', jsonoddsID)
    .where('status', '==', 'active')
    .get();
  
  return snap.docs.map(d => ({ id: d.id, ...d.data() }));
}
```

### 4. Query On-Chain Games Only (Optional Helper)
```javascript
// Helper to find games that ARE on-chain (for lock-in flow)
async function getOnChainGamesWithInterests() {
  const snap = await db.collection('agent_interests')
    .where('status', '==', 'active')
    .where('contestId', '!=', '')  // Has on-chain ID
    .get();
  
  return snap.docs.map(d => ({ id: d.id, ...d.data() }));
}
```

## Testing
1. ✅ Manually created test doc in Firebase
2. Run query script for that jsonoddsID - should return the doc

## Notes
- Schema matches your actual Firebase structure from screenshot
- `contestId` can be empty string for games not yet on-chain
- Filtering by side/team deferred to M17 (query interface handles that)
- Don't worry about indexes yet - add if queries slow
- Expiration can be manual for now (update status field)
- Don't need to hook up to agent yet - just storage layer works

## Done When
You can write an interest to Firebase and query it back by jsonoddsID. The test document you created counts - just verify you can query it.
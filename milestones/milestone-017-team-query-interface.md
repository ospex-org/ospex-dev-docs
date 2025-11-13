# Milestone 017: Team Name Query Interface
*Started: November 12, 2025*
*Completed: November 13, 2025*
*Status: ✅ Complete*

## Problem
Users need a simple way to express betting intent. Traditional sportsbook navigation (browse leagues → find game → select bet) doesn't match Ospex's model.

## What You're Building
Text input on agents page where user types team name and sees matching agent interests + existing unmatched positions.

## Success Criteria
- [X] Text input component renders on agent page
- [X] User can type and submit team name
- [X] Backend parses team name and finds matching game
- [X] Queries Firebase for matching agent interests
- [ ] Queries Firebase for existing unmatched positions (skipped)
- [X] Results display (even if just console.log for now)

## Implementation

### 1. Add Text Input to Agents page
```tsx
export default function AgentPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(null);
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setLoading(true);
    
    const data = await fetch('/api/query-agents', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ teamName: query })
    }).then(r => r.json());
    
    setResults(data);
    setLoading(false);
  }

  return (
    <div className="max-w-2xl mx-auto mt-20">
      <form onSubmit={handleSubmit} className="space-y-4">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Who are you on tonight?"
          className="w-full px-6 py-4 text-xl rounded-lg"
        />
        <button 
          type="submit"
          disabled={loading || !query}
          className="px-8 py-3 bg-blue-600 text-white rounded-lg"
        >
          {loading ? 'Searching...' : 'Find Bets'}
        </button>
      </form>
      
      {results && (
        <pre className="mt-8 p-4 bg-gray-100 rounded">
          {JSON.stringify(results, null, 2)}
        </pre>
      )}
    </div>
  );
}
```

### 2. Create API Route (skipped)
```typescript
// /api/query-agents/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  const { teamName } = await req.json();
  
  // Step 1: Find matching games
  const games = await findGamesForTeam(teamName);
  
  if (games.length === 0) {
    return NextResponse.json({ error: 'No games found' });
  }
  
  // For now, just take first match
  const game = games[0];
  
  // Step 2: Get agent interests for this game
  const interests = await getAgentInterests(game.jsonoddsID, teamName);
  
  // Step 3: Get existing unmatched positions
  const positions = await getUnmatchedPositions(game.contestId);
  
  return NextResponse.json({
    game,
    agentInterests: interests,
    unmatchedPositions: positions
  });
}
```

### 3. Team Name Matching
```typescript
async function findGamesForTeam(teamName: string) {
  const normalized = teamName.toLowerCase().trim();
  
  // Query upcoming games
  const snap = await db.collection('contests')
    .where('MatchTime', '>=', Timestamp.now())
    .where('MatchTime', '<=', Timestamp.fromMillis(Date.now() + 7 * 24 * 60 * 60 * 1000))
    .get();
  
  const matches = [];
  snap.forEach(doc => {
    const data = doc.data();
    const home = data.HomeTeam?.toLowerCase() || '';
    const away = data.AwayTeam?.toLowerCase() || '';
    
    // Simple exact match for now
    if (home.includes(normalized) || away.includes(normalized)) {
      matches.push({
        id: doc.id,
        ...data,
        matchedTeam: home.includes(normalized) ? data.HomeTeam : data.AwayTeam
      });
    }
  });
  
  return matches;
}
```

### 4. Query Agent Interests
```typescript
async function getAgentInterests(jsonoddsID: string, teamName: string) {
  const normalized = teamName.toLowerCase().trim();
  
  const snap = await db.collection('agent_interests')
    .where('jsonoddsID', '==', jsonoddsID)
    .where('status', '==', 'active')
    .get();
  
  const interests = [];
  snap.forEach(doc => {
    const data = doc.data();
    // Check if this interest is for the OPPOSITE side of what user wants
    if (!data.side.toLowerCase().includes(normalized)) {
      interests.push(data);
    }
  });
  
  return interests;
}
```

### 5. Query Unmatched Positions (skipped)
```typescript
async function getUnmatchedPositions(contestId: string) {
  // Use your existing position querying logic
  // Filter for positions where user could take opposite side
  const snap = await db.collection('positions')
    .where('contestId', '==', contestId)
    .where('unmatchedAmount', '>', 0)
    .get();
  
  return snap.docs.map(d => d.data());
}
```

## Testing
1. Type "Eagles" in text box
2. Submit - should find PHI game
3. Console/UI should show:
   - Game details
   - Agent interests for opposite side (if Dan wants Eagles, user gets offered that)
   - Any existing unmatched positions
4. Try different team names
5. Try team not in upcoming games - should handle gracefully

## Notes
- Team matching is simple for now (substring match)
- Don't worry about multiple games with same team
- Don't worry about perfect UI - JSON dump is fine for now
- Focus on data flow working end-to-end

## Done When
You can type "Packers" and see results showing the game, agent interests, and unmatched positions (even if displayed as raw JSON).

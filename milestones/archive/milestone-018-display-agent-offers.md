# Milestone 018: Polish Agent Query Interface
*Created: November 13, 2025*
*Completed: November 13, 2025*
*Status: ✅ Complete*

## Problem
TeamQuery component works but looks unpolished. Search box spans entire screen, odds display is cluttered with irrelevant markets, suggestions cause layout shift, and overall styling doesn't match the dapp's professional aesthetic.

## What You're Building
Professional, focused UI for agent interests that shows only relevant information in a clean, fixed layout.

## Success Criteria
- [X] Centered, inviting search box (lovable.dev inspired)
- [X] Real-time team suggestions in fixed-height container (no layout shift)
- [X] Clean agent interest cards showing only relevant bet details
- [X] Odds display matches existing Purchase Contracts modal style (decided to replicate Order Book style)
- [X] Shows: agent name, specific bet (e.g., "Phoenix Suns -4.0"), focused odds, amount, timestamp
- [X] "Create position" button either removed or shows informative state
- [X] No conviction score displayed (internal metric only)

## Implementation

### 1. Search Box Redesign (suggested code)
```tsx
// Center the search box, make it inviting
<div className="max-w-2xl mx-auto pt-12 pb-8">
  <Input
    placeholder="Who do you like today?" // Static for now, animated optional
    value={text}
    onChange={(e) => {
      setText(e.target.value);
      // Debounced suggestions trigger
    }}
    className="text-lg py-6" // Larger, more prominent
  />
  
  {/* Fixed height suggestions container - no layout shift */}
  <div className="h-16 mt-2"> 
    {suggestions.length > 0 && (
      <div className="flex flex-wrap gap-2">
        {suggestions.map(sug => (
          <Badge 
            key={sug.value} 
            variant="secondary" 
            className="cursor-pointer"
            onClick={() => handlePickSuggestion(sug.value)}
          >
            {sug.label}
          </Badge>
        ))}
      </div>
    )}
  </div>
</div>
```

Animated Placeholder (Optional - only if <30 min):
```tsx
const placeholders = [
  "Who do you like today?",
  "Which team are you backing?",
  "Got a hot take?"
];
// Rotate every 3 seconds
```

### 2. Agent Interest Card - Focused Display (suggested code)
Key principle: Only show odds/details for the SPECIFIC bet type the agent wants, not all markets for the game.
```tsx
function AgentInterestCard({ interest, game }) {
  const market = getMarket(interest.speculationScorer); // "moneyline", "spread", "total"
  const odds = getOddsDecimal(interest);
  const amount = interest.maxWagerUSDC;
  const timestamp = interest.createdAt; // Format nicely
  
  // Transform theNumber the same way as amoyPositionsv2.3
  const line = parseTheNumber(interest.theNumber);
  const displayLine = line != null ? normalizeLineToHalf(line) : null;
  
  // Build focused bet description
  const betDescription = formatFocusedBet(interest, game, market, displayLine);
  
  return (
    <Card className="p-6">
      {/* Agent Identity */}
      <div className="flex items-center gap-3 mb-4">
        <Avatar className="w-12 h-12">
          <AvatarFallback>{interest.agentName?.charAt(0) || 'A'}</AvatarFallback>
        </Avatar>
        <div>
          <div className="font-semibold text-lg">{interest.agentName}</div>
          <div className="text-sm text-muted-foreground">AI Agent</div>
        </div>
      </div>
      
      {/* Specific Bet - Clear and Focused */}
      <div className="mb-4">
        <div className="text-xl font-bold mb-1">{betDescription}</div>
        <div className="text-sm text-muted-foreground">
          {formatTimestamp(timestamp)}
        </div>
      </div>
      
      {/* Odds Display - Match Purchase Contracts Modal Style */}
      <div className="grid grid-cols-2 gap-4 mb-6">
        <div className="border rounded-lg p-3">
          <div className="text-xs text-muted-foreground mb-1">Odds</div>
          <div className="text-2xl font-bold">
            {odds ? `${odds.toFixed(2)}x` : '-'}
          </div>
        </div>
        <div className="border rounded-lg p-3">
          <div className="text-xs text-muted-foreground mb-1">Max Bet</div>
          <div className="text-2xl font-bold">
            {amount} <span className="text-base">USDC</span>
          </div>
        </div>
      </div>
      
      {/* Action - Defer implementation details for now */}
      <Button 
        className="w-full" 
        variant="outline"
        disabled
      >
        Coming Soon
      </Button>
      <p className="text-xs text-muted-foreground text-center mt-2">
        Full transaction flow in next milestone
      </p>
    </Card>
  );
}

function formatFocusedBet(interest, game, market, displayLine) {
  const userSide = interest.positionType === "1" ? "home" : "away";
  const team = userSide === "home" ? game.homeTeam : game.awayTeam;
  
  if (market === "moneyline") {
    return `${team} to Win`;
  }
  
  if (market === "spread") {
    const sign = displayLine >= 0 ? "+" : "";
    return `${team} ${sign}${displayLine.toFixed(1)}`;
  }
  
  if (market === "total") {
    const side = interest.positionType === "0" ? "Over" : "Under";
    return `${side} ${displayLine.toFixed(1)}`;
  }
  
  return `${team} - ${market}`;
}
```

### 3. Game Header - Clean Summary (suggested code)
```tsx
<div className="text-center pb-6 mb-6 border-b">
  <h2 className="text-2xl font-bold mb-2">
    {game.awayTeam} @ {game.homeTeam}
  </h2>
  <div className="text-sm text-muted-foreground">
    {formatGameTime(game.startTime)}
  </div>
</div>
```

Don't show all the market odds in the header. User is already seeing focused odds in each agent card.

### 4. Fixed Layout Structure (suggested code)
```tsx
<div className="space-y-6">
  {/* Search box - centered, max-w-2xl */}
  <div className="max-w-2xl mx-auto">
    {/* Input + fixed suggestions container */}
  </div>
  
  {/* Results - full width */}
  {game && (
    <div className="max-w-6xl mx-auto">
      <GameHeader game={game} />
      
      <div className="space-y-4">
        <h3 className="text-lg font-semibold">
          Available Agent Offers
        </h3>
        
        <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
          {interests.map(interest => (
            <AgentInterestCard 
              key={interest.id} 
              interest={interest} 
              game={game} 
            />
          ))}
        </div>
      </div>
    </div>
  )}
</div>
```

### 5. Real-time Suggestions (suggested code)
```tsx
// Debounce typing to avoid too many queries
const debouncedSearch = useMemo(
  () => debounce((value: string) => {
    if (value.length >= 2) {
      fetchSuggestions(value);
    }
  }, 300),
  []
);

useEffect(() => {
  debouncedSearch(text);
}, [text]);
```

## What's Explicitly Out of Scope
- Transaction flow ("Create position" button functionality) → M19
- Agent data fixes (positionTypeString, theNumber in memory) → Separate mini-milestone
- Re-evaluation logic before transaction → Future feature
- API route instead of client-side queries → Future optimization
- Firebase security rules → Infrastructure task

## Testing
1. Type "Warriors" - suggestions should appear in fixed container (no shift)
2. Select suggestion or hit enter
3. Agent interest cards should show:
  - Clean, focused bet description (e.g., "Phoenix Suns -4.0")
  - Only relevant odds (not all markets)
  - Amount in USDC
  - Timestamp
  - Agent name
4. Button should be disabled with "Coming Soon" message
5. No conviction score visible anywhere
6. Layout should never jump/shift

## Notes
- Reuse `normalizeLineToHalf` from existing code
- Match Purchase Contracts modal aesthetic for odds display
- Keep conviction score in data but don't render it
- Animated placeholder: only if trivial to implement

## Done When
Agent query interface looks professional and polished. Search is centered and inviting, suggestions don't cause layout shift, and agent interests display as clean, focused cards that show only the relevant bet details - matching your existing dapp's aesthetic.

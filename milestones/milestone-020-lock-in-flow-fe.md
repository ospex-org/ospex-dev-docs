# Milestone 020: Lock-In Flow (Frontend Only)

*Created: November 17, 2025*  
*Completed: November 19, 2025*  
*Status: ✅ Complete*

## Problem
Users can see agent interests but can't act on them. Need a clear, guided flow for users to create the contest and position that allows the agent to match.

## What You're Building
2-card interface with step-by-step user actions on the left, real-time agent status on the right. Horizontal button flow for creating contest → position → waiting for agent match.

## Success Criteria
- [X] 2-column card layout (desktop) / stacked (mobile)
- [X] Left card: User bet info + 2-button horizontal flow
- [X] Right card: Agent status, quote, and profile
- [X] Existing "Create Market" modal reused (pre-selected for correct side)
- [X] 60-second countdown after position created
- [X] Real-time status updates via Firebase snapshot (note: currently non-functional if user adjusts their position)
- [X] Link to user profile when matched (pending)
- [X] Responsive layout (cards stack on mobile)

## Implementation

### 1. Card Layout Structure

**Desktop (≥768px):**
```
┌──────────────────────────────┬──────────────────────────────┐
│   YOUR BET (Left Card)       │   DEGEN DAN (Right Card)     │
│                              │                              │
│   Coastal Carolina +10.5     │   Status: 💤 Waiting...      │
│                              │                              │
│   [1.Contest] [2.Position]   │   "Georgia Southern -160..." │
│   [3. Waiting ⏳]            │   - Degen Dan                │
│                              │                              │
│                              │   About Dan: ...             │
└──────────────────────────────┴──────────────────────────────┘
```

**Mobile (<768px):**
```
┌────────────────────────────────────┐
│   YOUR BET                         │
│   Coastal Carolina +10.5           │
│   [1.Contest] [2.Position] [3.⏳]  │
└────────────────────────────────────┘
┌────────────────────────────────────┐
│   DEGEN DAN                        │
│   Status: 💤 Waiting...            │
│   "Georgia Southern -160..."       │
│   About Dan: ...                   │
└────────────────────────────────────┘
```

### 2. Left Card: User Actions

Note: for all code blocks below, this exact code does not need to be replicated, the code should be used as guidance for what we are trying to do, not exactly how we are doing it.

**Components:**

```tsx
<div className="flex flex-col gap-4">
  {/* Bet Header */}
  <div>
    <h3 className="text-xl font-bold">YOUR BET</h3>
    <p className="text-2xl">{teamName} {displayLine}</p>
    <p className="text-sm text-muted-foreground">
      Will {teamName} {spreadExplanation}?
    </p>
  </div>

  {/* Horizontal Button Flow */}
  <div className="flex gap-2 flex-wrap">
    <Button 
      variant={contestExists ? "outline" : "default"}
      disabled={contestExists}
      onClick={handleCreateContest}
    >
      {contestExists ? "✓ Contest Created" : "1. Create Contest"}
    </Button>

    <Button
      variant={positionCreated ? "outline" : "default"}
      disabled={!contestExists || positionCreated}
      onClick={handleCreatePosition}
    >
      {positionCreated ? "✓ Position Created" : "2. Create Position"}
    </Button>

    <Button
      variant="ghost"
      disabled
      className={flowState === 'waiting_agent' ? 'animate-pulse' : ''}
    >
      {getStep3ButtonText(flowState, countdown)}
    </Button>
  </div>

  {/* Match Details (when matched) */}
  {flowState === 'matched' && (
    <Alert>
      <CheckCircle2 className="h-4 w-4" />
      <AlertTitle>Position Matched!</AlertTitle>
      <AlertDescription>
        Dan completed your bet.
        <Link to="/profile" className="underline ml-2">
          View Your Position →
        </Link>
      </AlertDescription>
    </Alert>
  )}
</div>
```

**Step 3 Button States:**

```typescript
function getStep3ButtonText(
  state: FlowState, 
  countdown: number
): string {
  switch(state) {
    case 'need_contest':
    case 'need_position':
      return '3. Waiting...';
    
    case 'waiting_agent':
      if (countdown > 0) {
        return `3. Dan Reviewing (${countdown}s)`;
      } else {
        return '3. Dan Reviewing...';
      }
    
    case 'matched':
      return '✓ Matched!';
    
    case 'declined':
      return 'Dan Declined';
    
    default:
      return '3. Waiting...';
  }
}
```

### 3. Right Card: Agent Status

**Components:**

```tsx
<div className="flex flex-col gap-4">
  {/* Agent Header */}
  <div className="flex items-center gap-3">
    <Avatar>
      <AvatarFallback>D</AvatarFallback>
    </Avatar>
    <div>
      <h3 className="font-bold">Degen Dan</h3>
      <p className="text-sm text-muted-foreground">
        {formatRelativeTime(interest.analyzedAt)}
      </p>
    </div>
  </div>

  {/* Dynamic Status */}
  <div className="rounded-lg border p-4 bg-muted/50">
    <div className="flex items-center gap-2 mb-2">
      <span className="text-2xl">{status.icon}</span>
      <span className="font-semibold">{status.message}</span>
    </div>
    {status.subtext && (
      <p className="text-sm text-muted-foreground">{status.subtext}</p>
    )}
    {flowState === 'waiting_agent' && countdown > 0 && (
      <Progress value={(60 - countdown) / 60 * 100} className="mt-2" />
    )}
  </div>

  {/* Dan's Quote */}
  <div className="italic text-sm border-l-2 pl-4">
    "{interest.reasoning}"
    <p className="text-xs text-muted-foreground mt-1">- Degen Dan</p>
  </div>

  {/* About Dan */}
  <div className="text-sm">
    <h4 className="font-semibold mb-1">About Dan:</h4>
    <p className="text-muted-foreground">
      Aggressive bettor who loves favorites and trendy picks. 
      Prefers teams favored at -150 or better.
    </p>
    <p className="text-xs text-muted-foreground mt-2">
      Record: 0-0 (just getting started!)
    </p>
  </div>
</div>
```

**Status Logic:**

```typescript
type FlowState = 
  | 'need_contest'
  | 'need_position' 
  | 'waiting_agent'
  | 'matched'
  | 'declined';

interface StatusDisplay {
  message: string;
  subtext: string | null;
}

function getAgentStatus(
  flowState: FlowState, 
  countdown: number
): StatusDisplay {
  switch(flowState) {
    case 'need_contest':
      return {
        message: 'Waiting for you to create the contest',
        subtext: 'Click button 1 to get started'
      };
    
    case 'need_position':
      return {
        message: 'Waiting for you to create your position',
        subtext: 'Contest is ready! Click button 2'
      };
    
    case 'waiting_agent':
      if (countdown > 0) {
        return {
          message: `Reviewing your bet...`,
          subtext: 'Dan checks odds and game status'
        };
      } else {
        return {
          message: 'Dan is really mulling this over...',
          subtext: 'Typical response: 1-5 minutes'
        };
      }
    
    case 'matched':
      return {
        message: 'Matched!',
        subtext: 'Dan completed your bet'
      };
    
    case 'declined':
      return {
        message: 'Dan passed on this bet',
        subtext: 'Odds moved or game conditions changed'
      };
  }
}
```

### 4. Modal Integration

**Reuse existing Create Market modal:**

```typescript
function handleCreatePosition() {
  // Open existing modal with pre-filled data
  openCreateMarketModal({
    contestId: onChainContestId,
    speculationScorer: interest.speculationScorer,
    theNumber: interest.theNumber,
    
    // Pre-select the OPPOSITE side of agent
    // If agent is positionType "0" (Upper), user takes "1" (Lower)
    preSelectedSide: interest.positionType === '0' ? 'lower' : 'upper',
    
    // Lock the side selection (user can't change it)
    lockSide: true,
    
    // Pre-fill odds if available
    suggestedOdds: calculateOppositeOdds(interest.targetOddsDecimal),
    
    // Callback after successful position creation
    onSuccess: (positionId: string) => {
      setPositionId(positionId);
      setFlowState('waiting_agent');
      setCountdown(60);
      subscribeToPosition(positionId);
    }
  });
}
```

**Modal modifications needed:**
- Accept `preSelectedSide` and `lockSide` props
- Disable side toggle when `lockSide: true`
- Show helper text: "You're taking the opposite side of Dan's bet"

### 5. Real-Time Updates

**Firebase Snapshot:**

```typescript
function subscribeToPosition(positionId: string) {
  const unsubscribe = onSnapshot(
    doc(db, 'amoyPositionsv2.3', positionId),
    (snapshot) => {
      const data = snapshot.data();
      
      // Check if position was matched
      if (data?.matchedAmount && data.matchedAmount > 0) {
        setFlowState('matched');
        setCountdown(0);
        
        // Optional: Show toast notification
        toast.success('Dan matched your bet!');
      }
      
      // Check if position was cancelled (edge case)
      if (data?.status === 'cancelled') {
        setFlowState('need_position');
        toast.info('Position was cancelled');
      }
    },
    (error) => {
      console.error('Snapshot error:', error);
      // Fallback: poll every 10 seconds
      startPolling(positionId);
    }
  );
  
  return unsubscribe;
}

// Cleanup
useEffect(() => {
  if (positionId) {
    const unsub = subscribeToPosition(positionId);
    return () => unsub();
  }
}, [positionId]);
```

**Countdown Timer:**

```typescript
const [countdown, setCountdown] = useState(60);

useEffect(() => {
  if (flowState === 'waiting_agent' && countdown > 0) {
    const timer = setTimeout(() => {
      setCountdown(c => c - 1);
    }, 1000);
    return () => clearTimeout(timer);
  }
}, [flowState, countdown]);
```

### 6. Responsive Design

**Tailwind Classes:**

```tsx
{/* Container */}
<div className="grid grid-cols-1 md:grid-cols-2 gap-4">
  
  {/* Left Card */}
  <Card className="p-6">
    {/* User actions */}
  </Card>
  
  {/* Right Card */}
  <Card className="p-6">
    {/* Agent status */}
  </Card>
  
</div>

{/* Button Flow - wraps on mobile */}
<div className="flex gap-2 flex-wrap">
  <Button className="flex-1 min-w-[140px]">1. Contest</Button>
  <Button className="flex-1 min-w-[140px]">2. Position</Button>
  <Button className="flex-1 min-w-[140px]">3. Waiting</Button>
</div>
```

**Breakpoints:**
- Mobile (<768px): Cards stack vertically, buttons wrap
- Tablet (768px-1024px): 2 columns, buttons may wrap
- Desktop (>1024px): 2 columns, buttons horizontal

### 7. Contest Creation Flow

**Check if contest exists:**

```typescript
async function checkContestExists(jsonoddsID: string): Promise<boolean> {
  const snap = await db.collection('amoyContestsv2.3')
    .where('jsonoddsID', '==', jsonoddsID)
    .where('status', '==', 'Verified')
    .limit(1)
    .get();
  
  return !snap.empty;
}

// On component mount
useEffect(() => {
  async function init() {
    const exists = await checkContestExists(interest.jsonoddsID);
    setContestExists(exists);
    
    if (exists) {
      setFlowState('need_position');
      
      // Get on-chain contestId for modal
      const contestId = await getOnChainContestId(interest.jsonoddsID);
      setOnChainContestId(contestId);
    } else {
      setFlowState('need_contest');
    }
  }
  init();
}, [interest.jsonoddsID]);
```

**Handle contest creation:**

```typescript
async function handleCreateContest() {
  try {
    setCreatingContest(true);
    
    // Call existing contest creation function
    const txHash = await createContest(interest.jsonoddsID);
    
    // Wait for confirmation
    await waitForContestVerification(interest.jsonoddsID);
    
    // Update state
    setContestExists(true);
    setFlowState('need_position');
    
    // Fetch on-chain ID
    const contestId = await getOnChainContestId(interest.jsonoddsID);
    setOnChainContestId(contestId);
    
    toast.success('Contest created!');
  } catch (error) {
    console.error('Contest creation failed:', error);
    toast.error('Failed to create contest. Please try again.');
  } finally {
    setCreatingContest(false);
  }
}
```

### 8. File Structure

**New/Modified Files:**

```
ospex-lovable/src/
├── components/agents/
│   ├── TeamQuery.tsx (modified - use 2-card layout)
│   ├── AgentInterestCard.tsx (NEW - left card component)
│   ├── AgentStatusCard.tsx (NEW - right card component)
│   └── AgentFlowState.tsx (NEW - shared state/logic)
├── lib/data/hooks/ (add to file hooks.ts)
│   ├── useAgentFlow
│   └── usePositionSubscription
└── lib/data/ (add to file utils.ts)
    └── flow-helpers (added as needed)
```

### 9. Testing Checklist

**Happy Path:**
1. [X] User searches "Lakers" (note: changed so all agent interests display by default)
2. [X] Sees Dan's interest card (2-column layout)
3. [X] Contest doesn't exist → Button 1 enabled, Buttons 2-3 disabled (note: only 2 buttons now, plus status)
4. [X] Click "Create Contest" → TX succeeds
5. [X] Button 1 shows "Contest Created", Button 2 enabled
6. [X] Click "Create Position" → Modal opens
7. [X] Modal has correct side pre-selected and locked
8. [X] Submit modal → TX succeeds
9. [X] Button 3 shows countdown "Dan Reviewing (60s...59s...)"
10. [X] Progress bar animates
11. [X] After 60s, message changes to "Dan is mulling..."
12. [X] Manually run agent code
13. [X] Frontend updates to "Matched!"
14. [X] Alert shows with link to profile (there isn't literally an alert thankfully)
15. [X] Click link → redirects to /profile

**Edge Cases:**
- [X] Contest already exists → Skip to step 2
- [X] User cancels modal → State remains at 'need_position'
- [X] Transaction fails → Show error, don't advance state (unsure if error shows properly but state does not advance)
- [X] User navigates away → Subscription cleanup works
- [X] Mobile view → Cards stack, buttons wrap
- [ ] Multiple interests for same team → Each card independent (untested)

**Responsive:**
- [X] Desktop (>1024px): 2 columns, all buttons visible
- [X] Tablet (768-1024px): 2 columns, buttons may wrap
- [X] Mobile (<768px): Stacked cards, wrapped buttons

### 10. Out of Scope (Deferred to M21)

- ❌ Actual agent automation (manual trigger only)
- ❌ Agent re-evaluation logic
- ❌ Multiple simultaneous users
- ❌ Complex error recovery
- ❌ Dan's full profile page
- ❌ Historical bet records

## Done When

User can:
1. ✅ Search for a team and see Dan's interest in 2-card layout
2. ✅ Create contest via button 1
3. ✅ Create position via button 2 (modal pre-fills correctly)
4. ✅ See real-time countdown and status updates
5. ✅ Get notified when position matched (via snapshot)
6. ✅ Click link to view their position
7. ✅ Experience works on desktop, tablet, and mobile

**Test:** Complete full flow from search → match on your own, manually triggering agent after position creation. Frontend should feel smooth and informative throughout.

## Notes

- Agent matching happens manually for M20 (run `yarn agent:match` or similar)
- M21 will add automatic agent responses
- Focus on UI polish and clear user guidance
- Real-time updates via Firebase are critical for good UX
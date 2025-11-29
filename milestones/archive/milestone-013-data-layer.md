# Milestone 013: Data Layer Architecture
*Created: October 4, 2025*  
*Target Completion: October 8, 2025*
*Status: ✅ Mostly complete*

## Problem Statement
The production frontend lacks a coherent data layer, resulting in inconsistent transformations across pages, scattered conversion logic, duplicate code, and data shape mismatches. Timestamps are converted differently in different components, odds calculations vary by page, league IDs map inconsistently, and there's no single source of truth for how backend data should appear in the UI. This creates a maintenance nightmare where bugs fixed on one page break another, and adding new pages requires re-implementing the same transformations.

## Context
v2.2 test frontend revealed this pattern: build UI first, discover data inconsistencies, attempt to refactor while building features, end up with scattered utilities and incomplete type coverage. The smart contract development followed proper architecture (modularity, gas optimization, clear interfaces). The frontend needs the same discipline: design the data contracts first, then build UI that consumes them.

## Scope & Boundaries

### In Scope
- Complete TypeScript type definitions for all platform entities
- Centralized data transformers (Firebase → UI, Contract events → UI, External APIs → UI)
- Custom React hooks for data fetching and real-time updates
- Utility functions for formatting, conversion, and calculation
- Comprehensive testing of transformers with real backend data
- Data layer documentation and usage examples
- Migration of useful code from v2.2 types/utils

### Out of Scope
- UI components or page implementations
- Contract integration code (that's M14)
- Firebase configuration or listeners (create hooks interface only)
- Visual design or styling
- Performance optimization beyond basic efficiency
- Error recovery mechanisms (basic error handling only)
- State management libraries (use React hooks)

## Design Strategy: Architecture Before Implementation

**Core principle**: Define data contracts completely before touching UI
- No UI work until data layer is complete and tested
- Every backend data source gets a transformer
- Every entity gets a complete type definition
- All conversions centralized in single functions
- Test transformers with real data, not mock data

**Type design philosophy**: Complete, consistent, UI-ready
- Types represent what UI needs, not what backend provides
- All monetary values in human-readable USDC (not wei)
- All timestamps as Date objects
- All odds in multiple formats (american, decimal, probability)
- Enums for all categorical data (leagues, statuses, sides)

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│           Data Layer (This Milestone)               │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │    Types     │  │ Transformers │  │   Hooks   │  │
│  │              │  │              │  │           │  │
│  │ - Position   │  │ FB → UI      │  │ usePos... │  │
│  │ - Contest    │  │ Contract → UI│  │ useCon... │  │
│  │ - Spec...    │  │ API → UI     │  │ useLea... │  │
│  │ - Leaderb... │  │              │  │           │  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │            Utils (Formatting)                │   │
│  │ - Time, Odds, Currency, League mappings      │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                         ▲
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        │                │                │
   ┌────▼────┐     ┌─────▼─────┐    ┌────▼────┐
   │Firebase │     │ Contracts │    │ APIs    │
   │         │     │  Events   │    │ (odds)  │
   └─────────┘     └───────────┘    └─────────┘
```

## Implementation Plan

### Phase 1: Audit & Planning (3-4 hours)

**1.1 Review v2.2 Code**
- [X] Examine existing types in `/types` folder
- [X] Review all utils files (timeFormatting, roiFormatting, etc.)
- [X] Identify useful patterns to keep
- [X] Document inconsistencies and problems
- [X] List all data transformations currently happening

**1.2 Map Data Requirements**
- [X] List all entities (positions, contests, speculations, leaderboards, users)
- [X] For each entity, document:
  - What UI pages need it
  - What formats it comes from backend (Firebase schema, contract events, API responses)
  - What format UI needs (display requirements)
- [X] Identify all conversion types needed (time, odds, currency, leagues)

**1.3 Design Type System**
- [X] Sketch out primary types with all required fields
- [X] Define enums for categorical data
- [X] Plan for variant types (matched vs unmatched positions, etc.)
- [X] Document type relationships (Position references Contest, etc.)

### Phase 2: Core Types & Enums (4-5 hours)

**2.1 Create `/lib/data/types.ts`**
- [X] **Position type** (all variants: matched, unmatched, settled)
  - Basic info: id, contestId, speculationId
  - Sides: userSide, opponentSide (from perspective of connected wallet)
  - Odds: american, decimal, probability (all formats)
  - Money: stake, payout, potential return (always human-readable USDC)
  - Time: created, updated (always Date objects)
  - Status: active, pending, settled
  - Metadata: league, teams, event details

- [X] **Contest type**
  - Teams, sport, league
  - Date/time (Date object)
  - Status
  - Associated speculation IDs

- [X] **Speculation type**
  - Contest reference
  - Type (moneyline, spread, total)
  - Line/target values
  - Market status
  - Leaderboard eligibility

- [X] **Leaderboard type**
  - Basic info, prize pool, duration
  - Participants with ROI
  - Eligible speculations
  - Status and timing

- [X] **User type**
  - Wallet address
  - Positions
  - Leaderboard participations
  - Stats

**2.2 Create Enums**
- [X] League (NFL, NBA, MLB, CFB, etc.)
- [X] Sport (football, basketball, baseball)
- [X] PositionStatus (active, pending, settled, claimed)
- [X] PositionSide (yes, no)
- [X] SpeculationType (moneyline, spread, total)
- [X] ContestStatus (upcoming, live, complete)

### Phase 3: Transformers (5-6 hours)

**3.1 Create `/lib/data/transformers.ts`**
- [X] **Firebase Position → UI Position**
  - Handle wei to USDC conversion
  - Convert timestamps to Date
  - Calculate all odds formats
  - Map league IDs to League enum
  - Determine user vs opponent side

- [X] **Contract Event → UI Position**
  - Parse event data
  - Transform to same Position type
  - Handle different event structures

- [X] **Firebase Contest → UI Contest**
  - Convert timestamps
  - Map sport/league IDs
  - Handle team names consistently

- [X] **Firebase Speculation → UI Speculation**
  - Map speculation types
  - Handle line values
  - Convert status fields

- [X] **Leaderboard Data → UI Leaderboard**
  - Parse participant data
  - Calculate ROI consistently
  - Handle prize pool formatting

**3.2 Create `/lib/data/utils.ts`**
- [X] **Time utilities**
  - `formatTimestamp(unix: number): Date`
  - `formatRelativeTime(date: Date): string` (e.g., "2 hours ago")
  - `formatGameTime(date: Date): string` (e.g., "Oct 4, 10:30 PM EDT")
  
- [X] **Odds utilities**
  - `decimalToAmerican(decimal: number): string`
  - `americanToDecimal(american: string): number`
  - `oddsToProb(decimal: number): number`
  - `formatOdds(decimal: number): { american, decimal, probability }`

- [X] **Currency utilities**
  - `weiToUsdc(wei: bigint): number`
  - `usdcToWei(usdc: number): bigint`
  - `formatUsdc(amount: number): string` (e.g., "42.50 USDC")
  - `calculatePayout(stake: number, odds: number): number`

- [X] **League utilities**
  - `mapLeagueId(id: number, source: 'firebase' | 'api1' | 'api2'): League`
  - `mapSportToLeague(sport: string): League[]`

### Phase 4: Hooks (4-5 hours)

**4.1 Create `/lib/data/hooks/`**
- [X] **usePositions.ts**
  - Fetch user positions from Firebase
  - Transform using transformFirebasePosition
  - Handle real-time updates
  - Filter by status (active, settled, etc.)
  - Return consistent Position[]

- [X] **useContests.ts**
  - Fetch contests
  - Transform to UI format
  - Filter by status/date
  - Handle real-time updates

- [X] **useSpeculations.ts**
  - Fetch speculations for contest
  - Transform to UI format
  - Group by type (moneyline, spread, total)

- [X] **useLeaderboard.ts**
  - Fetch leaderboard data
  - Transform participants
  - Calculate rankings
  - Handle real-time position updates

- [X] **useUnmatchedPairs.ts**
  - Fetch available positions to match
  - Transform to UI format
  - Group by speculation/side

**4.2 Hook Patterns**
- All hooks return: `{ data, loading, error }`
- All hooks handle real-time Firebase listeners
- All hooks apply transformers before returning
- All hooks clean up listeners on unmount

### Phase 5: Testing & Validation (2-3 hours) (skipped)

**5.1 Transformer Tests**
- [ ] Create `/lib/data/__tests__/transformers.test.ts`
- [ ] Get real Firebase data samples
- [ ] Get real contract event samples
- [ ] Test each transformer with real data
- [ ] Verify output matches type definitions
- [ ] Test edge cases (null values, missing fields, etc.)

**5.2 Utils Tests**
- [ ] Test timestamp conversions with various formats
- [ ] Test odds conversions (decimal ↔ american)
- [ ] Test USDC/wei conversions with edge cases
- [ ] Test league mapping from different sources
- [ ] Verify formatting functions produce consistent output

**5.3 Integration Testing**
- [ ] Test hooks fetch real data
- [ ] Verify transformers applied correctly
- [ ] Check real-time updates work
- [ ] Confirm no data shape mismatches

### Phase 6: Documentation (1-2 hours) (skipped)

**6.1 Create `/lib/data/README.md`**
- [ ] Document all types with examples
- [ ] Show transformer usage patterns
- [ ] Provide hook usage examples
- [ ] Explain utility functions
- [ ] Document data flow (backend → transformer → UI)

**6.2 Create Usage Examples**
- [ ] Example: Fetching and displaying positions
- [ ] Example: Converting odds formats
- [ ] Example: Formatting timestamps
- [ ] Example: Handling real-time updates

## Success Criteria

### Type System
- [X] All entities have complete type definitions
- [ ] No `any` types except where genuinely necessary
- [X] Enums for all categorical data
- [X] Types match actual UI needs (not backend structure)
- [X] TypeScript compilation errors guide correct usage

### Transformers
- [X] Every backend data source has a transformer
- [X] All transformers output same type for same entity
- [X] Transformers tested with real data
- [X] Edge cases handled (null, undefined, missing fields)
- [X] No inline transformations needed in components

### Utilities
- [X] One function for each conversion type
- [X] Consistent output formats
- [X] Tested with edge cases
- [X] No duplicate conversion logic anywhere

### Hooks
- [X] Data fetching abstracted from UI
- [X] Real-time updates handled in hooks
- [X] Transformers applied automatically
- [X] Consistent return shape across all hooks
- [X] Loading and error states managed

### Testing
- [X] All transformers tested with real backend data
- [X] Utilities tested with edge cases
- [X] No type errors when using data layer
- [X] Sample data flows through entire system correctly

## Technical Requirements

### File Structure
```
src/
├── lib/
│   └── data/
│       ├── types.ts          (All type definitions)
│       ├── transformers.ts   (All data transformers)
│       ├── utils.ts          (Formatting utilities)
│       ├── hooks/
│       │   ├── usePositions.ts
│       │   ├── useContests.ts
│       │   ├── useSpeculations.ts
│       │   ├── useLeaderboards.ts
│       │   └── useUnmatchedPairs.ts
│       ├── __tests__/
│       │   ├── transformers.test.ts
│       │   └── utils.test.ts
│       └── README.md
```

### Type Definition Example
```typescript
// Core Position type
export interface Position {
  // Identifiers
  id: string;
  contestId: string;
  speculationId: string;
  
  // Sides (always from connected user's perspective)
  userSide: PositionSide;
  opponentSide: PositionSide;
  userAddress: string;
  opponentAddress: string;
  
  // Odds (all formats provided)
  odds: {
    american: string;      // "+150"
    decimal: string;       // "2.50"
    probability: number;   // 0.40
  };
  
  // Money (always human-readable USDC)
  stake: {
    usdc: number;          // 50.0
    formatted: string;     // "50.00 USDC"
  };
  payout: {
    usdc: number;
    formatted: string;
  };
  potentialReturn: {
    usdc: number;
    formatted: string;
  };
  
  // Timing (always Date objects)
  createdAt: Date;
  updatedAt: Date;
  settledAt?: Date;
  
  // Status
  status: PositionStatus;
  
  // Event context
  contest: {
    teamA: string;
    teamB: string;
    league: League;
    sport: Sport;
    gameTime: Date;
  };
  
  // Speculation context
  speculation: {
    type: SpeculationType;
    line?: number;        // For spreads/totals
    target?: number;      // For totals
  };
}
```

### Transformer Example
```typescript
export function transformFirebasePosition(
  fbData: any,
  connectedWallet: string
): Position {
  // Determine user vs opponent perspective
  const isUserSideA = fbData.userA.toLowerCase() === connectedWallet.toLowerCase();
  const userSide = isUserSideA ? fbData.sideA : fbData.sideB;
  
  return {
    id: fbData.id,
    contestId: fbData.contestId,
    speculationId: fbData.speculationId,
    
    userSide: userSide,
    opponentSide: userSide === 'yes' ? 'no' : 'yes',
    userAddress: connectedWallet,
    opponentAddress: isUserSideA ? fbData.userB : fbData.userA,
    
    odds: formatOdds(fbData.odds),
    
    stake: {
      usdc: weiToUsdc(fbData.amount),
      formatted: formatUsdc(weiToUsdc(fbData.amount))
    },
    
    payout: {
      usdc: calculatePayout(weiToUsdc(fbData.amount), fbData.odds),
      formatted: formatUsdc(calculatePayout(weiToUsdc(fbData.amount), fbData.odds))
    },
    
    createdAt: formatTimestamp(fbData.timestamp),
    updatedAt: formatTimestamp(fbData.updatedAt),
    
    status: mapPositionStatus(fbData.status),
    
    contest: {
      teamA: fbData.contest.teamA,
      teamB: fbData.contest.teamB,
      league: mapLeagueId(fbData.contest.leagueId, 'firebase'),
      sport: fbData.contest.sport,
      gameTime: formatTimestamp(fbData.contest.gameTime)
    },
    
    speculation: {
      type: fbData.speculation.type,
      line: fbData.speculation.line,
      target: fbData.speculation.target
    }
  };
}
```

### Hook Example
```typescript
export function usePositions(walletAddress?: string) {
  const [positions, setPositions] = useState<Position[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    if (!walletAddress) {
      setPositions([]);
      setLoading(false);
      return;
    }
    
    // Subscribe to Firebase
    const unsubscribe = subscribeToUserPositions(
      walletAddress,
      (fbPositions) => {
        // Transform all positions
        const transformed = fbPositions.map(fbPos => 
          transformFirebasePosition(fbPos, walletAddress)
        );
        setPositions(transformed);
        setLoading(false);
      },
      (err) => {
        setError(err);
        setLoading(false);
      }
    );
    
    return () => unsubscribe();
  }, [walletAddress]);
  
  return { positions, loading, error };
}
```

## Migration from v2.2

### What to Keep
- Type definitions that are complete and correct
- Working utility functions (after testing)
- Successful patterns from hooks

### What to Rebuild
- Scattered conversion logic
- Inconsistent transformations
- Incomplete type coverage
- Duplicate utilities

### Migration Process
1. Copy v2.2 `/types` and `/utils` to new project
2. Audit against new architecture requirements
3. Consolidate duplicate functions
4. Test all kept code with real data
5. Delete anything that doesn't fit new patterns

## Risk Assessment
- **Medium complexity**: Requires understanding of all data sources
- **Low risk**: No user-facing changes, isolated work
- **Critical value**: Foundation for all UI work, prevents M14 chaos
- **Time discipline**: Must resist urge to start UI before completion

## Development Workflow

### Daily Process
1. Work in isolated `/lib/data` directory
2. No UI components allowed in this milestone
3. Test each transformer as you build it
4. Document as you go
5. Use real backend data for testing

### Testing Approach
- Get real Firebase data snapshots
- Get real contract event logs
- Test transformers produce correct types
- Verify utilities handle edge cases
- Check hooks return consistent shapes

### Completion Checklist
- [X] All types compile without errors
- [X] All transformers tested with real data
- [X] All utilities tested with edge cases
- [X] All hooks tested with Firebase
- [ ] Documentation complete with examples
- [ ] No TODO comments remaining
- [X] Ready to be imported in M14 UI work

## Success Metrics
- Zero inline transformations needed in UI components
- One source of truth for each conversion type
- TypeScript catches data shape mismatches at compile time
- Adding new pages doesn't require new transformation logic
- Bugs in data display fixed in one place, reflected everywhere

## Completion Definition
Milestone complete when:
1. All entity types fully defined and documented
2. All backend data sources have working transformers
3. All hooks return consistent, typed data
4. All utilities tested with real data and edge cases
5. Sample data flows through transformers → hooks → types without errors
6. Documentation explains usage for M14 UI development
7. No UI work has been started (data layer only)

## Looking Ahead to Milestone 14
With data layer complete, M14 (UI wiring) becomes:
- Import hooks and use them
- Display typed data without transformation
- Trust that Position type has everything needed
- No conversion logic in components
- Faster development, fewer bugs

## Notes on completion level
~85% complete - core data layer works, event-driven writes work, real-time listeners work.

*This milestone is the foundation that makes M14 possible. Don't skip it.*
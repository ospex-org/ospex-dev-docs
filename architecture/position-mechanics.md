# Position Mechanics - Maker vs Taker System

## Core Concept
Ospex is an orderbook protocol with no fees. Makers create positions with desired odds, takers fill them with calculated inverse odds.

## Maker (Position Creator)

**Function**: `createUnmatchedPair()` in PositionModule.sol

**What Maker Specifies**:
- `odds`: The odds they want (e.g., 2.10 for 2.10x payout)
- `positionType`: Which side they want (0=Upper, 1=Lower)
- `amount`: How much USDC they're putting up

**What Maker Gets**: Exactly the odds they requested when creating the position.

## Position Types
- **Upper (0)**: Away team (moneyline) or Over (totals)
- **Lower (1)**: Home team (moneyline) or Under (totals)

## Smart Contract Calculation

When `createUnmatchedPair()` executes, the contract:
1. Takes the maker's requested odds
2. Calculates the inverse odds for the opposite side
3. Emits both `upperOdds` and `lowerOdds` in the POSITION_CREATED event

**Example**:
- Maker wants 2.10x odds on Upper (away team)
- Contract calculates ~1.91x odds for Lower (home team)
- Emits: `upperOdds: 21000000, lowerOdds: 19100000` (7 decimal precision)

## Taker (Position Matcher)

**Function**: `completeUnmatchedPair()` in PositionModule.sol

**What Taker Gets**:
- Automatically assigned the opposite position type
- Automatically gets the calculated inverse odds
- Can partially match (don't have to take full position)

## The Math Relationship

Simple equation: If maker wants X odds and puts up Y amount, the math determines:
1. How much taker needs to contribute for full match
2. What taker's odds will be (inverse relationship)

**Key Point**: The relationship is purely mathematical - no subjective decisions about "who gets what odds."

## Data Flow
1. Maker calls `createUnmatchedPair(speculationId, odds, positionType, amount, ...)`
2. Contract calculates `upperOdds` and `lowerOdds`
3. Contract emits both odds in POSITION_CREATED event
4. Frontend can display both sides accurately using emitted odds
5. Taker calls `completeUnmatchedPair()` and gets opposite side with inverse odds

## The OddsPairId System (Architecture)

**Problem**: Can't allow infinite odds precision - would be too complex and confusing.

**Solution**: Pre-determined system with exactly 20,000 acceptable odds pairs.

### Odds Range Structure
- **Decimal Range**: 1.01 to 101.00 
- **Increment**: 0.01 (so 1.01, 1.02, 1.03... 100.98, 100.99, 101.00)
- **Total Unique Odds**: 10,000 possible values

### OddsPairId Ranges
- **0 to 9,999**: Maker chose Upper (positionType 0) - Away/Over
- **10,000 to 19,999**: Maker chose Lower (positionType 1) - Home/Under

### Examples
- **oddsPairId 0**: Maker requested 1.01x odds, chose Upper (away/over)
- **oddsPairId 10,000**: Maker requested 1.01x odds, chose Lower (home/under)
- **oddsPairId 109**: Maker requested 2.10x odds, chose Upper (away/over)
- **oddsPairId 10,109**: Maker requested 2.10x odds, chose Lower (home/under)

### Why This System Exists
1. **Deterministic**: Every possible combination has a specific ID
2. **Manageable**: Limited but not restrictive (20,000 pairs covers virtually all use cases)
3. **Encoded Metadata**: The ID itself tells you the original odds AND which side maker chose
4. **Contract-First**: All calculations happen on-chain, frontend just displays results

### Historical Context (Frontend Complexity)
- **Before**: Frontend had to reverse-engineer oddsPairId to determine odds
- **Problem**: Rounding in contract made reverse-engineering nearly impossible
- **Solution**: Contract now explicitly emits `upperOdds` and `lowerOdds`
- **Result**: Frontend complexity eliminated - just use emitted values

## Display Logic (Frontend)
- **For any position**: Use the emitted `upperOdds` and `lowerOdds` from POSITION_CREATED
- **Maker display**: Show the odds for their chosen positionType
- **Taker display**: Show the odds for the opposite positionType
- **No hardcoding needed**: All odds are explicitly provided by the smart contract
- **No reverse-engineering**: Contract does all calculations, frontend displays results

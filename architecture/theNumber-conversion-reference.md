# theNumber Storage and Conversion Reference

## Overview

Solidity cannot handle decimals, so all spread/total lines ending in `.5` must be converted to whole numbers for smart contract storage. The spread scorer uses `>=` logic to eliminate pushes - one side always wins.

## Core Principle

**theNumber is ALWAYS stored with respect to the away team's perspective.**

The same `theNumber` can represent both sides of a bet - `positionType` determines which side wins.

---

## Spread Scoring Logic

```solidity
function scoreSpread(
    uint32 _awayScore,
    uint32 _homeScore,
    int32 _theNumber
) private pure returns (WinSide) {
    if (int32(_awayScore) + _theNumber >= int32(_homeScore)) {
        return WinSide.Away;  // Upper position wins
    } else {
        return WinSide.Home;  // Lower position wins
    }
}
```

**Key:** Away score + theNumber >= Home score → Away wins

---

## Spread Examples

### Example 1: Toronto Raptors @ Cleveland Cavaliers

**All possible agent positions:**

#### 1. Agent wants: Toronto +3.5 (Away team getting points)
- **positionType:** `"0"`
- **positionTypeString:** `"Upper"`
- **theNumber:** `"3"`

**Why "3"?**
- Final: Tor 97, Cle 100 → 97 + 3 >= 100 → **100 >= 100 ✓** (Agent wins)
- Final: Tor 96, Cle 100 → 96 + 3 >= 100 → **99 >= 100 ✗** (Agent loses)

#### 2. Agent wants: Toronto -3.5 (Away team favored)
- **positionType:** `"0"`
- **positionTypeString:** `"Upper"`
- **theNumber:** `"-4"`

**Why "-4"?**
- Final: Tor 100, Cle 96 → 100 + (-4) >= 96 → **96 >= 96 ✓** (Agent wins)
- Final: Tor 100, Cle 97 → 100 + (-4) >= 97 → **96 >= 97 ✗** (Agent loses)

#### 3. Agent wants: Cleveland +3.5 (Home team getting points)
- **positionType:** `"1"`
- **positionTypeString:** `"Lower"`
- **theNumber:** `"-4"`

**Why "-4"?** (Same as scenario 2, opposite position type)
- Final: Tor 100, Cle 96 → 100 + (-4) >= 96 → **TRUE** (Away wins, Agent loses ✗)
- Final: Tor 100, Cle 97 → 100 + (-4) >= 97 → **FALSE** (Home wins, Agent wins ✓)

#### 4. Agent wants: Cleveland -3.5 (Home team favored)
- **positionType:** `"1"`
- **positionTypeString:** `"Lower"`
- **theNumber:** `"3"`

**Why "3"?** (Same as scenario 1, opposite position type)
- Final: Tor 97, Cle 100 → 97 + 3 >= 100 → **TRUE** (Away wins, Agent loses ✗)
- Final: Tor 96, Cle 100 → 96 + 3 >= 100 → **FALSE** (Home wins, Agent wins ✓)

---

### Example 2: Memphis Grizzlies @ Boston Celtics

**Market Display:** Grizzlies +6.5 / Celtics -6.5

#### Both sides use theNumber = `"6"`

**Position 1: Grizzlies +6.5 (Away)**
- **positionType:** `"0"` (Upper)
- **theNumber:** `"6"`
- Final: Grizz 94, Celtics 100 → 94 + 6 >= 100 → **100 >= 100 ✓** (Covers)
- Final: Grizz 93, Celtics 100 → 93 + 6 >= 100 → **99 >= 100 ✗** (Doesn't cover)

**Position 2: Celtics -6.5 (Home)**
- **positionType:** `"1"` (Lower)
- **theNumber:** `"6"` (Same number!)
- Final: Grizz 93, Celtics 100 → 93 + 6 >= 100 → **FALSE** → Home wins ✓
- Final: Grizz 94, Celtics 100 → 94 + 6 >= 100 → **TRUE** → Away wins ✗

**This is why the UI shows both sides on the same card - they share theNumber!**

---

## Spread Conversion Formulas

### Agent Line → theNumber (Storage)

```typescript
function convertLineToTheNumber(agentLine: number, team: 'away' | 'home'): string {
  // agentLine must end in .5 (e.g., -3.5, +4.5)
  
  if (team === 'away') {
    // Away team perspective is direct
    return String(Math.floor(agentLine));
  } else {
    // Home team: negate and floor
    return String(Math.floor(-agentLine));
  }
}

// Examples:
// Away -3.5 → Math.floor(-3.5) → "-4"
// Away +3.5 → Math.floor(3.5) → "3"
// Home -3.5 → Math.floor(3.5) → "3"
// Home +3.5 → Math.floor(-3.5) → "-4"
```

### theNumber → Display Line (Frontend)

```typescript
function displaySpreadLine(theNumber: string, positionType: '0' | '1'): string {
  const num = parseInt(theNumber, 10);
  
  if (positionType === '0') {
    // Upper (Away): add 0.5 for underdog, subtract 0.5 for favorite
    const displayLine = num >= 0 ? num + 0.5 : num + 0.5;
    return displayLine >= 0 ? `+${displayLine}` : `${displayLine}`;
  } else {
    // Lower (Home): opposite of away
    const displayLine = num >= 0 ? -(num - 0.5) : -(num - 0.5);
    return displayLine >= 0 ? `+${displayLine}` : `${displayLine}`;
  }
}

// Examples:
// theNumber "3", positionType "0" → "+3.5"
// theNumber "-4", positionType "0" → "-3.5"
// theNumber "3", positionType "1" → "-3.5"
// theNumber "-4", positionType "1" → "+3.5"
```

---

## Total Scoring Logic

```solidity
function scoreTotal(
    uint32 _awayScore,
    uint32 _homeScore,
    int32 _theNumber
) private pure returns (WinSide) {
    if (int32(_awayScore + _homeScore) >= _theNumber) {
        return WinSide.Over;  // Upper position wins
    } else {
        return WinSide.Under; // Lower position wins
    }
}
```

**Key:** Combined score >= theNumber → Over wins

---

## Total Examples

### Example: Total 195.5

#### Agent wants: Over 195.5
- **positionType:** `"0"`
- **positionTypeString:** `"Upper"`
- **theNumber:** `"196"`

**Why "196"?**
- Final: 98 + 98 = 196 → 196 >= 196 → **TRUE** (Over wins ✓)
- Final: 97 + 98 = 195 → 195 >= 196 → **FALSE** (Over loses ✗)

#### Agent wants: Under 195.5
- **positionType:** `"1"`
- **positionTypeString:** `"Lower"`
- **theNumber:** `"196"` (Same number!)

**Why "196"?**
- Final: 97 + 98 = 195 → 195 >= 196 → **FALSE** (Under wins ✓)
- Final: 98 + 98 = 196 → 196 >= 196 → **TRUE** (Under loses ✗)

---

## Total Conversion Formulas

### Agent Line → theNumber (Storage)

```typescript
function convertTotalToTheNumber(agentLine: number): string {
  // agentLine must end in .5 (e.g., 195.5, 220.5)
  // Always ceil to eliminate pushes
  return String(Math.ceil(agentLine));
}

// Examples:
// 195.5 → Math.ceil(195.5) → "196"
// 220.5 → Math.ceil(220.5) → "221"
```

### theNumber → Display Line (Frontend)

```typescript
function displayTotalLine(theNumber: string): string {
  const num = parseInt(theNumber, 10);
  return (num - 0.5).toFixed(1);
}

// Examples:
// "196" → "195.5"
// "221" → "220.5"
```

---

## Agent Memory Requirements

**Agent must ONLY store lines ending in .5**

```json
// ✓ CORRECT
{
  "line": -3.5,
  "market": "spread",
  "pick": "away"
}

// ✗ WRONG - Agent should never want -4.0
{
  "line": -4.0,
  "market": "spread"
}
```

**Why?** Smart contracts eliminate pushes using `>=` logic. Lines ending in `.0` would be impossible to create on-chain with this system.

---

## Position Type Mapping

| positionType | positionTypeString | Spread Meaning | Total Meaning |
|--------------|-------------------|----------------|---------------|
| `"0"` | `"Upper"` | Away | Over |
| `"1"` | `"Lower"` | Home | Under |

---

## Firebase Storage Schema

```json
{
  "positionType": "0",           // String: "0" or "1"
  "positionTypeString": "Upper", // String: "Upper" or "Lower"
  "theNumber": "-4",             // String: whole number
  "awayTeam": "Toronto Raptors",
  "homeTeam": "Cleveland Cavaliers"
}
```

---

## Summary

1. **theNumber is always from away team's perspective**
2. **Same theNumber, different positionType = opposite sides of same bet**
3. **Spreads:** `Math.floor(agentLine)` with team consideration
4. **Totals:** `Math.ceil(agentLine)` always
5. **Agent memory:** Lines must end in `.5` only
6. **UI display:** Add/subtract 0.5 based on context
7. **No pushes possible:** `>=` logic ensures one side always wins
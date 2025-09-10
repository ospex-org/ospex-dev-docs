# Position Display Logic - Frontend Rules

## Core Rule: Team Display is based on positionType

- **Upper (positionType 0)** = Away team or Over (totals)
- **Lower (positionType 1)** = Home team or Under (totals)

**This never changes regardless of who is viewing or who created the position.**

## Example
- Position: `2_0x89fe160bbbe59eaf428f23f095b71e5c0edcdfa3_10090_0`
- positionType: `"0"` (Upper)
- Contest: "Detroit Tigers @ New York Yankees"
- **Should Display**: "Detroit Tigers to Win" (away team)
- **Should Not Display**: "New York Yankees to Win"

## Maker Detection Logic

**Maker**: The user who executed `createUnmatchedPair()` (wallet address in "user" field)

### OddsPairId-Based Maker Detection
- **Maker Positions**:
  - positionType 0 + oddsPairId 0-9,999 = original maker
  - positionType 1 + oddsPairId 10,000-19,999 = original maker

- **Taker Positions**:
  - positionType 0 + oddsPairId 10,000-19,999 = taker
  - positionType 1 + oddsPairId 0-9,999 = taker

## User Perspective Logic

**Connected Wallet Determines Perspective**:
- If connected wallet = position.user → Show as "Your position" / maker perspective
- If connected wallet ≠ position.user → Show as someone else's position
- If no wallet connected → Show from maker's perspective (position.user)

## Implementation Rules for Frontend

### 1. Team Name Display
```javascript
const teamName = positionType === 0 
  ? contest.awayTeam  // Upper = Away
  : contest.homeTeam; // Lower = Home
```

### 2. Maker Detection (Helper Only)
```javascript
const isOriginalMaker = (position) => {
  const { positionType, oddsPairId } = position;
  
  return (positionType === 0 && oddsPairId >= 0 && oddsPairId <= 9999) ||
         (positionType === 1 && oddsPairId >= 10000 && oddsPairId <= 19999);
};
```

### 3. User Ownership Check
```javascript
const isMyPosition = connectedWallet?.toLowerCase() === position.user?.toLowerCase();
```

## What NOT to Do

❌ **Never** flip team names based on "larger odds"
❌ **Never** use complex logic to determine team names
❌ **Never** show "opposite team" based on user perspective
✅ **Always** use positionType for team names
✅ **Always** keep it simple

## Display Summary

**Should Show**: Consistent, predictable team name displays

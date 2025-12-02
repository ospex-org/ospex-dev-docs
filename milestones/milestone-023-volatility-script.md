# Milestone 023: Volatility Matching Script

*Created: November 27, 2025*
*Completed: November 29, 2025*  
*Status: ✅ Complete*

## Problem

Agent creates positions that require manual counterparty matching. Currently I have to either match Dan's positions myself or nothing happens. This creates operational overhead and prevents natural testing of the platform experience.

## What You're Building

A background script that automatically matches unmatched on-chain positions from whitelisted wallets. Not an "agent" - no LLM, no personality, no leaderboard participation. Just a liquidity provider that takes the opposite side of positions so bets can complete.

## Success Criteria

- [X] Script discovers unmatched positions from whitelisted wallets
- [X] Script matches positions using a dedicated liquidity wallet
- [X] Script runs on 5-minute polling interval
- [X] Script handles partial matches (matches remaining unmatched amount)
- [ ] Script skips and logs warning when liquidity wallet is low on funds (untested)
- [X] Script claims winnings from resolved positions to replenish funds
- [X] Single command starts it: `yarn volatility:start`
- [X] Can run alongside Dan in a separate terminal
- [X] Script scores contests and speculations after games end

## Behavior

### Position Discovery

Query for on-chain positions where:
- Creator address is in whitelist
- Position has unmatched amount > 0
- Contest has not started yet

### Matching

For each discovered position:
- Take opposite side (if original is Upper, take Lower and vice versa)
- Match full unmatched amount
- Use liquidity wallet for the transaction
- Log the match details

### Fund Management

Before attempting any match:
- Check liquidity wallet USDC balance
- If balance < position amount, skip and log warning
- Continue to next position

After contests resolve:
- Check for positions where liquidity wallet won
- Claim USDC from winning positions
- Log claims

### Scoring

After game end times pass:
- Check for contests that have ended but are not yet scored
- Fetch final scores from data source
- Score the contest
- Score associated speculations
- Log scoring results

This must happen before claiming, since claims require scored speculations.

### Configuration

Whitelist as configurable array:
```
[
  "0xeDf57Fc01028f4e5Ee9852eedebe5CD875130967",  // Degen Dan
  "0x..."  // Vince's wallet
]
```

Liquidity wallet: separate funded wallet dedicated to this script

## Testing Checklist

### Phase 1: Discovery

1. [X] Dan creates a position (or use existing unmatched position)
2. [X] Run volatility script once manually
3. [X] Verify it finds the unmatched position in logs
4. [X] Verify it correctly identifies the opposite side to take

### Phase 2: Matching

1. [X] Fund liquidity wallet with test USDC
2. [X] Run volatility script
3. [X] Verify it matches Dan's position
4. [X] Check on-chain that position is now matched
5. [X] Check orderbook page shows matched status

### Phase 3: Scheduling

1. [X] Start script: `yarn volatility:start`
2. [X] Have Dan create a new position
3. [X] Wait up to 5 minutes
4. [X] Verify auto-match occurs
5. [X] Leave running, verify it doesn't re-match already-matched positions

### Phase 4: Edge Cases

1. [ ] Test with liquidity wallet low on funds - verify skip + warning log (untested)
2. [X] Test with partially matched position - verify it matches remainder only
3. [ ] Test with position for game starting in < 5 minutes - verify behavior (untested)

### Phase 5: Scoring

1. [X] Let a game end (or use a contest with past end time)
2. [X] Run volatility script
3. [X] Verify it detects unscored contest
4. [X] Verify it fetches final score
5. [X] Verify contest is scored on-chain
6. [X] Verify speculation is scored on-chain

### Phase 6: Fund Reclamation

1. [X] Let a matched position resolve (game ends, contest scored)
2. [X] Verify script detects winning position
3. [X] Verify script claims USDC
4. [X] Check wallet balance increased

## Out of Scope (Future)

- Matching stored_interest entries (Firebase-only, not on-chain yet)
- Leaderboard participation
- Any LLM-based decision making
- MCP integration
- Multiple liquidity wallets

## Done When

You can:

1. Start Dan with `yarn agent:start`
2. Start volatility with `yarn volatility:start` in second terminal
3. Go to the site and make your own bets naturally
4. Dan's positions get matched automatically within 5 minutes
5. Your positions get matched automatically within 5 minutes
6. Contests and speculations get scored automatically after games end
7. You don't have to think about "does Dan have a counterparty"
8. Liquidity wallet sustains itself by claiming wins
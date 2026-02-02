Phase 1: Static Analysis Security Report                                                                                                                                                                    
                                                                                                                                                                                                                CRITICAL FINDINGS (potential fund loss)                                                                                                                                                                     

  [C-1] Matching math can leave the contract insolvent — _completeUnmatchedPair
  PositionModule.sol:898-908

  The taker deposits amount tokens. The maker's consumed amount is calculated as:
  makerAmountConsumed = (amount * (oppositeOdds - ODDS_PRECISION)) / ODDS_PRECISION

  But the payout on claim is:
  payout = (matchedAmount * odds) / ODDS_PRECISION

  The question is: does makerAmountConsumed + takerAmount always equal makerPayout or takerPayout (whichever wins)? If the winner's payout exceeds what was deposited by both sides, the contract becomes     
  insolvent.

  Example walkthrough: Maker creates Upper position at 2.00 odds (20,000,000). Inverse = 2.00. Maker deposits 10 USDC unmatched. Taker matches with 10 USDC.
  - matchableAmount = 10 * (20000000 - 10000000) / 10000000 = 10 (correct)
  - makerAmountConsumed = 10 * (20000000 - 10000000) / 10000000 = 10 (all maker unmatched consumed)
  - Maker's matchedAmount = 10. Winner payout = 10 * 20000000 / 10000000 = 20. Total deposited = 20. OK at 2.00.

  But with asymmetric odds and rounding from roundOddsToNearestIncrement, there could be dust-level insolvency over many matches. This needs a Foundry fuzz test with extreme odds values (e.g., 1.01 and     
  101.00) to verify solvency invariants hold.

  [C-2] claimPosition doesn't check payout > 0 before transfer when unmatchedAmount > 0
  PositionModule.sol:619-635

  A losing position with matchedAmount > 0 and unmatchedAmount > 0 will have payout = 0 + unmatchedAmount, which is correct — unmatched funds return. But a position with matchedAmount = 0 and
  unmatchedAmount = 0 is already guarded by line 619. This is actually fine on re-inspection. The real concern is whether a user can claim the same position by calling claimPosition from different
  oddsPairIds that map to the same underlying position. Since the mapping is speculationId => user => oddsPairId => positionType, each combination is unique. No issue here on further analysis.

  HIGH FINDINGS

  [H-1] settleSpeculation is permissionless — anyone can trigger settlement
  SpeculationModule.sol:251

  Anyone can call settleSpeculation. This is by design (the scorer service calls it), but it means a malicious actor could front-run the scorer to settle a speculation using stale or manipulated contest    
  scores. However, since scores come from the oracle via setScores (which is onlyOracleModule), and settleSpeculation reads from already-stored scores, this is actually safe. The scores can't be manipulated
   by the caller. Not exploitable, but worth documenting.

  [H-2] fulfillRequest only validates requestId == s_lastRequestId — concurrent oracle requests break
  OracleModule.sol:391

  If two oracle requests are sent before either callback arrives, only the last one will be accepted. The first callback will revert with UnexpectedRequestId. This means LINK is spent but the callback is   
  wasted. This isn't a fund-loss vulnerability but could cause operational issues (contests stuck unverified). Operational risk, not security risk.

  [H-3] Secondary market: listing persists after position is partially sold via transferPosition
  SecondaryMarketModule.sol:313-343

  When a buyer purchases a partial amount, the listing's amount is reduced. But the listing's price stays the same (it's the total price for the original listing amount). The purchase price calculation at  
  line 323:
  purchasePrice = (listing.price * amount) / listing.amount
  This is correct for partial fills — it's pro-rata. Not a vulnerability.

  [H-4] transferPosition sets toPos.unmatchedAmount = 0 — could zero out buyer's existing unmatched
  PositionModule.sol:563

  If the buyer (recipient of transferPosition) already has an unmatched position at the same speculation/oddsPairId/positionType, their unmatchedAmount gets zeroed. This is a real issue — buying a position 
  on the secondary market could wipe out the buyer's pending unmatched funds. Those funds would be stuck in the contract with no way to withdraw.

  MEDIUM FINDINGS

  [M-1] OddsPair storage is global, not per-speculation
  PositionModule.sol:86

  s_oddsPairs is a flat mapping from oddsPairId to OddsPair. The oddsPairId is derived purely from the odds value and position type. This means all speculations share the same odds pairs. This is fine      
  architecturally (odds pairs are just math), but worth confirming that no logic assumes odds pairs are speculation-specific.

  [M-2] Rounding in inverse odds calculation could create unfair asymmetry
  PositionModule.sol:726-737

  calculateAndRoundInverseOdds uses integer division which truncates. The inverse is then rounded to nearest increment. For example, at 1.05 odds (10,500,000):
  - numerator = 1e7 * 1e7 = 1e14
  - denominator = 10500000 - 10000000 = 500000
  - exactInverse = 1e14 / 500000 + 1e7 = 200000000 + 10000000 = 210000000 = 21.00

  That's correct. But at odds like 1.03 (10,300,000):
  - denominator = 300000
  - exactInverse = 1e14 / 300000 + 1e7 = 333333333 + 10000000 = 343333333 = 34.33
  - Rounded to nearest 0.01 = 34.33

  Then the combined implied probability: 1/1.03 + 1/34.33 = 0.9709 + 0.0291 = 1.0000. Almost exactly 1.0, which is correct for a zero-vig market. The math checks out at these extremes. But a fuzz test would
   be prudent to confirm no edge cases create solvency issues.

  [M-3] setAdmin in OspexCore is a one-step transfer with no confirmation
  OspexCore.sol:101-108

  The admin transfer immediately revokes the old admin and grants to the new one. If the wrong address is provided, admin access is permanently lost. A two-step transfer pattern (propose/accept) would be   
  safer. Not a fund-loss risk but an operational risk.

  [M-4] Leaderboard prize pool share calculation uses integer division, remainder lost
  LeaderboardModule.sol:754
  uint256 share = treasuryModule.getPrizePool(leaderboardId) / scoring.winners.length;
  If the pool is 10 and there are 3 winners, each gets 3, and 1 token is stuck forever. The adminSweep function uses the same division, so the remainder is never recoverable. Dust amount, but worth noting. 

  [M-5] submitLeaderboardROI — ROI of exactly 0 treated as "not submitted"
  LeaderboardModule.sol:641
  if (leaderboardScoring.userROIs[msg.sender] != 0) {
      revert LeaderboardModule__ROIAlreadySubmitted();
  }
  If a user's actual ROI calculates to exactly 0 (broke even), they can submit repeatedly. Each submission would recalculate and potentially get re-added to the winners list (though the duplicate check at  
  line 678 prevents actual duplicate entries). The main issue is they can call submitLeaderboardROI infinitely if their ROI is exactly 0. It just wastes gas and emits redundant events.

  [M-6] s_linkDenominator can be set to 0 by admin, causing division by zero
  OracleModule.sol:111
  uint256 payment = LINK_DIVISIBILITY / s_linkDenominator;
  If admin sets s_linkDenominator = 0, all oracle calls revert. DoS vector, but admin-only.

  LOW FINDINGS

  [L-1] No event for setAdmin in OspexCore when revoking old admin
  The AdminChanged event is emitted, which is sufficient.

  [L-2] scoreTotal returns Over when combined score equals theNumber
  TotalScorerModule.sol:75 — >= means exact match is Over. This is a design choice, not a bug, but is different from some sportsbook conventions where exact match is a Push. Document this clearly for users.

  [L-3] scoreSpread returns Away when tied with spread
  SpreadScorerModule.sol:76 — >= means if awayScore + spread == homeScore, it's Away. Same design-choice consideration as above.

  [L-4] No ReentrancyGuard on ContributionModule.handleContribution
  It does a safeTransferFrom on the contribution token, which is an external call. Since the contribution token is admin-configured and is likely USDC (not a reentrant token), this is low risk. But if an   
  ERC-777 or similar hook-enabled token were used as the contribution token, reentrancy could be possible.

  POSITIVE OBSERVATIONS (things done well)

  1. ReentrancyGuard used on all critical fund-moving functions in PositionModule and SecondaryMarketModule
  2. SafeERC20 used consistently for all token transfers
  3. Checks-Effects-Interactions pattern followed in claimPosition (state zeroed before transfer at lines 630-632, then transfer at 635)
  4. Solidity 0.8.19+ provides built-in overflow/underflow protection
  5. Access control is well-structured — OspexCore as registry, modules check authorization properly
  6. Source hash validation on oracle JS code prevents tampered oracle requests
  7. Void cooldown provides a safety net for unscored contests
  8. No delegatecall patterns that could introduce proxy vulnerabilities

  RECOMMENDED PHASE 2 EXPLOIT TESTS (priority order)

  1. Solvency fuzz test: Create positions at every possible odds value (1.01 to 101.00), match them, resolve, claim — verify contract token balance never goes negative
  2. H-4 exploit: Create unmatched position, buy a position on secondary market at same speculation/oddsPairId, verify unmatched amount is zeroed
  3. Rounding attack: Attempt to extract value through repeated small positions at extreme odds
  4. DoS via array: Create many unmatched pairs for the same speculation, check gas limits on batch operations
  5. Double-claim attempt: Try claiming same position twice, try claiming with different oddsPairId combinations

  ---

● That's the full Phase 1 report. The most actionable finding is H-4 (transferPosition zeroing unmatchedAmount on the buyer) — that's a concrete bug where a user could lose funds. C-1 (solvency invariant
  with rounding) needs a fuzz test to confirm or dismiss.
                                                                                                                                                                                                                Want me to move to Phase 2 and start writing Foundry exploit test scripts? I'd begin with the solvency fuzz test and the H-4 exploit reproduction. You'd run them locally with forge test.
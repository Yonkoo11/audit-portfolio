# `changeRanges` silently sends all liquidity to treasury when price is out of range

**Severity:** Medium
**Contest:** Olas — Code4rena (Jan 2026)
**Submission:** [S-993](https://code4rena.com/audits/2026-01-olas/submissions/S-993)
**Official report:** [M-06 in the Olas C4 report](https://code4rena.com/reports/2026-01-olas#m-06-changeranges-silently-fails-when-price-is-out-of-tick-range-sending-all-liquidity-to-treasury-instead-of-creating-new-position)

## File

`autonolas-tokenomics/contracts/pol/LiquidityManagerCore.sol` L761

Function: `changeRanges`.

## Bug

`changeRanges` rebalances the protocol-owned Uniswap v3 position. The flow:

1. Read the current pool price via `checkPoolAndGetCenterPrice`.
2. Compute new `tickLower` / `tickUpper` from that price.
3. Withdraw all liquidity from the old range.
4. Mint a new position at the new range.

The function never re-checks that the current pool price still falls inside the new range at mint time. Between step 1 and step 4 the price can move, or the range computation itself can produce bounds that the price already sits outside of.

When the mint fails to take in the supplied amounts (Uniswap v3 returns zero liquidity if both bounds are on the same side of the current tick), the fall-through path forwards the withdrawn token balance to the treasury instead of leaving it in the manager or reverting.

## Impact

A single `changeRanges` call run with a stale or skewed price moves the entire protocol-owned LP stash into the treasury in one tx. The protocol loses fee-earning concentrated exposure and cannot recover the position from inside the same call. Recovering requires a separate governance flow to pull funds back from treasury and re-mint, with all in-between fees lost.

The trigger does not require an attacker; ordinary price volatility between block proposal and execution is enough on volatile pairs. On a calm pair it still fires when an operator picks a tick width narrower than the in-block price drift.

## Recommendation

After computing `tickLower` / `tickUpper`, assert `tickLower <= currentTick < tickUpper` and revert with a descriptive error if not.

Stronger fix: remove the treasury fall-through inside `changeRanges`. On mint-zero-liquidity, revert the whole rebalance and leave the old position untouched. A rebalance that cannot create the intended position should not silently relocate the liquidity somewhere else.

# D3 liquidation seizes collateral at manipulable pool price while HF uses oracle price

**Severity:** Medium
**Contest:** Fluid DEX v2 — Sherlock #1225 (Jan 2026)
**Submission:** [audits.sherlock.xyz/contests/1225/voting/389](https://audits.sherlock.xyz/contests/1225/voting/389)

## Files

- `protocols/moneyMarket/core/liquidateModule/main.sol`
- `protocols/moneyMarket/core/other/helpers.sol`
- `protocols/dexV2/dexTypes/common/d3d4common/swapModuleInternals.sol`

Functions in play: `liquidate`, `_getD3SupplyAmounts`, `_getHfInfo`, `_calculateSqrtPriceX96`.

## Bug

Liquidations of D3-pool collateral compute the seized token0/token1 split at the live pool price (`AT_POOL_PRICE`) inside `_getD3SupplyAmounts`. The same liquidation values the position with an oracle-derived sqrtPrice in `_getHfInfo` / `_calculateSqrtPriceX96`. The two prices are independent inputs.

The per-call sqrtPrice movement is bounded by `MAX_SQRT_PRICE_CHANGE_PERCENTAGE` (20%), but the bound resets each `swapIn` / `swapOut`. A liquidator chains swaps in the same block to push the pool price arbitrarily far from oracle. The health-factor check still reads the oracle and passes (or barely passes), then the seizure runs against the manipulated pool state.

## Impact

A liquidator who plans the swap sequence extracts more than the protocol-defined liquidation penalty by choosing which side of the pool to be paid in. The position holder eats the difference.

In stressed-pool conditions the same asymmetry can flip the other way and seize collateral from positions that the oracle still considers solvent.

## Recommendation

Use the same sqrtPrice for both HF valuation and collateral payout. Either:

1. Reuse the oracle sqrtPrice inside `_getD3SupplyAmounts` for the supply split, or
2. Reject the liquidation when `|poolSqrtPrice - oracleSqrtPrice|` exceeds a threshold tied to the liquidation penalty (so manipulation costs more than it earns).

Capping the per-block aggregate sqrtPrice movement (rather than per-call) also closes the chained-swap path.

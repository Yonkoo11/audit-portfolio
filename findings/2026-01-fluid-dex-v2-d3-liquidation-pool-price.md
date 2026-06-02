# D3 liquidation penalty manipulation via pool price

**Severity:** Medium
**Contest:** Fluid DEX v2 — Sherlock #1225 (Jan 2026)
**Submission:** [#389](https://audits.sherlock.xyz/contests/1225/voting/389)

## Summary

D3 liquidation computes the supply-amount composition (`token0` vs `token1` value seized) at the actual DEX pool price (`AT_POOL_PRICE`) rather than the oracle price. A liquidator pushes the pool price toward the token with the higher liquidation penalty before calling `liquidate()`, raises the weighted-average penalty, and extracts more collateral from the victim than the protocol's configured penalty allows.

## Two prices, one liquidation

The same `liquidate` call reads two different prices for two closely related calculations.

**Health-factor check uses the oracle price.** `other/helpers.sol:899`:

```solidity
uint256 sqrtPriceX96_ = _calculateSqrtPriceX96(
    vhf_.token0Price, vhf_.token1Price,
    token0SupplyExchangePrice_, token1SupplyExchangePrice_,
    token0Decimals_, token1Decimals_
);
```

This feeds `getAmountsForLiquidity` to decide how much `token0` vs `token1` the position holds for HF purposes. Oracle-derived, manipulation-resistant.

**Penalty calculation uses the pool price.** `liquidateModule/main.sol:379`:

```solidity
) = _getD3SupplyAmounts(
    _getDexId(dexKey_),
    GetD3D4AmountsParams({
        tickLower: s_.tickLower,
        tickUpper: s_.tickUpper,
        positionSalt: s_.positionSalt,
        token0Decimals: vl_.token0Decimals,
        token1Decimals: vl_.token1Decimals,
        token0ExchangePrice: token0SupplyExchangePrice_,
        token1ExchangePrice: token1SupplyExchangePrice_,
        sqrtPriceX96: AT_POOL_PRICE  // live DEX pool price, manipulable
    })
);
```

`AT_POOL_PRICE` resolves inside `_getD3SupplyAmounts` (`other/helpers.sol:467-469`) to the current stored pool price:

```solidity
if (params_.sqrtPriceX96 == AT_POOL_PRICE) {
    params_.sqrtPriceX96 = (dexVariables_ >> DSL.BITS_DEX_V2_VARIABLES_CURRENT_SQRT_PRICE) & X72;
    params_.sqrtPriceX96 = BM.fromBigNumber(params_.sqrtPriceX96, DEFAULT_EXPONENT_SIZE, DEFAULT_EXPONENT_MASK);
}
```

That stored value is updated by every swap (`swapModuleInternals.sol:315-323`). The resulting supply amounts drive the weighted-average penalty (`liquidateModule/main.sol:413-416`):

```solidity
averageLiquidationPenalty_ = (
    (vl_.token0LiquidationPenalty * supplyValue0_) +
    (vl_.token1LiquidationPenalty * supplyValue1_)
) / supplyValue_;
```

## Why the existing guards do not stop this

**Per-swap sqrtPrice cap (`MAX_SQRT_PRICE_CHANGE_PERCENTAGE` = 20%).** `swapModuleInternals.sol:267` bounds each swap, but the pool lock releases after each swap (`swapModule.sol:126`), so an attacker chains N swaps in one tx for cumulative `1.2^N`. A single 20% sqrtPrice shift already moves a concentrated D3 position from a 50/50 composition to ~95/4, which is enough to drive the penalty differential.

**`maxLiquidationPenalty` cap (`liquidateModule/main.sol:60-64`).** Caps individual token penalties at `(collateralValue - debtValue) * 1000 / debtValue`. With a typical LT of 800 (80%), a position can be liquidatable (HF < 1.0) while collateral still exceeds debt by 10-25%. With `collateral = $115K`, `debt = $100K`, the cap is 150 (15%) — far above per-token configurations like 8% / 3%. The differential the attack relies on is preserved.

**Post-liquidation HF check (`if (_getHfInfo(...).hf > hfLimit_) revert`).** Extracting more collateral *lowers* the post-liquidation HF, so this check is the wrong direction to catch the abuse.

## Attack flow

1. Liquidator finds an underwater D3 position with asymmetric token penalties (e.g. ETH 8%, stablecoin 3%).
2. Flash-loans capital. Runs one or more swaps in the D3 pool to push the stored pool price toward the higher-penalty token. Each swap stays inside the 20% per-swap cap; the pool lock releases between swaps.
3. Calls `liquidate()`. The penalty weighting at L413-416 reads the manipulated pool price via `AT_POOL_PRICE`, yielding a higher average penalty.
4. Swaps back to repay the flash loan, closing the sandwich.

The swap and the liquidation hit different contracts (DEX vs Money Market), so the per-`dexId` pool lock does not interfere.

## Impact

Direct loss of funds for the liquidated user. The liquidator extracts more collateral than the protocol's penalty configuration is supposed to allow.

Worked example. Tick range `[-5000, 5000]`, token0 penalty 8%, token1 penalty 3%, position value $115K, debt $100K (HF = 0.92, LT = 800). `maxLiquidationPenalty` = 15% — does not cap either of 8% or 3%.

| Scenario | token0 fraction | token1 fraction | Avg penalty | Bonus on $100K |
|---|---|---|---|---|
| Normal (50/50) | 50% | 50% | 5.5% | $5,500 |
| 1 swap (20% sqrt down) | 95% | 4% | 7.7% | $7,700 |
| 2 swaps (36% sqrt down) | 96% | 3% | 7.8% | $7,800 |

Extra extraction per $100K liquidated: $2,200 to $2,300 depending on how many swaps are chained. The cost to the attacker is only the round-trip swap slippage (a few basis points), which is dwarfed by the extra penalty extracted. The gain scales linearly with size and with the penalty differential.

The protocol already uses oracle-derived pricing for HF in the same liquidation flow. The penalty calculation should be consistent.

## Proof of concept

Foundry test using the same `LiquidityAmounts` and `TickMath` libraries as Fluid DEX v2.

```
[PASS] test_singleSwapManipulation()
  === NORMAL (pool price at center) ===
    token0 value %: 50
    token1 value %: 50
    avgPenalty (THREE_DECIMALS): 55
  === AFTER 1 SWAP (20% sqrtPrice decrease) ===
    token0 value %: 95
    token1 value %: 4
    avgPenalty (THREE_DECIMALS): 77
  === IMPACT per $100K debt liquidated ===
    Normal penalty bonus ($): 5500
    Attacked penalty bonus ($): 7700
    Extra extracted ($): 2200
    Penalty increase (%): 40

[PASS] test_chainedSwapManipulation()
  === AFTER 2 CHAINED SWAPS (36% sqrtPrice decrease) ===
    token0 value %: 96
    token1 value %: 3
    avgPenalty (THREE_DECIMALS): 78
  === IMPACT per $100K debt liquidated (2 chained swaps) ===
    Extra extracted ($): 2300

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Test file: `test/foundry/poc/MED001_D3LiquidationPenaltyManipulation.t.sol`.

## Recommendation

Use the oracle-derived `sqrtPriceX96` for the D3 supply-amount calculation during liquidation, matching what HF already does:

```solidity
sqrtPriceX96: _calculateSqrtPriceX96(
    vl_.token0Price, vl_.token1Price,
    token0SupplyExchangePrice_, token1SupplyExchangePrice_,
    vl_.token0Decimals, vl_.token1Decimals
)
```

This makes the token composition used for penalty weighting oracle-derived and removes the manipulation vector. Alternative: cap the per-block aggregate sqrtPrice movement (not per-call) so chained swaps cannot bypass the limit.

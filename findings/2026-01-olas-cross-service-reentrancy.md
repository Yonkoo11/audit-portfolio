# Cross-service reentrancy in `StakingBase._withdraw` drains balance accounting

**Severity:** High
**Contest:** Olas — Code4rena (Jan 2026)
**Submission:** [S-946](https://code4rena.com/audits/2026-01-olas/submissions/S-946)
**Official report:** [H-05 in the Olas C4 report](https://code4rena.com/reports/2026-01-olas#h-05-insolvency-via-cross-service-reentrancy-in-stakingbase_withdraw)

## File

`autonolas-registries/contracts/staking/StakingBase.sol` L929–L948

External state-changing functions affected: `stake`, `unstake`, `claim`, `checkpointAndClaim`, `forcedUnstake`.

## Bug

`_withdraw` caches the global `balance` into a local `updatedBalance`, loops over agent instances paying each one, and writes `updatedBalance` back to storage at the end. No reentrancy guard sits on the public entry points that reach `_withdraw`.

An attacker who controls two services can reenter mid-transfer. Sequence:

1. Outer call enters `unstake(serviceA)`. `balance` (say 100) is read into local `updatedBalance`.
2. The transfer to a service-owned receiver hands control to the attacker.
3. Attacker calls `unstake(serviceB)`. The inner call reads the **unchanged** global `balance` (still 100), computes its own decrement, writes the result (say 90), and returns.
4. The outer call resumes, finishes its loop, and writes its stale local `updatedBalance` back to storage — overwriting the inner write.

The inner decrement is erased. Both services received tokens, but the contract's accounting reflects only one of them.

## Impact

Direct insolvency. The on-chain `balance` ends the block higher than the contract's actual token holdings. Subsequent legitimate unstakes either drain the contract below its obligations or revert once the next checkpoint underflows. Loss of staked principal for users who unstake after the attack.

## Recommendation

1. Inherit `ReentrancyGuard` on `StakingBase` and apply `nonReentrant` to `stake`, `unstake`, `claim`, `checkpointAndClaim`, `forcedUnstake`.
2. Refactor `_withdraw` to write the new `balance` to storage **before** the transfer loop, not after. The local `updatedBalance` should never outlive a single external call.

Both fixes together close the class. Either alone is enough for this finding, but the CEI fix is also a code-quality win independent of the guard.

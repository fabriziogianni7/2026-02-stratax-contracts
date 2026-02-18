# Unwind Uses Liquidation Threshold Instead of LTV for Collateral Calculation

## Description

* When unwinding a leveraged position, the contract repays Aave debt and withdraws the collateral that backed that debt. The collateral-to-debt relationship is defined by the LTV (Loan-to-Value) ratio: when opening, `borrowValue = collateralValue * LTV`, so when unwinding, `collateralToWithdraw = debtValue / LTV`.

* The code fetches the 3rd return value from `getReserveConfigurationData` (liquidation threshold) instead of the 2nd (LTV). Liquidation threshold is typically higher than LTV (e.g., 85% vs 80%). Using it divides by a larger number, so the contract withdraws less collateral than it should.

```solidity
            // Get LTV from Aave for the collateral token
@>          (,, uint256 liqThreshold,,,,,,,) =
@>              aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
            ...
            // Calculate collateral to withdraw: (debtAmount * debtPrice * collateralDec * LTV_PRECISION) / (collateralPrice * debtDec * ltv)
            uint256 collateralToWithdraw = (
                _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
@>          ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
```

## Risk

**Likelihood (high)**:

* Every unwind operation uses this code path.

* The open flow correctly uses LTV (line 386); the unwind flow incorrectly uses liquidation threshold.

**Impact (medium)**:

* With less collateral withdrawn, the swap yields fewer debt tokens. The unwind can revert at `require(returnAmount >= totalDebt, "Insufficient funds to repay flash loan")`, preventing users from unwinding positions.

* If the unwind succeeds (e.g., favorable price movement or slippage), excess collateral remains in Aave. The user effectively over-collateralizes and cannot easily recover the surplus.

**Severity (medium)**:

## Proof of Concept
repaying 1000 USDC debt with ETH collateral:
- LTV = 8000 (80%)
- liqThreshold = 8500 (85%)
- Correct collateral to withdraw = 1000 / 0.80 = $1250 worth of ETH
- Bug: contract uses 8500 → collateral = 1000 / 0.85 ≈ $1176 worth of ETH
- Withdrawn: ~$1176 ETH instead of $1250 → ~$74 less collateral
- Swap $1176 ETH → may get < $1000 + premium USDC → REVERT
- Or if swap succeeds, ~$74 worth of ETH left in Aave, user cannot access it via this unwind


## Recommended Mitigation

**Rationale**: The unwind reverses the open flow. On open, `borrowValue = collateralValue × LTV`, so on unwind, `collateralValue = debtValue / LTV`. Liquidation threshold is unrelated—it defines when a position becomes liquidatable, not the collateral-to-debt ratio. Using LTV restores symmetry with `calculateOpenParams` (line 386), which already fetches LTV for the same collateral token.

**Changes**:
1. Destructure the 2nd return value (`ltv`) instead of the 3rd (`liquidationThreshold`).
2. Add `require(ltv > 0)` to guard against division by zero and invalid Aave config (matches open flow).
3. Use `ltv` in the formula denominator.

```diff
-            // Get LTV from Aave for the collateral token
-            (,, uint256 liqThreshold,,,,,,,) =
-                aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
+            // Get LTV from Aave for the collateral token (must match open flow: collateralValue = debtValue / LTV)
+            (, uint256 ltv,,,,,,,,) =
+                aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
+            require(ltv > 0, "Asset not usable as collateral");
             ...
-            ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
+            ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * ltv);
```

**Verification**: After the fix, unwinding a position opened with `calculateOpenParams` should withdraw exactly the collateral that backed the repaid debt. Consider adding a test that opens a position, unwinds it, and asserts the contract holds no leftover collateral (or the expected surplus).

**Location**: `src/Stratax.sol:567-579`

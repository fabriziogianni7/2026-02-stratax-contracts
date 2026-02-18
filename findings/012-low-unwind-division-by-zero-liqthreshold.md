# Unwind Division by Zero When liqThreshold Is Zero

## Description

* When unwinding a leveraged position, the contract repays Aave debt via flash loan, then calculates how much collateral to withdraw. The formula derives collateral from the debt-to-collateral ratio: `collateralValue = debtValue / ratio`, where the ratio comes from Aave's reserve configuration.

* The contract fetches `getReserveConfigurationData(unwindParams.collateralToken)` and uses the 3rd return value (`liquidationThreshold`, stored as `liqThreshold`) in the denominator of the collateral withdrawal formula. The denominator is `collateralTokenPrice * (10 ** debtDecimals) * liqThreshold`.

* There is no validation that `liqThreshold > 0`. If Aave returns 0 for the liquidation threshold, the division `numerator / (denom * 0)` reverts with a division-by-zero error. The unwind fails inside the flash loan callback, causing the entire transaction to revert.

* The open flow (`calculateOpenParams`, line 386) correctly guards against this by using `require(ltv > 0, "Asset not usable as collateral")` when fetching LTV. The unwind flow has no equivalent check for `liqThreshold`.

## Affected Code

**Location**: `src/Stratax.sol:574-588`

```solidity
            // hack 006 fetches liqThreshold (3rd) but formula needs LTV (2nd); use (, uint256 ltv,,,,,,,,)
            (,, uint256 liqThreshold,,,,,,,) =
                aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);

            // Get prices and decimals
            uint256 debtTokenPrice = IStrataxOracle(strataxOracle).getPrice(_asset);
            uint256 collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(unwindParams.collateralToken);
            require(debtTokenPrice > 0 && collateralTokenPrice > 0, "Invalid prices");

            // Calculate collateral to withdraw: (debtAmount * debtPrice * collateralDec * LTV_PRECISION) / (collateralPrice * debtDec * ltv)
            uint256 collateralToWithdraw = (
                _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
@>          ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
```

## Risk

**Likelihood (low)**:

* Aave reserves are typically configured with non-zero liquidation threshold (e.g. 8500 = 85% for ETH).

* A zero liquidation threshold could occur if: (1) a reserve is newly added and misconfigured, (2) Aave admin sets it to 0 to disable collateral usage, (3) a future Aave upgrade changes default behavior, or (4) the protocol integrates with a fork or alternative deployment with different config.

**Impact (medium)**:

* The revert occurs inside `executeOperation`, the Aave flash loan callback. The flash loan has already been disbursed; the revert causes the entire callback to fail, so the flash loan is not repaid and the whole transaction reverts.

* The unwind cannot be executed. The position remains leveraged; the owner cannot close it until Aave config is fixed or a workaround is deployed.

**Severity (low)**:

## Proof of Concept

```solidity
// Scenario: Aave returns liquidationThreshold = 0 for unwindParams.collateralToken
// (e.g. reserve was disabled or misconfigured)

// 1. Flash loan of 1000 USDC is received
// 2. Aave debt is repaid successfully
// 3. getReserveConfigurationData returns (18, 8000, 0, 10500, ...)  // liqThreshold = 0
// 4. Formula: collateralToWithdraw = numerator / (collateralTokenPrice * 10^6 * 0)
// 5. Division by zero → Solidity reverts
// 6. executeOperation reverts → flash loan callback fails → entire tx reverts
// 7. Position cannot be unwound
```

## Recommended Mitigation

Add an explicit guard before the division. When fixing finding 006 (use LTV instead of liqThreshold), include the same guard for LTV:

```diff
             (,, uint256 liqThreshold,,,,,,,) =
                 aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
+            require(liqThreshold > 0, "Asset not usable as collateral");
             // Get prices and decimals
```

**Note**: Per finding 006, the correct fix is to use `ltv` instead of `liqThreshold` in the formula. The same `require(ltv > 0, "Asset not usable as collateral")` guard should be added, matching the open flow at line 387.

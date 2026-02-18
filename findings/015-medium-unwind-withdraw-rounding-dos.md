# Unwind DoS When Aave withdraw Returns Less Than Requested

## Description

* The contract computes `collateralToWithdraw` and calls `aavePool.withdraw(collateralToken, collateralToWithdraw, address(this))`.

* Aave's `withdraw` can return less than the requested amount (rounding, dust, interest accrual, or protocol-specific behavior).

* The returned `withdrawnAmount` is used for the 1inch swap. If it is less than expected, `returnAmount` may be below `totalDebt`, causing a revert at the flash loan repayment check.

```solidity
            withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
        }
        // ...
        IERC20(unwindParams.collateralToken).approve(address(oneInchRouter), withdrawnAmount);
@>      uint256 returnAmount = _call1InchSwap(unwindParams.oneInchSwapData, _asset, unwindParams.minReturnAmount);
        // ...
@>      require(returnAmount >= totalDebt, "Insufficient funds to repay flash loan");
```

## Risk

**Likelihood (medium)**:

* Aave V3 withdraw can round down; low-decimal tokens or dust can produce smaller amounts.

* The owner sets `minReturnAmount` based on expected collateral; they may not account for withdraw rounding.

**Impact (medium)**:

* Unwind reverts inside the flash loan callback.

* Position cannot be unwound; user/protocol may be stuck until conditions improve.

**Severity (medium)**:

## Proof of Concept

Aave's `withdraw` returns the actual amount transferred, which can be less than the requested amount. This happens because Aave converts between underlying and aToken shares with rounding that favors the protocol (rounds down when burning aTokens). For low-decimal tokens or positions with dust, the difference can be non-trivial. The contract assumes `withdrawnAmount` equals `collateralToWithdraw` when building the swap and setting `minReturnAmount`; it does not account for Aave returning less.

The chain of effects: less collateral withdrawn → less input to the 1inch swap → less debt token received → `returnAmount < totalDebt` → revert at the flash loan repayment check. The owner typically sets `minReturnAmount` based on the formula-derived `collateralToWithdraw` (or `calculateUnwindParams`), not on the actual `withdrawnAmount`. If the swap passes `minReturnAmount` (because the owner used a conservative buffer) but still returns less than `totalDebt`, the unwind reverts. The revert occurs inside the flash loan callback, so the entire transaction fails and the position remains leveraged.

```solidity
// 1. Formula yields collateralToWithdraw = 1.5e18 (e.g. 1.5 WETH)
// 2. aavePool.withdraw(..., 1.5e18, ...) returns 1.499999999e18 due to aToken/share rounding
// 3. We swap 1.499999999e18 WETH → ~999 USDC (slightly less than 1000 + premium)
// 4. totalDebt = 1000 + 9 = 1009 USDC
// 5. require(999 >= 1009) reverts with "Insufficient funds to repay flash loan"
// 6. executeOperation reverts → flash loan callback fails → entire tx reverts
// 7. Position cannot be unwound; owner is stuck until prices move or they set a lower minReturnAmount
```

## Recommended Mitigation

**Option 1: Scale `minReturnAmount` by actual withdrawn amount (on-chain fix)**

When `withdrawnAmount` is less than `collateralToWithdraw`, scale the owner's `minReturnAmount` proportionally so the swap does not revert with "Insufficient return amount from swap" when the swap would have returned enough. This helps when the swap overperforms (e.g. better routing) despite less collateral input. Note: `collateralToWithdraw` is scoped inside the block, so it must be moved out or the scaling done inside the block. The `require(returnAmount >= totalDebt)` still enforces repayment—if the swap returns less than `totalDebt`, the unwind will still revert.

```diff
         uint256 withdrawnAmount;
+        uint256 collateralToWithdraw;
         {
             (,, uint256 liqThreshold,,,,,,,) =
                 aaveDataProvider.getReserveConfigurationData(unwindParams.collateralToken);
             uint256 debtTokenPrice = IStrataxOracle(strataxOracle).getPrice(_asset);
             uint256 collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(unwindParams.collateralToken);
             require(debtTokenPrice > 0 && collateralTokenPrice > 0, "Invalid prices");

-            uint256 collateralToWithdraw = (
+            collateralToWithdraw = (
                 _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
             ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);

             withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
         }
+        require(withdrawnAmount > 0, "No collateral withdrawn");
+
+        // Scale minReturnAmount by actual withdrawn amount to account for Aave rounding
+        uint256 adjustedMinReturn = collateralToWithdraw > 0
+            ? (unwindParams.minReturnAmount * withdrawnAmount) / collateralToWithdraw
+            : unwindParams.minReturnAmount;
         // Step 3: Swap collateral to debt token to repay flash loan
         IERC20(unwindParams.collateralToken).approve(address(oneInchRouter), withdrawnAmount);
-        uint256 returnAmount = _call1InchSwap(unwindParams.oneInchSwapData, _asset, unwindParams.minReturnAmount);
+        uint256 returnAmount = _call1InchSwap(unwindParams.oneInchSwapData, _asset, adjustedMinReturn);
```

**Option 2: Add helper to compute recommended `minReturnAmount`**

The owner must set `minReturnAmount` to at least the amount needed to repay the flash loan (`totalDebt`). Setting it lower would allow swaps that return less than `totalDebt`, causing a revert at the repayment check. The helper should return the estimated `totalDebt` so the owner sets `minReturnAmount` correctly (e.g. `minReturnAmount = recommendedMinReturn` or slightly lower for swap slippage—but not so low that we accept less than `totalDebt`).

```diff
     function calculateUnwindParams(address _collateralToken, address _borrowToken)
         public
         view
-        returns (uint256 collateralToWithdraw, uint256 debtAmount)
+        returns (uint256 collateralToWithdraw, uint256 debtAmount, uint256 recommendedMinReturn)
     {
         // ... existing logic ...
-        return (collateralToWithdraw, debtAmount);
+        // recommendedMinReturn = estimated totalDebt (debt + flash loan premium); owner should set minReturnAmount >= this
+        uint256 estimatedPremium = (debtAmount * flashLoanFeeBps) / FLASHLOAN_FEE_PREC;
+        recommendedMinReturn = debtAmount + estimatedPremium;
+        return (collateralToWithdraw, debtAmount, recommendedMinReturn);
     }
```

**Option 3: Document owner responsibility**

Document that the owner must set `minReturnAmount` to at least the estimated `totalDebt` (debt + flash loan premium). Setting it lower risks accepting a swap that returns insufficient funds, causing a revert at "Insufficient funds to repay flash loan". The owner cannot fix withdraw rounding by lowering `minReturnAmount`—if Aave returns less collateral, the swap will return less, and the unwind will revert. This relies on the owner and is not enforced on-chain.

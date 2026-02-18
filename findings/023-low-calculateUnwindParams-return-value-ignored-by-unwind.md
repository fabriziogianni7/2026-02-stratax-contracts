# calculateUnwindParams Return Value Ignored by Unwind Flow

## Description

* `calculateUnwindParams` returns `collateralToWithdraw` and `debtAmount` for unwinding a position. Callers (e.g. frontend, tests) use these values to build 1inch swap data and pass them to `unwindPosition`. The function also applies a 5% slippage buffer to `collateralToWithdraw` before returning.

* `unwindPosition` accepts `_collateralToWithdraw` and stores it in `UnwindParams`, but `_executeUnwindOperation` completely ignores it. The callback recalculates `collateralToWithdraw` using a different formula (with LTV/liqThreshold from Aave). The 5% slippage in `calculateUnwindParams` only affects the 1inch swap input size (built from the returned value), while the actual withdrawal uses the internal formula. This creates a mismatch: the swap is prepared for one amount, but a different amount is withdrawn. If the internal formula withdraws less, the swap receives less input than expected and the unwind can revert at `require(returnAmount >= totalDebt)`.

```solidity
// calculateUnwindParams returns collateralToWithdraw with 5% slippage buffer
@>        collateralToWithdraw = (collateralToWithdraw * 1050) / 1000;
@>        return (collateralToWithdraw, debtAmount);
```

```solidity
// unwindPosition stores it but _executeUnwindOperation ignores unwindParams.collateralToWithdraw
            UnwindParams memory params = UnwindParams({
                collateralToken: _collateralToken,
@>              collateralToWithdraw: _collateralToWithdraw,  // never used
                ...
            });
```

```solidity
// _executeUnwindOperation recalculates with a different formula
@>            uint256 collateralToWithdraw = (
                _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
            ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);

            withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
```

## Risk

**Likelihood (high)**:

* Every unwind flow uses this pattern: call `calculateUnwindParams`, build 1inch swap from the result, then call `unwindPosition`. The tests (e.g. `Stratax.t.sol:202-206`) follow this exact flow.

* The two formulas differ: `calculateUnwindParams` uses a pure price-based conversion; `_executeUnwindOperation` uses LTV/liqThreshold. Per finding 006, liqThreshold > LTV causes the internal formula to withdraw less collateral than the price-based formula would suggest.

**Impact (medium)**:

* Swap is built for `collateralToWithdraw` from `calculateUnwindParams`, but actual withdrawal is the recalculated amount. If recalculated amount is less, swap receives less input → may return less than `totalDebt` → unwind reverts.

* User-facing confusion: the displayed/returned collateral amount does not match what the contract actually withdraws.

**Severity (low)**:

## Proof of Concept

```solidity
// 1. User calls calculateUnwindParams(USDC, WETH) → returns collateralToWithdraw = 1500e6 (with 5% buffer)
// 2. User builds 1inch swap for 1500e6 USDC → collateral token
// 3. User calls unwindPosition(USDC, 1500e6, WETH, debtAmount, swapData, minReturn)
// 4. _executeUnwindOperation recalculates with liqThreshold → collateralToWithdraw = 1200e6 (less, due to liqThreshold > LTV)
// 5. aavePool.withdraw returns 1200e6 (not 1500e6)
// 6. Swap receives 1200e6 as input; 1inch swap was built for 1500e6 → routing may be suboptimal or minReturn may fail
// 7. If returnAmount < totalDebt → revert "Insufficient funds to repay flash loan"
```

## Recommended Mitigation

**Option A**: Use `unwindParams.collateralToWithdraw` in `_executeUnwindOperation` instead of recalculating. Align the formula in `calculateUnwindParams` with the unwind logic (use LTV, not liqThreshold) so both produce the same result.

**Option B**: Remove `collateralToWithdraw` from `calculateUnwindParams` return and document that it is a view helper for display/estimation only. Have the frontend build the 1inch swap using the same formula as `_executeUnwindOperation` (or expose an internal view that returns the actual amount that will be withdrawn).

```diff
// Option A: Use the passed value when formulas are aligned
-            uint256 collateralToWithdraw = (
-                _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
-            ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
-
-            withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
+            withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, unwindParams.collateralToWithdraw, address(this));
```

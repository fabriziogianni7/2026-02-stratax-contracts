# Unwind Missing Validation of _asset vs unwindParams.debtToken

## Description

* The flash loan callback receives `_asset` (the flash-loaned token) and decodes `unwindParams` containing `debtToken`.

* The contract never checks that `_asset == unwindParams.debtToken`. If params are mis-encoded or the wrong flash loan is initiated, the logic could use mismatched tokens.

* While `_asset` is used for repay and flash loan repayment (correct), `unwindParams.collateralToken` is used for withdraw and swap. A mismatch could lead to incorrect behavior or confusing failures.

```solidity
        (, address user, UnwindParams memory unwindParams) = abi.decode(_params, (OperationType, address, UnwindParams));
@>      // No validation: _asset == unwindParams.debtToken
        // Step 1: Repay the Aave debt using flash loaned tokens
        IERC20(_asset).approve(address(aavePool), _amount);
```

## Risk

**Likelihood (low)**:

* The contract encodes params when initiating the flash loan; encoding bugs are rare.

* Only the owner can call `unwindPosition`.

**Impact (medium)**:

* Mis-encoded params could cause wrong token flows, failed swaps, or incorrect accounting.

**Severity (low)**:

## Proof of Concept

The mismatch can occur if the owner passes inconsistent arguments to `unwindPosition`, or if a bug in the encoding layer produces params that do not match the flash loan. For example, a script or frontend might swap the order of `_collateralToken` and `_debtToken`, or the `UnwindParams` struct could be built from stale or wrong data.

```solidity
// Scenario: Owner intends to unwind WETH collateral / USDC debt, but accidentally passes
// _debtToken = WETH and _collateralToken = USDC (swapped), or a script bug encodes
// unwindParams.debtToken = USDC while flashLoanSimple is called with WETH.

// unwindPosition(USDC, amount1, WETH, amount2, swapData, minReturn);
// Flash loan: aavePool.flashLoanSimple(..., WETH, amount2, ...)  → _asset = WETH in callback

// But if params were built with debtToken = USDC (bug):
// - _asset = WETH (from Aave callback)
// - unwindParams.debtToken = USDC
// - unwindParams.collateralToken = WETH (or whatever was encoded)

// Execution:
// 1. We repay _asset (WETH) to Aave — correct for the flash loan
// 2. We withdraw unwindParams.collateralToken — if this is WETH, we withdraw WETH
// 3. We swap collateral → _asset; oneInchSwapData was built for USDC→WETH, not WETH→WETH
// 4. Swap fails or returns wrong token; require(returnAmount >= totalDebt) may revert
// 5. Without the check, failures are opaque; with it, we fail fast with a clear error
```

The validation ensures that the token we receive from the flash loan (`_asset`) is the same token we expect to repay and that we encoded in params (`unwindParams.debtToken`). Without it, a mismatch produces confusing reverts (e.g. from the swap or from insufficient return amount) instead of an explicit "Asset mismatch" error at the start of the callback.

## Recommended Mitigation

Add an explicit check immediately after decoding params. This provides defense-in-depth: even though `unwindPosition` uses `_debtToken` for both the flash loan and the params, the callback cannot assume they match—e.g. if the encoding logic changes, or if `executeOperation` is ever invoked through a different path. The check costs minimal gas and yields a clear, early failure with a descriptive message.

```diff
         (, address user, UnwindParams memory unwindParams) = abi.decode(_params, (OperationType, address, UnwindParams));
+        require(_asset == unwindParams.debtToken, "Asset mismatch");
         // Step 1: Repay the Aave debt using flash loaned tokens
```

**Rationale**: `_asset` is the token Aave flash-loaned to the contract; it must be the debt token we intend to repay. `unwindParams.debtToken` is what the owner encoded when calling `unwindPosition`. Requiring them to match ensures we are operating on the intended token pair and prevents silent misuse or confusing downstream failures.

# Wrong Asset Passed to 1inch Swap Fallback in Open Flow Causes DoS

## Description

* In `_executeOpenOperation`, the swap converts borrowed tokens to collateral (e.g., WETH → USDC). The `_call1InchSwap` function expects the second parameter to be the **output** token (the asset being swapped to), used when the 1inch router returns no data to compute `returnAmount` via `balanceOf(_asset)`.

* The open flow incorrectly passes `flashParams.borrowToken` (the input token) instead of `_asset` (the collateral/output token). When the router returns no data (`result.length == 0`), the fallback reads `balanceOf(borrowToken)`, which is ~0 after the swap. This yields `returnAmount ≈ 0` and causes the transaction to revert even though the swap succeeded and the contract received sufficient collateral.

```solidity
        // Step 3: Swap borrowed tokens via 1inch to get back the collateral token
        IERC20(flashParams.borrowToken).approve(address(oneInchRouter), flashParams.borrowAmount);

        // Execute swap via 1inch
@>      uint256 returnAmount =
@>          _call1InchSwap(flashParams.oneInchSwapData, flashParams.borrowToken, flashParams.minReturnAmount);
```

```solidity
        // Decode the return amount from the swap
        if (result.length > 0) {
            (returnAmount,) = abi.decode(result, (uint256, uint256));
        } else {
            // If no return data, check balance
@>          returnAmount = IERC20(_asset).balanceOf(address(this));
        }
```

## Risk

**Likelihood (medium)**:

* The 1inch API returns calldata for multiple router versions and swap paths. Some paths (e.g., simple pool swaps, certain aggregator routes) use functions that do not return `(returnAmount, spentAmount)` and leave `result` empty.

* Router upgrades or API changes can alter which paths return data. Integrators cannot guarantee that all swap routes will return data.

* The unwind flow correctly passes `_asset` (line 586); only the open flow has the bug, indicating an oversight rather than a design choice.

**Impact (medium)**:

* DoS for users attempting to open leveraged positions when the chosen swap path returns no data.

* The revert message ("Insufficient funds to repay flash loan" or "Insufficient return amount from swap") is misleading—the swap succeeded but the fallback used the wrong token.

* No fund loss; the swap executes correctly and tokens are received. The failure occurs in post-swap validation.

**Severity**: Medium

## Proof of Concept

`_call1InchSwap` uses the second parameter only when the 1inch router returns an empty result. In that case it computes `returnAmount = balanceOf(_asset)`. The second parameter must be the **output** token (what we receive), not the **input** token (what we spend). The open flow incorrectly passes `borrowToken` (input).

**Example with numbers:** Flash loan 5,000 USDC (`_asset`), user collateral 1,000 USDC, borrow 2 WETH from Aave (`flashParams.borrowToken`). Swap 2 WETH → ~5,100 USDC succeeds; `totalDebt` (flash loan + premium) ≈ 5,004.5 USDC. Some 1inch paths do not return `(returnAmount, spentAmount)` and leave `result` empty.

**Correct behavior:** With `_asset` passed, the fallback would use `balanceOf(USDC)` (~5,100), returnAmount would be correct, and the tx would succeed.

```solidity
// 1. Flash loan gives _amount of USDC (_asset)
// 2. Supply USDC to Aave, borrow WETH (flashParams.borrowToken)
// 3. Swap WETH → USDC via 1inch; swap succeeds, router returns empty result
// 4. _call1InchSwap fallback: returnAmount = IERC20(borrowToken).balanceOf(this)
//    = IERC20(WETH).balanceOf(this) = 0 (we swapped all WETH away)
// 5. require(returnAmount >= totalDebt) → REVERTS (0 < 5,004.5)
// 6. Transaction reverts despite successful swap; contract actually holds ~5,100 USDC
```

## Recommended Mitigation

Pass the **output token** (the asset being swapped to) as the second argument to `_call1InchSwap`. In the open flow, the swap is `borrowToken → collateral`; the collateral equals `_asset` (the flash loan token). The second parameter is used when the router returns no data to compute `returnAmount` via `balanceOf(_asset)`—it must be the token the contract receives from the swap, not the token it spends.

```diff
        // Step 3: Swap borrowed tokens via 1inch to get back the collateral token
        IERC20(flashParams.borrowToken).approve(address(oneInchRouter), flashParams.borrowAmount);

        // Execute swap via 1inch
        uint256 returnAmount =
-           _call1InchSwap(flashParams.oneInchSwapData, flashParams.borrowToken, flashParams.minReturnAmount);
+           _call1InchSwap(flashParams.oneInchSwapData, _asset, flashParams.minReturnAmount);
```

**Note**: `_asset` and `flashParams.collateralToken` are identical in the open flow (both are the flash loan token). Either is correct; `_asset` is preferred for consistency with the unwind flow (line 586).

**Location**: `src/Stratax.sol:514-516`

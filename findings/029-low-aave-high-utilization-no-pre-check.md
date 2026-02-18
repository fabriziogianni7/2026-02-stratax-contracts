# Aave High Utilization Causes Unwind/Open Failures

## Description

* When Aave pool utilization is high, available liquidity is low. `withdraw()` returns only the amount actually available (which can be less than requested), and `borrow()` can revert or fail when liquidity is insufficient.

* The protocol does not check utilization before `createLeveragedPosition` or `unwindPosition`. In the unwind flow, `aavePool.withdraw()` returns `withdrawnAmount`, which may be less than `collateralToWithdraw` when utilization is high. The swap then receives less input than expected, and `require(returnAmount >= totalDebt)` reverts. The flash loan callback fails; the user cannot unwind.

```solidity
// Stratax.sol - _executeUnwindOperation
@>          withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
        // ...
        uint256 returnAmount = _call1InchSwap(unwindParams.oneInchSwapData, _asset, unwindParams.minReturnAmount);
        require(returnAmount >= totalDebt, "Insufficient funds to repay flash loan");
```

## Risk

**Likelihood (low)**:

* High utilization occurs during stress (e.g., mass liquidations, large withdrawals, or concentrated borrowing). Mainnet Aave pools for USDC/WETH typically have deep liquidity.

**Impact (medium)**:

* Users cannot open or unwind positions. Unwind reverts inside the flash loan callback, leaving positions stuck until utilization drops. Open flow can revert at `borrow()` or `supply()`.

**Severity (low)**:

## Proof of Concept

* Aave USDC reserve reaches 99% utilization. User calls `unwindPosition` to unwind a USDC-collateralized position. `aavePool.withdraw(USDC, collateralToWithdraw, ...)` returns only 10% of the requested amount due to lack of liquidity. The 1inch swap receives far less USDC than needed to repay the flash loan. `require(returnAmount >= totalDebt)` reverts. The entire transaction fails; the user cannot unwind.

```solidity
// High utilization: withdraw returns less than requested
uint256 requested = 1_000_000e6;
uint256 withdrawn = aavePool.withdraw(USDC, requested, address(this)); // returns 50_000e6
// Swap receives 50k USDC, needs 1M+ to repay flash loan -> revert
```

## Recommended Mitigation

* Use `aaveDataProvider` (IProtocolDataProvider) before initiating operations. Obtain reserve token addresses via `getReserveTokensAddresses(asset)`, then compute utilization from aToken and debt token `totalSupply()`:

```diff
  // In createLeveragedPosition, before flashLoanSimple:
+ (address aToken, address stableDebtToken, address variableDebtToken) =
+     aaveDataProvider.getReserveTokensAddresses(_flashLoanToken);
+ uint256 totalLiquidity = IERC20(aToken).totalSupply();
+ uint256 totalDebt = IERC20(stableDebtToken).totalSupply() + IERC20(variableDebtToken).totalSupply();
+ uint256 utilization = (totalDebt * 10000) / (totalLiquidity + totalDebt);
+ require(utilization <= 9500, "Reserve utilization too high"); // e.g. 95% max

  // Same check for borrow token before flash loan
+ (aToken, stableDebtToken, variableDebtToken) = aaveDataProvider.getReserveTokensAddresses(_borrowToken);
+ totalLiquidity = IERC20(aToken).totalSupply();
+ totalDebt = IERC20(stableDebtToken).totalSupply() + IERC20(variableDebtToken).totalSupply();
+ utilization = (totalDebt * 10000) / (totalLiquidity + totalDebt);
+ require(utilization <= 9500, "Borrow reserve utilization too high");

  aavePool.flashLoanSimple(address(this), _flashLoanToken, _flashLoanAmount, encodedParams, 0);
```

* For `unwindPosition`, add the same utilization check for both collateral and debt tokens before initiating the flash loan. Use `aaveDataProvider.getReserveTokensAddresses` as above.

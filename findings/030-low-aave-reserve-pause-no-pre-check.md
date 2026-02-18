# Aave Reserve Pause Causes DoS for Open and Unwind

## Description

* Aave reserves can be paused or frozen. When `isFrozen` is true, new supply, borrow, withdraw, and repay operations are disabled. When `isActive` is false, the reserve is disabled entirely.

* The protocol does not check reserve state before `createLeveragedPosition` or `unwindPosition`. If Aave freezes or deactivates a reserve, the flash loan callback's `supply`, `borrow`, `withdraw`, or `repay` calls revert. The entire transaction fails; users cannot open or unwind positions.

```solidity
// Stratax.sol - _executeOpenOperation
@>        aavePool.supply(_asset, totalCollateral, address(this), 0);
        // ...
@>        aavePool.borrow(flashParams.borrowToken, flashParams.borrowAmount, 2, 0, address(this));

// Stratax.sol - _executeUnwindOperation
@>        aavePool.repay(_asset, _amount, 2, address(this));
@>        withdrawnAmount = aavePool.withdraw(unwindParams.collateralToken, collateralToWithdraw, address(this));
```

## Risk

**Likelihood (low)**:

* Aave pauses reserves during emergencies (e.g., exploit, oracle manipulation, or governance decision). This is rare but possible.

**Impact (medium)**:

* Users cannot open or unwind positions. Transactions revert inside the flash loan callback. Positions may be stuck until the reserve is unfrozen. Similar to oracle revert DoS (finding 010).

**Severity (low)**:

## Proof of Concept

* Aave freezes the WETH reserve due to a suspected oracle issue. A user attempts to unwind a WETH-collateralized position. `unwindPosition` initiates a flash loan of the debt token. In the callback, `aavePool.withdraw(WETH, ...)` reverts because the reserve is frozen. The flash loan callback fails; the transaction reverts. The user cannot unwind.

```solidity
// Reserve frozen
aavePool.withdraw(WETH, amount, address(this)); // REVERT: reserve frozen
```

## Recommended Mitigation

* Use `aaveDataProvider.getReserveConfigurationData` (IProtocolDataProvider) before initiating operations. The function returns `isActive` and `isFrozen`. Revert if the reserve is not available:

```diff
  // In createLeveragedPosition, before flashLoanSimple:
+ (, , , , , , , , bool collateralActive, bool collateralFrozen) =
+     aaveDataProvider.getReserveConfigurationData(_flashLoanToken);
+ require(collateralActive && !collateralFrozen, "Collateral reserve not available");
+ (, , , , , , , , bool borrowActive, bool borrowFrozen) =
+     aaveDataProvider.getReserveConfigurationData(_borrowToken);
+ require(borrowActive && !borrowFrozen, "Borrow reserve not available");

  aavePool.flashLoanSimple(address(this), _flashLoanToken, _flashLoanAmount, encodedParams, 0);
```

* For `unwindPosition`, add the same checks for `_collateralToken` and `_debtToken` before `aavePool.flashLoanSimple`. Use `aaveDataProvider.getReserveConfigurationData` for both assets.

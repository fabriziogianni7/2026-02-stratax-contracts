# Aave Isolated Asset Max Debt Can Cause Borrow Revert

## Description

* Aave V3 reserves can have a borrow cap. When `borrowCap > 0`, total borrows cannot exceed the cap. Isolated assets also have a debt ceiling. When the cap or ceiling is reached, `borrow()` reverts.

* The protocol does not check borrow capacity before `createLeveragedPosition`. It calls `aavePool.borrow()` without verifying that the reserve has room for the requested amount. If the borrow cap is reached, the call reverts inside the flash loan callback; the entire open operation fails.

```solidity
// Stratax.sol - _executeOpenOperation
@>        aavePool.borrow(
            flashParams.borrowToken,
            flashParams.borrowAmount,
            2, // Variable interest rate mode
            0,
            address(this)
        );
```

## Risk

**Likelihood (low)**:

* Main assets (USDC, WETH) on Aave mainnet typically have no borrow cap or a very high cap. Isolated or niche assets are more likely to hit the cap.

**Impact (medium)**:

* `createLeveragedPosition` reverts inside the flash loan callback. The user's collateral was already transferred to the contract; the flash loan was drawn. The revert causes the entire transaction to fail. No funds are lost, but the user cannot open the position and must retry when capacity is available.

**Severity (low)**:

## Proof of Concept

* An isolated asset has borrowCap = 1,000,000. Current total borrows = 999,000. User attempts to open a position with borrowAmount = 50,000. `aavePool.borrow(asset, 50000, ...)` reverts because 999,000 + 50,000 > 1,000,000. The flash loan callback reverts; the open fails.

```solidity
// borrowCap reached
aavePool.borrow(asset, 50000, 2, 0, address(this)); // REVERT: borrow cap exceeded
```

## Recommended Mitigation

* Use `aaveDataProvider` (IProtocolDataProvider) to check borrow capacity before initiating the flash loan. The Aave ProtocolDataProvider contract implements `getReserveCaps(asset)` returning `(borrowCap, supplyCap)`. When `borrowCap > 0`, ensure `totalDebt + borrowAmount <= borrowCap`. Obtain current debt via `getReserveTokensAddresses` and the variable (and stable) debt token `totalSupply()`:

```diff
  // In createLeveragedPosition, before flashLoanSimple:
  // Add getReserveCaps to IProtocolDataProvider to match Aave's ProtocolDataProvider
+ (uint256 borrowCap,) = aaveDataProvider.getReserveCaps(_borrowToken);
+ if (borrowCap > 0) {
+     (,, address variableDebtToken) = aaveDataProvider.getReserveTokensAddresses(_borrowToken);
+     uint256 currentDebt = IERC20(variableDebtToken).totalSupply();
+     require(currentDebt + _borrowAmount <= borrowCap, "Borrow cap exceeded");
+ }

  aavePool.flashLoanSimple(address(this), _flashLoanToken, _flashLoanAmount, encodedParams, 0);
```

* Add `getReserveCaps(address asset) external view returns (uint256 borrowCap, uint256 supplyCap)` to `IProtocolDataProvider` so the call compiles. The deployed Aave ProtocolDataProvider already implements this function.

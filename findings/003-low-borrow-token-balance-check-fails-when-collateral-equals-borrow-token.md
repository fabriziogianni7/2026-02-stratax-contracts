# Incorrect Borrow Token Balance Check When Collateral Equals Borrow Token

## Description

* The contract verifies that all borrowed tokens are used in the swap by comparing the borrow token balance before the borrow to the balance after the swap. The invariant is: `afterSwapBorrowTokenbalance == prevBorrowTokenBalance`, which assumes the contract holds no borrow tokens before the borrow.

* When the flash loan asset (collateral token) equals the borrow token, the contract already holds borrow tokens before the borrow—from the flash loan and user collateral. After the swap (borrow token → collateral, same token), the contract receives borrow tokens back. The final balance will exceed `prevBorrowTokenBalance`, causing the check to revert even when the operation is valid.

```solidity
        // Store initial balance to verify all borrowed tokens are used in swap
@>      uint256 prevBorrowTokenBalance = IERC20(flashParams.borrowToken).balanceOf(address(this));
        // Step 2: Borrow against the supplied collateral
        aavePool.borrow(
            flashParams.borrowToken,
            flashParams.borrowAmount,
            2, // Variable interest rate mode
            0,
            address(this)
        );

        // Step 3: Swap borrowed tokens via 1inch to get back the collateral token
        IERC20(flashParams.borrowToken).approve(address(oneInchRouter), flashParams.borrowAmount);

        // Execute swap via 1inch
        uint256 returnAmount =
            _call1InchSwap(flashParams.oneInchSwapData, flashParams.borrowToken, flashParams.minReturnAmount);

        // Ensure all borrowed tokens were used in the swap
@>      uint256 afterSwapBorrowTokenbalance = IERC20(flashParams.borrowToken).balanceOf(address(this));
@>      require(afterSwapBorrowTokenbalance == prevBorrowTokenBalance, "Borrow token left in contract");
```

## Risk

**Likelihood (low)**:

* `createLeveragedPosition` accepts `_flashLoanToken` and `_borrowToken` as separate parameters with no validation that they differ.

* An owner or integrator can call with `_flashLoanToken == _borrowToken` (e.g., flash loan USDC and borrow USDC against USDC collateral).

* The protocol may expand to support same-token leverage strategies where collateral and borrow token are identical.

**Impact (low)**:

* Valid operations revert with "Borrow token left in contract" when collateral token equals borrow token.

* DoS for users attempting same-token leverage positions.

* Confusing failure mode: the revert suggests leftover tokens when the real cause is the check's incorrect assumption.

## Proof of Concept

```solidity
// Scenario: _flashLoanToken == _borrowToken (e.g., both USDC)

// 1. Flash loan gives _amount of USDC
// 2. User transfers collateralAmount of USDC
// 3. prevBorrowTokenBalance = _amount + collateralAmount (we already hold USDC = borrow token)
// 4. Borrow borrowAmount of USDC → balance = prevBorrowTokenBalance + borrowAmount
// 5. Swap borrowAmount USDC for USDC (same token) → receive ~borrowAmount USDC back
// 6. afterSwapBorrowTokenbalance = prevBorrowTokenBalance + returnAmount
// 7. require(afterSwapBorrowTokenbalance == prevBorrowTokenBalance) → REVERTS
```

## Recommended Mitigation

**Option A**: Explicitly disallow collateral == borrow token at entry point:

```diff
    function createLeveragedPosition(
        address _flashLoanToken,
        ...
    ) public onlyOwner {
        require(_collateralAmount > 0, "Collateral Cannot be Zero");
+       require(_flashLoanToken != _borrowToken, "Collateral and borrow token must differ");
        // Transfer the user's collateral to the contract
```

**Option B**: Make the balance check conditional when tokens differ, or use a different invariant (e.g., require leftover borrow tokens ≤ prevBorrowTokenBalance when same-token flows are supported):

```diff
        // Ensure all borrowed tokens were used in the swap
        uint256 afterSwapBorrowTokenbalance = IERC20(flashParams.borrowToken).balanceOf(address(this));
-       require(afterSwapBorrowTokenbalance == prevBorrowTokenBalance, "Borrow token left in contract");
+       if (_asset != flashParams.borrowToken) {
+           require(afterSwapBorrowTokenbalance == prevBorrowTokenBalance, "Borrow token left in contract");
+       } else {
+           require(afterSwapBorrowTokenbalance >= prevBorrowTokenBalance, "Insufficient borrow token after swap");
+       }
```

**Severity**: Low

**Location**: `src/Stratax.sol:499-519`

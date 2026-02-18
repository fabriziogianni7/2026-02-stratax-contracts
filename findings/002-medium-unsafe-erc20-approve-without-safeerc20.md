# Unsafe ERC20 approve() Without SafeERC20

## Description

The contract uses raw `IERC20.approve()` in 8 locations instead of OpenZeppelin's `SafeERC20.forceApprove()`. The standard ERC20 `approve()` has inconsistent behavior across token implementations.

Some tokens (e.g., USDT) do not return a boolean on success—they revert on failure or return nothing. Others return `false` on failure instead of reverting. Raw `approve()` does not handle these cases: it may succeed silently when approval failed, or revert unexpectedly with non-standard tokens.

```solidity
// _executeOpenOperation - Line 495
IERC20(_asset).approve(address(aavePool), totalCollateral);

// _executeOpenOperation - Line 510
IERC20(flashParams.borrowToken).approve(address(oneInchRouter), flashParams.borrowAmount);

// _executeOpenOperation - Lines 530, 534
IERC20(_asset).approve(address(aavePool), returnAmount - totalDebt);
IERC20(_asset).approve(address(aavePool), totalDebt);

// _executeUnwindOperation - Line 559
IERC20(_asset).approve(address(aavePool), _amount);

// _executeUnwindOperation - Line 583
IERC20(unwindParams.collateralToken).approve(address(oneInchRouter), withdrawnAmount);

// _executeUnwindOperation - Lines 593, 597
IERC20(_asset).approve(address(aavePool), returnAmount - totalDebt);
IERC20(_asset).approve(address(aavePool), totalDebt);
```

## Risk

**Likelihood**:

* Aave and 1inch integrations typically use standard ERC20 tokens (USDC, WETH, etc.) that behave correctly with raw `approve()`.
* If the protocol expands to support non-standard tokens (e.g., USDT) or tokens that return `false` on failure, approvals may fail silently or revert unpredictably.
* The issue affects every flash loan operation (open and unwind) that involves token approvals.

**Impact**:

* Silent approval failures: transactions may succeed while approval did not, leading to downstream reverts in `supply()`, `repay()`, or swap calls with confusing error messages.
* Incompatibility with USDT and similar tokens that require allowance to be set to 0 before setting a new value.
* Inconsistent behavior across different token implementations, reducing protocol robustness.

## Proof of Concept

```solidity
// USDT on mainnet: approve() returns void. Some wrappers expect a bool.
// A token that returns false on failure would not revert; the next call (supply/repay/swap) would fail instead.

// Example: User opens position with USDT as collateral. approve() succeeds (no return check).
// supply() is called. If the token has non-standard behavior, the flow may revert late with an opaque error.
```

## Recommended Mitigation

Use OpenZeppelin's `SafeERC20.forceApprove()` for all approval calls. In OpenZeppelin v5.x, `safeApprove` was deprecated; `forceApprove` handles tokens that return false, return nothing, or require approve(0) before approve(value).

```diff
+ import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

- IERC20(_asset).approve(address(aavePool), totalCollateral);
+ SafeERC20.forceApprove(IERC20(_asset), address(aavePool), totalCollateral);

- IERC20(flashParams.borrowToken).approve(address(oneInchRouter), flashParams.borrowAmount);
+ SafeERC20.forceApprove(IERC20(flashParams.borrowToken), address(oneInchRouter), flashParams.borrowAmount);

- IERC20(_asset).approve(address(aavePool), returnAmount - totalDebt);
+ SafeERC20.forceApprove(IERC20(_asset), address(aavePool), returnAmount - totalDebt);

- IERC20(_asset).approve(address(aavePool), totalDebt);
+ SafeERC20.forceApprove(IERC20(_asset), address(aavePool), totalDebt);

- IERC20(_asset).approve(address(aavePool), _amount);
+ SafeERC20.forceApprove(IERC20(_asset), address(aavePool), _amount);

- IERC20(unwindParams.collateralToken).approve(address(oneInchRouter), withdrawnAmount);
+ SafeERC20.forceApprove(IERC20(unwindParams.collateralToken), address(oneInchRouter), withdrawnAmount);
```

Apply the same pattern for all 8 approval locations listed above.

**Severity**: Low–Medium (depends on supported tokens; higher if non-standard tokens are in scope)

**Locations**: `src/Stratax.sol` lines 495, 510, 530, 534, 559, 583, 593, 597

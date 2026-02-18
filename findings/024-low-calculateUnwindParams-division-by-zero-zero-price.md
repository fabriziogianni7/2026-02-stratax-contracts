# calculateUnwindParams Division by Zero When collateralTokenPrice Is Zero

## Description

* `calculateUnwindParams` computes the collateral to withdraw using oracle prices. The formula divides by `collateralTokenPrice * 10 ** borrowDecimals`. There is no validation that `collateralTokenPrice > 0`.

* When the oracle returns 0 for `collateralTokenPrice` (e.g. deprecated feed, misconfigured token, or Chainlink returning 0 for an invalid state), the division reverts with a division-by-zero error. The function becomes unusable and callers cannot obtain unwind params.

```solidity
        uint256 debtTokenPrice = IStrataxOracle(strataxOracle).getPrice(_borrowToken);
        uint256 collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(_collateralToken);

@>        collateralToWithdraw = (debtTokenPrice * debtAmount * 10 ** IERC20(_collateralToken).decimals())
@>            / (collateralTokenPrice * 10 ** IERC20(_borrowToken).decimals());
```

## Risk

**Likelihood (low)**:

* Chainlink feeds typically return non-zero prices for active assets. A zero price can occur when a feed is deprecated, a token is newly added without proper config, or the oracle contract has a bug.

* The open flow and `_executeUnwindOperation` both guard with `require(collateralTokenPrice > 0, "Invalid prices")`. `calculateUnwindParams` is the only place that uses the price without this check.

**Impact (medium)**:

* Division by zero reverts the entire call. Users cannot obtain unwind params, blocking the unwind flow before it even starts.

* Unlike `_executeUnwindOperation`, the revert occurs before any flash loan is taken, so no funds are at risk. The impact is limited to DoS of the helper function.

**Severity (low)**:

## Proof of Concept

```solidity
// Scenario: Oracle returns 0 for collateral token (e.g. USDC feed deprecated or misconfigured)

// 1. User calls calculateUnwindParams(USDC, WETH)
// 2. getPrice(USDC) returns 0
// 3. collateralTokenPrice = 0
// 4. Denominator = 0 * 10^18 = 0
// 5. Division by zero → Solidity reverts
// 6. User cannot get unwind params; unwind flow is blocked
```

## Recommended Mitigation

Add an explicit guard before the division, matching `_executeUnwindOperation`:

```diff
        uint256 debtTokenPrice = IStrataxOracle(strataxOracle).getPrice(_borrowToken);
        uint256 collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(_collateralToken);

+        require(debtTokenPrice > 0 && collateralTokenPrice > 0, "Invalid prices");

        collateralToWithdraw = (debtTokenPrice * debtAmount * 10 ** IERC20(_collateralToken).decimals())
            / (collateralTokenPrice * 10 ** IERC20(_borrowToken).decimals());
```

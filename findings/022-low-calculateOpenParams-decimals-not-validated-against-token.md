# calculateOpenParams Trusts Caller-Provided Decimals Without Validation

## Description

* `calculateOpenParams` uses `collateralTokenDec` and `borrowTokenDec` from `TradeDetails` in all value conversions: `totalCollateralValueUSD`, `borrowAmount`, and `borrowValueInCollateral`. These decimals drive the entire leverage and borrow calculation.

* The caller provides decimals directly; the function does not validate them against the actual token decimals from `IERC20(token).decimals()`. A wrong value (e.g. 18 for USDC which has 6 decimals) produces incorrect `flashLoanAmount`, `borrowAmount`, and fails the `borrowValueInCollateral >= minRequiredAfterSwap` check or creates suboptimal positions.

```solidity
        // Calculate total collateral value in USD (with proper decimal handling)
        // totalCollateralValueUSD = (totalCollateral * collateralPrice) / (10^collateralDec)
@>      uint256 totalCollateralValueUSD =
            (totalCollateral * details.collateralTokenPrice) / (10 ** details.collateralTokenDec);
        ...
        // borrowAmount = (borrowValueUSD * 10^borrowTokenDec) / borrowTokenPrice
@>      borrowAmount = (borrowValueUSD * (10 ** details.borrowTokenDec)) / details.borrowTokenPrice;
        ...
@>      uint256 borrowValueInCollateral = (borrowAmount * details.borrowTokenPrice * (10 ** details.collateralTokenDec))
            / (details.collateralTokenPrice * (10 ** details.borrowTokenDec));
```

## Risk

**Likelihood (low)**:

* Frontends typically read decimals from the token contract; manual or cached values can be wrong.

* Integrators or scripts may hardcode decimals incorrectly for new tokens.

**Impact (low)**:

* Wrong decimals → incorrect `borrowAmount` and `flashLoanAmount` → failed `createLeveragedPosition` (revert at health factor or swap check) or suboptimal under-leveraged positions.

* Owner leaves value on the table or wastes gas on reverting transactions.

**Severity (low)**:

## Proof of Concept

```solidity
// Caller passes wrong decimals: USDC has 6, but passes 18
TradeDetails memory details = TradeDetails({
    collateralToken: address(USDC),
    borrowToken: address(WETH),
    desiredLeverage: 30000,
    collateralAmount: 1000e6,
    collateralTokenPrice: 2e8,
    borrowTokenPrice: 2000e8,
    collateralTokenDec: 18,  // Wrong: USDC has 6
    borrowTokenDec: 18
});

(uint256 flashLoanAmount, uint256 borrowAmount) = stratax.calculateOpenParams(details);
// totalCollateralValueUSD is 12x smaller (divided by 10^18 instead of 10^6)
// borrowAmount and borrowValueInCollateral are wrong
// May revert "Insufficient borrow to repay flash loan" or produce unusable params
```

## Recommended Mitigation

Fetch decimals from the token contracts instead of trusting caller input:

```diff
        require(details.desiredLeverage >= LEVERAGE_PRECISION, "Leverage must be >= 1x");
        require(details.collateralAmount > 0, "Collateral must be > 0");

+       uint256 collateralTokenDec = IERC20(details.collateralToken).decimals();
+       uint256 borrowTokenDec = IERC20(details.borrowToken).decimals();

        // If collateral token price is zero, fetch it from the oracle
        ...
-       uint256 totalCollateralValueUSD =
-           (totalCollateral * details.collateralTokenPrice) / (10 ** details.collateralTokenDec);
+       uint256 totalCollateralValueUSD =
+           (totalCollateral * details.collateralTokenPrice) / (10 ** collateralTokenDec);
        ...
-       borrowAmount = (borrowValueUSD * (10 ** details.borrowTokenDec)) / details.borrowTokenPrice;
+       borrowAmount = (borrowValueUSD * (10 ** borrowTokenDec)) / details.borrowTokenPrice;
        ...
-       uint256 borrowValueInCollateral = (borrowAmount * details.borrowTokenPrice * (10 ** details.collateralTokenDec))
-           / (details.collateralTokenPrice * (10 ** details.borrowTokenDec));
+       uint256 borrowValueInCollateral = (borrowAmount * details.borrowTokenPrice * (10 ** collateralTokenDec))
+           / (details.collateralTokenPrice * (10 ** borrowTokenDec));
```

* Optionally remove `collateralTokenDec` and `borrowTokenDec` from `TradeDetails` for this function, or document that they are ignored when fetched on-chain.

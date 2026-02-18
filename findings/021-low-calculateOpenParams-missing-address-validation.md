# calculateOpenParams Missing Address Validation for Token Parameters

## Description

* `calculateOpenParams` accepts `TradeDetails` with `collateralToken` and `borrowToken` addresses and uses them to fetch LTV from Aave and prices from the oracle.

* There is no validation that `collateralToken` or `borrowToken` are non-zero addresses. Passing `address(0)` leads to unclear revert behavior: `getReserveConfigurationData(address(0))` may return zeros (ltv = 0) causing a generic "Asset not usable as collateral" revert, and `getPrice(address(0))` may revert or return invalid data depending on the oracle implementation.

```solidity
        // Get LTV from Aave for the collateral token
@>      (, uint256 ltv,,,,,,,,) = aaveDataProvider.getReserveConfigurationData(details.collateralToken);
        require(ltv > 0, "Asset not usable as collateral");
        ...
@>      details.collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.collateralToken);
        ...
@>      details.borrowTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.borrowToken);
```

## Risk

**Likelihood (low)**:

* Frontends or integrators may pass uninitialized or default struct fields when constructing `TradeDetails`.

* Buggy off-chain logic or copy-paste errors can pass `address(0)` for token parameters.

**Impact (low)**:

* Unclear revert messages make debugging difficult; users receive "Asset not usable as collateral" instead of "Invalid token address".

* Oracle may revert on `getPrice(address(0))`, causing DoS for callers who pass invalid addresses.

**Severity (low)**:

## Proof of Concept

```solidity
TradeDetails memory details = TradeDetails({
    collateralToken: address(0),  // Invalid
    borrowToken: address(WETH),
    desiredLeverage: 30000,
    collateralAmount: 1000e6,
    collateralTokenPrice: 0,
    borrowTokenPrice: 0,
    collateralTokenDec: 6,
    borrowTokenDec: 18
});

// Reverts with "Asset not usable as collateral" (misleading) or oracle revert
(uint256 flashLoanAmount, uint256 borrowAmount) = stratax.calculateOpenParams(details);
```

## Recommended Mitigation

```diff
    {
        // Get LTV from Aave for the collateral token
+       require(details.collateralToken != address(0) && details.borrowToken != address(0), "Invalid token address");
        (, uint256 ltv,,,,,,,,) = aaveDataProvider.getReserveConfigurationData(details.collateralToken);
        require(ltv > 0, "Asset not usable as collateral");
```

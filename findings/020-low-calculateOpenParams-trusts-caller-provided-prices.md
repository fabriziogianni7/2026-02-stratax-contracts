# calculateOpenParams Trusts Caller-Provided Prices for Leverage Calculation

## Description

* `calculateOpenParams` is a view function that returns `flashLoanAmount` and `borrowAmount` for opening a leveraged position. These values drive the entire leverage calculation and are used downstream by `createLeveragedPosition`.

* When `collateralTokenPrice` or `borrowTokenPrice` are non-zero, the function uses the caller-provided values directly instead of fetching from the oracle. This violates the principle that critical price-dependent calculations should use the oracle as the single source of truth.

```solidity
        // If collateral token price is zero, fetch it from the oracle
@>      if (details.collateralTokenPrice == 0) {
            require(strataxOracle != address(0), "Oracle not set");
            // hack 010 no try/catch; oracle revert DoS
            details.collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.collateralToken);
        }
        require(details.collateralTokenPrice > 0, "Collateral token price must be > 0");

        // If borrow token price is zero, fetch it from the oracle
@>      if (details.borrowTokenPrice == 0) {
            require(strataxOracle != address(0), "Oracle not set");
            details.borrowTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.borrowToken);
        }
```

* The same prices are used in `totalCollateralValueUSD`, `borrowValueUSD`, `borrowAmount`, and `borrowValueInCollateral` (lines 422–441). In contrast, `calculateUnwindParams` and `_executeUnwindOperation` always fetch prices from the oracle.

## Risk

**Likelihood (medium)**:

* Frontends or scripts commonly pass cached or off-chain prices for gas optimization, assuming zero triggers oracle fetch.

* A malicious or buggy caller can pass arbitrary non-zero prices; no validation ensures they match the oracle.

**Impact (low)**:

* Inflated prices → larger `borrowAmount` → over-leverage attempt → `require(healthFactor > 1e18)` in `_executeOpenOperation` reverts; no position created, gas wasted.

* Deflated prices → smaller `borrowAmount` → suboptimal under-leveraged position; owner leaves value on the table.

* Stale prices → wrong simulation results; users may receive incorrect `flashLoanAmount`/`borrowAmount` and experience failed transactions or suboptimal positions.

* Impact is limited because `createLeveragedPosition` is `onlyOwner`; over-leverage is rejected by Aave's health factor check.

**Severity (low)**:

## Proof of Concept

* Caller passes an inflated `collateralTokenPrice` (e.g., 2x real price) to get a larger `borrowAmount`. They then call `createLeveragedPosition` with the returned amounts. The position would be over-leveraged; Aave's health factor (using real oracle prices) would be below 1, and the transaction reverts at the health check. No funds lost, but the design trusts untrusted input for critical calculations.

```solidity
// Caller passes fake collateral price (2x real)
TradeDetails memory details = TradeDetails({
    collateralToken: USDC,
    borrowToken: WETH,
    desiredLeverage: 30000, // 3x
    collateralAmount: 1000e6,
    collateralTokenPrice: 4e8,  // Fake: $4 instead of $2
    borrowTokenPrice: 0,       // Fetched from oracle
    collateralTokenDec: 6,
    borrowTokenDec: 18
});

(uint256 flashLoanAmount, uint256 borrowAmount) = stratax.calculateOpenParams(details);
// borrowAmount is inflated due to fake collateralTokenPrice

// Owner calls createLeveragedPosition with these amounts
// _executeOpenOperation: healthFactor < 1e18 → revert
```

## Recommended Mitigation

* Always fetch prices from the oracle in `calculateOpenParams`; do not trust caller-provided values. This aligns with `calculateUnwindParams` and `_executeUnwindOperation`.

```diff
        require(details.desiredLeverage >= LEVERAGE_PRECISION, "Leverage must be >= 1x");
        require(details.collateralAmount > 0, "Collateral must be > 0");

-       // If collateral token price is zero, fetch it from the oracle
-       if (details.collateralTokenPrice == 0) {
-           require(strataxOracle != address(0), "Oracle not set");
-           details.collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.collateralToken);
-       }
+       require(strataxOracle != address(0), "Oracle not set");
+       details.collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.collateralToken);
        require(details.collateralTokenPrice > 0, "Collateral token price must be > 0");

-       // If borrow token price is zero, fetch it from the oracle
-       if (details.borrowTokenPrice == 0) {
-           require(strataxOracle != address(0), "Oracle not set");
-           details.borrowTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.borrowToken);
-       }
+       details.borrowTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.borrowToken);
        require(details.borrowTokenPrice > 0, "Borrow token price must be > 0");
```

* Optionally remove `collateralTokenPrice` and `borrowTokenPrice` from the `TradeDetails` struct for this function, or document that they are ignored.

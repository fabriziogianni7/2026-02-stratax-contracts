# Oracle Reverts Cause Denial of Service

## Description

* The StrataxOracle `getPrice` function performs a direct call to the Chainlink price feed. If the price feed reverts (e.g., deprecated feed, contract upgrade, or unexpected failure), the entire call reverts. Stratax calls `getPrice` in `calculateOpenParams`, `calculateUnwindParams`, and `_executeUnwindOperation` without try/catch or fallback.

* There is no mechanism to handle oracle failures. A single oracle revert cascades and reverts the entire transaction, causing DoS for open, unwind, and parameter calculation flows.

```solidity
// StrataxOracle.sol - no try/catch
@>      (, int256 answer,,,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
```

```solidity
// Stratax.sol - direct calls, no fallback
@>      details.collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.collateralToken);
@>      details.borrowTokenPrice = IStrataxOracle(strataxOracle).getPrice(details.borrowToken);
@>      uint256 debtTokenPrice = IStrataxOracle(strataxOracle).getPrice(_borrowToken);
@>      uint256 collateralTokenPrice = IStrataxOracle(strataxOracle).getPrice(_collateralToken);
```

## Risk

**Likelihood (low)**:

* Chainlink feeds are generally reliable. Reverts can occur when feeds are deprecated, during network issues, or if the feed contract is upgraded incorrectly.

**Impact (high)**:

* Users cannot open or unwind positions. `calculateOpenParams` and `calculateUnwindParams` become unusable. Positions may be stuck if the oracle fails during an active unwind flow (e.g., inside `_executeUnwindOperation` after the flash loan has been taken).

**Severity (medium)**:

## Proof of Concept

* Chainlink deprecates the WBTC/USD feed and the new feed address is not yet configured. All calls to `getPrice(WBTC)` revert. Users cannot open WBTC-collateralized positions or unwind existing ones. The protocol is effectively DoS'd for that asset.

* If the revert happens inside `_executeUnwindOperation` after repaying Aave debt but before completing the swap, the flash loan callback reverts. The flash loan is not repaid; the operation fails. The user's unwind is blocked.

```solidity
// Oracle reverts (e.g., feed deprecated)
uint256 price = IStrataxOracle(strataxOracle).getPrice(token); // REVERT
// Entire tx reverts; no fallback path
```

## Recommended Mitigation

* In StrataxOracle: wrap the Chainlink call in try/catch. On failure, revert with a clear error or return a sentinel value that callers can handle.

* In Stratax: consider try/catch around oracle calls with a fallback (e.g., cached price, secondary oracle, or explicit revert with user-friendly message). For critical paths (e.g., inside flash loan callback), ensure the failure mode is predictable and does not leave the protocol in an inconsistent state.

```diff
  // StrataxOracle.sol
  function getPrice(address _token) public view returns (uint256 price) {
      ...
-     (, int256 answer,,,) = priceFeed.latestRoundData();
-     require(answer > 0, "Invalid price from oracle");
-     price = uint256(answer);
+     try priceFeed.latestRoundData() returns (uint80, int256 answer, uint256, uint256, uint80) {
+         require(answer > 0, "Invalid price from oracle");
+         price = uint256(answer);
+     } catch {
+         revert("Oracle call failed");
+     }
  }
```

* Alternatively, return `(uint256 price, bool success)` and let callers decide how to handle failure.

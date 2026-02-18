# calculateUnwindParams Uses Hardcoded 5% Slippage Buffer

## Description

* `calculateUnwindParams` returns the collateral amount to withdraw when unwinding a position. The function applies a fixed 5% buffer to account for swap slippage before returning `collateralToWithdraw`.

* The slippage buffer is hardcoded and not configurable. In volatile markets, 5% may be insufficient and unwinds can revert or fail. In stable conditions, 5% may be excessive and withdraw more collateral than needed, leaving dust or affecting position health.

```solidity
        collateralToWithdraw = (debtTokenPrice * debtAmount * 10 ** IERC20(_collateralToken).decimals())
            / (collateralTokenPrice * 10 ** IERC20(_borrowToken).decimals());

@>      // Account for 5% slippage in swap
@>      collateralToWithdraw = (collateralToWithdraw * 1050) / 1000;

        return (collateralToWithdraw, debtAmount);
```

## Risk

**Likelihood (low)**:

* Most unwinds use stable pairs (e.g., USDC/WETH) where 5% is often adequate.

* Volatile markets or illiquid pairs can experience >5% slippage; the buffer becomes insufficient and the unwind may revert at `require(returnAmount >= totalDebt)`.

**Impact (low)**:

* Too tight: unwinds fail in volatile conditions; owner must retry or wait for calmer markets.

* Too loose: excess collateral withdrawn; dust left in Aave or suboptimal health factor adjustment.

**Severity (low)**:

## Proof of Concept

* Owner calls `calculateUnwindParams(USDC, WETH)` to get `collateralToWithdraw` and `debtAmount`. The function returns `collateralToWithdraw` with a 5% buffer (e.g., 1050 instead of 1000).

* Owner builds 1inch swap for `collateralToWithdraw` and calls `unwindPosition`. During a volatile period, the swap returns 4% less than expected due to price movement. The 5% buffer is barely sufficient; in worse conditions (e.g., 6% slippage), the unwind reverts.

* Conversely, in stable conditions the swap may return 1% less than nominal. The 5% buffer is excessive; the protocol withdraws more collateral than needed.

```solidity
// Volatile: collateralToWithdraw = 1050 (5% buffer)
// Swap returns 995 (5.2% slippage) < totalDebt 1000 → revert

// Stable: collateralToWithdraw = 1050 (5% buffer)
// Swap returns 1045 (0.5% slippage) → 45 units excess withdrawn
```

## Recommended Mitigation

Make the slippage buffer configurable. Option A: add a parameter to `calculateUnwindParams`. Option B: add an owner-set state variable (e.g., `unwindSlippageBps`).

```diff
    function calculateUnwindParams(address _collateralToken, address _borrowToken)
        public
        view
        returns (uint256 collateralToWithdraw, uint256 debtAmount)
    {
        ...
        collateralToWithdraw = (debtTokenPrice * debtAmount * 10 ** IERC20(_collateralToken).decimals())
            / (collateralTokenPrice * 10 ** IERC20(_borrowToken).decimals());

-       // Account for 5% slippage in swap
-       collateralToWithdraw = (collateralToWithdraw * 1050) / 1000;
+       // Account for configurable slippage in swap (e.g., unwindSlippageBps = 500 = 5%)
+       collateralToWithdraw = (collateralToWithdraw * (10000 + unwindSlippageBps)) / 10000;

        return (collateralToWithdraw, debtAmount);
    }
```

* Add state variable: `uint256 public unwindSlippageBps = 500;` and setter `setUnwindSlippageBps(uint256)`.

**Location**: `src/Stratax.sol:469-470`

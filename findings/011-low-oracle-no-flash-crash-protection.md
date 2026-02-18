# Oracle Missing Flash Crash Protection

## Description

* During flash crashes, oracle prices can temporarily deviate significantly from the true market price. Chainlink aggregates multiple sources and has some built-in resistance, but extreme moves can still produce prices that are inaccurate for a short window.

* The protocol uses oracle prices for leverage calculations, collateral valuation, and unwind logic. There is no check that the returned price lies within an expected range (e.g., bounds relative to a recent price or max deviation). A flash-crash price can cause incorrect borrow amounts, unhealthy positions, or failed unwinds.

```solidity
@>      (, int256 answer,,,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
        price = uint256(answer);
        // No bounds check: price could be 10x or 0.1x of expected during flash crash
```

## Risk

**Likelihood (low)**:

* Flash crashes are rare. Chainlink's aggregation provides some protection. The vulnerability requires an extreme market event coinciding with a protocol interaction.

**Impact (medium)**:

* If the oracle returns an anomalously low price during a flash crash, collateral is undervalued. Users may borrow less than intended or the protocol may reject valid operations. If the oracle returns an anomalously high price, collateral is overvalued, leading to overborrowing and subsequent liquidation when prices normalize.

**Severity (low)**:

## Proof of Concept

* During a brief flash crash, ETH momentarily trades at $1500 (vs normal $3000). The Chainlink feed (or a lag in aggregation) returns $1500. A user opens a position: collateral is undervalued, borrow capacity is reduced. Alternatively, the feed lags and still shows $3000 while the market has crashed; the user overborrows and is liquidated when the feed updates.

```solidity
// Flash crash: market ETH = $1500, oracle still shows $3000 (or vice versa)
uint256 price = strataxOracle.getPrice(ETH); // No sanity check
// calculateOpenParams uses price → incorrect leverage/borrow
```

## Recommended Mitigation

* Implement optional bounds checking: compare the oracle price to a recent cached price or a configurable min/max. Revert if the deviation exceeds a threshold (e.g., 20% in a single update).

* This adds complexity and may block legitimate operations during real market moves. Consider making it configurable per asset or using a conservative deviation threshold.

```diff
+   mapping(address => uint256) public lastKnownPrice;
+   uint256 public constant MAX_PRICE_DEVIATION_BPS = 2000; // 20%

    function getPrice(address _token) public view returns (uint256 price) {
        ...
        (, int256 answer,,,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
        price = uint256(answer);
+       uint256 lastPrice = lastKnownPrice[_token];
+       if (lastPrice > 0) {
+           uint256 deviation = price > lastPrice
+               ? (price - lastPrice) * 10000 / lastPrice
+               : (lastPrice - price) * 10000 / lastPrice;
+           require(deviation <= MAX_PRICE_DEVIATION_BPS, "Price deviation too high");
+       }
        return price;
    }
```

* Note: Storing `lastKnownPrice` requires a state-changing update (e.g., in a separate function called after successful operations, or via a keeper). A simpler approach is to add min/max price bounds per token, configurable by the owner.

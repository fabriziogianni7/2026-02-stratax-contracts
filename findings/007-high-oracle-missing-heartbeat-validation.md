# Oracle Missing Heartbeat / Staleness Validation

## Description

* Chainlink price feeds return `updatedAt` (timestamp when the round was updated). To ensure accurate price usage, the last update timestamp should be checked against a predefined maximum delay (heartbeat).

* The `getPrice` function in StrataxOracle ignores `updatedAt` entirely. If a price feed stops updating (e.g., Chainlink deprecates the feed, network issues, or the feed is paused), the contract will continue returning the last known price, which may be arbitrarily stale.

```solidity
    function getPrice(address _token) public view returns (uint256 price) {
        ...
@>      (, int256 answer,,,) = priceFeed.latestRoundData();
@>      require(answer > 0, "Invalid price from oracle");
        price = uint256(answer);
    }
```

## Risk

**Likelihood (medium)**:

* Chainlink deprecates price feeds periodically; when a feed is deprecated, it stops updating.

* Network congestion or Chainlink node issues can cause temporary staleness beyond the expected heartbeat.

**Impact (high)**:

* Stale prices lead to incorrect leverage calculations. Users may open positions with overvalued collateral or undervalued debt, resulting in immediate liquidation or unhealthy positions.

* During unwind, stale prices cause incorrect collateral-to-debt conversion, potentially reverting unwinds or leaving excess collateral locked.

**Severity (high)**:

## Proof of Concept

* ETH/USD feed stops updating at 12:00. At 14:00, ETH has crashed 20% but the oracle still returns the 12:00 price. A user opens a leveraged position using the stale price. The position is immediately underwater; the next price update (or liquidation) reveals the loss.

* Alternatively, during a flash crash, the oracle may return a pre-crash price while the market has moved. The protocol uses outdated data for critical financial decisions.

```solidity
// Oracle returns price from 2 hours ago; no check
uint256 price = strataxOracle.getPrice(ETH);
// price is stale; calculateOpenParams uses it → incorrect borrow amount
```

## Recommended Mitigation

```diff
+   uint256 public constant MAX_PRICE_AGE = 1 hours; // Adjust per feed heartbeat

    function getPrice(address _token) public view returns (uint256 price) {
        address priceFeedAddress = priceFeeds[_token];
        require(priceFeedAddress != address(0), "Price feed not set for token");

        AggregatorV3Interface priceFeed = AggregatorV3Interface(priceFeedAddress);

-       (, int256 answer,,,) = priceFeed.latestRoundData();
+       (, int256 answer,, uint256 updatedAt,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
+       require(block.timestamp - updatedAt <= MAX_PRICE_AGE, "Stale price");
        price = uint256(answer);
    }
```

**Note**: Set `MAX_PRICE_AGE` based on the actual heartbeat of each feed (e.g., ETH/USD ~1h on mainnet; some feeds may differ).

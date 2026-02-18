# StrataxOracle setPriceFeeds Unbounded Loop

## Description

* `StrataxOracle.setPriceFeeds` accepts an array of token addresses and price feeds, then iterates over them in a loop to set each price feed. The `_tokens.length` is not bounded.

* When the owner passes a very large array (e.g. hundreds or thousands of tokens), the transaction can exceed the block gas limit. The entire call reverts, and no price feeds are updated. The owner must either split the operation into smaller batches or reduce the array size.

* The function is `onlyOwner`, so the risk is limited to misconfiguration or a script that accidentally generates a huge array. There is no malicious actor who can trigger this without owner privileges.

```solidity
    function setPriceFeeds(address[] calldata _tokens, address[] calldata _priceFeeds) external onlyOwner {
        require(_tokens.length == _priceFeeds.length, "Array length mismatch");

@>      for (uint256 i = 0; i < _tokens.length; i++) {
            _setPriceFeed(_tokens[i], _priceFeeds[i]);
            emit PriceFeedUpdated(_tokens[i], _priceFeeds[i]);
        }
        // No upper bound on _tokens.length; large arrays can hit gas limit
    }
```

**Location**: `src/StrataxOracle.sol:38-45`

## Risk

**Likelihood (low)**:

* Requires the owner to pass a large array—either intentionally (e.g. bulk onboarding many tokens) or accidentally (e.g. a script bug that generates a large list).

* Typical use cases set a handful of price feeds per call.

**Impact (low)**:

* The transaction reverts when gas is exhausted. No state is corrupted; the owner can retry with a smaller array.

* The owner may need to split the operation into multiple transactions, which is an operational inconvenience.

**Severity**: Informational

## Proof of Concept

* Owner runs a script to onboard 500 tokens from a config file. The script calls `setPriceFeeds(tokens, priceFeeds)` with 500 elements. Each iteration: `_setPriceFeed` does external calls to `priceFeed.decimals()`, and `emit PriceFeedUpdated` writes to logs. The total gas exceeds the block limit; the transaction reverts. The owner must either split into 50 batches of 10 or reduce the scope.

```solidity
address[] memory tokens = new address[](500);
address[] memory feeds = new address[](500);
// ... populate arrays ...

strataxOracle.setPriceFeeds(tokens, feeds);
// Reverts: out of gas
```

## Recommended Mitigation

```diff
+   uint256 public constant MAX_BATCH_SIZE = 50;

    function setPriceFeeds(address[] calldata _tokens, address[] calldata _priceFeeds) external onlyOwner {
        require(_tokens.length == _priceFeeds.length, "Array length mismatch");
+       require(_tokens.length <= MAX_BATCH_SIZE, "Batch size too large");

        for (uint256 i = 0; i < _tokens.length; i++) {
            _setPriceFeed(_tokens[i], _priceFeeds[i]);
            emit PriceFeedUpdated(_tokens[i], _priceFeeds[i]);
        }
    }
```

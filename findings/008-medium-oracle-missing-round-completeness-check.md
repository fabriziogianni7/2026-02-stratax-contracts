# Oracle Missing Round Completeness Check (answeredInRound)

## Description

* Chainlink's `latestRoundData()` returns `roundId`, `answer`, `startedAt`, `updatedAt`, and `answeredInRound`. When a round is in progress, `answeredInRound` can be less than `roundId`, indicating the answer belongs to a previous round and the current round is incomplete.

* The oracle only validates `answer > 0` and ignores `answeredInRound` and `roundId`. If the contract reads during an incomplete round, it may receive stale or incorrect data from a previous round.

```solidity
@>      (, int256 answer,,,) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
        price = uint256(answer);
```

## Risk

**Likelihood (low)**:

* The race window is narrow (seconds between rounds). An attacker would need to time a transaction to land exactly when a round is incomplete.

* Chainlink rounds typically complete quickly; the condition is rare but possible during network stress.

**Impact (high)**:

* Returning data from an incomplete round can yield incorrect prices. Incorrect prices propagate to leverage calculations, collateral valuations, and unwind logic, potentially causing liquidations or fund loss.

**Severity (medium)**:

## Proof of Concept

* During a Chainlink round transition, `roundId` advances before the new answer is available. `answeredInRound < roundId` indicates the answer is from the previous round. The contract accepts it without checking. In edge cases, the previous round's answer could be stale or anomalous.

```solidity
// Attacker times tx during round transition
(uint80 roundId, int256 answer,,, uint80 answeredInRound) = priceFeed.latestRoundData();
// answeredInRound = 100, roundId = 101 → incomplete round 101
// Oracle returns answer from round 100; no validation
```

## Recommended Mitigation

```diff
-       (, int256 answer,,,) = priceFeed.latestRoundData();
+       (uint80 roundId, int256 answer,,, uint80 answeredInRound) = priceFeed.latestRoundData();
        require(answer > 0, "Invalid price from oracle");
+       require(answeredInRound >= roundId, "Incomplete round");
        price = uint256(answer);
```

**Reference**: Chainlink recommends this check in their documentation for consuming price feed data.

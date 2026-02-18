# Oracle Missing Sequencer Uptime Check for L2 Deployments

## Description

* On Arbitrum and Optimism, the sequencer can go offline. When it is down, transactions are delayed and the mempool is not processed. Oracles may update on L1 while the L2 sequencer is offline, creating a situation where L2 state (including oracle prices) is stale.

* An attacker can observe L1 price updates, then submit transactions when the sequencer comes back online. The L2 oracle may still have the pre-update price, allowing arbitrage or manipulation.

* StrataxOracle has no sequencer uptime feed integration. If deployed on Arbitrum or Optimism, the protocol is vulnerable to stale price exploitation when the sequencer restarts.

```solidity
    function getPrice(address _token) public view returns (uint256 price) {
        ...
@>      (, int256 answer,,,) = priceFeed.latestRoundData();
        // No check: is the L2 sequencer up? Has enough time passed since it came back?
        require(answer > 0, "Invalid price from oracle");
        price = uint256(answer);
    }
```

## Risk

**Likelihood (low)**:

* Only applies if the protocol is deployed on Arbitrum or Optimism (or other L2s with sequencer uptime feeds).

* Sequencer downtime is infrequent but occurs during upgrades and incidents.

**Impact (high)**:

* When the sequencer restarts, there is a grace period where prices can be stale. Attackers can front-run legitimate users with knowledge of L1 price movements, opening or unwinding positions at incorrect prices.

**Severity (medium)**:

## Proof of Concept

* Protocol deployed on Arbitrum. Sequencer goes offline for 30 minutes. During that time, ETH drops 10% on L1. Sequencer comes back. The L2 Chainlink feed has not yet updated. Attacker opens a leveraged position using the stale (higher) ETH price. Position is immediately underwater when the feed updates.

```solidity
// On Arbitrum: sequencer was down, just came back
// L1 ETH price: $2000, L2 oracle still shows $2200 (stale)
uint256 price = strataxOracle.getPrice(ETH); // 2200e8
// User opens position with inflated collateral value → overborrow → liquidation when price updates
```

## Recommended Mitigation

* If deploying on Arbitrum or Optimism, integrate the Chainlink sequencer uptime feed. Reject prices (or revert) when the sequencer is down or when the grace period after sequencer restart has not elapsed.

```diff
+   // L2 only: sequencer uptime feed (Arbitrum/Optimism)
+   address public sequencerUptimeFeed; // Set per chain

    function getPrice(address _token) public view returns (uint256 price) {
        ...
+       if (sequencerUptimeFeed != address(0)) {
+           (, int256 answer, uint256 startedAt,,) = AggregatorV3Interface(sequencerUptimeFeed).latestRoundData();
+           require(answer == 0, "Sequencer down"); // 0 = up, 1 = down
+           require(block.timestamp - startedAt > GRACE_PERIOD, "Sequencer restarted recently");
+       }
        (, int256 answer,,,) = priceFeed.latestRoundData();
        ...
    }
```

* On Ethereum mainnet, `sequencerUptimeFeed` can be `address(0)` to skip the check.

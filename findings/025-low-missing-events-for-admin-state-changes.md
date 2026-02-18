# Missing Events for Admin State Changes

## Description

* Stratax emits events for user-facing operations (`LeveragePositionCreated`, `PositionUnwound`) but does not emit events for administrative state changes.

* The four admin functions that modify critical protocol state—`transferOwnership`, `setStrataxOracle`, `setFlashLoanFee`, and `recoverTokens`—do not emit events. Off-chain indexers, monitoring, and governance tooling cannot track these changes without parsing transaction logs for non-standard patterns.

```solidity
    function setStrataxOracle(address _strataxOracle) external onlyOwner {
        require(_strataxOracle != address(0), "Invalid oracle address");
@>      strataxOracle = _strataxOracle;
        // No event emitted
    }

    function setFlashLoanFee(uint256 _flashLoanFeeBps) external onlyOwner {
        require(_flashLoanFeeBps < FLASHLOAN_FEE_PREC, "Fee must be < 100%");
@>      flashLoanFeeBps = _flashLoanFeeBps;
        // No event emitted
    }

    function recoverTokens(address _token, uint256 _amount) external onlyOwner {
@>      IERC20(_token).transfer(owner, _amount);
        // No event emitted
    }

    function transferOwnership(address _newOwner) external onlyOwner {
        require(_newOwner != address(0), "Invalid address");
@>      owner = _newOwner;
        // No event emitted (StrataxOracle emits OwnershipTransferred; Stratax does not)
    }
```

## Risk

**Likelihood (low)**:

* Admin state changes occur infrequently; the lack of events is a design oversight rather than an active attack vector.

**Impact (low)**:

* Indexers and governance dashboards cannot reliably detect oracle changes, fee changes, ownership transfers, or token recoveries without custom transaction parsing.

* Incident response and audit trails are degraded; reconstructing protocol state history requires manual transaction inspection.

**Severity**: Low

## Proof of Concept

* Owner changes the oracle address via `setStrataxOracle(newOracle)`. Off-chain monitoring expects an `OracleUpdated` or similar event to update cached state. No such event exists; the indexer continues to show the old oracle address until the next transaction is manually inspected.

```solidity
// Owner calls setStrataxOracle(address(0x123))
stratax.setStrataxOracle(address(0x123));

// No event emitted; indexers cannot detect the change
```

## Recommended Mitigation

```diff
+   event OracleUpdated(address indexed previousOracle, address indexed newOracle);
+   event FlashLoanFeeUpdated(uint256 previousFeeBps, uint256 newFeeBps);
+   event TokensRecovered(address indexed token, address indexed to, uint256 amount);
+   event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    function setStrataxOracle(address _strataxOracle) external onlyOwner {
        require(_strataxOracle != address(0), "Invalid oracle address");
+       address previousOracle = strataxOracle;
        strataxOracle = _strataxOracle;
+       emit OracleUpdated(previousOracle, _strataxOracle);
    }

    function setFlashLoanFee(uint256 _flashLoanFeeBps) external onlyOwner {
        require(_flashLoanFeeBps < FLASHLOAN_FEE_PREC, "Fee must be < 100%");
+       uint256 previousFee = flashLoanFeeBps;
        flashLoanFeeBps = _flashLoanFeeBps;
+       emit FlashLoanFeeUpdated(previousFee, _flashLoanFeeBps);
    }

    function recoverTokens(address _token, uint256 _amount) external onlyOwner {
        IERC20(_token).transfer(owner, _amount);
+       emit TokensRecovered(_token, owner, _amount);
    }

    function transferOwnership(address _newOwner) external onlyOwner {
        require(_newOwner != address(0), "Invalid address");
+       address previousOwner = owner;
        owner = _newOwner;
+       emit OwnershipTransferred(previousOwner, _newOwner);
    }
```

**Locations**: `src/Stratax.sol` — `setStrataxOracle`, `setFlashLoanFee`, `recoverTokens`, `transferOwnership`

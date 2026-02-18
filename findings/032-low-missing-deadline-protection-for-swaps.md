# Missing Deadline Protection for Swaps Enables Sandwich Attacks

## Description

* `createLeveragedPosition` and `unwindPosition` execute 1inch swaps inside a flash loan callback. Both functions accept swap calldata (`_oneInchSwapData`) and minimum return amount (`_minReturnAmount`) but have no deadline parameter.

* Without a deadline, a pending transaction can be held in the mempool until the market moves against the user. An attacker can front-run with a swap that worsens the price, then back-run after the victim's swap executes at an unfavorable rate. The protocol relies entirely on the 1inch API to include a deadline in its encoded calldata; Stratax does not enforce one at the protocol level.

```solidity
    function createLeveragedPosition(
        address _flashLoanToken,
        uint256 _flashLoanAmount,
        uint256 _collateralAmount,
        address _borrowToken,
        uint256 _borrowAmount,
        bytes calldata _oneInchSwapData,
        uint256 _minReturnAmount
    ) public onlyOwner {
@>      // No deadline parameter; no require(block.timestamp <= deadline)
        ...
        aavePool.flashLoanSimple(address(this), _flashLoanToken, _flashLoanAmount, encodedParams, 0);
    }

    function unwindPosition(
        address _collateralToken,
        uint256 _collateralToWithdraw,
        address _debtToken,
        uint256 _debtAmount,
        bytes calldata _oneInchSwapData,
        uint256 _minReturnAmount
    ) external onlyOwner {
@>      // No deadline parameter; no require(block.timestamp <= deadline)
        ...
        aavePool.flashLoanSimple(address(this), _debtToken, _debtAmount, encodedParams, 0);
    }
```

## Risk

**Likelihood (low)**:

* The 1inch API typically includes a deadline in its swap calldata, which mitigates the risk when integrators use the API correctly.

* Integrators building swap data manually or via custom tooling may omit the deadline. Custom scripts or future integrations may not enforce one.

**Impact (medium)**:

* Pending transactions can be held until prices move unfavorably; the owner receives worse execution than expected.

* Sandwich attacks: attacker front-runs with a swap that moves the price, victim's swap executes at worse rate, attacker back-runs to capture profit.

**Severity (low)**:

## Proof of Concept

* Owner prepares `createLeveragedPosition` with swap data and `minReturnAmount` based on current market. Transaction is submitted but not mined immediately due to congestion or low gas.

* Attacker sees the pending tx, front-runs with a large swap that worsens the price for the victim's swap path.

* Victim's tx is mined; the 1inch swap executes at the manipulated price. If the swap data had no deadline (or a far-future one), the tx succeeds but the owner receives less than expected. Attacker back-runs to capture profit.

```solidity
// Attacker front-run: swap large amount to move price
// Victim tx mines: createLeveragedPosition executes swap at worse rate
// Attacker back-run: swap back to capture profit
// Owner receives suboptimal execution; no protocol-level deadline to reject stale txs
```

## Recommended Mitigation

Add a `deadline` parameter to both functions and enforce it before initiating the flash loan:

```diff
    function createLeveragedPosition(
        address _flashLoanToken,
        uint256 _flashLoanAmount,
        uint256 _collateralAmount,
        address _borrowToken,
        uint256 _borrowAmount,
        bytes calldata _oneInchSwapData,
-       uint256 _minReturnAmount
+       uint256 _minReturnAmount,
+       uint256 _deadline
    ) public onlyOwner {
+       require(block.timestamp <= _deadline, "Expired");
        require(_collateralAmount > 0, "Collateral Cannot be Zero");
        ...
    }

    function unwindPosition(
        address _collateralToken,
        uint256 _collateralToWithdraw,
        address _debtToken,
        uint256 _debtAmount,
        bytes calldata _oneInchSwapData,
-       uint256 _minReturnAmount
+       uint256 _minReturnAmount,
+       uint256 _deadline
    ) external onlyOwner {
+       require(block.timestamp <= _deadline, "Expired");
        ...
    }
```

**Location**: `src/Stratax.sol:314-339`, `src/Stratax.sol:236-256`

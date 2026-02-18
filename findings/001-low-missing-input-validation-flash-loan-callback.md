# Missing Input Validation in Flash Loan Callback

## Description

`executeOperation` is the Aave flash loan callback. It validates that the caller is the Aave Pool and that the initiator is this contract, but it does **not** validate `_amount`, `_premium`, or `_params`.

Under normal operation, Aave passes correct values and the contract encodes `_params` correctly. However, the lack of input validation violates defense-in-depth and could lead to confusing failures if Aave interface changes, encoding bugs occur, or upgrades introduce errors.

```solidity
    function executeOperation(
        address _asset,
        uint256 _amount,
        uint256 _premium,
        address _initiator,
        bytes calldata _params
    ) external returns (bool) {
        require(msg.sender == address(aavePool), "Caller must be Aave Pool");
        require(_initiator == address(this), "Initiator must be this contract");
        // hack should add validation here
        // likelyhood: low | impact low | severity low
        // Decode operation type
        OperationType opType = abi.decode(_params, (OperationType));
```

## Risk

**Likelihood**:

* Aave is trusted and the callback is only reachable from the Aave Pool.
* `_params` is encoded by this contract when initiating the flash loan.
* Invalid values would likely cause reverts in downstream logic rather than silent misuse.

**Impact**:

* Malformed or unexpected inputs could cause confusing reverts instead of clear, early failures.
* No protection against future Aave interface changes or encoding bugs.
* Inconsistent with other entry points that validate inputs.

## Proof of Concept

```solidity
// If _amount is 0, the flash loan is effectively empty. Logic may behave unexpectedly.
// If _params is empty, abi.decode reverts with an opaque error.
// No explicit bounds check on _premium (e.g., vs _amount).

// Current flow: validation only on msg.sender and _initiator
// Missing: _amount > 0, _params.length > 0, optional _premium sanity check
```

## Recommended Mitigation

```diff
        require(msg.sender == address(aavePool), "Caller must be Aave Pool");
        require(_initiator == address(this), "Initiator must be this contract");
+       require(_amount > 0, "Invalid flash loan amount");
+       require(_params.length > 0, "Invalid params");
        // Decode operation type
        OperationType opType = abi.decode(_params, (OperationType));
```

**Severity**: Low (best practice / defense-in-depth)

**Location**: `src/Stratax.sol:209-214`

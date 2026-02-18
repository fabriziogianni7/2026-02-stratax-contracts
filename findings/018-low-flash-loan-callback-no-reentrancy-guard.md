# Flash Loan Callback Lacks Reentrancy Guard

## Description

* `executeOperation` is the Aave flash loan callback and performs multiple external calls (repay, withdraw, 1inch swap, supply).

* The 1inch swap uses a low-level `call` that can execute arbitrary code. There is no `nonReentrant` modifier on `executeOperation`.

* While the contract has no obvious callback that would benefit an attacker, defense-in-depth is missing.

```solidity
    function executeOperation(
        address _asset,
        uint256 _amount,
        uint256 _premium,
        address _initiator,
        bytes calldata _params
@>  ) external returns (bool) {
        require(msg.sender == address(aavePool), "Caller must be Aave Pool");
        require(_initiator == address(this), "Initiator must be this contract");
        // ...
        } else {
            return _executeUnwindOperation(_asset, _amount, _premium, _params);
        }
    }
```

## Risk

**Likelihood (low)**:

* 1inch swap paths would need to callback into this contract.

* The contract has no `receive()` or useful callback; reentrancy paths are limited.

**Impact (low)**:

* If a callback path existed, reentrancy could lead to unexpected behavior or fund loss.

**Severity (low)**:

## Proof of Concept

* 1inch router executes swap; DEX in the path calls back to our contract. Without `nonReentrant`, that callback could re-enter before state is finalized. Current design has no useful callback, so exploit path is unclear.

## Recommended Mitigation

```diff
+ import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

- contract Stratax is Initializable {
+ contract Stratax is Initializable, ReentrancyGuard {

    function executeOperation(...)
+       nonReentrant
        external returns (bool) {
```

Note: If the contract is upgradeable, ensure ReentrancyGuard storage is compatible with the upgrade pattern.

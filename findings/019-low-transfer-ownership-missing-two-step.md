# Missing Two-Step Ownership Transfer

## Description

* Ownership transfer is a critical privilege: the owner controls all tokens, Aave positions, `recoverTokens`, `createLeveragedPosition`, `unwindPosition`, and oracle/fee settings.

* The current `transferOwnership` uses a single-step transfer. The new owner is set immediately with no confirmation from the recipient.

* A typo, wrong address, or social engineering could result in ownership being transferred to an unintended address.

```solidity
    /**
     * @notice Updates the owner address
     * @param _newOwner The new owner address
     */
    function transferOwnership(address _newOwner) external onlyOwner {
        require(_newOwner != address(0), "Invalid address");
@>      owner = _newOwner;
    }
```

## Risk

**Likelihood (low)**:

* Requires owner error (typo, copy-paste mistake, or social engineering).

* Single-step transfers are a common pattern; accidental transfers are rare but possible.

**Impact (high)**:

* If ownership is transferred to an unintended address, the new owner gains full control over the protocol.

* The owner can recover all tokens via `recoverTokens`, unwind positions, and drain funds.

* No recovery mechanism exists once the transfer is committed.

**Severity (low)**:

## Proof of Concept

* Owner intends to transfer ownership to address `0x1234...abcd` but accidentally types `0x1234...abce` (wrong last character). The transaction succeeds. The unintended recipient now owns the protocol and can call `recoverTokens` to drain all aTokens and user collateral.

```solidity
// Owner calls transferOwnership(wrongAddress)
stratax.transferOwnership(0x1234...abce); // typo in address

// wrongAddress is now owner; no way to revert
// wrongAddress can: recoverTokens, unwindPosition, etc.
```

## Recommended Mitigation

* Implement a two-step ownership transfer (Ownable2Step pattern):

```diff
+  address public pendingOwner;

    function transferOwnership(address _newOwner) external onlyOwner {
        require(_newOwner != address(0), "Invalid address");
-       owner = _newOwner;
+       pendingOwner = _newOwner;
    }

+   function acceptOwnership() external {
+       require(msg.sender == pendingOwner, "Not pending owner");
+       owner = pendingOwner;
+       delete pendingOwner;
+   }
```

* Alternatively, use OpenZeppelin's `Ownable2Step` if compatible with the upgradeable pattern.

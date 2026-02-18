# Unwind Excess Collateral Stays in Protocol (Design Note)

## Description

* When the swap returns more than needed to repay the flash loan (`returnAmount > totalDebt`), the excess is supplied to Aave via `aavePool.supply(_asset, returnAmount - totalDebt, address(this), 0)`.

* The excess remains in the contract's Aave position. The owner does not receive it directly and would need a separate withdrawal mechanism.

* This may be intentional (protocol retains surplus) but is worth documenting.

```solidity
        if (returnAmount - totalDebt > 0) {
            IERC20(_asset).approve(address(aavePool), returnAmount - totalDebt);
@>          aavePool.supply(_asset, returnAmount - totalDebt, address(this), 0);
        }
```

## Risk

**Likelihood (high)**:

* Occurs whenever the swap yields more than `totalDebt`.

**Impact (low)**:

* Owner may expect to receive excess; it stays in the protocol.

* Requires separate flow (e.g. `recoverTokens` on aTokens) to extract surplus.

**Severity (informational)**:

## Proof of Concept

* Swap returns 1050 USDC, totalDebt is 1009 USDC. Excess 41 USDC is supplied to Aave. Contract holds aTokens; owner does not receive 41 USDC directly.

## Recommended Mitigation

* Document that excess stays in the protocol and how the owner can recover it (e.g. via `recoverTokens` on aTokens).

* If the design intends the owner to receive excess, add a transfer to the owner instead of (or in addition to) supplying to Aave.

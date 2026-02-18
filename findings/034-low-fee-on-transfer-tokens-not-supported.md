# Fee-on-Transfer Tokens Not Supported

## Description

* The protocol assumes that the amount transferred via `transferFrom` equals the amount received by the contract. This holds for standard ERC20 tokens but fails for fee-on-transfer tokens, which charge a fee on each transfer so the receiver gets less than the sent amount.

* When opening a leveraged position, the contract computes `totalCollateral = _amount + flashParams.collateralAmount` and supplies it to Aave. The `collateralAmount` is the value passed by the caller, not the actual amount received. With fee-on-transfer tokens, the contract receives less than `collateralAmount`, so it does not hold enough tokens to supply `totalCollateral`. The `supply` call reverts due to insufficient balance.

```solidity
        // Transfer the user's collateral to the contract
@>      IERC20(_flashLoanToken).transferFrom(msg.sender, address(this), _collateralAmount);

        FlashLoanParams memory params = FlashLoanParams({
            collateralToken: _flashLoanToken,
            collateralAmount: _collateralAmount,
            ...
        });
        ...
        // _executeOpenOperation
        uint256 totalCollateral = _amount + flashParams.collateralAmount;

@>      IERC20(_asset).approve(address(aavePool), totalCollateral);
@>      aavePool.supply(_asset, totalCollateral, address(this), 0);
```

## Risk

**Likelihood (low)**:

* Aave reserves and typical leverage flows use standard tokens (USDC, WETH) that are not fee-on-transfer.

* Protocol expansion or integration with staking/reward tokens (e.g. some PAXG, deflationary tokens) would expose this.

**Impact (low)**:

* `createLeveragedPosition` reverts when the collateral token is fee-on-transfer.

* No fund loss; operations fail with "insufficient balance" or similar revert.

**Severity (low)**:

## Proof of Concept

```solidity
// Fee-on-transfer token: 1% fee on transfer
// User approves and calls createLeveragedPosition with collateralAmount = 1000e18
// transferFrom sends 1000e18, contract receives 990e18 (1% fee)
// totalCollateral = flashLoanAmount + 1000e18
// supply(_asset, totalCollateral, ...) attempts to transfer totalCollateral from contract
// Contract only holds flashLoanAmount + 990e18 → revert (insufficient balance)
```

## Recommended Mitigation

**Option A** — Measure actual received amount and use it in accounting:

```diff
        require(_collateralAmount > 0, "Collateral Cannot be Zero");
        // Transfer the user's collateral to the contract
+       uint256 balanceBefore = IERC20(_flashLoanToken).balanceOf(address(this));
        IERC20(_flashLoanToken).transferFrom(msg.sender, address(this), _collateralAmount);
+       uint256 actualReceived = IERC20(_flashLoanToken).balanceOf(address(this)) - balanceBefore;
+       require(actualReceived > 0, "No collateral received");

        FlashLoanParams memory params = FlashLoanParams({
            collateralToken: _flashLoanToken,
-           collateralAmount: _collateralAmount,
+           collateralAmount: actualReceived,
            ...
        });
```

**Option B** — Document and enforce: explicitly disallow fee-on-transfer tokens in supported token list; add validation or whitelist if such tokens must be excluded.

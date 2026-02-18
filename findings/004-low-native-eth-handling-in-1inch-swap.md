# Native ETH Handling in 1inch Swap Causes Swap Failure or Revert

## Description

* The `_call1InchSwap` function executes token swaps via a low-level `call` to the 1inch router. The protocol is designed for ERC20 tokens (USDC, WETH) and tests only use wrapped tokens.

* The protocol does not explicitly state that native ETH is unsupported. The README lists "ERC20 tokens" and "Standard ERC20 implementation" but never documents that native ETH is disallowed. There is no validation in the code that rejects native ETH addresses at entry points. Users and integrators may reasonably assume native ETH is supported, especially since 1inch and Aave support it.

* When swap flows involve native ETH (either as input or output), the implementation fails in three ways: (1) no ETH is forwarded when swapping from native ETH, (2) the contract cannot receive native ETH when swapping to it, and (3) the fallback balance check uses `IERC20(_asset)` which reverts when `_asset` is the native ETH sentinel address.

```solidity
        // Execute the 1inch swap using low-level call with the calldata from the API
@>      (bool success, bytes memory result) = address(oneInchRouter).call(_swapParams);
        require(success, "1inch swap failed");

        // Decode the return amount from the swap
        if (result.length > 0) {
            (returnAmount,) = abi.decode(result, (uint256, uint256));
        } else {
            // If no return data, check balance
@>          returnAmount = IERC20(_asset).balanceOf(address(this));
        }
```

* The 1inch `swap` function is `payable` (IAggregationRouter), so swaps involving native ETH require the caller to send value or accept ETH. The Stratax contract has no `receive()` or `fallback()`.

## Risk

**Likelihood**:

* Protocol expansion or integrator configuration may add native ETH as collateral, borrow token, or flash loan asset. The 1inch API supports native ETH (e.g., sentinel `0xEeeE...`).

* Off-chain swap data generation can produce calldata for native ETH swaps when the configured token is native ETH instead of WETH.

* No validation rejects native ETH addresses at entry points (`createLeveragedPosition`, `unwindPosition`).

**Impact**:

* Swaps that require native ETH as input revert because `call(_swapParams)` sends 0 wei; the router receives no funds.

* Swaps that return native ETH revert because the contract cannot receive ETH (no `receive()`/`fallback()`).

* When `result.length == 0` and `_asset` is the native ETH sentinel, `IERC20(_asset).balanceOf(...)` reverts—native ETH is not an ERC20.

* DoS for users attempting native-ETH leverage flows; confusing failure mode ("1inch swap failed" or low-level revert).

## Proof of Concept

```solidity
// Scenario 1: Swap FROM native ETH (e.g., flash loan ETH, borrow ETH, swap ETH for USDC)
// - call(_swapParams) sends 0 wei
// - 1inch router expects msg.value > 0
// - Swap reverts

// Scenario 2: Swap TO native ETH (e.g., swap USDC for ETH)
// - 1inch router sends ETH to address(this)
// - Stratax has no receive() or fallback()
// - ETH transfer reverts, entire swap fails

// Scenario 3: result.length == 0 and _asset = 0xEeeE... (1inch native ETH sentinel)
// - returnAmount = IERC20(0xEeeE...).balanceOf(address(this));
// - Call to non-ERC20 address reverts
```

## Recommended Mitigation

**Option A** — Explicitly disallow native ETH (document and enforce):

```diff
    function _call1InchSwap(bytes memory _swapParams, address _asset, uint256 _minReturnAmount)
        internal
        returns (uint256 returnAmount)
    {
+       require(_asset != address(0), "Native ETH not supported");
+       // Optionally: require(_asset != 0xEeeE... , "Native ETH not supported");
        // Execute the 1inch swap using low-level call with the calldata from the API
        (bool success, bytes memory result) = address(oneInchRouter).call(_swapParams);
```

**Option B** — Support native ETH when required:

```diff
    function _call1InchSwap(bytes memory _swapParams, address _asset, uint256 _minReturnAmount)
        internal
        returns (uint256 returnAmount)
    {
        // Execute the 1inch swap using low-level call with the calldata from the API
-       (bool success, bytes memory result) = address(oneInchRouter).call(_swapParams);
+       (bool success, bytes memory result) = address(oneInchRouter).call{value: msg.value}(_swapParams);
        require(success, "1inch swap failed");

        // Decode the return amount from the swap
        if (result.length > 0) {
            (returnAmount,) = abi.decode(result, (uint256, uint256));
        } else {
            // If no return data, check balance
-           returnAmount = IERC20(_asset).balanceOf(address(this));
+           returnAmount = _asset == 0xEeeE... ? address(this).balance : IERC20(_asset).balanceOf(address(this));
        }
```

* Add `receive() external payable {}` if swaps can return native ETH.

**Severity**: Low (current flows use WETH/USDC only; impact increases if native ETH is supported)

**Location**: `src/Stratax.sol:613-631`

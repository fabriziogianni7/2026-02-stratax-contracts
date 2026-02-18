# Stratax Implementation Missing _disableInitializers

## Description

* Stratax is deployed as a Beacon Proxy implementation. The implementation contract is used only as the target for delegatecalls; the proxy holds its own storage and state.

* OpenZeppelin's upgradeable pattern recommends that the implementation contract call `_disableInitializers()` in its constructor so that the implementation itself cannot be initialized. This prevents anyone from calling `initialize()` directly on the implementation address and taking ownership of the implementation's storage.

* Stratax has no constructor and does not call `_disableInitializers()`. The implementation contract can be initialized by anyone who calls `initialize()` on the implementation address. That would set `owner` and other state on the implementation's storage—which is not used by the proxy. The proxy's storage remains separate. The impact is limited because the implementation does not hold funds and is not used as a standalone contract, but the pattern violates upgradeable best practices and can cause confusion or unexpected behavior in edge cases.

```solidity
contract Stratax is Initializable {
    // No constructor
    // Missing: constructor() { _disableInitializers(); }

    function initialize(...) external initializer {
        aavePool = IPool(_aavePool);
        aaveDataProvider = IProtocolDataProvider(_aaveDataProvider);
        oneInchRouter = IAggregationRouter(_oneInchRouter);
        USDC = _usdc;
        strataxOracle = _strataxOracle;
@>      owner = msg.sender;
        flashLoanFeeBps = 9;
    }
```

**Location**: `src/Stratax.sol` (implementation contract)

## Risk

**Likelihood (low)**:

* Requires someone to intentionally or accidentally call `initialize()` on the implementation contract address rather than the proxy.

* The implementation address is typically not exposed in frontends; users interact with the proxy.

**Impact (low)**:

* The implementation's storage is separate from the proxy's. Initializing the implementation does not affect the proxy's state or funds.

* If the implementation is initialized, its `owner` could theoretically call functions on the implementation directly—but the implementation has no `receive()` or payable functions, and no funds are sent to it. The implementation contract itself is not designed to be used standalone.

* The main risk is a violation of the upgradeable pattern: future code or tooling may assume the implementation is locked, and the lack of `_disableInitializers()` could lead to unexpected behavior in edge cases.

**Severity**: Low

## Proof of Concept

* Attacker deploys the Stratax implementation (or the implementation is already deployed as part of the beacon setup). After deployment, the attacker calls `initialize(attacker, attacker, attacker, attacker, attacker)` on the implementation address. The implementation's storage is now initialized with `owner = attacker`. The proxy's storage is unchanged. The proxy continues to work correctly. The impact is limited to the implementation contract having a non-default state.

```solidity
// Implementation deployed at 0xImpl
Stratax impl = new Stratax();

// Attacker calls initialize directly on implementation
impl.initialize(AAVE_POOL, DATA_PROVIDER, ROUTER, USDC, ORACLE);
// impl.owner = msg.sender (attacker)
// Proxy is unaffected; proxy has its own storage
```

## Recommended Mitigation
apply this change to the implementation contract:
```diff
contract Stratax is Initializable {
+   constructor() {
+       _disableInitializers();
+   }

    function initialize(
```

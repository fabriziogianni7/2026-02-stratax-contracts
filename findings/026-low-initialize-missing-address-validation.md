# Initialize Missing Address Validation

## Description

* `initialize` is the single entry point for configuring the Stratax proxy. It sets the Aave pool, data provider, 1inch router, USDC address, and oracle—all of which are critical for correct protocol behavior.

* The function does not validate that any of these addresses are non-zero. Passing `address(0)` for any parameter results in a broken proxy: subsequent calls to `createLeveragedPosition`, `unwindPosition`, and view functions will revert or behave incorrectly when they interact with the zero address.

* Deployment scripts and deploy scripts typically pass correct values, but the lack of validation violates defense-in-depth. A typo in a config file, a bug in a deployment script, or an accidental use of an uninitialized variable can leave the proxy in an invalid state with no clear error message.

```solidity
    function initialize(
        address _aavePool,
        address _aaveDataProvider,
        address _oneInchRouter,
        address _usdc,
        address _strataxOracle
    ) external initializer {
@>      aavePool = IPool(_aavePool);
@>      aaveDataProvider = IProtocolDataProvider(_aaveDataProvider);
@>      oneInchRouter = IAggregationRouter(_oneInchRouter);
@>      USDC = _usdc;
@>      strataxOracle = _strataxOracle;
        owner = msg.sender;
        flashLoanFeeBps = 9;
        // No validation that any address != address(0)
    }
```

**Location**: `src/Stratax.sol:173-186`

## Risk

**Likelihood (low)**:

* Deployment is typically done via a script that reads addresses from config or env. A typo or misconfiguration in the script can produce a zero address.

* Multi-chain deployments or script reuse across networks can introduce wrong addresses when a chain-specific config is missing.

* Automated deployment pipelines may pass uninitialized or default struct fields when constructing initialization arguments.

**Impact (medium)**:

* A zero address for `aavePool` causes `executeOperation` to fail the `require(msg.sender == address(aavePool))` check when Aave calls back—flash loans never succeed.

* A zero address for `strataxOracle` causes `require(strataxOracle != address(0), "Oracle not set")` in `calculateOpenParams` and `calculateUnwindParams` to revert—users cannot calculate positions.

* A zero address for `oneInchRouter` causes `_call1InchSwap` to perform a low-level call to `address(0)`, which succeeds but returns empty data—swaps fail with confusing revert messages.

* A zero address for `USDC` or `_aaveDataProvider` leads to similar failures in dependent logic. The proxy becomes unusable and may require a redeployment or upgrade to fix.

**Severity**: Low

## Proof of Concept

* Deploy script incorrectly passes `address(0)` for `_strataxOracle` due to a missing env variable. The proxy deploys successfully. The first call to `createLeveragedPosition` (or any flow that uses `calculateOpenParams`) reverts with "Oracle not set". The proxy is stuck; the owner must either redeploy or upgrade.

```solidity
// Deployment script bug: config.strataxOracle is unset, defaults to address(0)
bytes memory initData = abi.encodeWithSelector(
    Stratax.initialize.selector,
    AAVE_POOL,
    AAVE_PROTOCOL_DATA_PROVIDER,
    INCH_ROUTER,
    USDC,
    address(0)  // Typo or missing config
);

proxy = new BeaconProxy(address(beacon), initData);
stratax = Stratax(address(proxy));

// Later: user calls calculateOpenParams
// Reverts: require(strataxOracle != address(0), "Oracle not set")
// Proxy is unusable until upgrade
```

## Recommended Mitigation

```diff
    function initialize(
        address _aavePool,
        address _aaveDataProvider,
        address _oneInchRouter,
        address _usdc,
        address _strataxOracle
    ) external initializer {
+       require(_aavePool != address(0), "Invalid aave pool");
+       require(_aaveDataProvider != address(0), "Invalid data provider");
+       require(_oneInchRouter != address(0), "Invalid 1inch router");
+       require(_usdc != address(0), "Invalid USDC");
+       require(_strataxOracle != address(0), "Invalid oracle");
        aavePool = IPool(_aavePool);
        aaveDataProvider = IProtocolDataProvider(_aaveDataProvider);
        oneInchRouter = IAggregationRouter(_oneInchRouter);
        USDC = _usdc;
        strataxOracle = _strataxOracle;
        owner = msg.sender;
        flashLoanFeeBps = 9;
    }
```

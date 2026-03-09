# Vanguard - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect hook flag `BEFORE_INITIALIZE_FLAG` is set in deploy script `deployLaunchHook.s.sol`.](#H-01)
    - ### [H-02. The `_resetPerAddressTracking` function that is called in `_beforeSwap` never resets address tracking variables, causing penalty fees for users.](#H-02)
    - ### [H-03. Sender is used instead of the actual user address, causing **EOA addresses** that use the same Router to accumulate penalties based on shared stats.](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Mismatch of the current phase calculation in `_afterSwap` function and `getCurrentPhase`, providing incorrect information to user.](#M-01)
    - ### [M-02. Inaccurate swap amount accounting, causing misapplied limits/penalties for users.](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #56

### Dates: Jan 29th, 2026 - Feb 5th, 2026

[See more contest details here](https://codehawks.cyfrin.io/c/2026-01-vanguard)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect hook flag `BEFORE_INITIALIZE_FLAG` is set in deploy script `deployLaunchHook.s.sol`.            



# Root + Impact

## Description

* During deployment, hook flags must match the hook permissions.

* However, in this case the BEFORE\_INITIALIZE\_FLAG flag was set instead of AFTER\_INITIALIZE\_FLAG.

```solidity
    function run() public {
 @>       uint160 flags = uint160(Hooks.BEFORE_SWAP_FLAG | Hooks.BEFORE_INITIALIZE_FLAG);

        // Mine a salt that will produce a hook address with the correct flags
        .
        .
        .
```

## Risk

**Likelihood**:

* Deployment issue that could lead to a broken hook contract if successfully deployed.

**Impact**:

* This issue could lead to a hidden problem that only appears after a pool with the hook is deployed, potentially requiring redeployment of both the pool and the hook.

## Proof of Concept

Permissions from `TokenLaunchHook.sol`

```solidity
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: true, // AFTER_INITIALIZE_FLAG
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: true, // BEFORE_SWAP_FLAG
            afterSwap: false,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }
```

## Recommended Mitigation

Replace `BEFORE_INITIALIZE_FLAG` with `AFTER_INITIALIZE_FLAG`;

```diff
    function run() public {
-       uint160 flags = uint160(Hooks.BEFORE_SWAP_FLAG | Hooks.BEFORE_INITIALIZE_FLAG);
+       uint160 flags = uint160(Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_INITIALIZE_FLAG);
        // Mine a salt that will produce a hook address with the correct flags
        .
        .
        .
```

## <a id='H-02'></a>H-02. The `_resetPerAddressTracking` function that is called in `_beforeSwap` never resets address tracking variables, causing penalty fees for users.            



# Root + Impact

## Description

* The `_resetPerAddressTracking` function is supposed to reset address tracking `addressSwappedAmount` and `addressLastSwapBlock` when the currentPhase changes in order to reset previous phase stats.

* However, these variables are never reset to 0, as `_resetPerAddressTracking` only resets values for `address(0)`.

```solidity
 function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        .
        .
        .
         if (newPhase != currentPhase) {
@>           _resetPerAddressTracking();
            currentPhase = newPhase;
            lastPhaseUpdateBlock = block.number;
        }
        .
        .
        .

    }
@>    function _resetPerAddressTracking() internal {
        addressSwappedAmount[address(0)] = 0;
        addressLastSwapBlock[address(0)] = 0;
    }
```

## Risk

**Likelihood**:

* Likelihood is high as there are no specific steps required for this issue to occur.

* Users swap within the normal flow, performing swaps during various phases.

**Impact**:

* Users reach the limits of the next phase sooner than expected, based on stats from the previous phase.

* Users are penalized for normal operation due to a flaw in the hook logic.

* Causes denial of service, loss of users as it becomes less interesting for users or potentially more expensive than using other pools.

## Proof of Concept

This POC shows that addressSwappedAmount did not reset to 0 after the phase changed.
Add this snippet of code to `test/TokenLaunchHookUnit.t.sol`

```solidity
    function test_PhaseTransition_Phase1ToPhase2_limitsNotReseted_POC() public {
        assertEq(antiBotHook.getCurrentPhase(), 1, "Should start in phase 1");

        vm.deal(address(this), 1 ether);
        SwapParams memory params = SwapParams({
            zeroForOne: true, amountSpecified: -int256(0.001 ether), sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});
        swapRouter.swap{value: 0.001 ether}(key, params, testSettings, ZERO_BYTES);

        // Check that swapRouter has swapped (since sender is the router)
        uint256 addressSwappedAmountPhase1 = antiBotHook.addressSwappedAmount(address(swapRouter));

        vm.roll(block.number + phase1Duration + 1);

        uint256 currentPhase = antiBotHook.getCurrentPhase();
        uint256 addressSwappedAmountPhase2 = antiBotHook.addressSwappedAmount(address(swapRouter));

        assertEq(currentPhase, 2, "Should be in phase 2");
        assertEq(
            addressSwappedAmountPhase2,
            addressSwappedAmountPhase1,
            "SwappedAmount is equal, proves reset on phase change did not happen"
        );
    }
```

## Recommended Mitigation

1. Add sender address input parameter to \_resetPerAddressTracking, and replace address(0) with this new input parameter.
2. Replace \_resetPerAddressTracking(); with \_resetPerAddressTracking(sender); in the \_beforeSwap function.

```diff
function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        .
        .
        .
        if (newPhase != currentPhase) {
-            _resetPerAddressTracking();
+            _resetPerAddressTracking(sender);
            currentPhase = newPhase;
            lastPhaseUpdateBlock = block.number;
        }
        .
        .
        .
    }

-    function _resetPerAddressTracking() internal {
-        addressSwappedAmount[address(0)] = 0;
-        addressLastSwapBlock[address(0)] = 0;
+    function _resetPerAddressTracking(address sender) internal {
+        addressSwappedAmount[sender] = 0;
+        addressLastSwapBlock[sender] = 0;
    }
```

## <a id='H-03'></a>H-03. Sender is used instead of the actual user address, causing **EOA addresses** that use the same Router to accumulate penalties based on shared stats.            



# Root + Impact

## Description

* The phase limit and penalties should apply to individual addresses based on their own swap activity.

* However, since `_beforeSwap` uses the `sender` input parameter as the user identification address, all stats are accumulated on the **Router address** instead of the actual user.

```solidity
@> function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        if (launchStartBlock == 0) revert PoolNotInitialized();

        if (initialLiquidity == 0) {
            uint128 liquidity = StateLibrary.getLiquidity(poolManager, key.toId());
            initialLiquidity = uint256(liquidity);
        }

        uint256 blocksSinceLaunch = block.number - launchStartBlock;
        uint256 newPhase;
        if (blocksSinceLaunch <= phase1Duration) {
            newPhase = 1;
        } else if (blocksSinceLaunch <= phase1Duration + phase2Duration) {
            newPhase = 2;
        } else {
            newPhase = 3;
        }
        if (newPhase != currentPhase) {
            _resetPerAddressTracking();
            currentPhase = newPhase;
            lastPhaseUpdateBlock = block.number;
        }
        if (currentPhase == 3) {
            return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, LPFeeLibrary.OVERRIDE_FEE_FLAG);
        }
        uint256 phaseLimitBps = currentPhase == 1 ? phase1LimitBps : phase2LimitBps;
        uint256 phaseCooldown = currentPhase == 1 ? phase1Cooldown : phase2Cooldown;
        uint256 phasePenaltyBps = currentPhase == 1 ? phase1PenaltyBps : phase2PenaltyBps;
        uint256 swapAmount =
            params.amountSpecified < 0 ? uint256(-params.amountSpecified) : uint256(params.amountSpecified);
        uint256 maxSwapAmount = (initialLiquidity * phaseLimitBps) / 10000;
        bool applyPenalty = false;
        if (addressLastSwapBlock[sender] > 0) {
@>            uint256 blocksSinceLastSwap = block.number - addressLastSwapBlock[sender];
            if (blocksSinceLastSwap < phaseCooldown) {
                applyPenalty = true;
            }
        }
@>        if (!applyPenalty && addressSwappedAmount[sender] + swapAmount > maxSwapAmount) {
            applyPenalty = true;
        }

@>        addressSwappedAmount[sender] += swapAmount;
@>        addressLastSwapBlock[sender] = block.number;

        uint24 feeOverride = 0;
        if (applyPenalty) {
            feeOverride = uint24((phasePenaltyBps * 100));
        }
        return (
            BaseHook.beforeSwap.selector,
            BeforeSwapDeltaLibrary.ZERO_DELTA,
            feeOverride | LPFeeLibrary.OVERRIDE_FEE_FLAG
        );
    }

```

## Risk

**Likelihood**:

* Does not require any specific action to occur during normal swap interactions.

* High likelihood, as most users route through the same router, causing stats to hit limits much faster.

**Impact**:

* Core functionality meant to mitigate and penalize bot activity does not work.

* Regular users and bots pay penalty fees based on shared activity.

* Users are likely to leave the protocol due to this buggy logic.

## Proof of Concept

This test verifies that the hook does not track individual user addresses and only tracks the router address.

Add this snippet of code to `test/TokenLaunchHookUnit.t.sol`

```solidity
    function test_usersShare_address_tracking_stats_POC() public {
        uint256 user1SwapAmount = 0.001 ether;
        uint256 user2SwapAmount = 0.01 ether;

        vm.deal(user1, 1 ether);
        vm.startPrank(user1);
        SwapParams memory paramsUser1 = SwapParams({
            zeroForOne: true, amountSpecified: -int256(user1SwapAmount), sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        PoolSwapTest.TestSettings memory testSettingsUser1 =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});
        swapRouter.swap{value: user1SwapAmount}(key, paramsUser1, testSettingsUser1, ZERO_BYTES);
        vm.stopPrank();

        vm.deal(user2, 1 ether);
        vm.startPrank(user2);
        SwapParams memory paramsUser2 = SwapParams({
            zeroForOne: true, amountSpecified: -int256(user2SwapAmount), sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        PoolSwapTest.TestSettings memory testSettingsUser2 =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});
        swapRouter.swap{value: user2SwapAmount}(key, paramsUser2, testSettingsUser2, ZERO_BYTES);
        vm.stopPrank();

        assertEq(antiBotHook.addressSwappedAmount(user1), 0, "User1 swapped amount should be 0");

        assertEq(antiBotHook.addressSwappedAmount(user2), 0, "User2 swapped amount should be 0");

        assertEq(
            antiBotHook.addressSwappedAmount(address(swapRouter)),
            user1SwapAmount + user2SwapAmount,
            "swapRouter swapped amount should equal user1 + user2 swap amounts"
        );
    }
```

## Recommended Mitigation

Based on <https://docs.uniswap.org/contracts/v4/guides/accessing-msg.sender-using-hook>

```diff
+// Define interface
+interface IMsgSender {
+    function msgSender() external view returns (address);
+}

+// Add mapping of trusted routers
+mapping(address => bool) public verifiedRouters;

+// Add setter functions for trusted routers (with proper access control)
+function addRouter(address _router) external {
+    verifiedRouters[_router] = true;
+    // console.log("Router added:", _router);    ← remove in production
+}

+function removeRouter(address _router) external {
+    verifiedRouters[_router] = false;
+    // console.log("Router removed:", _router);  ← remove in production
+}

function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
    internal
    override
    returns (bytes4, BeforeSwapDelta, uint24)
{
+    address user = address(0);
+
+    if (verifiedRouters[sender]) {
+        try IMsgSender(sender).msgSender() returns (address swapper) {
+            user = swapper;
+        } catch {
+            revert("Router does not implement msgSender()");
+        }
+    } else {
+        user = sender;
+    }
+
+    if (user == address(0)) revert("Invalid user address");

    ...
-    if (addressLastSwapBlock[sender] > 0) {
+    if (addressLastSwapBlock[user] > 0) {
-        uint256 blocksSinceLastSwap = block.number - addressLastSwapBlock[sender];
+        uint256 blocksSinceLastSwap = block.number - addressLastSwapBlock[user];
        ...
    }

-    if (!applyPenalty && addressSwappedAmount[sender] + swapAmount > maxSwapAmount) {
+    if (!applyPenalty && addressSwappedAmount[user] + swapAmount > maxSwapAmount) {
        applyPenalty = true;
    }

-    addressSwappedAmount[sender] += swapAmount;
-    addressLastSwapBlock[sender] = block.number;
+    addressSwappedAmount[user] += swapAmount;
+    addressLastSwapBlock[user] = block.number;
    ...
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Mismatch of the current phase calculation in `_afterSwap` function and `getCurrentPhase`, providing incorrect information to user.            



# Root + Impact

## Description

* The `_afterSwap` function calculation of current phase and `getCurrentPhase` should match

* Two functions have similar calculations with small variation, the `_afterSwap` counts for phase equality between `blocksSinceLaunch` and phase durations, however `getCurrentPhase` does not, comparing only greater or lower values.

The code snippet from `_beforeSwap` function.

```solidity
   function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {

@>    if (blocksSinceLaunch <= phase1Duration) {
            newPhase = 1;
@>        } else if (blocksSinceLaunch <= phase1Duration + phase2Duration) {
            newPhase = 2;
        } else {
            newPhase = 3;
        }
 .
 .
 .
}
```

The code snippet from `getCurrentPhase` function.

```solidity
  function getCurrentPhase() public view returns (uint256) {
        if (launchStartBlock == 0) return 0;
        uint256 blocksSinceLaunch = block.number - launchStartBlock;
@>        if (blocksSinceLaunch < phase1Duration) {
            return 1;
@>        } else if (blocksSinceLaunch < phase1Duration + phase2Duration) {
            return 2;
        } else {
            return 3;
        }
    }
```

## Risk

**Likelihood**:

* Possibility is medium as there is no need for specific action for mismatch to occur as it is based on number of blocks, however the situation is rare.

**Impact**:

* Impact is low, mostly and does not cause loss of funds but may affect user decision to commit swap.

* In case of deterministic approach, getter function should reflect the actual logic if provided.

## Proof of Concept

```solidity
    function test_swapPhase_does_not_match_getCurrentPhase() public {
        vm.roll(block.number + phase1Duration + phase2Duration);

        uint256 initialPhase = antiBotHook.currentPhase();
        uint256 currentPhase = antiBotHook.getCurrentPhase();

        uint256 user1SwapAmount = 1 ether;
        vm.deal(user1, user1SwapAmount);
        vm.startPrank(user1);
        SwapParams memory paramsUser1 = SwapParams({
            zeroForOne: true, amountSpecified: -int256(user1SwapAmount), sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });
        PoolSwapTest.TestSettings memory testSettingsUser1 =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});
        swapRouter.swap{value: user1SwapAmount}(key, paramsUser1, testSettingsUser1, ZERO_BYTES);
        vm.stopPrank();

        uint256 swapPhase = antiBotHook.currentPhase();

        console.log("swap phase: ", swapPhase);
        console.log("intial phase state: ", initialPhase);
        console.log("current phase state: ", currentPhase);

        // proves that swap phase does not match getCurrentPhase()
        assertNotEq(antiBotHook.currentPhase(), antiBotHook.getCurrentPhase(), "Should be equal");
    }
```

result:

```bash
  swap phase:  2
  intial phase state:  1
  current phase state:  3
```

## Recommended Mitigation

Add equality to `getCurrentPhase` function comparison of blocks passed and phase duration.

```diff
  function getCurrentPhase() public view returns (uint256) {
        if (launchStartBlock == 0) return 0;
        uint256 blocksSinceLaunch = block.number - launchStartBlock;
-        if (blocksSinceLaunch < phase1Duration) {
+        if (blocksSinceLaunch <= phase1Duration) {
            return 1;
-        } else if (blocksSinceLaunch < phase1Duration + phase2Duration) {
+        } else if (blocksSinceLaunch <= phase1Duration + phase2Duration) {
            return 2;
        } else {
            return 3;
        }
    }
```

## <a id='M-02'></a>M-02. Inaccurate swap amount accounting, causing misapplied limits/penalties for users.            



# Root + Impact

## Description

* Uniswap has several parameters that indicate swap direction `zeroForOne` and positive/negative `amountSpecified`. Negative `amountSpecified` represents the exact amount of token0 (or token1 in case `zeroForOne=false`) the user is willing to spend for a swap, and positive `amountSpecified` represents how much of token1 (or token0 in case `zeroForOne=true`) the user wants to receive. Both negative and positive `amountSpecified` provide amounts in token-specific units, which can cause incorrect calculation of swapped amount tracking and cause users additional penalties.

* However, in the `_beforeSwap` function, the `swapAmount` calculation treats negative and positive `amountSpecified` equally, adjusting both to positive values. Meaning, in the case of exact input for output it counts the correct swap amount, but in the case of exact output for input it counts the other token amount, not the actual spend amount.

```solidity
    function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        .
        .
        .
        uint256 swapAmount =
@>            params.amountSpecified < 0 ? uint256(-params.amountSpecified) : uint256(params.amountSpecified);
        uint256 maxSwapAmount = (initialLiquidity * phaseLimitBps) / 10000;
        .
        .
        .
 }    
```

## Risk

**Likelihood**:

* Does not require specific actions, standard swap flow for exact input for output, and exact output for input.

* Occurs on any swap; the swap amount is just added to a storage slot to be used for penalty calculation later in the protocol functionality.

**Impact**:

* Can force users to hit phase limits by accounting for different token-specific amounts, or vice versa prevent users and bots from reaching limits.

## Proof of Concept

This test proves that the recorded swap amount in `addressSwappedAmount` does not account for correct swap amounts.

Add this snippet of code to `test/TokenLaunchHookUnit.t.sol`

```solidity
function test_POC_ExactOutput_SwapAmount_Misaccounted() public {
        uint256 amountOutToken = 1_000_000; // 1 token with 6 decimals

        vm.deal(user1, 1 ether);
        vm.startPrank(user1);

        SwapParams memory params = SwapParams({
            zeroForOne: true, amountSpecified: int256(amountOutToken), sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });

        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        uint256 recordedBefore = antiBotHook.addressSwappedAmount(address(swapRouter));
        BalanceDelta delta = swapRouter.swap{value: 1 ether}(key, params, testSettings, ZERO_BYTES);
        uint256 recordedAfter = antiBotHook.addressSwappedAmount(address(swapRouter));

        uint256 recordedSwapAmount = recordedAfter - recordedBefore;
        uint256 actualInputEth = uint256(int256(-delta.amount0()));

        assertEq(recordedSwapAmount, amountOutToken, "Recorded swap amount uses amountSpecified");
        assertTrue(actualInputEth != amountOutToken, "Actual input differs from recorded amount");

        console.log("actualInputEth:     ",actualInputEth);
        console.log("recordedSwapAmount: ",recordedSwapAmount);

        vm.stopPrank();
    }
```

## Recommended Mitigation

Adjust `addressSwappedAmount` based on the `afterSwap` `BalanceDelta`. It would not fully remove the risk of miscalculation, but it would definitely reduce the number of negative scenarios.

```diff
    function _beforeSwap(address sender, PoolKey calldata key, SwapParams calldata params, bytes calldata)
        internal
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        .
        .
        .
-        addressSwappedAmount[sender] += swapAmount;
-        addressLastSwapBlock[sender] = block.number;
+        if(params.amountSpecified < 0) {
+            addressSwappedAmount[sender] += swapAmount;
+            addressLastSwapBlock[sender] = block.number;
         }
        .
        .
        .
    }
.
.
.
+ function _afterSwap(address, PoolKey calldata, SwapParams calldata, BalanceDelta delta, bytes calldata)
+         internal
+         virtual
+         returns (bytes4, int128)
+     {
+     if(params.amountSpecified > 0) {
+            addressSwappedAmount[sender] += delta.amount1();
+            addressLastSwapBlock[sender] = block.number;
+          }
+     }
```






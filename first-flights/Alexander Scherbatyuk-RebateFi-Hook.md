# RebateFi Hook - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The `ReFiSwapRebateHook::_beforeInitialize` function check allow ReFi as currency 1 token only, preventing other tokens to be used as currency 1 token and currency 0 token to be ReFi token](#H-01)
    - ### [H-02. he `ReFiSwapRebateHook::_isReFiBuy` returns wrong value, causing `ReFiSwapRebateHook::_beforeSwap` function to apply fee on ReFi token buy.](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Unchecked transfer in the `ReFiSwapRebateHook::withdrawTokens`, does not check return value](#M-01)
- ## Low Risk Findings
    - ### [L-01. The `ReFiSwapRebateHook::withdrawTokens` function emits `TokensWithdrawn` event with wrong parameters order.](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #53

### Dates: Nov 20th, 2025 - Nov 27th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-11-rebatefi-hook)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. The `ReFiSwapRebateHook::_beforeInitialize` function check allow ReFi as currency 1 token only, preventing other tokens to be used as currency 1 token and currency 0 token to be ReFi token            



# ReFi must be one of the two pool tokens → Only ReFi as currency1 is accepted due to duplicated condition

## Description

* The purpose of `_beforeInitialize` in `ReFiSwapRebateHook` is to ensure that the ReFi token is present in every pool that uses this hook (so rebates can be paid in ReFi).

* The current check contains a copy-paste error: it compares `currency1` with ReFi twice, making the condition always true when `currency1` is not ReFi and completely ignoring `currency0`. As a result, pools where ReFi is `currency0` (the conventional ordering for most pairs) are rejected, while only pools with ReFi as `currency1` are allowed.

```solidity
    // Root cause in the codebase with @> marks to highlight the relevant section
    function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
@>        if (Currency.unwrap(key.currency1) != ReFi && Currency.unwrap(key.currency1) != ReFi) {
            revert ReFiNotInPool();
        }
        return BaseHook.beforeInitialize.selector;
    }
```

## Risk

**Likelihood**:

* Every pool creation where ReFi is sorted as currency0 (the vast majority of pairs, because Uniswap V4 canonical ordering puts the lower-address token as currency0) triggers the revert

* Any front-end or script that follows the standard token ordering will fail to create the pool

**Impact**:

* Intended pools (e.g., USDC/ReFi, WETH/ReFi) cannot be initialized when ReFi has the higher address and therefore becomes currency1

* The hook becomes effectively limited to only exotic pairs where ReFi is the lower-address token, severely restricting usability

* Users and integrators waste time and gas on failed transactions

## Proof of Concept

Add the following code snippet to the `RebateFiHookTest.t.sol` test file.

This snippet of code is to demonstrate that the pool will revert when trying to initialize the pool with ReFi token as currency 0 and ERC20 token as currency 1.

```solidity
function test_ReFiNotInPool() public {
        // Deploy the ReFi token
        reFiToken = new MockERC20("ReFi Token", "ReFi", 18);
        reFiCurrency = Currency.wrap(address(reFiToken));

        // Deploy the ERC20 token
        token = new MockERC20("TOKEN", "TKN", 18);
        tokenCurrency = Currency.wrap(address(token));

        // Get creation code for hook
        bytes memory creationCode = type(ReFiSwapRebateHook).creationCode;
        bytes memory constructorArgs = abi.encode(manager, address(reFiToken));

        // Find a salt that produces a valid hook address
        uint160 flags = uint160(Hooks.BEFORE_INITIALIZE_FLAG | Hooks.AFTER_INITIALIZE_FLAG | Hooks.BEFORE_SWAP_FLAG);

        (address hookAddress, bytes32 salt) = HookMiner.find(address(this), flags, creationCode, constructorArgs);

        // Deploy the hook with the mined salt
        rebateHook = new ReFiSwapRebateHook{salt: salt}(manager, address(reFiToken));
        require(address(rebateHook) == hookAddress, "Hook address mismatch");

        // Expect revert when trying to initialize the pool with ReFi token as currency 0 and ERC20 token as currency 1
        vm.expectRevert();
        // Initialize the pool with ReFi token as currency 0 and ERC20 token as currency 1 using DYNAMIC_FEE_FLAG
        (key,) = initPool(reFiCurrency, tokenCurrency, rebateHook, LPFeeLibrary.DYNAMIC_FEE_FLAG, SQRT_PRICE_1_1_s);
    }
```

## Recommended Mitigation

Possible mitigation is to check if the currency 0 token is ReFi.

```diff
    function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
-	    if (Currency.unwrap(key.currency1) != ReFi && Currency.unwrap(key.currency1) != ReFi) {
+       if (Currency.unwrap(key.currency1) != ReFi && Currency.unwrap(key.currency0) != ReFi) {
            revert ReFiNotInPool();
        }
        return BaseHook.beforeInitialize.selector;
    }
```

## <a id='H-02'></a>H-02. he `ReFiSwapRebateHook::_isReFiBuy` returns wrong value, causing `ReFiSwapRebateHook::_beforeSwap` function to apply fee on ReFi token buy.            



# Incorrect ReFi buy detection → Fees and ReFiSold event are applied when users buy ReFi (opposite of intended behavior)

## Description

* The hook is designed to incentivise buying ReFi by charging little/no fee on ReFi purchases and a higher fee when users sell ReFi.

* `_isReFiBuy` determines whether the current swap direction is a ReFi purchase. It returns `true` when the user is receiving ReFi (buy) and `false` when the user is sending ReFi (sell).

* Due to the logic being inverted in both branches, the function returns the opposite value in every possible pool configuration, causing `_beforeSwap` to apply the sell fee and emit `ReFiSold` whenever a user buys ReFi, and vice-versa.

```solidity
    // Root cause in the codebase with @> marks to highlight the relevant section
    function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {
@>        bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
        if (IsReFiCurrency0) {
            return zeroForOne;
        } else {
            return !zeroForOne;
        }
```

## Risk

**Likelihood**:

* Every swap executed on any pool that contains ReFi triggers the bug

* The inversion affects 100% of buy and sell transactions regardless of token ordering

**Impact**:

* Users are charged the higher “sell” fee when they buy ReFi — directly contradicting the core product incentive

* The ReFiSold event is emitted on buys instead of sells, making on-chain analytics and rebate accounting completely backwards

## Proof of Concept

Add the following code snippet to the `RebateFiHookTest.t.sol` test file.

This snippet of code is to demonstrate that on ReFi buy the `_beforeSwap` function will apply fee on ReFi token buy and emit the `ReFiSold` event.

```solidity
    function test_BuyReFi_EmitsReFiSoldEvent() public {
        uint256 ethAmount = 0.01 ether;
        // Fund user and record initial balances
        vm.deal(user1, 1 ether);

        //rebateHook.ChangeFee(true, 0, true, 0);
        (uint24 buyFee, uint24 sellFee) = rebateHook.getFeeConfig();
        console.log("buyFee", buyFee);
        console.log("sellFee", sellFee);

        vm.startPrank(user1);

        SwapParams memory params = SwapParams({
            zeroForOne: true, // ETH -> ReFi
            amountSpecified: -int256(ethAmount),
            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
        });

        PoolSwapTest.TestSettings memory testSettings =
            PoolSwapTest.TestSettings({takeClaims: false, settleUsingBurn: false});

        vm.recordLogs();
        // Perform swap
        swapRouter.swap{value: ethAmount}(key, params, testSettings, ZERO_BYTES);

        Vm.Log[] memory entries = vm.getRecordedLogs();
        bool foundReFiSold = false; // Move outside the loop
        bytes32 expectedSig = keccak256("ReFiSold(address,uint256,uint256)");

        for (uint256 i = 0; i < entries.length; i++) {
            Vm.Log memory logEntry = entries[i];
            console.log("logEntry", address(uint160(uint256(logEntry.topics[1]))));

            // Check if this log entry matches the ReFiSold event signature
            if (logEntry.topics.length > 0 && logEntry.topics[0] == expectedSig) {
                foundReFiSold = true;
                break; // Found it, no need to continue
            }
        }

        // Assert after checking all logs
        assertTrue(foundReFiSold, "ReFiSold event not emitted in logs");
        vm.stopPrank();
    }
```

## Recommended Mitigation

Possible mitigation is to modify the `_isReFiBuy` function according.

```diff
    function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {
        bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
        if (IsReFiCurrency0) {
-					  return zeroForOne;
+					  return !zeroForOne;
        } else {
-					  return !zeroForOne;
+					  return zeroForOne;
        }
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unchecked transfer in the `ReFiSwapRebateHook::withdrawTokens`, does not check return value            



# Unchecked ERC20 transfer in withdrawTokens → Silent failure + incorrect event emission on failed withdrawals

## Description

* The `withdrawTokens` function is intended to allow the owner to withdraw any ERC20 tokens collected by the hook (e.g., fees or rebates).

* The current implementation calls `IERC20(token).transfer(to, amount)` directly without checking the returned `bool`. Many ERC20 tokens (USDT, older tokens, some fee-on-transfer tokens) return `false` instead of reverting on failure, or have non-standard behavior.

* As a result, a failed transfer is treated as successful and the `TokensWithdrawn` event is still emitted, creating an inconsistent on-chain state.

```solidity
// Root cause in the codebase with @> marks to highlight the relevant section
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
@>        IERC20(token).transfer(to, amount);
        emit TokensWithdrawn(to, token, amount);
    }
```

## Risk

**Likelihood**:

* Any non-standard or legacy ERC20 token deposited into the hook will trigger the issue

* Tokens that return false on failure (e.g., USDT before CPI upgrade, certain proxy tokens) are still widely used

* Fee-on-transfer or rebasing tokens may also cause transfer to return false

**Impact**:

* Tokens can become permanently stuck in the hook if the transfer silently fails

* The `TokensWithdrawn` event lies about a successful withdrawal, breaking off-chain accounting and monitoring

* Owner may believe funds were sent when they were not, leading to loss of funds or delayed recovery attempts

## Proof of Concept

Add the following code snippet to the `RebateFiHookTest.t.sol` test file.

This is a mock ERC20 token that will revert on the first transfer. Add this contract to the `RebateFiHookTest.t.sol` file.

```solidity
contract FailedMockERC20 is MockERC20 {
    uint256 public countTransfers;

    constructor(string memory name, string memory symbol, uint8 decimals) MockERC20(name, symbol, decimals) {}

    function mint(address to, uint256 amount) public override {
        _mint(to, amount);
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        if (countTransfers == 0) {
            countTransfers++;
            return super.transfer(to, amount);
        }
        return false;
    }
}
```

This is a test function that will test the `ReFiSwapRebateHook::withdrawTokens` function with a failed token. Add this test to the `RebateFiHookTest.t.sol` file.

```solidity
    function test_WithdrawTokens_Failed() public {
        FailedMockERC20 failedToken = new FailedMockERC20("Failed", "FAIL", 18);
        uint256 transferAmount = 1 ether;

        failedToken.mint(address(this), transferAmount);
        failedToken.approve(address(rebateHook), transferAmount);

        failedToken.transfer(address(rebateHook), transferAmount);

        uint256 initialBalance = failedToken.balanceOf(address(this));
        uint256 hookBalanceBefore = failedToken.balanceOf(address(rebateHook));

        console.log("initialBalance", initialBalance);
        console.log("hookBalanceBefore", hookBalanceBefore);

        uint256 withdrawAmount = 0.5 ether;
        rebateHook.withdrawTokens(address(failedToken), address(this), withdrawAmount);

        uint256 finalBalance = failedToken.balanceOf(address(this));

        console.log("finalBalance", finalBalance);

        assertEq(finalBalance, initialBalance + withdrawAmount, "Should receive withdrawn tokens");
    }
```

## Recommended mitigation

ERC20 functions may not behave as expected. For example: return values are not always meaningful. It is recommended to use OpenZeppelin's SafeERC20 library.

```diff
+	import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
.
.
.
	contract ReFiSwapRebateHook is BaseHook, Ownable {
+    using SafeERC20 for IERC20;
.
.
.
       function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
-        IERC20(token).transfer(to, amount);
+        IERC20(token).safeTransfer(to, amount);
        emit TokensWithdrawn(token, to, amount);
       }
	}
	
```


# Low Risk Findings

## <a id='L-01'></a>L-01. The `ReFiSwapRebateHook::withdrawTokens` function emits `TokensWithdrawn` event with wrong parameters order.            



# TokensWithdrawn event emitted with swapped parameters, indexers and frontends parse the wrong addresses (token vs recipient)

## Description

* The contract defines the `TokensWithdrawn` event with the standard and expected parameter order:\
  `event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);`

* However, when emitting, the arguments are passed in reverse order (`to, token, amount`).

* As a result, off-chain tools that rely on the event signature and ABI (The Graph, Dune, Etherscan, wallets, internal dashboards) will interpret the first indexed topic as the token address being the recipient, and the second as the token address — completely swapping the meaning.

```solidity
// Root cause in the codebase with @> marks to highlight the relevant section
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).transfer(to, amount);
@>        emit TokensWithdrawn(to, token, amount);
    }
```

## Risk

**Likelihood**:

* Every successful withdrawal by the owner triggers the incorrectly ordered event

* All existing and future indexers already use the declared event ABI

**Impact**:

* Analytics show withdrawals going to the token contract address and tokens being sent to the actual recipient

* Monitoring alerts (e.g., “large withdrawal of USDC”) fire on the wrong address

* Frontends and explorers display misleading or completely incorrect withdrawal data

* Breaks compatibility with any tool that auto-parses this common event pattern

## Proof of Concept

Event parameters should be in the defined order.

```Solidity
event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);
```

## Recommended Mitigation

Change the event emission to the correct order: first the `token` address, then the `to` receiver address, and finally the amount.

```diff
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).transfer(to, amount);
-		emit TokensWithdrawn(to, token, amount);
+       emit TokensWithdrawn(token, to, amount);
    }
```




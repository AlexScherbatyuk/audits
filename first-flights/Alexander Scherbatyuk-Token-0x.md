# Token-0x - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Missing Balance Check in `ERC20Internals::_burn` Function, causing underflow inflating balance and total supply.](#H-01)

- ## Low Risk Findings
    - ### [L-01. Contract Should Be Abstract - Missing Public Mint/Burn Functions](#L-01)
    - ### [L-02. Missing Transfer Events in _mint and _burn - ERC20 Standard Violation](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #54

### Dates: Dec 4th, 2025 - Dec 11th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-12-token0x)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Missing Balance Check in `ERC20Internals::_burn` Function, causing underflow inflating balance and total supply.            



# Root + Impact

## Description

**Description:** The `_burn` function does not verify that the account has sufficient balance before burning tokens. This allows burning more tokens than the account holds, which can lead to:

* **Underflow in balance calculation** - While Solidity 0.8+ will revert on underflow, the explicit check should be present for clarity and to emit proper errors

* **Invariant violation** - Breaks the critical ERC20 invariant: `sum(userBalances) == totalSupply`

* **State corruption** - If balance underflows, the totalSupply will be reduced but the balance becomes corrupted

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
    function _mint(address account, uint256 value) internal {
        assembly ("memory-safe") {
            if iszero(account) {
                mstore(0x00, shl(224, 0xec442f05))
                mstore(add(0x00, 4), 0x00)
                revert(0x00, 0x24)
            }

            let ptr := mload(0x40)
            let balanceSlot := _balances.slot
            let supplySlot := _totalSupply.slot

 @>           let supply := sload(supplySlot)
 @>           sstore(supplySlot, add(supply, value))

            mstore(ptr, account)
            mstore(add(ptr, 0x20), balanceSlot)

 @>           let accountBalanceSlot := keccak256(ptr, 0x40)
 @>           let accountBalance := sload(accountBalanceSlot)
            sstore(accountBalanceSlot, add(accountBalance, value))
        }
    }
```

## Risk

**Likelihood**:

* Direct call to the function with an incorrect amount.

* Any access restrictions would not prevent this situation, since the internal function does not mitigate such an error.

**Impact**:

* Users can burn more tokens than they own, corrupting the token's accounting

* Protocols relying on the invariant `sum(balances) == totalSupply` will break

* Total supply can become less than the sum of all balances

* In Yul assembly, `sub()` will wrap around on underflow (not revert), potentially causing silent corruption

## Proof of Concept

This test demonstrates the overflow in the \`\_burn\` function.

```Solidity
    function test_burnMoreThanBalance() public {
        uint256 amount = 100e18;
        address account = makeAddr("account");
        token.mint(account, amount);

        uint256 balance = token.balanceOf(account);
        assertEq(balance, amount);
        assertEq(token.totalSupply(), amount);

        token.burn(account, amount + 1);
        balance = token.balanceOf(account);
        assertEq(balance, type(uint256).max);
    }
```

## Recommended Mitigation

Add a balance check in the \_burn function before burning tokens to prevent underflow.

```diff
    function _burn(address account, uint256 value) internal {
        assembly ("memory-safe") {
            if iszero(account) {
                mstore(0x00, shl(224, 0x96c6fd1e))
                mstore(add(0x00, 4), 0x00)
                revert(0x00, 0x24)
            }

            let ptr := mload(0x40)
            let balanceSlot := _balances.slot
            let supplySlot := _totalSupply.slot

-           let supply := sload(supplySlot)
-           sstore(supplySlot, sub(supply, value))

-           mstore(ptr, account)
-           mstore(add(ptr, 0x20), balanceSlot)

-           let accountBalanceSlot := keccak256(ptr, 0x40)
-           let accountBalance := sload(accountBalanceSlot)
+
+           // Load account balance first
+           mstore(ptr, account)
+           mstore(add(ptr, 0x20), balanceSlot)
+           let accountBalanceSlot := keccak256(ptr, 0x40)
+           let accountBalance := sload(accountBalanceSlot)
+
+           // Check balance is sufficient
+           if lt(accountBalance, value) {
+            mstore(0x00, shl(224, 0xe450d38c))
+            mstore(add(0x00, 4), account)
+            mstore(add(0x00, 0x24), accountBalance)
+            mstore(add(0x00, 0x44), value)
+            revert(0x00, 0x64)
+           }
+
+          let supply := sload(supplySlot)
+          sstore(supplySlot, sub(supply, value))
+          sstore(accountBalanceSlot, sub(accountBalance, value))
        }
    }
```

    


# Low Risk Findings

## <a id='L-01'></a>L-01. Contract Should Be Abstract - Missing Public Mint/Burn Functions            



# Root + Impact

## Description

The `ERC20` contract is not marked as `abstract`, yet it lacks public `mint` and `burn` functions. This means:

* The contract can be deployed directly, but will have 0 total supply with no way to create tokens

* The contract is essentially unusable when deployed standalone

* This contradicts the intended design pattern (similar to OpenZeppelin's ERC20) where it should be used as a base class

For example: OpenZeppelin's ERC20 implementation is explicitly marked as abstract with documentation stating: "This implementation is agnostic to the way tokens are created. This means that a supply mechanism has to be added in a derived contract using {\_mint}."

```solidity
// Root cause in the codebase with @> marks to highlight the relevant section
```

## Risk

**Likelihood**:

* Nothing preventing user to driectly deploy the contract expecting them to work standalone

**Impact**:

* Users can deploy a useless token contract (0 supply, no way to mint)

* Confusion about intended usage pattern

* Inconsistency with industry standards (OpenZeppelin pattern)

## Proof of Concept

This test verifies that a token can be deployed directly with 0 supply and there is no way to mint more tokens after deployment.

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {ERC20} from "../src/ERC20.sol";

contract Token0xTest is Test {
    ERC20 public token;

    function setUp() public {
        token = new ERC20("Token0x", "TKN0x");
    }

    function test_metadataAndMintFunctionsExists() public {
        assertEq(token.name(), "Token0x");
        assertEq(token.symbol(), "TKN0x");
        assertEq(token.decimals(), 18);
        assertEq(token.totalSupply(), 0);

        (bool success,) =
            address(token).call(abi.encodeWithSignature("mint(address,uint256)", makeAddr("account"), 100e18));
        assertEq(success, true);
    }
}

```

## Recommended Mitigation

Mark the contract as `abstract`, and extend the token documentation with the note: "This implementation is agnostic to the way tokens are created. This means that a supply mechanism has to be added in a derived contract using `_mint`."

```diff
- contract ERC20 is IERC20Errors, ERC20Internals, IERC20 {
+ abstract contract ERC20 is IERC20Errors, ERC20Internals, IERC20 {
    // ... rest of implementation
}
```

## <a id='L-02'></a>L-02. Missing Transfer Events in _mint and _burn - ERC20 Standard Violation            



# Root + Impact

## Description

According to the ERC20 standard (EIP-20), minting and burning operations **MUST** emit `Transfer` events

* Minting **SHOULD** emit: `Transfer(address(0), account, value)` - "A token contract which creates new tokens SHOULD trigger a Transfer event with the \_from address set to 0x0 when tokens are created."
* Burning **SHOULD** emit: `Transfer(account, address(0), value)` - When tokens are destroyed, should emit Transfer with `_to = 0x0`

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
  function _mint(address account, uint256 value) internal {
.
.
.
@>          sstore(accountBalanceSlot, add(accountBalance, value))
@>          // emit event
  }
.
.
.
    function _burn(address account, uint256 value) internal {
.
.
.
@>         sstore(accountBalanceSlot, sub(accountBalance, value))
@>        // emit event
        }
    }
```

## Risk

**Likelihood**:

* Any direct or indirect call to the \_mint function.

* Any direct or indirect call to the \_burn function.

**Impact**:

* **Standard Compliance Violation** - Direct violation of ERC20 standard recommendations
* **Off-chain systems broken** - Indexers, block explorers, and analytics tools cannot track mint/burn operations
* **Incorrect token supply** - Wallets and DApps will show wrong total supply (missing minted tokens, including burned tokens)
* **DeFi integration failures** - Protocols that monitor Transfer events for supply changes will fail
* **User confusion** - Tokens can appear/disappear without traceable events
* **Audit trail broken** - No on-chain record of token creation/destruction

## Proof of Concept

These tests demonstrate that the \_mint and \_burn functions do not emit the standard Transfer event.

```Solidity

    function test_mintEmitsTransferEvent() public {
        uint256 amount = 100e18;
        address account = makeAddr("account");
        vm.expectEmit(true, true, false, true);
        emit Transfer(address(0), account, amount);
        token.mint(account, amount);
    }

    function test_burnEmitsTransferEvent() public {
        uint256 amount = 100e18;
        address account = makeAddr("account");
        token.mint(account, amount);
        vm.expectEmit(true, true, false, true);
        emit Transfer(address(0), account, amount);
        token.burn(account, amount);
    }
```

## Recommended Mitigation

Add Transfer event emissions to the `_burn` and `_mint` functions.

```diff
    function _burn(address account, uint256 value) internal {
.
.
.
            sstore(accountBalanceSlot, sub(accountBalance, value))
+           mstore(ptr, value)
+           log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, account, 0)
        }
    }


    function _mint(address account, uint256 value) internal {
.
.
.
            sstore(accountBalanceSlot, add(accountBalance, value))
+           mstore(ptr, value)
+           log3(ptr, 0x20, 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef, 0, account)
        }
    }
```




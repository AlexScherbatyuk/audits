# MultiSig Timelock - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Timelock bypass via multiple small transactions, reducing protocol security and breaking the timelock functionality.](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Revoked Signer's Confirmations Remain Active on Pending Transactions](#M-01)
    - ### [M-02. No transaction cancellation mechanism, no way to prevent transfer to a malicious address.](#M-02)
- ## Low Risk Findings
    - ### [L-01. OpenZeppelin AccessControl `renounceRole()` function not implemented, causing ghost signers.](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #55

### Dates: Dec 18th, 2025 - Dec 25th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-12-multisig-timelock)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 2
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Timelock bypass via multiple small transactions, reducing protocol security and breaking the timelock functionality.            



# Root + Impact

## Description

Large transactions require multi-day timelocks, but the timelock can be completely bypassed by splitting large amounts into many small transactions (<1 ETH each), which have NO timelock.

```solidity
    function _getTimelockDelay(uint256 value) internal pure returns (uint256) {
        uint256 sevenDaysTimeDelayAmount = 100 ether;
        uint256 twoDaysTimeDelayAmount = 10 ether;
        uint256 oneDayTimeDelayAmount = 1 ether;

        if (value >= sevenDaysTimeDelayAmount) {
            return SEVEN_DAYS_TIME_DELAY;
        } else if (value >= twoDaysTimeDelayAmount) {
            return TWO_DAYS_TIME_DELAY;
        } else if (value >= oneDayTimeDelayAmount) {
            return ONE_DAY_TIME_DELAY;
        } else {
@>            return NO_TIME_DELAY;
        }
    }
```

## Risk

**Likelihood**:

* Splitting the transaction value among several transactions does not directly violate any protocol terms, as the protocol only enforces timelocks on a single transaction's value.

* Such a case can occur intentionally to avoid a longer timelock, or unintentionally, as proposed transactions can accumulate over time.

**Impact**:

* Complete bypass of the timelock protection

* High-value transactions can be executed immediately

* Emergency response window eliminated

* Timelock becomes security theater

## Proof of Concept

Add this snipped of code to the `MultiSigTimelockTest.t.sol` test file.

```solidity
function test_timelockBypassWithSplitedTransactions() public grantSigningRoles {
        // ARRANGE
        vm.deal(address(multiSigTimelock), OWNER_BALANCE_THREE);

        console2.log("SPENDER_ONE balance before: ", SPENDER_ONE.balance);
        // first transaction proposal below 10 ether
        int256 txnId;
        for (uint256 i = 0; i < 10; i++) {
            vm.prank(OWNER);
            uint256 txnId = multiSigTimelock.proposeTransaction(SPENDER_ONE, 0.5 ether, hex"");

            // ACT
            // get 3 / 5 confirmations for first transaction
            vm.prank(OWNER);
            multiSigTimelock.confirmTransaction(txnId);
            vm.prank(SIGNER_TWO);
            multiSigTimelock.confirmTransaction(txnId);
            vm.prank(SIGNER_THREE);
            multiSigTimelock.confirmTransaction(txnId);

            // execute transaction immediately without timelock
            vm.prank(OWNER);
            multiSigTimelock.executeTransaction(txnId);
        }

        console2.log("SPENDER_ONE balance after: ", SPENDER_ONE.balance);
        assertEq(SPENDER_ONE.balance, 5 ether, "SPENDER_ONE should have received 5 ether in total");
    }
```

Even though the example transfers only 5 ether rather than 100, it still uncovers the opportunity to bypass any value eventually.

```bash
forge test --mt test_timelockBypassWithSplitedTransactions -vvvv
```

## Recommended Mitigation

Enforce a minimum timelock for all transactions to implement security measures.

```diff
+ uint256 private constant MINIMUM_TIMELOCK = 1 hours;
.
.
.
    function _getTimelockDelay(uint256 value) internal pure returns (uint256) {
        uint256 sevenDaysTimeDelayAmount = 100 ether;
        uint256 twoDaysTimeDelayAmount = 10 ether;
        uint256 oneDayTimeDelayAmount = 1 ether;

        if (value >= sevenDaysTimeDelayAmount) {
            return SEVEN_DAYS_TIME_DELAY;
        } else if (value >= twoDaysTimeDelayAmount) {
            return TWO_DAYS_TIME_DELAY;
        } else if (value >= oneDayTimeDelayAmount) {
            return ONE_DAY_TIME_DELAY;
        } else {
-           return NO_TIME_DELAY;
+           return MINIMUM_TIMELOCK;
        }
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Revoked Signer's Confirmations Remain Active on Pending Transactions            



# Root + Impact

## Description

When `revokeSigningRole()` is called to remove a compromised or malicious signer, their existing confirmations on pending transactions remain active. The `s_signatures[txnId][signer]` mapping is never cleared, and the `confirmations` counter is never decremented. This allows revoked signers to contribute to transaction execution post-revocation.

```solidity
function revokeSigningRole(address _account) external nonReentrant onlyOwner noneZeroAddress(_account) {
    .
    .
    .
        // Clear the last position and decrement count
        s_signers[s_signerCount - 1] = address(0);
        s_signerCount -= 1;

@>      // MISSING: No cleanup of existing confirmations
@>      // MISSING: No loop through s_signatures[txnId][_account]
@>      // MISSING: No decrement of s_transactions[txnId].confirmations

        s_isSigner[_account] = false;
        _revokeRole(SIGNING_ROLE, _account);
    }
```

## Risk

**Likelihood**:

* Direct call of the protocol function revokeSigningRole() (standard flow).

**Impact**:

* Compromised signers can help malicious transactions even after revocation

* Security assumption broken: "only active signers can confirm"

* Transactions may have fewer real confirmations than `REQUIRED_CONFIRMATIONS`

* No way to audit which confirmations came from currently-active signers

## Proof of Concept

Add this snipped of code to the `MultiSigTimelockTest.t.sol` test file.

```solidity
function test_RevokeSignerCheckTransactionConfirmationRemains() public grantSigningRoles {
        // ARRANGE
        vm.deal(address(multiSigTimelock), OWNER_BALANCE_TWO);

        // first transaction proposal
        vm.prank(OWNER);
        uint256 txnId = multiSigTimelock.proposeTransaction(SPENDER_ONE, OWNER_BALANCE_TWO, hex"");

        // ACT
        // get 3 / 5 confirmations for first transaction
        vm.prank(OWNER);
        multiSigTimelock.confirmTransaction(txnId);
        vm.prank(SIGNER_TWO);
        multiSigTimelock.confirmTransaction(txnId);
        vm.prank(SIGNER_THREE);
        multiSigTimelock.confirmTransaction(txnId);

        vm.prank(OWNER);
        multiSigTimelock.revokeSigningRole(SIGNER_TWO);

        // ASSERT
        MultiSigTimelock.Transaction memory txn = multiSigTimelock.getTransaction(txnId);
        assertEq(txn.confirmations, 3, "Confirmations should remain 3 after revoking a signer");
    }
```

How to execute:

```bash
forge test --mt test_RevokeSignerCheckTransactionConfirmationRemains -vvvv
```

## Recommended Mitigation

Clear all pending confirmations from this signer

```diff
function revokeSigningRole(address _account) external nonReentrant onlyOwner {
    // ... existing validation ...
    
+    // Clear all pending confirmations from this signer
+     for (uint256 txnId = 0; txnId < s_transactionCount; txnId++) {
+         if (!s_transactions[txnId].executed && s_signatures[txnId][_account]) {
+             s_signatures[txnId][_account] = false;
+             s_transactions[txnId].confirmations -= 1;
+             emit TransactionRevoked(txnId, _account);
+         }
+     }
    
    // ... rest of function
}
```

## <a id='M-02'></a>M-02. No transaction cancellation mechanism, no way to prevent transfer to a malicious address.            



# Root + Impact

## Description

The current statistics on protocol funds lost confirm that, even with multisig, such attacks succeed. Functionality for emergency blocking or cancellation of a transaction is required.

```solidity
@> // function cancelTransaction(uint256 txnId) - does not exist

```

## Risk

**Likelihood**:

* Even though the likelihood is low by itself, human errors do happen.

* Not every signer might be suspicious of every transaction; transactions might be confirmed without a check.

* Also, an attack vector to trick a signer into transferring funds to a malicious address that resembles a previously verified one has become common and, in some cases, successful.

**Impact**:

* Loss of funds intended for internal protocol transfers might cause severe damage to the protocol, as such transfers are by nature less suspicious.

* Cannot recover from a compromised owner.

* Erroneous transactions cannot be cancelled.

* Must trust a single signer to never confirm malicious transactions.

* Pending malicious transactions remain a permanent threat.

## Proof of Concept

1. One of the signers has been tricked into transferring funds to a malicious address that looks like the intended one.
2. It gets 2 confirmations before detection.
3. No way to cancel – must rely on the 3rd signer indefinitely.
4. Accidents or errors in transaction parameters cannot be fixed.

## Recommended Mitigation

Allow any signer to cancel suspicious transactions before they are fully confirmed.

```diff
+ function cancelTransaction(uint256 txnId) 
+     external 
+    onlyRole(SIGNING_ROLE) 
+     transactionExists(txnId) 
+     notExecuted(txnId) 
+ {
+     require(
+         s_transactions[txnId].confirmations < REQUIRED_CONFIRMATIONS,
+         "Cannot cancel fully confirmed transaction"
+    );
+     
+     // Clear all confirmations and signatures
+     for (uint256 i = 0; i < s_signerCount; i++) {
+         address signer = s_signers[i];
+         if (s_signatures[txnId][signer]) {
+             s_signatures[txnId][signer] = false;
+         }
+     }
+     
+     s_transactions[txnId].confirmations = 0;
+     s_transactions[txnId].executed = true;  // Mark as cancelled
+     
+     // Release reserved funds if implemented
+     s_totalReserved -= s_reservedFunds[txnId];
+    delete s_reservedFunds[txnId];
+     
+     emit TransactionCancelled(txnId);
+ }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. OpenZeppelin AccessControl `renounceRole()` function not implemented, causing ghost signers.            



# Root + Impact

## Description

OpenZeppelin's `AccessControl` allows any role holder to call `renounceRole(bytes32 role, address account)` where `account` must be `msg.sender`. When a signer calls this function, they lose the `SIGNING_ROLE` but remain in the `s_signers[]` array with `s_isSigner[signer] = true` and are still counted in `s_signerCount`. This creates "gost signers" who:

* Cannot confirm transactions (no role)

* Still count toward `MAX_SIGNER_COUNT`

* Block addition of new functional signers

* Are not removed from `s_signers[]` array

```solidity
// Opene Zeppelin AccessControl library
@>    function renounceRole(bytes32 role, address callerConfirmation) public virtual {
        if (callerConfirmation != _msgSender()) {
            revert AccessControlBadConfirmation();
        }

        _revokeRole(role, callerConfirmation);
    }
```

## Risk

**Likelihood**:

* The `renounceRole()` function is a standard function from the inherited library, therefore it could be called intentionally or unintentionally.

**Impact**:

* Only the OpenZeppelin implementation of a role is renounced, leaving all state variables unchanged; the signer itself remains active but useless.

* New signers cannot be added when compromised signers exist.

* Reduces effective signer count (5→4→3).

* At 3 signers, becomes 3-of-3 (no redundancy, single point of failure).

* Ghost signers create confusion and management overhead.

* Inconsistent state between AccessControl and multisig tracking

## Proof of Concept

Add this snipped of code to the `MultiSigTimelockTest.t.sol` test file.

```solidity
    function test_signerCallsRenounceRoleGostSignerRemains() public grantSigningRoles {
        // ARRANGE
        // ACT
        vm.startPrank(SIGNER_TWO);
        multiSigTimelock.renounceRole(multiSigTimelock.getSigningRole(), SIGNER_TWO);
        vm.stopPrank();

        // ASSERT
        // after renounce signers still counted
        assertEq(multiSigTimelock.getSignerCount(), 5, "Should still be 5 signers");
        assertFalse(multiSigTimelock.hasRole(multiSigTimelock.getSigningRole(), SIGNER_TWO));
    }
```

How to execute:

```bash
forge test --mt test_signerCallsRenounceRoleGhostSignerRemains -vvvv
```

## Recommended Mitigation

Override \`renounceRole\` to properly delete signer from the protocol.

```diff
+ function renounceRole(bytes32 role, address account) public virtual override {
+     require(account == msg.sender, "Can only renounce own roles");
+     
+     if (role == SIGNING_ROLE) {
+         // Integrate with multisig state
+         if (s_signerCount <= 1) {
+             revert MultiSigTimelock__CannotRevokeLastSigner();
+         }
+         
+         // Find and remove from array (same logic as revokeSigningRole)
+         uint256 indexToRemove = type(uint256).max;
+         for (uint256 i = 0; i < s_signerCount; i++) {
+             if (s_signers[i] == account) {
+                 indexToRemove = i;
+                 break;
+             }
+         }
+         
+         if (indexToRemove < s_signerCount - 1) {
+             s_signers[indexToRemove] = s_signers[s_signerCount - 1];
+         }
+         s_signers[s_signerCount - 1] = address(0);
+         s_signerCount -= 1;
+         s_isSigner[account] = false;
+     }
+     
+     super.renounceRole(role, account);
+ }
```




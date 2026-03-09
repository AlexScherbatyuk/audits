# Company Simulator - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Predictable and Manipulable Pseudo-Randomness](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #51

### Dates: Oct 23rd, 2025 - Oct 30th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-10-company-simulator)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Predictable and Manipulable Pseudo-Randomness            



# Root + Impact

## Description

* The normal behavior calculates customer demand size pseudo-randomly using a seed from block.timestamp and msg.sender, hashed to determine base (1-5) and potential extra items based on reputation.

* The specific issue is that the seed is predictable (timestamp is known pre-submission, sender is fixed), allowing users or miners to time transactions for favorable outcomes, manipulating demand size for optimal reputation effects or failures.

```Python
# In CustomerEngine.trigger_demand:
@> seed: uint256 = convert(
@>     keccak256(
@>         concat(
@>             convert(block.timestamp, bytes32), convert(msg.sender, bytes32)
@>         )
@>     ),
@>     uint256,
@> )
base: uint256 = seed % 5
if (seed % 100) < (rep - 50):
    extra_item_chance = 1
```

## Risk

**Likelihood**:High

* Users time transactions to achieve desired timestamps, directly influencing the seed and demand outcome.

* Miners or validators front-run or delay blocks to manipulate timestamp for self-benefit in the simulation.

**Impact**:Low

* Minor unfairness in game mechanics, such as consistently high/low demands without real randomness.

* No direct fund loss, but subtle influence on reputation and sales patterns.

## Proof of Concept

This test demonstrates the vulnerability by funding the company (10 ETH) and producing 10 items as OWNER, advancing time to a known timestamp (target\_ts), then simulating the contract's seed calculation offline using keccak(timestamp || sender) to predict demand size (predicted\_requested via base % 5 + extra chance based on reputation). It triggers a demand as PATRICK, measures inventory reduction (actual\_requested), and asserts they match, proving attackers can pre-compute and time transactions for favorable outcomes (e.g., max items for rep gains).

```Python
import boa
from eth_utils import to_wei, keccak

def test_predictable_pseudo_randomness(customer_engine_contract, industry_contract, OWNER, PATRICK):
    # Arrange: Fund and produce inventory
    boa.env.set_balance(OWNER, to_wei(10, "ether"))
    with boa.env.prank(OWNER):
        industry_contract.fund_cyfrin(0, value=to_wei(10, "ether"))
        industry_contract.produce(10)

    # Set known future timestamp to avoid underflow
    current_ts = boa.env.evm.patch.timestamp
    target_ts = current_ts + 1
    boa.env.time_travel(1)  # Advance to target_ts

    # Predict requested: Simulate seed calculation
    timestamp = target_ts
    sender_hex = str(PATRICK)[2:].lower()
    sender_bytes = bytes.fromhex(sender_hex)
    sender_bytes = b'\x00' * (32 - len(sender_bytes)) + sender_bytes
    ts_bytes = timestamp.to_bytes(32, "big")
    concat = ts_bytes + sender_bytes
    seed = int.from_bytes(keccak(concat), "big")
    base = seed % 5
    rep = industry_contract.reputation()
    extra = 1 if (seed % 100) < (rep - 50) else 0
    predicted_requested = min(base + 1 + extra, 5)

    # Act: Trigger demand
    inventory_before = industry_contract.inventory()
    with boa.env.prank(PATRICK):
        customer_engine_contract.trigger_demand(value=to_wei(0.1, "ether"))

    # Assert: Actual requested matches predicted (inventory reduction = requested)
    actual_requested = inventory_before - industry_contract.inventory()
    assert actual_requested == predicted_requested, f"Predicted {predicted_requested}, actual {actual_requested}"
```

## Recommended Mitigation

Mitigations, in order of security:

1\) Chainlink VRF (production)

* Provably random values with cryptographic proof

Implementation example (via a Chainlink VRF service):

2\) Commit–reveal scheme

* Two-phase randomness with no trusted third party

* Requires multiple transactions

```Python
# State variables needed
commits: public(HashMap[address, bytes32])
nonces: public(HashMap[address, uint256])
reveal_deadline: public(uint256)

# Phase 1: Commit
@external
def commit_randomness(commitment: bytes32):
    # User commits to a value without revealing it
    self.commits[msg.sender] = commitment

# Phase 2: Reveal after deadline
@external
def reveal_and_compute():
    # After deadline, users reveal their values
    # Use XOR of all revealed values as randomness
```

3\) Enhanced on-chain entropy (weaker, faster)

\*Add less predictable inputs (e.g., block.prevrandao if available)

\*Reduce predictability but not as robust as VRF

```diff
@external
def trigger_demand():
    # ... existing checks ...
    
    # Enhanced seed - but STILL vulnerable to timing attacks
    seed: uint256 = convert(
        keccak256(
            concat(
-                convert(block.timestamp, bytes32), convert(msg.sender, bytes32)
-         )
-     ),
-     uint256,
- )
+               convert(msg.sender, bytes32),
+                convert(self.last_trigger[msg.sender], bytes32),  # Historical data
+                convert(industry_contract.reputation(), bytes32), # State-dependent
+            )
+        ),
+       uint256,
+    )
    # ... rest of logic ...
}
```






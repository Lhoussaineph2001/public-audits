# [High] Internal Balance Pre-Poisoning Vulnerability in BEX Vault (Balancer V2 Fork) Allowing Fund Theft When Adding Newly-Deployed Tokens


## Description

Bera’s Vault forks Balancer V2 and includes internal balances, a mechanism to store tokens on behalf of users. Due to a gas optimization in `_callOptionalReturn()`, the Vault does not check whether a token contract exists at a given address before updating internal balances.

This allows an attacker to assign arbitrary internal balances to a token that does not yet exist on-chain. If that token is later deployed and added to a pool, the attacker can use the pre-assigned internal balances to trade against the pool, effectively draining value from Liquidity Providers (LPs). Tokens that are already deployed on-chain are not affected.

---

## Vulnerability Details

The low-level execution logic is processed within the following internal Vault function:

```solidity
function _callOptionalReturn(address token, bytes memory data) private {    
    (bool success, bytes memory returndata) = token.call(data);    
    assembly {        
        if eq(success, 0) {            
            returndatacopy(0, 0, returndatasize())            
            revert(0, returndatasize())        
        }    
    }    
    _require(        
        returndata.length == 0 || abi.decode(returndata, (bool)),        
        Errors.SAFE_ERC20_CALL_FAILED    
    );
}

```

### The Core Flaw

* `_callOptionalReturn()` executes a low-level call via `token.call(data)` without verifying contract existence using `token.code.length > 0`.
* In the Ethereum Virtual Machine (EVM), low-level calls to non-existent accounts (addresses with no code) **implicitly return true** with empty return data.
* Because `returndata.length == 0`, the execution path bypasses the `abi.decode` sanity check and treats the transaction as valid. As a result, the Vault records state modifications for non-existent tokens.

### Exploit Scenario

1. An attacker identifies or calculates a token address that will be deployed in the future (e.g., via predictable `CREATE` or `CREATE2` salts).
2. The attacker triggers internal balance updates within the Vault for this undeployed address.
3. The token is subsequently deployed on-chain.
4. A Balancer-style pool is initialized incorporating this token asset.
5. The attacker utilizes the pre-populated internal balance state to execute highly asymmetric trades against the pool, extracting legitimate assets deposited by LPs.

### Scope / Constraints

* **Undeployed Tokens Only:** The flaw exclusively affects tokens that are completely non-existent at the time the internal balance state modification is called.
* **Live Tokens Secure:** Tokens already deployed on-chain maintain true balance invariants because low-level calls to active contracts will execute actual code and fail if the parameters or permissions are invalid.

---

## Impact Details

### Severity: Direct Theft of User Funds (At-Rest or In-Motion)

* **Liquidity Drainage:** Initial protocol liquidity deposited into pools utilizing newly deployed tokens can be completely drained.
* **Arbitrary Price Manipulation:** Fake internal balances act as infinite virtual liquidity pools for the attacker, enabling them to alter the relative price of the pool assets arbitrarily.
* **LP Financial Risk:** Financial loss is absorbed fully by the protocol's LPs upon execution of the malicious swap.

---

## Mitigation

### Contract Level

The Vault infrastructure is immutable; therefore, an on-chain architectural fix cannot be applied directly to the core routing logic.

### Operational Mitigation (Standard Balancer Protocol Framework)

* **Token Allowlisting:** Implement a strict governance allowlist or manual approval phase for all token assets prior to pool initialization.
* **Pre-Seeding Invariant Checks:** Deploy off-chain monitoring scripts to query the Vault state for existing internal balances associated with a token address *before* adding liquidity or launching its respective pool.
* **Controlled Deployment Sequences:** Ensure pools are seeded only after confirming that the corresponding token address has a clean balance sheet within the Vault.

---

## References

* **Balancer V2 Token Front-Run Vulnerability Disclosure:** [https://forum.balancer.fi/t/balancer-v2-token-frontrun-vulnerability-disclosure/6309](https://forum.balancer.fi/t/balancer-v2-token-frontrun-vulnerability-disclosure/6309)

---

## Proof of Concept

The general progression to confirm the on-chain vector consists of the following phases:

1. **Target Identification:** Attacker pre-computes or observes a token contract address that does not yet possess on-chain code.
2. **State Seeding:** Attacker executes standard deposit or transfer functions pointing to the undeployed address to record an arbitrary `internalBalance[attacker][token]` credit within the Vault contract.
3. **Token Creation:** The legitimate project deploys the token and registers the pool asset pair within the ecosystem.
4. **Liquidity Extraction:** Attacker invokes swap or withdrawal logic using their pre-set internal balance allocations to drain the physical tokens provided by LPs.
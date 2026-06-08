
# [High] Permissionless SPL transfer to pool vault decouples bonding curve state, enabling complete drain of USDF from any live pool

### Target

`https://github.com/code-payments/flipcash-program`

---

## Vulnerability Details

The Flipcash bonding curve program calculates token pricing using two independent values derived from raw vault account balances. Both values are read directly from on-chain SPL Token account balances—balances that any wallet can modify via a standard permissionless transfer instruction, completely bypassing the program logic.

### 1. Supply Axis State Derivation

In `program/src/instruction/sell.rs` $\rightarrow$ `sell_common()`, the bonding curve supply axis calculation uses the raw balance of Vault A:

```rust
let tokens_left_raw = target_vault.amount();      // raw on-chain vault A balance
let supply_from_bonding = MAX_TOKEN_SUPPLY
    .checked_mul(QUARKS_PER_TOKEN)
    .ok_or(ProgramError::InvalidArgument)?
    .checked_sub(tokens_left_raw)                 //@audit supply = MAX − vault_A.amount()
    .ok_or(ProgramError::InvalidArgument)?;

```

### 2. Value Axis State Derivation

In `program/src/instruction/sell.rs` $\rightarrow$ `sell_common()`, the independent value axis calculation reads directly from Vault B:

```rust
let value_left_raw = base_vault.amount()          // raw on-chain vault B balance
    .checked_sub(pool.fees_accumulated)
    .ok_or(ProgramError::InvalidArgument)?;

```

### 3. Payout Formula

The program calculates the payout by evaluating the gap between these two decoupled axes:

```rust
let curve     = DiscreteExponentialCurve::default();
let zero      = UnsignedNumeric::zero();
let new_value = curve.tokens_to_value(&zero, &new_supply)
    .ok_or(ProgramError::InvalidArgument)?;

let mut total_sell_value = if new_value.less_than(&value_left) {
    value_left
        .checked_sub(&new_value)                  //@audit pays out: value_left − new_value
        .ok_or(ProgramError::InvalidArgument)?
} else {
    UnsignedNumeric::zero()
};

```

An identical vulnerable architecture exists in `program/src/instruction/buy.rs` $\rightarrow$ `buy_common()`:

```rust
let tokens_left_raw = target_vault.amount();      // raw on-chain vault A balance
let supply_from_bonding = MAX_TOKEN_SUPPLY
    .checked_mul(QUARKS_PER_TOKEN)
    .ok_or(ProgramError::InvalidArgument)?
    .checked_sub(tokens_left_raw)
    .ok_or(ProgramError::InvalidArgument)?;

let current_value_raw = base_vault.amount()       // raw on-chain vault B balance
    .checked_sub(pool.fees_accumulated)
    .ok_or(ProgramError::InvalidArgument)?;

```

### Core Flaw

The variables `supply_from_bonding` and `value_left_raw`/`current_value_raw` are treated as independent sources of truth for the curve's $X$ and $Y$ axes. Because any external account can transfer tokens directly into `vault_a_pda` (or USDF into `vault_b_pda`) via a standard SPL token instruction, an attacker can manipulate one axis without modifying the other.

This completely breaks the curve invariant, causing the payout formula `value_left - new_value` to yield nearly the entire pool balance in response to a negligible sell order. This is a classic **donation attack**, typically mitigated in Automated Market Makers (AMMs) by storing internal reserve states that update exclusively via validated swap handlers rather than relying on live `amount()` or `balanceOf()` readings.

---

## Impact Details

### Critical Curve Disruption & Capital Theft

* **State Decoupling:** An attacker can manipulate pricing calculations permissionlessly by transferring tokens directly into the underlying pool accounts.
* **Capital Drain:** The price calculation mechanics can be distorted to allow the complete extraction of USDF backing from any active pool.
* **Direct Loss:** Legitimate liquidity providers and token buyers cannot exit or recover their principal assets because the vault is depleted during the exploit.

---

## Attack Steps

### Step 1: Pool Initialization

* `InitializeCurrencyIx` creates the mint PDA and currency configuration.
* `InitializePoolIx` creates `pool_pda`, `vault_a_pda` (tokens), and `vault_b_pda` (USDF).
* **State:** `vault_a` = `MAX_TOKEN_SUPPLY` quarks, `vault_b` = 0 USDF quarks.

### Step 2: Attacker Position Entry

* Attacker calls `BuyTokensIx` with 1,000 USDF. `buy_common()` evaluates the raw balances to execute the swap.
* **Result:** Attacker receives 958,588,639,750,412 token quarks. `vault_b` grows to 1,000,000,000 USDF quarks.

### Step 3: Capital Seeding (Victim Entry)

* A victim executes a `BuyTokensIx` order with 50,000 USDF.
* **Result:** Victim receives 18,421,475,041,300,305 token quarks. `vault_b` grows to 51,000,000,000 USDF quarks (~$51,000).

### Step 4: Curve Axis Manipulation

* The attacker executes a raw `spl_token::transfer` from their Associated Token Account (ATA) directly to `vault_a_pda`, bypassing the Flipcash program entirely.
* **Amount Transferred:** 958,588,639,750,312 quarks (leaving exactly 100 quarks in their balance).
* **State Impact in `sell_common()`:**
* `tokens_left_raw` increases artificially by the donation amount.
* `supply_from_bonding` collapses toward 0.
* `value_left_raw` remains static because Vault B was untouched.
* The curve axes are now completely decoupled.



### Step 5: Exploitation (Asymmetric Sell Order)

* Attacker calls `SellTokensIx` with an `in_amount` of exactly 100 quarks.
* **Mathematical Output:**
* `new_supply` $\approx 0$ (due to the artificial donation collapsing the calculated supply metric).
* `new_value = curve.tokens_to_value(0, new_supply)` $\approx 0$.
* `total_sell_value = value_left - new_value` $\approx 51,000$ USDF.


* **Result:** The attacker extracts ~4,982,022,262 USDF quarks (~$4,982) in exchange for liquidating just 100 quarks ($10^{-8}$ tokens).

### Step 6: Value Destruction

* The victim attempts to call `SellTokensIx` to salvage their position.
* Because Vault B has been systematically drained, the contract fails to return the original value, ensuring permanent capital loss for the victim.

---

## Validation Steps

### Environment Configuration

1. Copy the Proof of Concept file (`poc.rs`) into the `flipcash-program/program/tests/` directory.
2. Execute the test suite with the following terminal command to verify the execution flow:

```bash
cargo test test_attack_drain_usdf_by_donating_tokens -- --nocapture

```
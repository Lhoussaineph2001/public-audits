# [High] Fee Loss on Rejected Collateral Reservation Due to Transfer Failure
## Description

In the `rejectCollateralReservation` flow, if the `transferNATAllowFailure` call fails due to gas constraints, the full fee sent by the user (reservation + executor fee) is burned, resulting in irrecoverable user loss. A malicious or careless agent owner can deliberately induce this failure, exploiting users who attempt to mint.

---

## Vulnerability Details

A malicious agent owner can grief users by invoking `rejectCollateralReservation` with insufficient gas (specifically less than `TRANSFER_GAS_ALLOWANCE`), causing the native token refund transfer to fail.

### Code Execution Flow

When the execution path hits `_rejectOrCancelCollateralReservation`, it triggers a refund using a capped transfer mechanism:

```solidity
// High-level conceptual execution flow within CollateralReservations.sol
bool success = Transfers.transferNATAllowFailure(user, refundAmount, TRANSFER_GAS_ALLOWANCE);
if (!success) {
    Agents.burnDirectNAT(totalFee); //@audit Burning entire user fee on failure
}

```

### The Core Flaw

* `transferNATAllowFailure` absorbs the failure instead of reverting the transaction state.
* If the gas supplied to the low-level call is constrained or if the user's receiving contract fallback logic exceeds the `TRANSFER_GAS_ALLOWANCE` threshold, the transfer returns `false`.
* Instead of keeping the funds in the contract for a pull-based withdrawal pattern, the protocol handles the failure state by calling `Agents.burnDirectNAT(totalFee)`.
* This permanently burns the entire reservation and executor fee, leading to an absolute loss of user funds without any functional benefit or cost to the attacker beyond transaction fees.

---

## Impact Details

### High Severity: Griefing & Forced User Capital Destruction

* **User Fund Loss:** Complete loss of the user's upfront fees (`msg.value`) despite the minting process being rejected.
* **Malicious Trigger Vector:** Agent owners retain full control over transaction execution properties and can force this loss state deterministically via gas manipulation or smart contract wallet structures.
* **Lack of Safe Fallbacks:** No alternative refund mechanism, escrow, or pull-based recovery pattern exists to safeguard users against failed push transfers.
* **Centralization Risk:** Introduces censorship and economic sabotage hazards where malicious agents can clear out queue reservations and financially penalize targeted users.

---

## References

* **CollateralReservations.sol (Line 115):** [`rejectCollateralReservation`](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol%23L115)
* **CollateralReservations.sol (Line 313):** [`_rejectOrCancelCollateralReservation`](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol%23L313)
* **CollateralReservations.sol (Line 328 & 331):** [Refund and Burn branches](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/library/CollateralReservations.sol#L328)
* **Transfers.sol (Line 57):** [`transferNATAllowFailure`](https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/utils/lib/Transfers.sol%23L57)

---

## Proof of Concept

The programmatic flow demonstrating the forced fee destruction proceeds as follows:

1. **Reservation Phase:** A user calls `reserveCollateral`, providing an attached `msg.value` that fulfills both the required `reservationFee` and `executorFee`.
2. **Rejection Phase:** The agent owner calls `rejectCollateralReservation`, which internally cascades to `_rejectOrCancelCollateralReservation`.
3. **Execution Capping:** The underlying refund logic leverages `Transfers.transferNATAllowFailure` with a hard-coded gas limit parameter (`TRANSFER_GAS_ALLOWANCE`).
4. **Gas Starvation:** The owner deliberately parameters the call gas limit so that the sub-call to the user address runs out of gas, or the target user is a smart contract whose fallback function demands more gas than the allowed ceiling.
5. **Irreversible Deficit:** The transfer returns `false`, bypassing the successful return route and hitting `Agents.burnDirectNAT(totalFee)`.
6. **Result:** The user's entire deposit asset allocation is permanently removed from the circulating supply, leaving the user with a total financial deficit and no minted assets.
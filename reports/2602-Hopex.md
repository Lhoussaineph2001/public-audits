
# Findings

## High

###  Incorrect Slippage Parameter in _sendCrossChainPurchase Causes Persistent DoS

#### Summary

The `_sendCrossChainPurchase` function sets `minAmountLD` equal to `ticketPrice`. However, LayerZero’s `_debitView` removes "dust" from the `amountLD` to match the shared decimal precision across chains. This ensures `amountReceivedLD` will almost always be less than `ticketPrice` if the price contains non-granular units, triggering a `SlippageExceeded` revert.

---

#### Vulnerability Detail

In the `SendParam` struct, `minAmountLD` acts as a strict floor for the transaction.

1. The contract defines `minAmountLD: ticketPrice`.
2. The OFT's `_debitView` calls `_removeDust(_amountLD)`.
3. If the token's local decimals (e.g., 18) are higher than the shared decimals (e.g., 6), any value in the lower 12 decimal places is truncated.
4. `amountReceivedLD` becomes `ticketPrice - dust`.
5. The check `if (amountReceivedLD < _minAmountLD)` evaluates to `true` because `(ticketPrice - dust) < ticketPrice`.

#### Impact

**Severity: High**
The cross-chain purchase functionality is broken for any `ticketPrice` that is not perfectly aligned with the OFT’s decimal granularity. Users will be unable to buy tickets as `quoteSend` and `send` will consistently revert with `SlippageExceeded`.

---

#### Code Snippet

```solidity
    function _sendCrossChainPurchase(
        // ... parameters
    ) private {
        // ... message building

        SendParam memory sendParam = SendParam({
            dstEid: lotturaCoreEid, 
            to: _addressToBytes32(address(lotturaCore)),
            amountLD: ticketPrice,
            //@audit BUG: ticketPrice is used as the minimum. 
            //@audit If _removeDust reduces amountLD by even 1 wei, the check below fails.
            minAmountLD: ticketPrice, 
            extraOptions: options,
            composeMsg: composeMsg,
            oftCmd: ""
        });

        // This call triggers _debitView internally
        MessagingFee memory fee = IOFT(paymentToken).quoteSend(sendParam, false);
    }

    // Internal OFT Logic (The Revert Trigger)
    function _debitView(uint256 _amountLD, uint256 _minAmountLD, uint32) internal view virtual 
        returns (uint256 amountSentLD, uint256 amountReceivedLD) {
        
        amountSentLD = _removeDust(_amountLD); 
        amountReceivedLD = amountSentLD; 

        //@audit Reverts here because amountReceivedLD (stripped of dust) < _minAmountLD (original ticketPrice)
        if (amountReceivedLD < _minAmountLD) {
            revert SlippageExceeded(amountReceivedLD, _minAmountLD);
        }
    }

```

---

#### Tool Used

Manual Review

---

#### Recommendation

Calculate the expected amount after dust removal and use that value for `minAmountLD`. 

### Improper Source Authentication in lzCompose Enables Cross-Chain Message Spoofing

#### Summary

The `lzCompose()` implementation does not verify the `_oApp` parameter provided by the LayerZero endpoint.
Instead, authentication is performed using a `composeFrom` address extracted directly from the `_message` payload.

Because `_message` is user-controlled on the source chain, an attacker can forge the `composeFrom` field to impersonate a trusted hub contract and bypass cross-chain authentication checks.

This results in a critical trust boundary violation and enables unauthorized execution of privileged logic such as ticket purchases.

---

#### Vulnerability Detail

The contract processes LayerZero compose messages as follows:

1. `lzCompose()` verifies that the caller is the LayerZero `endpoint`.
2. It forwards `_oApp` and `_message` to `_handleCompose()`.
3. `_handleCompose()` extracts `composeFrom` from `_message`.
4. It compares this extracted address against `hubs[chainId]`.

However:

* `_oApp` is the authentic source OApp address provided by the LayerZero endpoint.
* `_message` is constructed on the source chain and can be controlled by the sending OApp.
* The contract does **not verify `_oApp`** against the trusted hub mapping.
* Authentication relies entirely on `composeFrom` extracted from `_message`.

The issue is that `composeFrom` is not guaranteed to reflect the true sender. An attacker can:

* Deploy a malicious OApp on a source chain.
* Craft a compose message.
* Set the `composeFrom` bytes offset to a legitimate hub address.
* Provide a valid `chainId`.
* Bypass the check:

  ```
  if (address(hubs[chainId]) != sourceLotturaHub)
  ```

Since `_oApp` is never validated, the contract trusts a user-controlled value for authentication.

This directly contradicts LayerZero’s integration security guidance, which states that:

* `_oApp` must be validated against trusted peers.
* Message payload fields must not be trusted for authentication.

---

#### Impact

This vulnerability allows an attacker to:

* Spoof a trusted hub contract
* Bypass cross-chain authentication
* Trigger `_purchaseTicket()` without originating from a legitimate hub
* Manipulate lottery entries
* Corrupt draw state
* Potentially bypass payment assumptions
* Break cross-chain accounting integrity

Because this function executes financial and state-changing logic, the vulnerability represents a **cross-chain authentication bypass**.



Reasoning:

* Trust boundary violation across chains
* Unauthorized execution of core business logic
* Potential financial and state corruption impact
* Direct violation of LayerZero security model

---

#### Code Snippet

#### Vulnerable Authentication Logic

```solidity
address sourceLotturaHub = _bytes32ToAddress(
    OFTComposeMsgCodec.composeFrom(_message)
);

if (address(hubs[chainId]) != sourceLotturaHub) {
    revert CallerNotLotturaHub();
}
```

#### composeFrom Extraction (User-Controlled Data)

```solidity
function composeFrom(bytes calldata _msg) internal pure returns (bytes32) {
    return bytes32(_msg[AMOUNT_LD_OFFSET:COMPOSE_FROM_OFFSET]); //@audit struct from the message attack controlle
}
```

#### Missing Critical Check

The `_oApp` parameter is never validated:

```solidity
function lzCompose(
    address _oApp, //@audit miss check https://docs.layerzero.network/v2/tools/integration-checklist#check-lzcompose-security
    ...
) external payable override {
    if (_msgSender() != address(endpoint))
        revert OnlyEndpointExecutesLZCompose();

    _handleCompose(_oApp, _message);
}
```

---

#### Tool Used

Manual Review

---

#### Recommendation

Authentication must rely on `_oApp`, not on values embedded inside `_message`.

#### Required Fix

Add explicit verification of `_oApp` inside `lzCompose()`:

```diff
function lzCompose(
    address _oApp,
    ...
) external payable override {
    if (_msgSender() != address(endpoint))
        revert OnlyEndpointExecutesLZCompose();

+    if (_oApp != address(hubs[chainId])) {
+        revert CallerNotLotturaHub();
+    }

    _handleCompose(_message);
}
```

### Architectural Ambiguity in lotturaCore Destination

#### Summary

The contract uses the same `lotturaCore` address for local calls and cross-chain destination targets, which is architecturally inconsistent.

#### Vulnerability Detail

The contract logic branches based on `isOnPolygonChain`.

1.  **If on Polygon**: It makes a direct contract call to `lotturaCore.purchaseTicket(...)`. This implies `lotturaCore` is a local address on the Polygon network.


2.  **If NOT on Polygon**: It uses the same `lotturaCore` address as the `to` field in a cross-chain `SendParam`. This implies `lotturaCore` is the address of the contract on the *destination* chain.



Using the same state variable for both a local contract instance and a remote address assumes the `LotturaCore` contract is deployed at the exact same address across all chains (a "vanity" or "deterministic" deployment). If the addresses differ between chains, one of these operations will always fail.

#### Impact

* **Deployment Rigidity**: The protocol cannot be deployed unless the `LotturaCore` address is identical on both the Hub and Core chains.
*  **Message Failure**: If the address is incorrect on the destination chain, tokens and messages sent via LayerZero will be lost or unrecoverable at the destination.



#### Code Snippet

```solidity
// Local call
lotturaCore.purchaseTicket(tokenId, _msgSender(), numbers, drawId, hashedRandomness, ticketPrice, commission);

// Cross-chain destination
to: _addressToBytes32(address(lotturaCore)),

```

#### Tool Used

Manual Review

#### Recommendation

Separate these variables. Use `lotturaCore` for local calls (when `isOnPolygonChain` is true) and a separate `remoteCoreAddress` variable for cross-chain `SendParam` targets to ensure flexibility and safety.


### Native Fee Drainage (Griefing/DOS)


#### Summary

The `LotturaHub` contract pays for LayerZero cross-chain message fees using its own internal balance. This allows any user to deplete the contract's native gas reserves by purchasing tickets.

#### Vulnerability Detail

In the `purchaseTickets` function, the contract transfers the `ticketPrice` from the user in the form of a payment token. However, when calling `_sendCrossChainPurchase`, the contract calculates the `nativeFee` required by LayerZero and pays it directly from its own balance (`address(this).balance`).



Because `purchaseTickets` is an external, unpermissioned function, a malicious actor can call it repeatedly with large ticket batches. Since the user only pays the `ticketPrice` (which they might value or even control via the `paymentToken`), but the contract pays the `nativeFee`, the contract's `fundGas` reserves can be exhausted by an attacker, leading to a Denial of Service (DOS) for legitimate users who will see the transaction revert with `InsufficientLayerZeroFee`.

#### Impact

* **Financial Loss**: The contract owner's deposited gas funds are consumed by users rather than being covered by the transaction initiator.
* 
**Service Disruption**: Once the balance is drained, all cross-chain purchases and claims will fail until the owner manually refills the contract via `fundGas`.



#### Code Snippet

* **Fee Payment**:

```solidity
  function _sendCrossChainPurchase(
        uint256 tokenId,
        address buyer,
        uint8[6] memory numbers,
        uint256 drawId,
        bytes32 hashedRandomness
    ) private {
        bytes memory composeMsg = abi.encode(
            MSG_TYPE_PURCHASE,
            _chainId,
            tokenId,
            buyer,
            numbers,
            drawId,
            hashedRandomness,
            ticketPrice,
            commission
        );

        bytes memory options = _buildOptionsWithCompose();

        SendParam memory sendParam = SendParam({
            dstEid: lotturaCoreEid, 
            to: _addressToBytes32(address(lotturaCore)),
            amountLD: ticketPrice,
            minAmountLD: ticketPrice, 
            extraOptions: options,
            composeMsg: composeMsg,
            oftCmd: ""
        });

        MessagingFee memory fee = IOFT(paymentToken).quoteSend(
            sendParam,
            false
        );

        if (address(this).balance < fee.nativeFee)
            revert InsufficientLayerZeroFee();

        IOFT(paymentToken).send{value: fee.nativeFee}( // @audit attacker can drain this by forcing this one to pay alot of txs with big data 
            sendParam,
            fee,
            address(this) 
        );
    }

```
* **Balance Check**:
The logic for paying this fee is defined in the internal `_payNative` override:

```solidity
function _payNative(uint256 _nativeFee) internal virtual override returns (uint256 nativeFee) {
    if (address(this).balance < _nativeFee)
        revert NotEnoughNative(address(this).balance);
    return _nativeFee;
}

```
#### Tool Used

Manual Review

#### Recommendation

Modify `purchaseTickets` and `claimTicket` to be `payable`. Require the caller to provide the `nativeFee` as `msg.value`. The contract should then use the provided `msg.value` to pay LayerZero, ensuring the user—not the protocol—covers the cost of their cross-chain activity.


## Medium

### Protocol Gas Exhaustion via Unrestricted Cross-Chain Claims

#### Summary

The `LotturaCore` contract pays for cross-chain LayerZero messaging fees using its own internal balance instead of charging the user. Because the `claimTicket` function (via `_claimTicket`) can be triggered by users, an attacker with many small winning tickets can intentionally drain the contract's native token balance.

#### Vulnerability Detail

When a user claims a winning cross-chain ticket:

1. The `claimTicket` function is called.


2. This triggers `_claimTicket`, which identifies the ticket as cross-chain and calls `_sendCrossChainPrize`.


3. Inside `_sendCrossChainPrize`, the contract calculates the `MessagingFee`.


4. It checks if `address(this).balance` is sufficient and then pays the `nativeFee` out of the **contract's own funds**.



#### Impact

A malicious actor can purchase numerous tickets across various chains. Even if they win the lowest tier prize, they can force the contract to pay the cross-chain gas fees for every single claim. If the gas cost for the cross-chain message is significant, the contract's gas reserve (funded via `fundGas`) will be depleted.


#### Code Snippet

```solidity
if (address(this).balance < fee.nativeFee)
    revert InsufficientGasForCrossChainPrize();
paymentToken.send{value: fee.nativeFee}(sendParam, fee, address(this));

```

#### Tool Used

Manual Review

#### Recommendation

Modify `claimTicket` and `_claimTicket` to be `payable`. Require the user to provide the `nativeFee` required for the cross-chain message. Use `paymentToken.quoteSend` to determine the fee and ensure `msg.value` covers it, refunding any excess to the `claimer`.

### Lack of receive/fallback Function for LayerZero Fee Refunds in LotturaCore

#### Summary

The `LotturaCore` contract lacks a `receive()` or `fallback()` function to handle returned native tokens. Furthermore, it designates `address(this)` as the refund recipient for LayerZero messaging fee overpayments. Because there is no mechanism to withdraw these "trapped" native tokens other than an emergency function, protocol funds are effectively locked.

#### Vulnerability Detail

In `_sendCrossChainPrize`, the contract executes `paymentToken.send{value: fee.nativeFee}(...)`. The third parameter in the `send` function is the `refundAddress`, which is set to `address(this)`.

1. If the `nativeFee` provided is higher than the actual cost required by the LayerZero endpoint, the excess is sent back to the `refundAddress` (`address(this)`).


2. The contract does not implement `receive() external payable {}` or `fallback() external payable {}` . While some versions of Solidity/contracts allow receiving native via certain paths, the lack of a explicit `receive` often causes transfers to fail or makes the balance inaccessible to standard logic.


3. Although `nativeEmergencyWithdraw` exists, it is restricted to the `ADMIN_ROLE`, meaning protocol gas funds cannot be recovered or reused automatically by the system logic.



#### Impact

Excess gas fees paid by the contract are trapped within the contract. Over time, this results in a significant "leak" of native tokens (POL/ETH) that were intended for protocol operations (VRF and cross-chain messaging).

#### Code Snippet

```solidity
paymentToken.send{value: fee.nativeFee}(sendParam, fee, address(this));

```

#### Tool Used

Manual Review

#### Recommendation

1. Add a `receive()` function to the contract to safely handle native token inflows.


#### Insufficient Gas Limit Sanity & Hardcoded Options


#### Summary

The contract uses static/hardcoded gas options for cross-chain messages, which does not account for destination chain gas volatility or the complexity of the `LotturaCore` execution.

#### Vulnerability Detail

The `LotturaHub` utilizes `_buildOptionsWithCompose()` and `_buildOptionsWithoutCompose()` to generate the `extraOptions` for LayerZero.

```solidity
    function _buildOptionsWithCompose() private pure returns (bytes memory) {
        return
            OptionsBuilder
                .newOptions()
                .addExecutorLzReceiveOption(200000, 0)
                .addExecutorLzComposeOption(0, 700000, 0); //@audit // Static gas limit
    }


```

The guideline explicitly warns against "Gas Limit Sanity" issues. Hardcoding 500,000 gas for the `composeMsg` (the actual purchase logic on the destination) is dangerous. If the destination chain (e.g., Polygon or an L2) experiences a gas spike or if the `LotturaCore.purchaseTicket` logic expands in complexity, the message will fail on the destination chain.

Because LayerZero V2 is non-blocking, a "failed" execution due to Out-of-Gas (OOG) means the tokens are moved, but the ticket is never recorded. This results in a "Lost Message" state where the user has paid but received nothing.

#### Impact

* **Permanent Loss of Funds**: Users pay the `ticketPrice` on the Hub chain, but the transaction fails on the Core chain due to OOG.
* **Protocol Desync**: The Hub chain believes a ticket exists (it may have minted a local receipt), but the Core chain (the source of truth for the draw) has no record of it.

#### Code Snippet

```solidity
    function _buildOptionsWithCompose() private pure returns (bytes memory) {
        return
            OptionsBuilder
                .newOptions()
                .addExecutorLzReceiveOption(200000, 0)
                .addExecutorLzComposeOption(0, 700000, 0);
    }


```

#### Tool Used

Manual Review (Reference: [BurraSec LayerZero Guideline](https://github.com/burrasec/Cross-Chain-Integration-Security-Review-Guideline))

#### Recommendation

1. **Dynamic Options**: Allow the `extraOptions` to be passed or adjusted by the owner/admin to respond to network conditions.
2. **Gas Buffer**: Increase the default gas limit and implement a "safety margin."
3. **User-Provided Gas**: Allow users to provide a `gasLimit` parameter in the `purchaseTickets` function to ensure their specific transaction has enough gas to execute on the destination.



#### Improper Refund Address (Lack of receive Function)


#### Summary

The contract designates itself as the `refundAddress` for LayerZero fees  but lacks a `receive()` or `fallback()` function, which may cause transactions to fail or trap native tokens.

#### Vulnerability Detail

When calling `IOFT(paymentToken).send`, the contract passes `address(this)` as the third argument, which represents the address where excess LayerZero fees are refunded. Similarly, in `_sendCrossChainClaim`, `address(this)` is used as the refund recipient.

For a contract to receive native Ether (refunds), it must implement a `receive()` or `fallback()` function. The provided code for `LotturaHub` does not include these functions. If the LayerZero endpoint attempts to refund surplus `nativeFee` to `LotturaHub` within the same transaction, the transfer will fail/revert because the contract cannot accept native tokens.

#### Impact

* **Transaction Reversion**: If the LayerZero implementation requires a successful refund to complete the `send` operation, purchases will revert.
* **Loss of Funds**: If refunds are processed via a separate mechanism, the native tokens may be trapped or lost because the contract cannot receive them.

#### Code Snippet

```solidity
    function _sendCrossChainPurchase(
        uint256 tokenId,
        address buyer,
        uint8[6] memory numbers,
        uint256 drawId,
        bytes32 hashedRandomness
    ) private {
        bytes memory composeMsg = abi.encode(
            MSG_TYPE_PURCHASE,
            _chainId,
            tokenId,
            buyer,
            numbers,
            drawId,
            hashedRandomness,
            ticketPrice,
            commission
        );

        bytes memory options = _buildOptionsWithCompose();

        SendParam memory sendParam = SendParam({
            dstEid: lotturaCoreEid, 
            to: _addressToBytes32(address(lotturaCore)),
            amountLD: ticketPrice,
            minAmountLD: ticketPrice,
            extraOptions: options,
            composeMsg: composeMsg,
            oftCmd: ""
        });

        MessagingFee memory fee = IOFT(paymentToken).quoteSend(
            sendParam,
            false
        );

        if (address(this).balance < fee.nativeFee)
            revert InsufficientLayerZeroFee();

        IOFT(paymentToken).send{value: fee.nativeFee}(
            sendParam,
            fee,
            address(this) //@audit this contract doesn't have receiver function, so it will be failed and the fee will be refunded
        );
    }
```
#### Tool Used

Manual Review

#### Recommendation

1. **Add Receive Function**: Implement a simple `receive() external payable {}` function to allow the contract to accept native token refunds.
2. **Redirect Refunds**: Alternatively, set the `refundAddress` to the `_msgSender()` so that any excess fees are returned directly to the user who initiated the transaction.

## Low

### Unchecked msg.value in LayerZero V2 Message Handlers

#### Summary

The contract’s `_lzReceive` and related cross-chain message handling do not enforce or validate the expected `msg.value` carried with inbound LayerZero messages. Per LayerZero’s integration checklist, **if a message execution option includes a native value, the destination contract must verify that the actual `msg.value` received matches what was intended**. Without this check, a verified message can be executed with insufficient native token value, leading to unintended logic execution or state changes. 

---

#### Vulnerability Detail

LayerZero allows specifying execution options, including the amount of native value (`msg.value`) to be provided when a cross-chain message is delivered. However, these options **are not automatically enforced by the protocol**, and any caller can execute a verified message on the destination chain with any `msg.value` amount. Therefore, **the destination contract should explicitly validate the received `msg.value` against what was encoded in the message payload.** 

In the provided `_lzReceive` implementation, the contract executes core logic to credit a ticket and emit events, but it **never validates `msg.value` against an expected encoded value** in the message.

---

#### Impact

Because `_lzReceive` does not check `msg.value`:

* A cross-chain message that expects a certain native value (e.g., to cover fees or pay for subsequent operations) **could be executed with less or no native token** than intended.
* This may lead to **unintended state changes, partial execution, or improper handling of economic assumptions** if the contract logic implicitly relies on native value presence.
* Attackers or arbitrary callers could repeatedly execute valid messages with insufficient native value, potentially breaking destination logic or causing inconsistent outcomes.

Although the message itself is verified by LayerZero, **the application layer must enforce native value requirements** to guarantee that downstream logic behaves as expected. 

---

#### Code Snippet

The vulnerable handler is:

```solidity
function _lzReceive(
    Origin calldata _origin,
    bytes32 _guid,
    bytes calldata _message,
    address,
    bytes calldata
) internal virtual override {
    address toAddress = _message.sendTo().bytes32ToAddress();
    uint256 tokenId = _message.tokenId();
    _credit(toAddress, tokenId, _origin.srcEid);
    TicketReceipt memory ticket = _message.ticketReceipt();
    _ticketReceipts[tokenId] = ticket;

    if (_message.isComposed()) {
        bytes memory composeMsg = ONFTComposeMsgCodec.encode(
            _origin.nonce,
            _origin.srcEid,
            _message.composeMsg()
        );

        endpoint.sendCompose(toAddress, _guid, 0, composeMsg);
    }
    emit ONFTReceived(_guid, _origin.srcEid, toAddress, tokenId);
}
```

There is **no enforcement of expected `msg.value`** before executing the handler logic.

---

#### Tool Used

Manual Review based on the *LayerZero V2 Integration Checklist* guidance that mandates enforcement of native value expectations in message handlers to prevent execution with incorrect value. https://docs.layerzero.network/v2/tools/integration-checklist#enforce-msg-value-in-_lzreceive-and-lzcompose

---

#### Recommendation

To comply with LayerZero best practices, modify the `_lzReceive` and `lzCompose` handlers as follows:

1. **Encode Expected Value on Source Side:** When sending cross-chain messages, include the expected native token amount in the encoded payload.

2. **Validate Received Value:** In `_lzReceive` and `lzCompose`, decode the expected value and enforce it against `msg.value`:

   ```solidity
   uint256 expectedValue = _message.value();
   require(msg.value >= expectedValue, "insufficient msg.value");
   ```

   This ensures that the message is not executed with insufficient native funds.


### Gas Optimization in _validateTicketNumbers

#### Technical Audit: `_validateTicketNumbers`

The provided function is functionally **accurate** but architecturally **inefficient** for an EVM environment. While it correctly identifies out-of-range values and duplicates, it does so using a nested loop structure that scales poorly.


```solidity

        for (uint256 i = 0; i < ticketsCount; i++) {
            bytes32 hashedRandomness = hashedRandomnessList[i];

            if (hashedRandomness == bytes32(0))
                revert InvalidParam(InvalidParams.ZeroHashedRandomness);

            uint8[6] calldata numbers = numbersList[i];

            _validateTicketNumbers(numbers); //@audit alot of gas may cause OUT OF GAS ( O(n3)
.......
    function _validateTicketNumbers(uint8[6] memory numbers) private pure {
        for (uint8 i = 0; i < NUMBERS_PER_TICKET; i++) { // 6 numbers per ticket
            if (numbers[i] < MIN_NUMBER || numbers[i] > MAX_NUMBER) { // 1 to 50
                revert InvalidParam(InvalidParams.TicketNumbersOutOfRange);
            }

            // Check duplicates
            for (uint8 j = i + 1; j < NUMBERS_PER_TICKET; j++) {
                if (numbers[i] == numbers[j])
                    revert InvalidParam(InvalidParams.TicketNumbersDuplicate);
            }
        }
    }
```
---

##### 1. Logical Accuracy

* **Range Validation:** Correct. It iterates through the array and checks against `MIN_NUMBER` and `MAX_NUMBER`.
* **Duplicate Detection:** Correct. It uses a "brute force" comparison ($O(n^2)$), comparing each element to every subsequent element.
* **Error Handling:** Uses custom errors (`InvalidParam`), which is gas-efficient compared to string reverts.

##### 2. Efficiency Issues (Gas Analysis)

In Solidity, gas is consumed for every operation. The current implementation performs approximately **15 comparisons** for the duplicate check alone ($5 + 4 + 3 + 2 + 1$).

* **Current Complexity:** $O(n^2)$
* **Optimal Complexity:** $O(n)$ using a bitmask.

##### 3. Recommended Optimization: Bitmasking

Since the numbers are restricted to a small range (1–50), you can use a single `uint256` as a "seen" set. Each bit position represents a number. This reduces the process to a single loop.

```solidity

function _validateTicketNumbers(uint8[6] memory numbers) private pure {
    uint256 seen; 
    for (uint256 i = 0; i < 6; i++) {
        uint8 num = numbers[i];
        
        // Range Check
        if (num < MIN_NUMBER || num > MAX_NUMBER) {
            revert InvalidParam(InvalidParams.TicketNumbersOutOfRange);
        }

        // Duplicate Check: Shift 1 by the value of 'num'
        // If that bit is already 1, it's a duplicate.
        uint256 bit = 1 << num;
        if (seen & bit != 0) {
            revert InvalidParam(InvalidParams.TicketNumbersDuplicate);
        }
        seen |= bit;
    }
}

```

##### 4. Comparison Summary

| Metric | Original Function | Bitmask Optimized |
| --- | --- | --- |
| **Iterations** | 21 total (Range + Nested) | 6 total |
| **Logic** | Brute force comparison | Single-pass bitwise check |
| **Scaling** | Becomes expensive as `n` grows | Extremely cheap for `n < 256` |
| **Readability** | High (Basic loops) | Moderate (Requires bitwise logic) |

---

##### Final Assessment

**Status:** **Pass (Logical)** / **Fail (Optimization)**.
If this function is called frequently (e.g., during a ticket minting process), the $O(n^2)$ approach will waste user gas. Transitioning to a bitmask is the industry standard for small-set validation in Solidity.

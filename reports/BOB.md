# [Medium] Gateway strategy wrappers use total contract balance instead of freshly minted output, enabling residual balance capture

### Description
Several Gateway strategy wrappers derive transfer amounts from `balanceOf(address(this))` instead of the specific delta attributable to the current operation. This allows any residual balance (from rewards, dust, or accidental transfers) to be "swept" by the next caller.

### Root Cause Analysis
1.  **Known Oversight:** `BedrockStrategy.sol` contains an explicit `// ToDo: Missing corner case to check Insufficient supply provided.`, confirming the logic was known to be incomplete at deployment.
2.  **Inconsistent Implementation:** Contracts like `AvalonLendingStrategy` and `IonicStrategy` correctly implement delta-based accounting. The vulnerable contracts are a result of inconsistent application of these safe patterns.

### Security Consequences
- **Accounting Corruption:** `TokenOutput` events log the total balance (Deposit + Residual), corrupting off-chain tracking, yield calculations, and protocol analytics.
- **Griefing / Denial of Service:** An attacker can "poison" a strategy by donating a tiny amount of tokens to cause accounting mismatches and `revert` calls in downstream integrator contracts that expect exact balances.
- **Insolvency Risk:** Downstream vaults calculating shares based on received amounts will over-mint shares for the attacker, diluting honest liquidity providers.

### Vulnerable Code (Example: BedrockStrategy.sol)
```solidity
vault.mint(address(tokenSent), amountIn);
IERC20 uniBTC = IERC20(vault.uniBTC());
uint256 uniBTCAmount = uniBTC.balanceOf(address(this)); //@audit  Vulnerable: includes residual balance
uniBTC.safeTransfer(recipient, uniBTCAmount);
```

### Recommendation
Always use the "Delta Pattern": Snapshot the balance before the external action and transfer only the difference.
```solidity
uint256 beforeBal = outputToken.balanceOf(address(this));
// ... perform deposit/mint ...
uint256 amountOut = outputToken.balanceOf(address(this)) - beforeBal;
outputToken.safeTransfer(recipient, amountOut);
```
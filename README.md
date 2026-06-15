# About muslimethguard

I'm an independent Web3 Security Researcher competing on Sherlock, C4, and Codehawks. Skilled in Rust, Solidity. Currently triaging at Pashove Audit Group.

I specialize in identifying vulnerabilities in blockchain protocols and EVM protocols/Bridges/Cross-chain

For collabs or security audits, reach out on X [muslimethguard](https://x.com/lhoussainePh).




# First Flights Audits

| ID | Protocol | Ecosystem | Description | Findings | Report |
| --- | --- | --- | --- | --- | --- |
| 1 | 2403-PasswordStore | Solidity - EVM | PasswordStore is a protocol dedicated to storage and retrieval of a user's passwords. The protocol is designed to be used by a single user, and is not designed to be used by multiple users. Only the owner should be able to set and access this password.  | 2 H, 1 I | [🔗](./reports/PasswordStore.md) |
| 2 | 2403-Puppy-Raffle | Solidity - EVM | Puppy Rafle is a protocol dedicated to raffling off puppy NFTs with variying rarities. A portion of entrance fees go to the winner, and a fee is taken by another address decided by the protocol owner.  | 3 H, 4 M , 1 L ,2 G, 6 I | [🔗](./reports/Puppy-Raffle.md) |
| 3 | 2405-TSwap | Solidity - EVM | TSWAP is an constant-product AMM that allows users permissionlessly trade WETH and any other ERC20 token set during deployment. Users can trade without restrictions, just paying a tiny fee in each swapping operation. Fees are earned by liquidity providers, who can deposit and withdraw liquidity at any time.  | 3 H, 3 M, 2 L, 5 I | [🔗](./reports/TSwap.md) |



# Private Audits

| ID | Protocol | Ecosystem | Description | Findings | Report |
| --- | --- | --- | --- | --- | --- |
| 1 | 2602-Hopex | Solidity - EVM | Cross-chain lottery protocol built on LayerZero V2                               | 4 H, 4 M, 2 L | [🔗](./reports/2602-Hopex.md) |



# Competitive Audits

| ID | Protocol | Ecosystem | Platform | Description | Findings | Report |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Hybra Finance | Solidity - EVM | C4 | Decentralized public liquidity layer for Hyperliquid | 1 H | [🔗](https://code4rena.com/audits/2025-10-hybra-finance/submissions/S-600)  |
| 2 | Notional Exponent | Solidity - EVM | Sherlock | Notional Exponent is a leveraged yield protocol. Notional Exponent enables users to borrow from Morpho to establish leveraged staking, leveraged PT, and leveraged liquidity strategies. | 2 M | [🔗](https://audits.sherlock.xyz/contests/1001) |
| 3 | Next-generation | Solidity - EVM | C4 | — | 1 M | [🔗](https://code4rena.com/audits/2025-01-next-generation/submissions/S-778) |
| 4 | Aave DIVA Wrapper | Solidity - EVM | Codehawks | — | 1 L | [🔗](https://codehawks.cyfrin.io/c/2025-01-diva/s/317) |



# Bug Bounty

| ID | Protocol | Ecosystem | Platform | Description | Finding | Status | Report |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | The Flipcash Reserve Contract | Solana - SPL | Hackenproof | Bonding curve / stablecoin reserve | Permissionless SPL transfer to pool vault decouples bonding curve state, enabling complete drain of USDF from any live pool | OOS | [🔗](./reports/Flipcash.md) |
| 2 | Ethena | Solidity - EVM | Immunefi | Staking / reward distribution | Privileged State Desynchronization in `redistributeLockedAmount()` Enables Double Asset Claims | DUP | [🔗](./reports/Ethena.md) |
| 3 | Berachain | Solidity - EVM | Immunefi | BEX Vault (Balancer V2 Fork) | Internal Balance Pre-Poisoning Vulnerability Allowing Fund Theft When Adding Newly-Deployed Tokens | DUP | [🔗](./reports/Berachain.md) |
| 4 | Flare \| FAssets Vault | Solidity - EVM | Immunefi (Audit Comp) | Collateral reservation | Fee Loss on Rejected Collateral Reservation Due to Transfer Failure | OOS | [🔗](./reports/Flare.md) |
| 5 | BOB | Solidity - EVM | Remmedy | Cross-chain, L2 | Gateway strategy wrappers use total contract balance instead of freshly minted output, enabling residual balance capture | DUP | [🔗](./reports/BOB.md) |
| 6 | PHUX | Solidity - EVM | External | AMM (Balancer V2 fork) | Token frontrun vulnerability | — | [🔗](https://forum.balancer.fi/t/balancer-v2-token-frontrun-vulnerability-disclosure/6309) |

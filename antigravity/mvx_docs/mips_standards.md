# MultiversX Improvement Proposals (MIPs) Standards

## Protocol Standards (Sharding & Sovereign Chains)

### MIP-7: Sovereign Shards

**Concept**: Dedicated "AppChains" that inherit MultiversX security but operate with independent execution shards.

- **Why**: High throughput (gaming), Custom VM/Fees, Sovereignty.
- **Architecture**:
  - **Validators**: Stake in a `SovereignValidatorsStaking` contract on Mainnet (EGLD).
  - **Rewards**: Earn Mainnet rewards + Sidechain fees.
  - **Slashing**: Mainnet contract handles slashing of EGLD stake based on sidechain proofs.

### MIP-12: Settlement & Data Availability

**Mechanism**: Sovereign Chains store state roots on MultiversX Mainnet.

- **Escape Hatch**: Users can prove funds on Mainnet using Merkle Proofs against the last trusted RootHash if the Sovereign Chain halts.
- **Rollup**: Evaluating ZK-proofs for full state validity (long-term).

### MIP-14: Chain Factory & Governance

**Deployment**:

- **Factory SC**: A "God Contract" without an admin key (DAO controlled) that deploys new Sovereign Chains.
- **Config**: Deterministic ChainIDs + ESDT Prefixes.
- **Controller**: A DAO contract that manages upgrades to the Factory.

### MIP-20: EthereumX (L2 via Sovereign Chains)

**Concept**: An Ethereum Layer 2 architecture built on MultiversX Sovereign Chains, offering EVM compatibility via SpaceVM (WASM abstraction).

- **Throughput**: Target ~4,000 TPS for DeFi/Gaming.
- **Security**:
  - **No Re-entrancy**: Inherits MultiversX's async model features where possible to prevent re-entrancy attacks.
  - **No Infinite Approvals**: Designed to prevent wallet drain attacks common in EVM.
- **Components**:
  - **Native Cross-Chain Operation**: Direct async calls between L2 and Mainnet.
  - **Atomic Wrapping**: ESDT ↔ ERC20 wrapping happens atomically during execution (no pegged token risk).
  - **Decentralized Sequencer**: No single point of failure; impervious to standard MEV vector.

### MIP-27: Supernova (Sub-second Finality)

**Mechanism**: Decouples execution from consensus to achieve rapid finality.

- **Propose-Vote-Execute**: Validators vote on headers before full execution is verified, trusting the "Execution-Result Inclusion Estimator" (EIE).
- **Slot Time**: Targets 600ms (down from 6s).
- **EIE**: A deterministic rule for safe notarization limits.

---

## Token Standards (ESDT & NFT 2.0)

### MIP-1: Composable NFTs/SFTs

**Feature**: NFTs that can "own" other tokens (ESDTs, NFTs).

- **Equipping**: Logic for treating attached NFTs as "equippable" items (gaming context).
- **Implementation**: Needs a `MergableNFT` standard where the parent NFT holds the child NFTs in its custody.
- **Royalties**: Automatic distribution to original creators upon sale of the composite asset.

### MIP-2: Fractional Ownership

**Concept**: Split an NFT into fungible "shares" (SFTs or MetaESDTs).

- **Logic**:
  - Contract holds the original NFT.
  - Mints `1000` SFT units representing 1/1000th ownership.
  - **Redemption**: Burn 1000 SFTs = Unlock original NFT.

### MIP-5: Dynamic Resources

**Problem**: Immutable attributes/URIs limit game logic.
**Solution**:

- **`ESDTModifyRoyalties`**: Change royalty per-token.
- **`ESDTModifyCreator`**: Transfer "Creator" role to another address.
- **Dynamic URIs**: Allow appending/removing URIs without burning/reminting (if `canModify` role is set).

---

## Staking & Governance

### MIP-30: Bot Wallet Automation

**Concept**: Allows delegators to keep cold wallets offline while authorizing a "Bot Wallet" to perform routine maintenance tasks.

- **Operations**: `ClaimRewards`, `ReDelegate` (Compound).
- **Architecture**:
  - **Delegation Manager**: Central registry of `Owner -> Bot` authorizations.
  - **Batch Endpoints**: `claimMultiOf`, `reDelegateMultiOf` for processing multiple users in one transaction (Gas limit considerations apply).
  - **Security**: Bot cannot transfer funds, only reinvest or claim to the Owner's address. Unstaking is NOT allowed by the bot.
- **Activation**: Controlled by `DelegationOnBehalfFlag`.

---

## User Experience Standards

### Transaction Simulation Standard

**Goal**: Wallets MUST show a "diff" of the transaction before signing (What goes in vs. What comes out).

- **Endpoint**: `/transaction/simulate` on Proxy/Gateway.
- **Features**:
  - **Asset Changes**: Detailed list of ESDT/NFTs leaving and entering the wallet.
  - **State Changes**: Updates to Storage Mappers (visible in simulation trace).
  - **Failure Prediction**: Warns if the transaction is likely to fail or run out of gas.
- **Security**: Critical defense against "blind signing" phishing attacks.

---

## Index of Known MIPs (Agora)

| MIP | Title | Category |
| :--- | :--- | :--- |
| **MIP-1** | Composable NFTs/SFTs | Token |
| **MIP-2** | Fractional Ownership | Token |
| **MIP-3** | Game Lobby SC Template | SC |
| **MIP-4** | TransferRole & Composability | Token |
| **MIP-5** | dynamic resources | Token |
| **MIP-6** | xCalendar & Indices | Meta |
| **MIP-7** | Sovereign Shards | Protocol |
| **MIP-12** | Settlement & Data Availability | Protocol |
| **MIP-14** | Chain Factory | Protocol |
| **MIP-15** | Sovereign Interoperability | Protocol |
| **MIP-19** | BitcoinX (BTC L2) | Protocol |
| **MIP-20** | EthereumX (EVM L2) | Protocol |
| **MIP-21** | SolanaX (Solana L2) | Protocol |
| **MIP-22** | Account Abstraction (Mobile) | UX |
| **MIP-27** | Supernova (Sub-second Finality) | Protocol |
| **MIP-28** | ZK Proofs & ECC Primitives | VM |
| **MIP-29** | Staking V5 (Economics) | Protocol |
| **MIP-30** | Bot Wallet Automation | Staking |

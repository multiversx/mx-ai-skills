---
name: mvx_protocol_experts
description: Expert knowledge of the MultiversX Protocol, Consensus (SPoS), Sharding, and Standard Implementations (MIPs).
---

# MultiversX Protocol Expertise

This skill encompasses deep knowledge of the core MultiversX protocol architecture, utilized for reviewing protocol-level changes, complex dApp architectures involving cross-shard logic, and sovereign chain integrations.

## 1. Core Architecture: Adaptive State Sharding
- **Sharding Types**:
    - **Network Sharding**: Nodes are distributed into shards.
    - **Transaction Sharding**: Transactions are processed by the shard containing the sender's account.
    - **State Sharding**: Each shard maintains a portion of the global state (account space).
- **Metachain**:
    - The coordinator shard.
    - Handles: Validator churning, Epoch/Round verification, Slashing, Header notarization.
    - **Does NOT** execute general smart contracts (except for system contracts like Staking, ESDT management).
- **Cross-Shard Transactions**:
    - **Async (Process -> Relay -> Execute)**: Sender Shard processes TX -> Generates internal transaction (Scratchpad) -> Relayer -> Receiver Shard executes.
    - **Atomicity**: Cross-shard is atomic but asynchronous.

## 2. Consensus: Secure Proof of Stake (SPoS)
- **Selection**: Deterministic selection of validators based on stake + rating.
- **BLS Signing**: Validators sign proposed blocks using BLS multi-signatures.
- **Finality**: Instant finality (within seconds) due to PBFT-like consensus within the shard.

## 3. Token Standards (ESDT)
- **Native Implementation**: Tokens are NOT smart contracts (unlike ERC-20). They are part of the protocol account state.
- **ESDT Properties**:
    - **CanFreeze**: Protocol can freeze assets if configured.
    - **CanWipe**: Protocol can wipe assets if configured.
    - **CanPause**: Protocol can pause transfers.
- **Roles**:
    - `ESDTRoleLocalMint`: Allows minting.
    - `ESDTRoleLocalBurn`: Allows burning.

## 4. Sovereign Chains & Interoperability
- **Sovereign Chains**: Dedicated blockchains extending MultiversX.
    - **Gateway Contract**: Handles bridging between Mainchain and Sovereign Chain.
    - **Settlement**: Sovereign Chains settle proofs to the MultiversX Mainchain (acting as Layer 1).
- **Interoperability Standards**:
    - **MIP-X**: Monitor [MIPs] for latest standard adoption.

## 5. Transaction Processing
- **Gas Schedule**: Operations have deterministic costs.
- **Built-in Functions**:
    - `ESDTTransfer`: Direct transfer.
    - `ESDTNFTTransfer`: Transfer of SFT/NFT.
    - `MultiESDTNFTTransfer`: Batch transfer (often used for Contract calls).

## 6. Critical Checks for Protocol Developers
- **Intra-shard vs Cross-shard**:
    - Always assume a call *might* be cross-shard unless verified co-location.
    - **Async**: If logic depends on the result of a call to another address, it MUST be an Async Call (Callbacks).
- **Reorg Handling**:
    - Microservices must handle block rollbacks (Standard: wait for 'Finalized' status, or handle re-org events).

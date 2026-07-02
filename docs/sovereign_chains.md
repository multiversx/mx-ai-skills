# Sovereign Chains Architecture

Sovereign Chains are the MultiversX solution for Layer 2 scaling and specialized application logic. They operate as independent blockchains that settle specific data on the MultiversX Mainnet.

## Core Concepts (MIP-7 & MIP-14)

### Sovereignty

- **Custom VM**: Can run specialized execution environments (e.g., SpaceVM for EVM support, SolanaVM).
- **Independent Gas**: Fee market is isolated from Mainnet.
- **Security Inheritance**: Security is derived from MultiversX Mainnet via the "Validator Staking" contract.

### Validator Model

- **Staking**: Validators stake EGLD on the MultiversX Mainnet (`SovereignValidatorsStaking` contract).
- **Slashing**: Evidence of fraud/misbehavior on the Sovereign Chain is submitted to the Mainnet contract, which slashes the EGLD stake.
- **Consensus**: The Sovereign Chain runs its own consensus (e.g., SPoS) but rotates its validator set based on the Mainnet contract's registry.

---

## Interoperability & Bridging

### Gravity Bridge (Ad-Astra)

The native bridging mechanism for transferring assets between Mainnet and Sovereign Chains.

#### Transfer Flow (Mainnet -> Sovereign)

1. **Lock**: User sends ESDT/EGLD to the `SovereignBridge` contract on Mainnet.
2. **Event**: Contract emits a `Deposit` event.
3. **Relay**: Sovereign Chain validators observe the event.
4. **Mint**: Corresponding tokens are minted on the Sovereign Chain users' address.

#### Transfer Flow (Sovereign -> Mainnet)

1. **Burn**: User "burns" or locks tokens on Sovereign Chain.
2. **Proof**: A batch of withdrawals is aggregated.
3. **Execute**: The batch proof is submitted to the Mainnet `SovereignBridge`.
4. **Unlock**: Mainnet contract verifies the proof and releases the original assets.

---

## Ecosystem Implementations

### EthereumX (MIP-20)

- **Goal**: Full EVM compatibility.
- **Tech**: SpaceVM (WASM abstraction layer).
- **Features**: Native Async calls to Mainnet, unbiasable random, atomic wrapping.

### BitcoinX (MIP-19)

- **Goal**: Bitcoin Layer 2.
- **Tech**: Uses Sovereign Chain finality to anchor Bitcoin transactions.

---

## Settlement (MIP-12)

- **State Roots**: Sovereign Chains submit their State Root Hash to Mainnet periodically.
- **Data Availability**: Can use Mainnet (expensive, secure) or external DA layers (cheaper).
- **ZK Rollups**: Future-proof design allows replacing the "Optimistic" or "PoS" settlement with Zero-Knowledge Validity Proofs.

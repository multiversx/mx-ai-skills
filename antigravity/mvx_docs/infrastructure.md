# Protocol Infrastructure & Architecture

## Adaptive State Sharding

MultiversX implements **Adaptive State Sharding**, a technique that combines three types of sharding to scale linearly with the number of nodes.

### 1. State Sharding

- **Concept**: The global state (accounts, smart contracts) is partitioned across shards.
- **Mechanism**: Account addresses determine their shard affinity. `shard_id = last_bits_of_pubkey`.
- **Benefit**: Nodes only need to store the state of their specific shard, not the entire ledger.

### 2. Network Sharding

- **Concept**: Processing nodes are grouped into shards.
- **Mechanism**: Validators are shuffled between shards at the end of each **Epoch** (24 hours) to prevent collusion.
- **Benefit**: Optimizes peer-to-peer communication; messages primarily propagate within the shard.

### 3. Transaction Sharding

- **Concept**: Transactions are mapped to shards based on the sender's address.
- **Cross-Shard**: If Sender and Receiver are in different shards, the transaction is processed in the Sender's shard, and a "MiniBlock" is propagated to the Receiver's shard for finalization.

---

## Secure Proof of Stake (SPoS)

SPoS is the consensus mechanism ensuring security and efficiency.

### Consensus Components

- **Randomness**: A "Randomness Beacon" is generated from the previous block's signature (BLS). It is unpredictable and unbiasable.
- **Block Proposer**: Selected deterministically based on the Randomness and their Stake. `Hash(PubKey + Randomness)` determines the priority.
- **Validators**: A consensus group (subset of the shard's validators) is selected to attest the block.

### Finality

- **Single-Block Finality**: MultiversX achieves probabilistic finality almost immediately and economic finality very quickly due to high redundancy & BLS signature aggregation.
- **Supernova Upgrade**: Aims for **Sub-second Finality** by decoupling block proposal from execution/verification.

---

## The Metachain

The Metachain is a specialized "Shard 0xFF" that coordinates the network.

- **Responsibilities**:
  - **Validator Management**: Registering/Unregistering nodes, Staking operations.
  - **Epoch Management**: Triggers the end of an epoch and calculates the validator shuffle (Cross-Shard reconfiguration).
  - **Header Notarization**: Stores the headers of all shard blocks to facilitate cross-shard communication and immutability.
  - **Slashing**: Processes evidence of misbehavior.
- **Not for User TXs**: It does *not* process standard user transfers or smart contracts unless they relate to staking/governance.

---

## Observer Infrastructure

To interact with the network without staking, "Observer" nodes are used.

### Observing Squad

A recommended production setup for dApps/Exchanges:

- **Components**: `N` Observers (one for each Shard) + 1 Observer (Metachain) + `mx-chain-proxy`.
- **Proxy**: Aggregates data. When you query an address, the Proxy routes the request to the specific Shard Observer that holds that address.
- **Modes**:
  - **Full Archive**: Stores entire history (Heavy disk usage).
  - **DbLookup**: Optimized for recent history and state queries.

### Elasticsearch

- **Use Case**: Complex queries (e.g., "All txs by user X in range Y-Z", "All NFT mints") that are inefficient to run against a standard Node API.

## Data Indexing (Elasticsearch)

The `mx-chain-es-indexer-go` plugin pushes blockchain data to Elasticsearch.

### 1. Operations Index (formerly transactions)

Stores all processed operations.

- **Key Fields**:
  - `sender`, `receiver`: Bech32 addresses.
  - `value`: Amount transferred (in Wei).
  - `status`: `success` or `fail`.
  - `function`: The SC function called.
  - `hasScResults`: Boolean, indicating if usage of Smart Contract Results (internal txs).

### 2. Accounts Index

Snapshot of account state (updated per block).

- **Key Fields**:
  - `address`: Bech32 address (`_id`).
  - `balance`: Current EGLD balance.
  - `shard`: Shard ID (0, 1, 2, or 4294967295/Metachain).
  - `developerRewards`: Accumulated developer rewards (if SC).

### 3. Events Index (formerly logs)

Crucial for dApps listening to contract updates.

- **Structure**:
  - `identifier`: The Event Name (e.g., `swap`, `deposit`).
  - `address`: The Contract Address emitting the event.
  - `topics`: Indexed parameters (base64 encoded). First topic usually matches `identifier`.
  - `data`: Non-indexed data payload.

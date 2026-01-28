# Testing & Simulation Guide

## 1. Chain Simulator (`mx-chain-simulator-go`)

A binary that mimics a local Testnet without consensus delays.

- **Speed**: Blocks generated on-demand (HTTP Trigger) or instantly.
- **Key Features**:
  - **No Consensus**: Single node, instant execution.
  - **State Injection**: Can set storage/balance for any address arbitrarily.

## 2. Chain Simulator API Reference

### Block Generation

- **`POST /simulator/generate-blocks/:num`**: Advance `num` blocks (per shard).
- **`POST /simulator/generate-blocks-until-transaction-processed/:txHash`**: Advance until `txHash` is executed. **Critical for Integration Tests**.

### State Manipulation

- **`POST /simulator/set-state`**: Inject balance or storage.

  ```json
  [
    {
      "address": "erd1...",
      "balance": "1000000000000000000",
      "keys": {
        "73756d": "0a" // key "sum" = 10
      }
    }
  ]
  ```

- **`POST /simulator/force-epoch-change`**: Jump to next epoch (trigger expiration/shuffles).

### Configuration (`config/config.toml`)

- `auto-generate-blocks`: Set to `false` for deterministic testing.
- `round-duration-in-milliseconds`: Set low (e.g. 6000) or high depending on test needs.

## 3. Testing Layers

### Level 1: Rust Unit Tests (RustVM)

- **Tool**: `cargo test` in contract folder.
- **Speed**: Instant (<1s).
- **Scope**: Logic verification, math errors, permission checks.
- **Mocking**: No IO, purely logic.

### Level 2: Scenario Integration Tests (RustVM / Go)

- **Tool**: Mandos (`.scen.json`) or Go scenarios.
- **Scope**: Interaction between multiple contracts, async calls, gas consumption estimates.
- **Runner**: `sc-meta test-gen` to generate Go tests from Scenarios.

### Level 3: Chain Simulator (Blackbox System Test)

- **Tool**: `mx-chain-simulator-go` + Python/Go scripts.
- **Scope**: Real transaction broadcasting, HTTP API responses, Indexing verification.
- **Ideal for**: `mvx-microservice-developer` testing their off-chain listeners.

### Level 4: Devnet / Testnet

- **Scope**: Final integration, Wallet connectivity, Latency checks.

## Best Practice Workflow

1. **Code**: Write logic.
2. **Unit Test**: 100% coverage on math/logic.
3. **Mandos**: Verify every endpoint & async flow.
4. **Simulator**: CI pipeline runs full system test.
5. **Devnet**: Manual verification & Beta testing.

---
name: mvx_testing_handbook
description: Guide to mandos (scenarios), rust-vm unit tests, and chain-simulator.
---

# MultiversX Testing Handbook

This skill provides expert guidance on the 3 layers of MultiversX testing: RustVM Unit Tests, Mandos Integration Tests, and Chain Simulation.

## 1. RustVM Unit Tests
- **Location**: `#[cfg(test)]` modules within contract files or `tests/` directory.
- **Speed**: Instant.
- **Scope**: Internal logic, strict math, private functions.
- **Mocking**: Use `multiversx_sc_scenario::imports::*` to mock the blockchain environment (`blockchain_mock.set_caller(addr)`).

## 2. Mandos Scenarios (`.scen.json`)
- **Location**: `scenarios/` directory.
- **Requirement**: "If it's an endpoint, it MUST have a Mandos test."
- **Execution**:
    - `cargo test-gen` generates Rust wrappers for scenarios.
    - `cargo test` runs them.
- **Key Concepts**:
    - `step: setState`: Initialize accounts and balances.
    - `step: scCall`: Execute endpoint.
    - `step: checkState`: Verify storage matches expected values.

## 3. Chain Simulator (`mx-chain-simulator-go`)
- **Location**: Separate system test suite (often Go or Python).
- **Scope**: Interaction with Node API, complex cross-shard reorgs, off-chain indexing.
- **Tool**: Use `POST /simulator/generate-blocks` to force execution.

## 4. Coverage Strategy
- **Money In/Out**: 100% Mandos coverage required.
- **View Functions**: Unit test coverage sufficient.
- **Access Control**: Explicit negative tests (expect error 4) for unauthorized calls.

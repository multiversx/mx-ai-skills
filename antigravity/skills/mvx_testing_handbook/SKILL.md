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
- **Endpoint Coverage**: Every `#[endpoint]` must have at least one Mandos scenario.
- **Payable Endpoints**: Every `#[payable]` endpoint must test: correct token, wrong token, zero amount, unauthorized caller.
- **Role-Gated Endpoints**: Every `#[only_owner]` / `#[only_role]` must have a negative test (unauthorized caller → expect error).
- **State Transitions**: State transitions must be verified with `checkState` in Mandos.

## 5. Test Quality Scoring Rubric

Score each category, then average for overall score.

| Category | 0-2 (Poor) | 3-5 (Partial) | 6-8 (Good) | 9-10 (Excellent) |
|----------|-----------|---------------|------------|-------------------|
| Unit Tests | No tests | Some functions tested | Core logic covered | All functions + edge cases |
| Mandos Coverage | No scenarios | Some endpoints covered | All endpoints covered | All endpoints + error paths |
| Access Control Tests | No auth tests | Owner-only tested | All roles tested | All roles + negative tests |
| Money Flow Tests | No payment tests | Happy path only | All payment endpoints | Happy + error + edge amounts |
| Chain Simulator | Not available | Setup exists, not run | Basic flows tested | Cross-shard + reorg tested |

### Scoring Formula
- Start at 5 (baseline for "tests exist and pass").
- +1 for each category rated Good (6-8).
- +2 for each category rated Excellent (9-10).
- -1 for each category rated Poor (0-2).
- -2 if Chain Simulator is not available.
- Cap at 10, floor at 1.

## 6. Output Format

### Test Quality Report
```
Test Execution:
  cargo test: [pass/fail/skip counts]
  Mandos scenarios: [pass/fail counts]
  Chain Simulator: [available: Y/N] [pass/fail if available]

Coverage Assessment:
  Endpoints with Mandos: [N/total] ([%])
  Payable endpoints fully tested: [N/total]
  Access control negative tests: [N/total]
  State transition checks: [N/total]

Category Scores:
  Unit Tests: [0-10]
  Mandos Coverage: [0-10]
  Access Control Tests: [0-10]
  Money Flow Tests: [0-10]
  Chain Simulator: [0-10]

Overall Score: [1-10]

Gaps:
- [list of endpoints without tests]
- [list of missing negative tests]
- [list of untested payment scenarios]
```

## Completion Criteria
Test quality assessment is complete when:
1. All tests have been executed (cargo test + Mandos).
2. Every category in the scoring rubric has been evaluated.
3. Overall score is calculated.
4. Gaps are documented.

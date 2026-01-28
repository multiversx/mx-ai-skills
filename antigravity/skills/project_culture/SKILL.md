---
name: project_culture
description: Assess code maturity based on MultiversX standards (docs presence, testing culture).
---

# Project Culture & Code Maturity Index

This skill helps you gauge the quality and reliability of a codebase based on "smells" and cultural indicators.

## 1. Documentation Quality
- **Docs Presence**: Is there a `README.md`? Does it explain *how* to build and test?
- **MultiversX Specifics**:
    - `mxpy.json`: Indicates use of standard build tools.
    - `snippets.sh` or `interaction/`: Scripts for deploying/interacting.
- **Specs**: Are there `.md` files in `specs/` or `whitepaper/`?

## 2. Testing Culture
- **Unit Tests**: Run `cargo test`. Are there tests? Do they cover edge cases or just "happy path"?
- **Integration Tests**: Look for `scenarios/` (Mandos).
    - *Gold Standard*: `scen.json` files covering every endpoint.
    - *Red Flag*: No scenarios, or only `deployment.scen.json`.
- **Simulation**: Is `mx-chain-simulator-go` mentioned in CI or scripts?

## 3. Code Hygiene
- **Linter**: Does `cargo clippy` pass?
- **Magic Numbers**: Are there raw integers (e.g., `86400`) instead of named constants (`SECONDS_PER_DAY`)?
- **Comments**: Are complex blocks explained?
- **Unwrap**: Excessive use of `unwrap()` instead of `sc_panic!` or checks.

## 4. Dependency Mangement
- **Lockfile**: Is `Cargo.lock` committed?
- **Version Pinning**: Are `multiversx-sc` versions pinned or wildcard `*`? (Wildcard is bad).

## Scoring (Mental Model)
- **High Maturity**: Full Mandos coverage, detailed specs, clean clippy, pinned versions. -> **Audit focus**: Business logic flaws.
- **Low Maturity**: No tests, spaghetti code, `unwrap()` everywhere. -> **Audit focus**: Basic safety, reentrancy, arithmetic overflow, DoS.

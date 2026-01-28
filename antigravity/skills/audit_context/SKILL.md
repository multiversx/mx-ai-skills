---
name: audit_context
description: Guidelines for establishing context before an audit.
---

# Audit Context Building

This skill helps you rapidly build a mental model of a codebase before diving into vulnerability hunting.

## 1. Reconnaissance
- **Identify the Core**: Where is the money / critical logic?
    - *MultiversX*: Look for `#[multiversx_sc::contract]`, `#[payable("*")]`, and `impl` blocks.
- **Identify Externalities**:
    - Which other contracts does this interact with?
    - Are there hardcoded addresses? (e.g., `sc:` smart contract literals).
- **Identify Documentation**:
    - `README.md`, `specs/`, `whitepaper.pdf`.
    - *MultiversX*: `mxpy.json` (build config), `multiversx.yaml`, `snippets.sh`.

## 2. System Mapping
Create a mental (or written) map of the system.
- **Roles**: Who can do what? (`Owner`, `Admin`, `User`, `Whitelisted`).
- **Assets**: What tokens are flowing? (EGLD, ESDT, NFT, SFT).
- **State**: What is stored? (`SingleValueMapper`, `VecMapper`).

## 3. Threat Modeling (Initial)
- **Asset at Risk**: If this contract fails, what is lost?
- **Attacker Profile**: External user? Malicious admin? Reentrant contract?
- **Entry Points**: List all `#[endpoint]` functions. Which ones are unchecked?

## 4. Environment Check
- **Language Version**: Is `cargo.toml` using a recent `multiversx-sc` version?
- **Test Suite**: Does `scenarios/` exist? Run `sc-meta test-gen` to see if tests are up to date.

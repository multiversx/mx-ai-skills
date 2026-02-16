---
name: audit_context
description: Guidelines for establishing context before an audit.
---

# Audit Context Building

This skill helps you rapidly build a mental model of a codebase before diving into vulnerability hunting.

## 1. Reconnaissance
- **Identify the Core**: Where is the money / critical logic?
    - *MultiversX*: Look for `#[multiversx_sc::contract]`, `#[payable]`, and `impl` blocks.
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

## Output Format

### Audit Context Report
```
Contract: [name]
Commit: [hash]
Framework: multiversx-sc [version from Cargo.toml]
Test Suite: [scenarios/ exists: Y/N] [test count]

System Overview:
- Core Logic: [1-2 sentence description of what the contract does]
- Value Flow: [how money/tokens move through the contract]

Roles:
| Role | Access Level | Endpoints |
|------|-------------|-----------|
| Owner | #[only_owner] | [list] |
| Admin | #[only_role] | [list] |
| User | Public | [list] |

Assets:
| Token | Type | Roles Held | Flow |
|-------|------|------------|------|
| [id] | EGLD/ESDT/NFT/SFT | Mint/Burn/Transfer | [in/out/both] |

External Dependencies:
| Contract/Service | Interaction Type | Risk |
|-----------------|-----------------|------|
| [address/name] | sync_call/async/proxy | [High/Medium/Low] |

Async Call Graph:
[contract A] --async_call--> [contract B] --callback--> [contract A]

Threat Summary:
- Assets at risk: [what can be stolen/locked/inflated]
- Attacker profiles: [external user / malicious admin / reentrant contract]
- Highest-risk entry points: [top 3 endpoints by risk]

Scope Determination:
- Upgrade: [Y/N]
- DeFi: [Y/N]
- Multi-contract: [Y/N]
```

## Completion Criteria
Context building is complete when:
1. All roles and their permissions are documented.
2. All assets and their flows are mapped.
3. All external dependencies are identified.
4. Threat summary identifies at least one risk per attacker profile.
5. Scope determination is filled (drives which auditor phases apply).

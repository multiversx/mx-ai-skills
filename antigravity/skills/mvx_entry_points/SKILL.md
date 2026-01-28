---
name: mvx_entry_points
description: Identify and analyze MultiversX Smart Contract entry points (#[endpoint], #[view], #[payable]).
---

# MultiversX Entry Point Analyzer

This skill helps you identify the attack surface of a smart contract by enumerating all public interaction points.

## 1. Identification
Scan for `multiversx_sc` macros that expose functions:
- **`#[endpoint]`**: Public write function. **High Risk**.
- **`#[view]`**: Public read function. Low risk (unless used on-chain).
- **`#[payable("*")]`**: Accepts EGLD/ESDT. **Critical Risk** (value handling).
- **`#[init]`**: Constructor.
- **`#[upgrade]`**: Upgrade handler. **Critical Risk** (migration logic).

## 2. Risk Classification

### A. Non-Payable Endpoints
Functions that change state but don't accept value.
- *Check*: Is there `require!`? Who can call this (Owner only?)?
- *Risk*: Unauthorized state change.

### B. Payable Endpoints (`#[payable("*")]`)
Functions receiving money.
- *Check*:
    - Does it use `self.call_value().all_esdt_transfers()`?
    - Is `amount > 0` checked?
    - Are token IDs validated?
- *Risk*: Stealing funds, accepted fake tokens.

### C. Views
- *Check*: Does it modify state? (It shouldn't, but Rust allows interior mutability or misuse).

## 3. Analysis Workflow
1.  **List all Entry Points**.
2.  **Tag Access Control**: `OnlyOwner`, `Whitelisted`, `Public`.
3.  **Tag Value handling**: `Refusable`, `Payable`.
4.  **Graph Data Flow**: Which storage mappers do they touch?

## 4. Specific Attacks
- **Privilege Escalation**: Is a sensitive endpoint accidentally public?
- **DoS**: public endpoint inserting into UnorderedSetMapper (unbounded growth).

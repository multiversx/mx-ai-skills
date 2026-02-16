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
- **`#[payable]`**: Accepts EGLD/ESDT. **Critical Risk** (value handling).
- **`#[init]`**: Constructor.
- **`#[upgrade]`**: Upgrade handler. **Critical Risk** (migration logic).

## 2. Risk Classification

### A. Non-Payable Endpoints
Functions that change state but don't accept value.
- *Check*: Is there `require!`? Who can call this (Owner only?)?
- *Risk*: Unauthorized state change.

### B. Payable Endpoints (`#[payable]`)
Functions receiving money.
- *Check*:
    - Does it use `self.call_value().all()` or `self.call_value().single()`?
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

## Output Format

### Entry Point Inventory
```
| # | Endpoint | Type | Payable | Access Control | Risk Level | Storage Touched | Location |
|---|----------|------|---------|----------------|------------|-----------------|----------|
| 1 | stake | endpoint | EGLD | Public | Critical | user_stake, total_staked | src/lib.rs:42 |
| 2 | claim | endpoint | No | Public | High | user_rewards | src/lib.rs:87 |
| 3 | set_fee | endpoint | No | #[only_owner] | Medium | fee_percent | src/admin.rs:12 |
| 4 | get_balance | view | No | Public | Low | - | src/views.rs:5 |
| 5 | init | init | No | Deploy only | Critical | all mappers | src/lib.rs:1 |
```

### Attack Surface Summary
```
Total endpoints: [N]
  Critical (payable/init/upgrade): [N]
  High (state-changing, public): [N]
  Medium (state-changing, restricted): [N]
  Low (views): [N]

Unchecked payable endpoints: [list or "none"]
Public state-changing without access control: [list or "none"]
Endpoints missing from Mandos scenarios: [list or "none"]
```

## Completion Criteria
Entry point analysis is complete when:
1. Every `#[endpoint]`, `#[view]`, `#[payable]`, `#[init]`, `#[upgrade]`, and `#[callback]` is in the inventory table.
2. Every entry has a risk classification.
3. Attack Surface Summary is filled.
4. Any endpoint that is payable without token validation is flagged.

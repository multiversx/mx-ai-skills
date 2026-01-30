---
description: Checklist of Best Practices for MultiversX Smart Contracts.
---

# MultiversX SC Best Practices Checklist

Use this workflow to audit or review any smart contract implementation.

## 1. Safety & Security

- [ ] **Arithmetic**: Are all financial calculations using `BigUint`?
  - *Bad*: `u64 + u64` (risk of overflow, though panic is safe, it handles poorly).
  - *Good*: `self.balance().get() + amount`.
- [ ] **Re-entrancy**: Are storage updates (Effects) happening *before* external calls (Interactions)?
  - *Check*: Look for `self.send().direct_...` or `contract_call`. Ensure state is settled before these lines.
- [ ] **Access Control**: Do admin endpoints have `#[only_owner]` or `#[only_admin]`?

## 2. Gas Optimization

- [ ] **Storage Mappers**:
  - Use `SingleValueMapper` for single items.
  - Use `UnorderedSetMapper` for collections where order doesn't matter (cheaper than Vec).
  - Avoid `VecMapper` for large lists (O(N) costs).
- [ ] **Data Types**:
  - Avoid `String` or `Vec<u8>` for passing data if `ManagedBuffer` can be used.
  - `ManagedBuffer` interacts directly with VM memory handle.

## 3. Correctness

- [ ] **Payable**: Do endpoints receiving funds have `#[payable("*")]` or `#[payable("EGLD")]`?
  - without this, transfers will fail.
- [ ] **Init**: Is `#[init]` initializing all necessary storage?
- [ ] **Views**: Are read-only functions marked with `#[view]`?

## 4. Testing

- [ ] **Scenario**: Is there a `.scen.json` (Mandos) test covering the success flow?
- [ ] **Error Path**: Is there a test covering the failure/revert cases?

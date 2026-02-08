---
name: mvx_flash_loan_patterns
description: Atomic lend-execute-verify pattern — reentrancy guards, shard validation, endpoint checks.
---

# MultiversX Atomic Lend-Execute-Verify Pattern

Lend assets, execute callback, verify repayment — all atomically in one transaction.

## When to Use

- Flash loans, atomic swaps, temporary grants
- NOT for cross-shard operations (breaks atomicity)

## Security Checklist

1. Reentrancy guard — prevent nested operations
2. Shard validation — target must be same shard
3. Endpoint validation — callback must not be built-in
4. Repayment verification — check contract balance after callback
5. Guard cleanup — always clear the flag

## Core Flow

```rust
#[endpoint(atomicOperation)]
fn atomic_operation(&self, asset: TokenIdentifier, amount: BigUint, target: ManagedAddress, callback: ManagedBuffer) {
    self.require_not_ongoing();
    self.require_same_shard(&target);
    require!(!callback.is_empty() && !self.blockchain().is_builtin_function(&callback), "Invalid endpoint");

    let fee = &amount * self.fee_bps().get() / 10_000u64;
    let balance_before = self.blockchain().get_sc_balance(&asset.clone().into(), 0);

    self.operation_ongoing().set(true);
    self.tx().to(&target).raw_call(callback).single_esdt(&asset, 0, &amount).sync_call();

    let balance_after = self.blockchain().get_sc_balance(&asset.into(), 0);
    require!(balance_after >= balance_before + &fee, "Repayment insufficient");

    self.operation_ongoing().set(false);
}
```

## Reentrancy Guard

```rust
#[storage_mapper("operationOngoing")]
fn operation_ongoing(&self) -> SingleValueMapper<bool>;

fn require_not_ongoing(&self) {
    require!(!self.operation_ongoing().get(), "Operation already in progress");
}
```

## Shard Validation

```rust
fn require_same_shard(&self, target: &ManagedAddress) {
    let target_shard = self.blockchain().get_shard_of_address(target);
    let self_shard = self.blockchain().get_shard_of_address(&self.blockchain().get_sc_address());
    require!(target_shard == self_shard, "Must be same shard");
}
```

**Why**: Cross-shard calls execute in different blocks, breaking atomicity.

## Anti-Patterns

- Checking storage value instead of actual `get_sc_balance` for repayment
- No shard validation (cross-shard sync_call becomes async silently)
- Built-in function callbacks bypassing repayment logic

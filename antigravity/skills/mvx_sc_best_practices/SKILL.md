---
name: mvx_sc_best_practices
description: Expert guidelines for developing, auditing, and optimizing MultiversX Smart Contracts (Rust).
---

# MultiversX Smart Contract Best Practices

This skill provides expert-level guidance on writing secure, gas-efficient, and idiomatic Smart Contracts on MultiversX using the `multiversx-sc` framework.

## 1. Storage Optimization (Critical)
Storage is the most expensive resource.

- **`SingleValueMapper`**: Use for individual items (flags, configs, IDs).
    - *Gas*: Cheapest (~1 slot write).
    - *Pattern*: `#[storage_mapper("myValue")] fn my_value(&self) -> SingleValueMapper<MyType>;`
- **`VecMapper`**: Use for ordered lists where you need index access.
    - *Warning*: **NEVER iterate** a `VecMapper` on-chain if it can grow indefinitely. This is a DoS vector (Gas Loop).
    - *Gas*: Medium.
- **`UnorderedSetMapper`**: Use for unique collections or whitelists.
    - *Gas*: Checks existence before insert. Good for `O(1)` membership checks.
- **`MapMapper`**: **AVOID** unless strictly necessary.
    - *Why*: It uses a linked-list structure (4 storage writes per entry). It is ~4x more expensive than `SingleValueMapper`.
    - *Alternative*: If you don't need to iterate keys, use a `SingleValueMapper` keyed by a hash or composite key.

## 2. Security Patterns

### Arithmetic Safety
- **Always use `BigUint`** for tokens, prices, and financial math.
    - *Why*: Prevents overflow/underflow and matches the VM's native big int implementation.
- Avoid `u64`/`u32` for money. Only use them for loop counters or small IDs.

### Reentrancy Protection
- **Checks-Effects-Interactions**:
    1.  **Checks**: Validate inputs (`require!`).
    2.  **Effects**: Update storage (deduct balance, update state).
    3.  **Interactions**: Send tokens or call other contracts.
- **Async Calls**: MultiversX async calls are safer than synchronous calls regarding reentrancy of the *same* execution context, but state changes happen in a separate transaction (callback).
- **Callback Verification**: Always validate the state in the `#[callback]` function. Do not assume the async call succeeded just because it was sent.

### Access Control
- Use `#[only_owner]` for admin functions.
- For fine-grained control, use the `only_admin` module from the `multiversx-sc-modules` crate. It provides a standard implementation for managing multiple admins.

## 3. Data Flow & Testing

### Transfer-Execute Pattern
- When sending tokens to a contract, prefer **`MultiESDTNFTTransfer`** (built-in function) over 2 transactions (Approve + TransferFrom).
- In the contract, use `#[payable]` to accept tokens and `self.call_value().all()` to inspect them.

### Testing (Mandos/Scenarios)
- **Mandos (`.scen.json`)** are mandatory for integration testing.
- Cover all pathways:
    - Happy path.
    - Error path (expect status `4`).
- **Whitebox Testing**: Use `#[cfg(test)]` modules with `multiversx_sc_scenario::imports::*` to test internal functions without deploying.

## 4. Code Structure
- **Endpoints**: Public functions `#[endpoint]`.
- **Views**: Read-only `#[view]`.
- **Private**: Helper functions (no annotation, or pure Rust).
- **Events**: `#[event]` for indexing, but don't store critical data solely in events.

## 5. Common Pitfalls / "Sharp Edges"
- **Token Identifier Validation**: Always validate `token_id`. Don't assume the user sent the correct token.
- **Gas Limit**: Be aware of the block gas limit (1.5B gas). Large loops will revert.
- **Managed Types**: Use `ManagedBuffer`, `ManagedAddress`, `ManagedVec` instead of standard Rust `Vec`, `String` to avoid serialization overhead.

## 6. Production Patterns (Advanced)

### Cache-Based Gas Optimization
Use a `Drop`-trait cache struct to batch storage reads/writes:

```rust
pub struct StorageCache<'a, C: crate::storage::StorageModule> {
    sc_ref: &'a C,
    pub field_a: BigUint<C::Api>,
    pub field_b: BigUint<C::Api>,
}

impl<'a, C: crate::storage::StorageModule> StorageCache<'a, C> {
    pub fn new(sc_ref: &'a C) -> Self {
        StorageCache {
            field_a: sc_ref.field_a().get(),
            field_b: sc_ref.field_b().get(),
            sc_ref,
        }
    }
}

impl<C: crate::storage::StorageModule> Drop for StorageCache<'_, C> {
    fn drop(&mut self) {
        self.sc_ref.field_a().set(&self.field_a);
        self.sc_ref.field_b().set(&self.field_b);
    }
}
```

### Error Constants Organization
```rust
// errors.rs — static byte strings for gas efficiency
pub static ERROR_NOT_ACTIVE: &[u8] = b"Not active";
pub static ERROR_UNAUTHORIZED: &[u8] = b"Unauthorized";
pub static ERROR_ZERO_AMOUNT: &[u8] = b"Zero amount";
```

### Event Trait Composition
```rust
#[multiversx_sc::module]
pub trait EventsModule {
    #[event("deposit")]
    fn deposit_event(&self, #[indexed] caller: &ManagedAddress, amount: &BigUint);
}
```

### View Endpoint Separation
Keep all `#[view]` endpoints in a dedicated `views.rs` module for clarity.

### Validation Module Pattern
Centralize all `require!` checks in a `validation.rs` module so security rules are auditable in one place.

### Cross-Contract Storage Reads
Use `#[storage_mapper_from_address("key")]` to read other contracts' storage without async call overhead:

```rust
#[storage_mapper_from_address("reserve")]
fn external_reserve(&self, addr: ManagedAddress, token: &TokenIdentifier)
    -> SingleValueMapper<BigUint, ManagedAddress>;
```

Only works same-shard. Read-only. Key must match target contract exactly.

---
name: mvx_cache_patterns
description: Drop-based write-back caches for gas optimization. Use for endpoints reading 3+ storage values.
---

# MultiversX Cache Patterns

## Why Cache?
- Storage operations are the most expensive in MultiversX contracts
- With cache: reads on entry + writes on exit, intermediate reads FREE (in-memory)

## Pattern 1: Write-Back Cache with Drop Trait

Load state into struct on entry, mutate in memory, commit on scope exit via `Drop`.

```rust
pub struct StorageCache<'a, C>
where
    C: crate::storage::StorageModule,
{
    sc_ref: &'a C,
    pub field_a: BigUint<C::Api>,
    pub field_b: BigUint<C::Api>,
    pub field_c: BigUint<C::Api>,
}

impl<'a, C> StorageCache<'a, C>
where
    C: crate::storage::StorageModule,
{
    pub fn new(sc_ref: &'a C) -> Self {
        StorageCache {
            field_a: sc_ref.field_a().get(),
            field_b: sc_ref.field_b().get(),
            field_c: sc_ref.field_c().get(),
            sc_ref,
        }
    }
}

impl<C> Drop for StorageCache<'_, C>
where
    C: crate::storage::StorageModule,
{
    fn drop(&mut self) {
        self.sc_ref.field_a().set(&self.field_a);
        self.sc_ref.field_b().set(&self.field_b);
        self.sc_ref.field_c().set(&self.field_c);
    }
}
```

## Pattern 2: Cache with Computed Methods

Add derived value methods on the cache struct — uses cached fields + contract module methods without extra storage reads:

```rust
impl<C> StateCache<'_, C>
where
    C: crate::storage::StorageModule + crate::math::MathModule,
{
    pub fn exchange_rate(&self) -> BigUint<C::Api> {
        if self.total_shares == 0u64 { return BigUint::from(1u64); }
        &self.total_deposited / &self.total_shares
    }
}
```

## Selective Write-Back

Only write back mutable fields — skip config/read-only fields in Drop to save gas.

## When to Use vs Direct Storage

| Scenario | Approach |
|---|---|
| Endpoint reads 3+ storage values | Use cache |
| Single storage read/write | Direct access is fine |
| View function reading multiple values | Read-only cache (no Drop) |
| Async call boundary | Manually drop cache BEFORE async call |

## Anti-Patterns

### 1. Caching Across Async Boundaries
```rust
// WRONG - async_call_and_exit() terminates execution, drop() never runs!
fn bad_async(&self) {
    let mut cache = StorageCache::new(self);
    cache.balance += &deposit;
    self.tx().to(&other).typed(Proxy).call()
        .callback(self.callbacks().on_done())
        .async_call_and_exit();
}

// CORRECT - manually drop cache before async call
fn good_async(&self) {
    {
        let mut cache = StorageCache::new(self);
        cache.balance += &deposit;
    } // cache.drop() fires here
    self.tx().to(&other).typed(Proxy).call()
        .callback(self.callbacks().on_done())
        .async_call_and_exit();
}
```

### 2. Forgetting Fields in Drop
### 3. Writing Back Immutable Config

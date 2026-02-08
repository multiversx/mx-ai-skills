---
name: mvx_cross_contract_storage
description: Read another contract's storage directly using storage_mapper_from_address for same-shard contracts.
---

# MultiversX Cross-Contract Storage Reads

Read another contract's storage mappers directly — zero gas overhead from proxy calls, no async complexity.

## When to Use

| Criteria | `storage_mapper_from_address` | Proxy Call |
|---|---|---|
| Same shard only | Yes (required) | Works cross-shard |
| Read-only | Yes | Read + Write |
| Needs computation on target | No | Yes |
| Gas cost | ~storage read cost | ~execution + storage |
| Requires knowing storage keys | Yes | No (uses ABI) |

## Core Pattern

```rust
#[multiversx_sc::module]
pub trait ExternalStorageModule {
    // Simple value
    #[storage_mapper_from_address("total_supply")]
    fn external_total_supply(&self, contract_address: ManagedAddress) -> SingleValueMapper<BigUint, ManagedAddress>;

    // Composite key (token-specific)
    #[storage_mapper_from_address("balance")]
    fn external_balance(&self, contract_address: ManagedAddress, token_id: &TokenIdentifier) -> SingleValueMapper<BigUint, ManagedAddress>;

    // Module-prefixed key
    #[storage_mapper_from_address("pause_module:paused")]
    fn external_paused(&self, contract_address: ManagedAddress) -> SingleValueMapper<bool, ManagedAddress>;
}
```

### Key Rules
1. String must EXACTLY match storage key in target contract
2. First param must be `ManagedAddress`
3. Return type's second generic must be `ManagedAddress`
4. Additional params become composite key parts

## How to Discover Storage Keys
1. Source code: `#[storage_mapper("key")]` in target
2. ABI: `.abi.json` lists all keys
3. Module keys: prefixed like `pause_module:paused`

## Security

- **Same-shard only**: Cross-shard reads return defaults silently — no error
- **Stale data**: Reflects target's state as of current block
- **Key changes**: If target renames keys in upgrade, reads break silently
- **Read-only**: Cannot write to another contract's storage

## Anti-Patterns
- Not validating empty results (get default 0/false silently)
- Not validating target is on same shard

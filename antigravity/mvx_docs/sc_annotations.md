# MultiversX Smart Contract Annotations

## Contract Structure
- `#[multiversx_sc::contract]`: Marks a trait as the main contract.
- `#[multiversx_sc::module]`: Marks a trait as a module.
- `#[multiversx_sc::proxy]`: Marks a trait as a proxy interface.

## Methods
- `#[init]`: Constructor, called once upon deployment.
- `#[upgrade]`: Called when upgrading the contract.
- `#[endpoint]`: Public method callable via transactions.
- `#[view]`: Read-only public method (currently synonymous with endpoint but indicates intent).
- `#[payable("*")]`: Allows the endpoint to receive value (EGLD/ESDT).

## Storage
Avoid manual storage access. Use annotations:
- `#[storage_get("key")]`
- `#[storage_set("key")]`
- `#[storage_mapper("key")]` (recommended for complex types)

Example:
```rust
#[storage_get("sum")]
fn get_sum(&self) -> BigUint;

#[storage_set("sum")]
fn set_sum(&self, sum: &BigUint);
```

## Events
Define events as trait methods with no body:
```rust
#[event("transfer")]
fn transfer_event(
    &self,
    #[indexed] from: &ManagedAddress,
    #[indexed] to: &ManagedAddress,
    #[indexed] token_id: u32,
    data: ManagedBuffer,
);
```
- `#[indexed]`: Topic (searchable).
- Non-indexed: Data field.

## Proxies
```rust
#[proxy]
fn vault_proxy(&self, to: Address) -> vault::Proxy<Self::Api>;
```

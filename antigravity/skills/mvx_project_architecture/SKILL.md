---
name: mvx_project_architecture
description: Production-grade project structure patterns for MultiversX smart contracts.
---

# MultiversX Project Architecture

## Single Contract Structure

```
my-contract/
├── Cargo.toml
├── src/
│   ├── lib.rs              # #[contract] trait — ONLY trait composition + init/upgrade
│   ├── storage.rs           # All #[storage_mapper] definitions
│   ├── views.rs             # All #[view] endpoints
│   ├── config.rs            # Admin configuration endpoints
│   ├── events.rs            # #[event] definitions
│   ├── validation.rs        # Input validation helpers
│   ├── errors.rs            # Static error constants
│   └── helpers.rs           # Pure business logic functions
├── meta/
├── wasm/
└── tests/
```

## lib.rs Pattern: Trait Composition Only

```rust
#![no_std]
multiversx_sc::imports!();

#[multiversx_sc::contract]
pub trait MyContract:
    storage::StorageModule
    + config::ConfigModule
    + views::ViewsModule
    + events::EventsModule
    + validation::ValidationModule
    + helpers::HelpersModule
{
    #[init]
    fn init(&self) { }

    #[upgrade]
    fn upgrade(&self) { }

    #[endpoint]
    fn main_operation(&self) {
        self.validate_payment(&payment);
        let mut cache = cache::StorageCache::new(self);
        self.process_operation(&mut cache, &payment);
    }
}
```

## errors.rs Pattern

```rust
pub static ERROR_NOT_ACTIVE: &[u8] = b"Contract is not active";
pub static ERROR_INVALID_AMOUNT: &[u8] = b"Invalid amount";
pub static ERROR_UNAUTHORIZED: &[u8] = b"Unauthorized";
pub static ERROR_ZERO_AMOUNT: &[u8] = b"Amount must be greater than zero";
```

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Contract crate | kebab-case | `liquidity-pool` |
| Module files | snake_case | `storage.rs` |
| Storage keys | camelCase | `"totalSupply"` |
| Error constants | SCREAMING_SNAKE | `ERROR_INVALID_AMOUNT` |
| Module traits | PascalCase | `StorageModule` |
| Endpoint names | snake_case | `fn deposit_tokens(&self)` |
| View names | camelCase (ABI) | `#[view(getBalance)]` |

## When to Create Common Workspace Crate

Same struct/math/errors/events in 2+ contracts → move to `common/` crate.

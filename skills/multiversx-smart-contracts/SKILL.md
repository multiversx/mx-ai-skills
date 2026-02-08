---
name: multiversx-smart-contracts
description: Build MultiversX smart contracts with Rust. Use when app needs blockchain logic, token creation, NFT minting, staking, crowdfunding, or any on-chain functionality requiring custom smart contracts.
---

# MultiversX Smart Contract Development

Build, test, and deploy MultiversX smart contracts using Rust, sc-meta, and mxpy.

## Prerequisites

Tools available on the VM:
- **Rust** (version 1.85.0+)
- **sc-meta** - Smart contract meta tool
- **mxpy** - MultiversX Python CLI for deployment

If sc-meta is not installed:
```bash
cargo install multiversx-sc-meta --locked
```

## Creating a New Contract

Use sc-meta to scaffold a new contract from templates:

```bash
# List available templates
sc-meta templates

# Create from template
sc-meta new --template adder --name my-contract

# Available templates:
# - empty: Minimal contract structure
# - adder: Basic arithmetic operations
# - crypto-zombies: NFT game example
```

## Project Structure

After creating a contract, you get this structure:

```
my-contract/
├── Cargo.toml           # Dependencies
├── src/
│   └── lib.rs           # Contract code
├── meta/
│   ├── Cargo.toml       # Meta crate dependencies
│   └── src/
│       └── main.rs      # Build tooling entry
├── wasm/
│   ├── Cargo.toml       # WASM output config
│   └── src/
│       └── lib.rs       # WASM entry point
└── scenarios/           # Test files (optional)
```

## Cargo.toml Configuration

```toml
[package]
name = "my-contract"
version = "0.0.0"
edition = "2024"

[lib]
path = "src/lib.rs"

[dependencies.multiversx-sc]
version = "0.64.0"

[dev-dependencies.multiversx-sc-scenario]
version = "0.64.0"
```

## Basic Contract Structure

```rust
#![no_std]

multiversx_sc::imports!();

#[multiversx_sc::contract]
pub trait MyContract {
    #[init]
    fn init(&self, initial_value: BigUint) {
        self.stored_value().set(initial_value);
    }

    #[upgrade]
    fn upgrade(&self) {
        // Called when contract is upgraded
    }

    #[endpoint]
    fn add(&self, value: BigUint) {
        self.stored_value().update(|v| *v += value);
    }

    #[view(getValue)]
    fn get_value(&self) -> BigUint {
        self.stored_value().get()
    }

    #[storage_mapper("storedValue")]
    fn stored_value(&self) -> SingleValueMapper<BigUint>;
}
```

## Core Annotations

### Contract & Module Level

| Annotation | Purpose |
|------------|---------|
| `#[multiversx_sc::contract]` | Marks trait as main contract (one per crate) |
| `#[multiversx_sc::module]` | Marks trait as reusable module |
| `#[multiversx_sc::proxy]` | Creates proxy for calling other contracts |

### Method Level

| Annotation | Purpose |
|------------|---------|
| `#[init]` | Constructor, called on deploy |
| `#[upgrade]` | Called when contract is upgraded |
| `#[endpoint]` | Public callable method |
| `#[view]` | Read-only public method |
| `#[endpoint(customName)]` | Endpoint with custom ABI name |
| `#[view(customName)]` | View with custom ABI name |

### Payment Annotations

| Annotation | Purpose |
|------------|---------|
| `#[payable]` | Accepts any token payment (shorthand for `#[payable("*")]` since SDK v0.64.0) |
| `#[payable("EGLD")]` | Accepts only EGLD |
| `#[payable("TOKEN-ID")]` | Accepts specific token |

### Event Annotations

| Annotation | Purpose |
|------------|---------|
| `#[event("eventName")]` | Defines contract event |
| `#[indexed]` | Marks event field as searchable topic |

### Callback Annotations

| Annotation | Purpose |
|------------|---------|
| `#[callback]` | Callback for legacy async calls |
| `#[promises_callback]` | Callback for promise-based async calls (used with `register_promise()`) |

## Storage Mappers

### SingleValueMapper
Stores a single value.

```rust
#[storage_mapper("owner")]
fn owner(&self) -> SingleValueMapper<ManagedAddress>;

// Usage
self.owner().set(caller);
let owner = self.owner().get();
self.owner().is_empty();
self.owner().clear();
self.owner().update(|v| *v = new_value);
```

### VecMapper
Stores indexed array (1-indexed).

```rust
#[storage_mapper("items")]
fn items(&self) -> VecMapper<BigUint>;

// Usage
self.items().push(&value);
let item = self.items().get(1); // 1-indexed!
self.items().set(1, &new_value);
let len = self.items().len();
for item in self.items().iter() { }
self.items().swap_remove(1);
```

### SetMapper
Unique collection with O(1) lookup, preserves insertion order.

```rust
#[storage_mapper("whitelist")]
fn whitelist(&self) -> SetMapper<ManagedAddress>;

// Usage
self.whitelist().insert(address); // Returns false if duplicate
self.whitelist().contains(&address); // O(1)
self.whitelist().remove(&address);
for addr in self.whitelist().iter() { }
```

### UnorderedSetMapper
Like SetMapper but more efficient when order doesn't matter.

```rust
#[storage_mapper("participants")]
fn participants(&self) -> UnorderedSetMapper<ManagedAddress>;
```

### MapMapper
Key-value pairs. **Expensive - avoid when iteration not needed.**

```rust
#[storage_mapper("balances")]
fn balances(&self) -> MapMapper<ManagedAddress, BigUint>;

// Usage
self.balances().insert(address, amount);
let balance = self.balances().get(&address); // Returns Option
self.balances().contains_key(&address);
self.balances().remove(&address);
for (addr, bal) in self.balances().iter() { }
```

### LinkedListMapper
Doubly-linked list for queue operations.

```rust
#[storage_mapper("queue")]
fn queue(&self) -> LinkedListMapper<BigUint>;

// Usage
self.queue().push_back(value);
self.queue().push_front(value);
self.queue().pop_front();
self.queue().pop_back();
```

### FungibleTokenMapper
Manages fungible token with built-in ESDT operations.

```rust
#[storage_mapper("token")]
fn token(&self) -> FungibleTokenMapper;

// Usage
self.token().issue_and_set_all_roles(...);
self.token().mint(amount);
self.token().burn(amount);
self.token().get_balance();
```

### NonFungibleTokenMapper
Manages NFT/SFT/META-ESDT tokens.

```rust
#[storage_mapper("nft")]
fn nft(&self) -> NonFungibleTokenMapper;

// Usage
self.nft().nft_create(amount, &attributes);
self.nft().nft_add_quantity(nonce, amount);
self.nft().get_all_token_data(nonce);
```

### WhitelistMapper
Non-iterable O(1) membership check. Lighter than SetMapper when iteration is not needed.

```rust
#[storage_mapper("allowedTokens")]
fn allowed_tokens(&self) -> WhitelistMapper<TokenId>;

// Usage
self.allowed_tokens().add(&token_id);
self.allowed_tokens().contains(&token_id); // O(1)
self.allowed_tokens().require_whitelisted(&token_id); // panics if missing
self.allowed_tokens().remove(&token_id);
```

### BiDiMapper
Bidirectional mapping — two-way lookups between keys and values (both must be unique).

```rust
#[storage_mapper("idToAddress")]
fn id_to_address(&self) -> BiDiMapper<u64, ManagedAddress>;

// Usage
self.id_to_address().insert(1u64, address.clone());
let addr = self.id_to_address().get_value(&1u64);
let id = self.id_to_address().get_id(&address);
self.id_to_address().contains_id(&1u64);
self.id_to_address().remove_by_id(&1u64);
```

### UniqueIdMapper
Manages a pool of unique IDs for random or sequential assignment.

```rust
#[storage_mapper("nftIds")]
fn nft_ids(&self) -> UniqueIdMapper<Self::Api>;

// Usage
self.nft_ids().set_initial_len(1000); // IDs 1..=1000
let id = self.nft_ids().swap_remove(index); // pop random ID
self.nft_ids().len(); // remaining IDs
```

### OrderedBinaryTreeMapper
Self-balancing binary search tree for ordered on-chain data.

```rust
#[storage_mapper("orderBook")]
fn order_book(&self) -> OrderedBinaryTreeMapper<Self::Api, u64>;
```

### QueueMapper
FIFO queue with push-back and pop-front.

```rust
#[storage_mapper("pending")]
fn pending(&self) -> QueueMapper<ManagedAddress>;

// Usage
self.pending().push_back(address);
let next = self.pending().pop_front(); // Option
self.pending().len();
```

### AddressToIdMapper
Bidirectional mapping between addresses and auto-incrementing `u64` IDs. Gas-efficient for contracts that track many users by numeric ID.

```rust
#[storage_mapper("users")]
fn users(&self) -> AddressToIdMapper;

// Usage
let id: u64 = self.users().get_or_create_id(&caller); // auto-assigns next ID
let addr: ManagedAddress = self.users().get_address(id);
self.users().contains(&caller); // O(1) check
```

### UserMapper
Similar to AddressToIdMapper but designed specifically for user management with count tracking.

```rust
#[storage_mapper("user")]
fn user_mapper(&self) -> UserMapper;

// Usage
let user_id = self.user_mapper().get_or_create_user(&address);
let user_count = self.user_mapper().get_user_count();
let address = self.user_mapper().get_user_address(user_id);
```

### MapStorageMapper
Map where values are themselves storage mappers (nested storage). Use when each key needs its own complex storage structure.

```rust
#[storage_mapper("vaults")]
fn vaults(&self) -> MapStorageMapper<ManagedAddress, SingleValueMapper<BigUint>>;
```

### TimelockMapper
Value with a time-lock — stores a current and future value with unlock timestamp. Value transitions automatically when block time passes the unlock point.

```rust
#[storage_mapper("admin")]
fn admin(&self) -> TimelockMapper<ManagedAddress>;

// Usage — schedule a change with delay
self.admin().update_and_lock(new_admin, unlock_timestamp);
let current = self.admin().get(); // returns current until unlock, then future
```

## Data Types

### Core Types

| Type | Description |
|------|-------------|
| `BigUint` | Unsigned arbitrary-precision integer |
| `BigInt` | Signed arbitrary-precision integer |
| `ManagedBuffer` | Byte array (strings, raw data) |
| `ManagedAddress` | 32-byte address |
| `EsdtTokenIdentifier` | ESDT token ID (e.g., "TOKEN-abc123") |
| `TokenId` | Unified token identifier for both EGLD and ESDT (EGLD represented as `EGLD-000000`) |
| `Payment` | Unified payment: `TokenId` + nonce + `NonZeroBigUint` amount |
| `NonZeroBigUint` | `BigUint` guaranteed to be non-zero at construction time |
| `EgldOrEsdtTokenIdentifier` | Legacy: Either EGLD or ESDT token ID (prefer `TokenId`) |
| `EsdtTokenPayment` | Legacy: Token ID + nonce + amount (prefer `Payment`) |

### Creating Values

```rust
// BigUint
let amount = BigUint::from(1000u64);
let zero = BigUint::zero();

// ManagedBuffer (strings)
let buffer = ManagedBuffer::from("hello");

// Address
let caller = self.blockchain().get_caller();

// Token identifier
let token = EsdtTokenIdentifier::from("TOKEN-abc123");

// Unified token identifier (EGLD or ESDT)
let token_id = TokenId::from("EGLD-000000"); // EGLD
let token_id = TokenId::from("TOKEN-abc123"); // ESDT

// NonZeroBigUint
let nz_amount = NonZeroBigUint::new_or_panic(BigUint::from(1000u64));
```

## Payment Types: `Payment` vs `EgldOrEsdtTokenPayment`

| Aspect | `Payment<M>` (v0.64+) | `EgldOrEsdtTokenPayment<M>` (Legacy) |
|---|---|---|
| Token ID | `TokenId` (EGLD = `EGLD-000000`) | `EgldOrEsdtTokenIdentifier` |
| Amount | `NonZeroBigUint` (zero rejected) | `BigUint` (zero allowed) |
| Status | Preferred for new code | Widely used in production |

```rust
// New (v0.64+): Payment with TokenId + NonZeroBigUint
let payment: Payment<Self::Api> = self.call_value().single();
// payment.token_identifier is TokenId, payment.amount is NonZeroBigUint

// Legacy: EgldOrEsdtTokenPayment with BigUint
let legacy = self.call_value().egld_or_single_esdt();
// legacy.token_identifier is EgldOrEsdtTokenIdentifier, legacy.amount is BigUint
```

## Payment Handling

### Bad
```rust
// DON'T: Use BigUint for payment amounts — allows zero-value transfers
let amount: BigUint = self.call_value().egld_value().clone_value();
let payment = EsdtTokenPayment::new(token, 0, amount); // Legacy type, no zero check
```

### Good
```rust
// DO: Use NonZeroBigUint and Payment — zero is rejected at the type level
let payment = self.call_value().single(); // Returns Payment with NonZeroBigUint
// payment.amount is NonZeroBigUint — guaranteed non-zero
```

### Bad
```rust
// DON'T: Use legacy EgldOrEsdtTokenIdentifier
let token: EgldOrEsdtTokenIdentifier = self.call_value().egld_or_single_esdt().token_identifier;
```

### Good
```rust
// DO: Use unified TokenId — handles both EGLD and ESDT uniformly
let token: TokenId = self.call_value().single().token_identifier;
// EGLD is represented as "EGLD-000000"
```

### Bad
```rust
// DON'T: Use legacy send() API for cross-contract calls
self.send().direct_esdt(&recipient, &token, 0, &amount);
self.send_raw().direct_egld_execute(&to, &amount, 0, &ManagedBuffer::new());
```

### Good
```rust
// DO: Use the Tx builder API for all transfers and cross-contract calls
self.tx().to(&recipient).payment(payment).transfer();
self.tx().to(&addr).typed(proxy::Proxy).some_endpoint(arg).sync_call();
```

### Receiving EGLD

```rust
#[payable("EGLD")]
#[endpoint]
fn deposit_egld(&self) {
    let payment = self.call_value().egld();
    // payment is a BigUint (EGLD amount)
}
```

### Receiving Any Single Payment (EGLD or ESDT, unified)

```rust
#[payable]
#[endpoint]
fn deposit(&self) {
    let payment = self.call_value().single();
    // payment.token_identifier : TokenId
    // payment.token_nonce : u64
    // payment.amount : NonZeroBigUint (guaranteed non-zero)
}
```

### Receiving Multiple Payments

```rust
#[payable]
#[endpoint]
fn multi_deposit(&self) {
    let payments = self.call_value().all();
    // Returns ManagedRef<PaymentVec> — unified EGLD + ESDT payments
    for payment in payments.iter() {
        // payment is a Payment
    }
}
```

### Receiving Exact N Payments

```rust
#[payable]
#[endpoint]
fn dual_deposit(&self) {
    let [token_a, token_b] = self.call_value().array();
    // Exactly 2 payments, crashes otherwise
}
```

### Optional Single Payment

```rust
#[payable]
#[endpoint]
fn optional_deposit(&self) {
    let maybe_payment = self.call_value().single_optional();
    // Returns Option<Ref<Payment>> — zero or one payment
}
```

### Sending Tokens

```rust
// Send EGLD
self.tx()
    .to(&recipient)
    .egld(&amount)
    .transfer();

// Send a Payment (requires NonZeroBigUint)
if let Some(amount_nz) = amount.into_non_zero() {
    self.tx()
        .to(&recipient)
        .payment(Payment::new(token_id, nonce, amount_nz))
        .transfer();
}

// Send multiple payments
self.tx()
    .to(&recipient)
    .payment(&payments)
    .transfer();

// Transfer only if non-empty
self.tx()
    .to(&recipient)
    .egld(&amount)
    .transfer_if_not_empty();
```

**Note:** Since SDK v0.55.0, EGLD and ESDT can be sent together in the same multi-transfer transaction.

## Events

```rust
#[event("deposit")]
fn deposit_event(
    &self,
    #[indexed] caller: &ManagedAddress,
    #[indexed] token: &TokenId,
    amount: &BigUint,
);

// Emit event
self.deposit_event(&caller, &token_id, &amount);
```

## Modules

Split large contracts into modules:

```rust
// In src/storage.rs
#[multiversx_sc::module]
pub trait StorageModule {
    #[storage_mapper("owner")]
    fn owner(&self) -> SingleValueMapper<ManagedAddress>;
}

// In src/lib.rs
mod storage;

#[multiversx_sc::contract]
pub trait MyContract: storage::StorageModule {
    #[init]
    fn init(&self) {
        self.owner().set(self.blockchain().get_caller());
    }
}
```

## Error Handling

```rust
// Using require!
#[endpoint]
fn withdraw(&self, amount: BigUint) {
    let caller = self.blockchain().get_caller();
    require!(
        caller == self.owner().get(),
        "Only owner can withdraw"
    );
    require!(amount > 0, "Amount must be positive");
}

// Using sc_panic!
if condition_failed {
    sc_panic!("Operation failed");
}
```

## Building Contracts

### Build All Contracts in Workspace

```bash
sc-meta all build
```

### Build Single Contract

```bash
cd my-contract/meta
cargo run build
```

### Build with Options

```bash
# Build with locked dependencies
sc-meta all build --locked

# Debug build with WAT output
cd meta && cargo run build-dbg
```

### Build Output

After building, find outputs in `output/`:
- `my-contract.wasm` - Contract bytecode
- `my-contract.abi.json` - Contract ABI

## Testing

### Rust-Based Tests

Create tests in `tests/` folder:

```rust
use multiversx_sc_scenario::*;

fn world() -> ScenarioWorld {
    let mut blockchain = ScenarioWorld::new();
    blockchain.register_contract(
        "mxsc:output/my-contract.mxsc.json",
        my_contract::ContractBuilder,
    );
    blockchain
}

#[test]
fn test_deploy() {
    let mut world = world();
    world.run("scenarios/deploy.scen.json");
}
```

### Running Tests

```bash
# Run all tests
sc-meta test

# Run specific test
cargo test test_deploy
```

## Deploying with mxpy

### Deploy New Contract

```bash
mxpy contract deploy \
    --bytecode output/my-contract.wasm \
    --proxy https://devnet-api.multiversx.com \
    --chain D \
    --pem wallet.pem \
    --gas-limit 60000000 \
    --arguments 1000 \
    --send
```

### Upgrade Existing Contract

```bash
mxpy contract upgrade <contract-address> \
    --bytecode output/my-contract.wasm \
    --proxy https://devnet-api.multiversx.com \
    --chain D \
    --pem wallet.pem \
    --gas-limit 60000000 \
    --send
```

### Call Contract Endpoint

```bash
mxpy contract call <contract-address> \
    --proxy https://devnet-api.multiversx.com \
    --chain D \
    --pem wallet.pem \
    --gas-limit 5000000 \
    --function "add" \
    --arguments 100 \
    --send
```

### Query View Function

```bash
mxpy contract query <contract-address> \
    --proxy https://devnet-api.multiversx.com \
    --function "getValue"
```

### Network Endpoints

| Network | Proxy URL | Chain ID |
|---------|-----------|----------|
| Devnet | `https://devnet-api.multiversx.com` | D |
| Testnet | `https://testnet-api.multiversx.com` | T |
| Mainnet | `https://api.multiversx.com` | 1 |

## Advanced Patterns

### Cross-Contract Calls with Proxy

```rust
// Define proxy trait
#[multiversx_sc::proxy]
pub trait OtherContract {
    #[endpoint]
    fn some_endpoint(&self, value: BigUint);
}

// Use proxy
#[endpoint]
fn call_other(&self, other_address: ManagedAddress, value: BigUint) {
    self.tx()
        .to(&other_address)
        .typed(other_contract_proxy::OtherContractProxy)
        .some_endpoint(value)
        .sync_call();
}
```

### Retrieving Back-Transfers from Sync Calls

When a called contract sends tokens back (e.g., DEX swap returns), capture them with `ReturnsBackTransfers`:

```rust
#[endpoint]
fn swap_and_store(&self, dex: ManagedAddress, token_in: TokenId, amount: NonZeroBigUint) {
    let back_transfers = self.tx()
        .to(&dex)
        .typed(dex_proxy::DexProxy)
        .swap(token_in, amount)
        .returns(ReturnsBackTransfersReset) // Reset avoids stale accumulation
        .sync_call();

    let payments = back_transfers.into_payment_vec();
    for payment in &payments {
        self.received_tokens(&payment.token_identifier)
            .update(|bal| *bal += payment.amount.as_big_uint());
    }
}
```

**Key rules:**
- Use `ReturnsBackTransfersReset` (not `ReturnsBackTransfers`) when making multiple sync calls in one endpoint — back-transfers accumulate otherwise.
- Use `ReturnsBackTransfersEGLD` when you only need the EGLD amount.
- Use `ReturnsBackTransfersSingleESDT` when expecting exactly one ESDT token back.
- Call `.into_payment_vec()` to get `PaymentVec` with v0.64 `Payment` items (`TokenId` + `NonZeroBigUint`).

### Async Calls with Callbacks

```rust
#[endpoint]
fn async_call(&self, other_address: ManagedAddress) {
    self.tx()
        .to(&other_address)
        .typed(other_contract_proxy::OtherContractProxy)
        .some_endpoint()
        .callback(self.callbacks().my_callback())
        .async_call_and_exit();
}

#[callback]
fn my_callback(&self, #[call_result] result: ManagedAsyncCallResult<BigUint>) {
    match result {
        ManagedAsyncCallResult::Ok(value) => {
            // Handle success
        }
        ManagedAsyncCallResult::Err(err) => {
            // Handle error
        }
    }
}
```

### Promises with BackTransfers

For cross-shard calls that return tokens, use promises + callback:

```rust
#[endpoint]
fn cross_shard_swap(&self, dex: ManagedAddress, token: EgldOrEsdtTokenIdentifier, nonce: u64, amount: BigUint) {
    let gas_limit = self.blockchain().get_gas_left() - 20_000_000;
    self.tx()
        .to(&dex)
        .typed(dex_proxy::DexProxy)
        .swap(token, nonce, amount)
        .gas(gas_limit)
        .callback(self.callbacks().swap_callback())
        .gas_for_callback(10_000_000)
        .register_promise();
}

#[promises_callback]
fn swap_callback(&self) {
    let back_transfers = self.blockchain().get_back_transfers();
    let payments = back_transfers.into_payment_vec();
    for payment in &payments {
        self.received_tokens(&payment.token_identifier)
            .update(|bal| *bal += payment.amount.as_big_uint());
    }
}
```

### Token Issuance (Modular Approach)

The recommended way to handle token issuance is by importing and inheriting the `EsdtModule` from the framework (`multiversx-sc-modules`). This module provides a unified `issue_token` method that can be used to issue any type of token on MultiversX (Fungible, NonFungible, SemiFungible, Meta, Dynamic).

```rust
#[multiversx_sc::contract]
pub trait MyContract: multiversx_sc_modules::esdt::EsdtModule {

    // Note: Only Fungible and Meta tokens have decimals
    // Example: Issuing a Fungible Token
    #[payable("EGLD")]
    #[endpoint(issueFungible)]
    fn issue_fungible(
        &self,
        token_display_name: ManagedBuffer,
        token_ticker: ManagedBuffer,
        num_decimals: usize,
    ) {
        // Calls the inherited issue_token method from EsdtModule
        self.issue_token(
            token_display_name,
            token_ticker,
            EsdtTokenType::Fungible,
            OptionalValue::Some(num_decimals),
        );
    }

    // Example: Issuing an NFT
    #[payable("EGLD")]
    #[endpoint(issueNft)]
    fn issue_nft(
        &self,
        token_display_name: ManagedBuffer,
        token_ticker: ManagedBuffer,
    ) {
        self.issue_token(
            token_display_name,
            token_ticker,
            EsdtTokenType::NonFungible,
            OptionalValue::None,
        );
    }
}
```

### Token Minting

Similarly, the `EsdtModule` provides a `mint` method to create new units of a token that has already been issued by the contract.

```rust
#[multiversx_sc::contract]
pub trait MyContract: multiversx_sc_modules::esdt::EsdtModule {

    #[endpoint(mintTokens)]
    fn mint_tokens(&self, amount: BigUint) {
        // Mints tokens using the inherited mint method from EsdtModule.
        // The token_id is managed by the module's storage.
        // For fungible tokens, the nonce is 0.
        self.mint(0, &amount);
    }
}
```

## Code Examples

### Crowdfunding Contract Pattern

```rust
#![no_std]
use multiversx_sc::{chain_core::types::TimestampSeconds, imports::*};

#[multiversx_sc::contract]
pub trait Crowdfunding {
    #[init]
    fn init(&self, target: BigUint, deadline: TimestampSeconds, token_id: TokenId) {
        self.target().set(target);
        self.deadline().set(deadline);
        require!(token_id.is_valid(), "Invalid token provided");
        self.cf_token_identifier().set(token_id);
    }

    #[payable]
    #[endpoint]
    fn fund(&self) {
        require!(self.blockchain().get_block_timestamp_millis() < self.deadline().get(), "Funding period ended");
        let payment = self.call_value().single();
        require!(payment.token_identifier == self.cf_token_identifier().get(), "Wrong token");
        self.deposit(&self.blockchain().get_caller())
            .update(|deposit| *deposit += payment.amount.as_big_uint());
    }

    #[endpoint]
    fn claim(&self) {
        require!(self.blockchain().get_block_timestamp_millis() >= self.deadline().get(), "Funding period not ended");
        let caller = self.blockchain().get_caller();
        let token_id = self.cf_token_identifier().get();

        if self.get_current_funds() >= self.target().get() {
            require!(caller == self.blockchain().get_owner_address(), "Not owner");
            if let Some(bal) = self.get_current_funds().into_non_zero() {
                self.tx().to(&caller).payment(Payment::new(token_id, 0, bal)).transfer();
            }
        } else {
            let deposit = self.deposit(&caller).get();
            require!(deposit > 0, "No deposit");
            self.deposit(&caller).clear();
            if let Some(dep) = deposit.into_non_zero() {
                self.tx().to(&caller).payment(Payment::new(token_id, 0, dep)).transfer();
            }
        }
    }

    #[view(getCurrentFunds)]
    fn get_current_funds(&self) -> BigUint {
        self.blockchain().get_sc_balance(&self.cf_token_identifier().get(), 0)
    }

    #[storage_mapper("target")]
    fn target(&self) -> SingleValueMapper<BigUint>;
    #[storage_mapper("deadline")]
    fn deadline(&self) -> SingleValueMapper<TimestampSeconds>;
    #[storage_mapper("tokenIdentifier")]
    fn cf_token_identifier(&self) -> SingleValueMapper<TokenId>;
    #[storage_mapper("deposit")]
    fn deposit(&self, donor: &ManagedAddress) -> SingleValueMapper<BigUint>;
}
```

## Critical Knowledge

### WRONG: Using MapMapper when not iterating

```rust
// WRONG - MapMapper is expensive (4*N + 1 storage entries)
#[storage_mapper("balances")]
fn balances(&self) -> MapMapper<ManagedAddress, BigUint>;
```

### CORRECT: Use SingleValueMapper with address key

```rust
// CORRECT - Efficient when you don't need to iterate
#[storage_mapper("balance")]
fn balance(&self, user: &ManagedAddress) -> SingleValueMapper<BigUint>;
```

### WRONG: Large modules and functions

```rust
// WRONG - Everything in one file
#[multiversx_sc::contract]
pub trait MyContract {
    // 500+ lines of code...
}
```

### CORRECT: Split into modules

```rust
// CORRECT - Organized modules
mod storage;
mod logic;
mod events;

#[multiversx_sc::contract]
pub trait MyContract:
    storage::StorageModule +
    logic::LogicModule +
    events::EventsModule
{
    #[init]
    fn init(&self) { }
}
```

### WRONG: Duplicated error messages

```rust
// WRONG - Duplicated strings increase contract size
require!(amount > 0, "Amount must be positive");
require!(other_amount > 0, "Amount must be positive");
```

### CORRECT: Static error messages

```rust
// CORRECT - Single definition
const ERR_AMOUNT_POSITIVE: &str = "Amount must be positive";

require!(amount > 0, ERR_AMOUNT_POSITIVE);
require!(other_amount > 0, ERR_AMOUNT_POSITIVE);
```

### Sending EGLD + ESDT Together (since v0.55.0)

```rust
// Supported: EGLD and ESDT can be combined in multi-transfers
// Use Payment with TokenId for unified handling
let mut payments = ManagedVec::new();
if let Some(egld_nz) = egld_amount.into_non_zero() {
    payments.push(Payment::new(TokenId::from("EGLD-000000"), 0, egld_nz));
}
if let Some(esdt_nz) = esdt_amount.into_non_zero() {
    payments.push(Payment::new(TokenId::from(token_id), 0, esdt_nz));
}
self.tx().to(&recipient).payment(&payments).transfer();
```

## Production Patterns

For production-grade contracts, these additional skills cover advanced patterns:

### Gas-Optimized Storage Caching
Use **Drop-based caches** to batch storage reads on entry and writes on exit. See the `multiversx-cache-patterns` skill.

```rust
// Load all state once, mutate in memory, commit on scope exit
let mut cache = StorageCache::new(self);
cache.total_supply += &amount;
cache.fee_reserve += &fee;
// Drop automatically writes both values back
```

### Cross-Contract Storage Reads
Read another contract's storage directly without async calls using `#[storage_mapper_from_address]`. See the `multiversx-cross-contract-storage` skill.

```rust
#[storage_mapper_from_address("reserve")]
fn external_reserve(
    &self,
    contract_address: ManagedAddress,
    token_id: &TokenIdentifier,
) -> SingleValueMapper<BigUint, ManagedAddress>;
```

### Production Project Structure
For multi-module contracts, follow the modular architecture pattern. See the `multiversx-project-architecture` skill.

```
src/
  lib.rs          # Trait composition only
  storage.rs      # All storage mappers
  cache/mod.rs    # Drop-based caches
  views.rs        # #[view] endpoints
  config.rs       # Admin config endpoints
  events.rs       # Event definitions
  validation.rs   # Input validation
  errors.rs       # Static error constants
  helpers.rs      # Business logic
```

### DeFi Financial Math
For lending, staking, or DEX contracts, use half-up rounding and standardized precision levels. See the `multiversx-defi-math` skill.

### Additional Specialized Skills
- `multiversx-flash-loan-patterns` — Flash loan implementation with security guards
- `multiversx-factory-manager` — Deploy and manage child contracts
- `multiversx-vault-pattern` — In-memory token tracking for multi-step operations

## Documentation Links

Always consult official documentation:

- **Smart Contracts Overview**: https://docs.multiversx.com/developers/smart-contracts
- **sc-meta Tool**: https://docs.multiversx.com/developers/meta/sc-meta
- **Storage Mappers**: https://docs.multiversx.com/developers/developer-reference/storage-mappers
- **Annotations**: https://docs.multiversx.com/developers/developer-reference/sc-annotations
- **Payments**: https://docs.multiversx.com/developers/developer-reference/sc-payments
- **Example Contracts**: https://github.com/multiversx/mx-sdk-rs/tree/master/contracts/examples
- **Framework Repository**: https://github.com/multiversx/mx-sdk-rs

## Verification Checklist

Before completion, verify:

- [ ] Contract created with `sc-meta new --template <template> --name <name>`
- [ ] `#![no_std]` at top of lib.rs
- [ ] `#[multiversx_sc::contract]` on main trait
- [ ] `#[init]` function defined for deployment
- [ ] Storage mappers use appropriate types (avoid MapMapper unless iterating)
- [ ] Payment endpoints have `#[payable(...)]` annotation
- [ ] Error messages use `require!` macro
- [ ] Contract builds successfully with `sc-meta all build`
- [ ] Output files exist: `output/<name>.wasm` and `output/<name>.abi.json`
- [ ] Tests pass with `sc-meta test` (if tests exist)
- [ ] Deployment tested on devnet before mainnet

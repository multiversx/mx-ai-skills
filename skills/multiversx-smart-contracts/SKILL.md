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
version = "0.65.0"

[dev-dependencies.multiversx-sc-scenario]
version = "0.65.0"
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

## Testing (Blackbox Tests)

Blackbox tests are the standard way to test MultiversX smart contracts. They run a full VM simulation using `ScenarioWorld` and verify contract behavior through typed proxy calls.

### Running Tests

```bash
# Run all tests
sc-meta test

# Run specific test
cargo test test_deploy
```

### Essential Imports

```rust
use my_contract::my_contract_proxy;      // Contract-specific proxy
use multiversx_sc_scenario::imports::*;  // Core testing framework
```

For contracts that need BLS or other crypto in tests:
```rust
use multiversx_sc_scenario::{
    imports::*,
    multiversx_chain_vm::crypto_functions_bls::verify_bls_aggregated_signature,
    scenario_model::TxResponseStatus,
};
```

### Generating the Contract Proxy

The typed proxy used in `.typed(...)` calls is auto-generated from the contract ABI. To enable generation, add an `sc-config.toml` at the contract root:

```toml
[[proxy]]
path = "src/my_contract_proxy.rs"
```

Then run:

```bash
sc-meta all proxy
```

This reads the ABI and writes the proxy file to the declared path. The file must be committed and kept in sync with the ABI.

> **Naming convention:** the proxy file name is derived from the crate name with hyphens replaced by underscores, e.g., `order-book-pair` → `src/order_book_pair_proxy.rs`.

Rebuild the proxy whenever the contract ABI changes (new endpoints, modified argument types, or changed return types).

### Top-Level Constants

Declare **all** addresses, token IDs, code paths, and binary constants at the top of the test file.

```rust
// Addresses
const OWNER_ADDRESS: TestAddress = TestAddress::new("owner");
const USER1_ADDRESS: TestAddress = TestAddress::new("acc1");
const USER2_ADDRESS: TestAddress = TestAddress::new("acc2");
const SC_ADDRESS: TestSCAddress = TestSCAddress::new("the-contract");

// Code path
const CODE_PATH: MxscPath = MxscPath::new("output/my-contract.mxsc.json");

// Token IDs – always use TestTokenId
const MY_TOKEN: TestTokenId = TestTokenId::new("MYTOKEN-123456");
const NFT_ID: TestTokenId = TestTokenId::new("NFT-123456");

// Duration/fee constants – declare with the typed wrapper directly, not as bare u64
const COOLDOWN_TIME: DurationMillis = DurationMillis::new(86_400_000);
const LEVEL_UP_FEE: u64 = 1_000_000_000_000_000;

// Binary constants – declare as typed arrays, never inline in tx calls
const DEPOSIT_KEY_01: [u8; 32] =
    hex!("d0474a3a065d3f0c0a62ae680ef6435e48eb482899d2ae30ff7a3a4b0ef19c60");

// ED25519 signatures are exactly 64 bytes
const SIGNATURE_01: [u8; 64] = hex!(
    "443c75ceadb9ec42acff7e1b92e0305182279446c1d6c0502959484c147a0430\
     d3f96f0b988e646f6736d5bf8e4a843d8ba7730d6fa7e60f0ef3edd225ce630f"
);

// Other byte-slice constants
const ACCEPT_FUNDS_FUNC_NAME: &[u8] = b"accept_funds";
const FOURTH_ATTRIBUTES: &[u8] = b"SomeAttributeBytes";
const FOURTH_URIS: &[&[u8]] = &[b"FirstUri", b"SecondUri"];
```

### World Setup

`world()` is responsible for **execution setup only**: registering the VM executor and the contract implementations. Account state is set up separately (in each test or in a dedicated helper).

```rust
fn world() -> ScenarioWorld {
    let mut blockchain = ScenarioWorld::new();
    blockchain.set_current_dir_from_workspace("contracts/examples/my-contract");
    blockchain.register_contract(CODE_PATH, my_contract::ContractBuilder);
    blockchain
}
```

For tests involving multiple contracts, register each one:
```rust
blockchain.register_contract(VAULT_PATH, vault::ContractBuilder);
blockchain.register_contract(FORWARDER_PATH, forwarder::ContractBuilder);
```

> **Unregistered contract binaries:** A contract can also call into another contract whose source is not registered and has no Rust `ContractBuilder`. In that case, the framework executes the compiled `.wasm` or `.mxsc.json` binary directly. This is useful for integration tests against already-deployed third-party contracts.
>
> Example from the multisig blackbox test:
> ```rust
> // In world() – only multisig is registered
> fn world() -> ScenarioWorld {
>     let mut blockchain = ScenarioWorld::new().executor_config(ExecutorConfig::full_suite());
>     blockchain.set_current_dir_from_workspace("contracts/examples/multisig");
>     blockchain.register_contract(MULTISIG_CODE_PATH, multisig::ContractBuilder);
>     blockchain
> }
> 
> // Deploy adder without registering it – uses binary execution
> fn deploy_adder_contract(&mut self) {
>     self.world
>         .tx()
>         .from(ADDER_OWNER_ADDRESS)
>         .typed(adder_proxy::AdderProxy)
>         .init(5u64)
>         .code(ADDER_CODE_PATH)  // "test-contracts/adder.mxsc.json"
>         .new_address(ADDER_ADDRESS)
>         .run();
> }
> ```
>
> **Required:** Add to your `Cargo.toml`:
> ```toml
> [dev-dependencies.multiversx-sc-scenario]
> features = ["wasmer-experimental"]
> ```

> **`ExecutorConfig::full_suite()`:** This mode runs every SC call through the real WASM executor instead of the Rust interpreter. It is required when using unregistered contract binaries, and is also useful for benchmarking or cross-compilation verification. Most projects do not need it.

### Setting Up Accounts

Account state lives in each test or a shared `init_accounts` helper – **not** inside `world()`.

**In a shared `init_accounts` helper (handwritten style):**
```rust
fn init_accounts(world: &mut ScenarioWorld) {
    world
        .account(OWNER_ADDRESS)
        .nonce(1)
        .balance(100_000);
    world
        .account(USER1_ADDRESS)
        .nonce(0)
        .balance(1_000_000)
        .esdt_balance(MY_TOKEN, 500);
}
```

**In a test-state struct `new()` (for complex contracts with shared mutable helpers):**
```rust
impl MyTestState {
    fn new() -> Self {
        let mut world = world();
        world.new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
        init_accounts(&mut world);
        Self { world }
    }
}
```

Additional account properties:

```rust
world.account(OWNER_ADDRESS)
    .nonce(1)
    .balance(100)
    .esdt_balance(TOKEN_ID, 500)
    .esdt_nft_balance(NFT_ID, 2, 1, ())           // (token, nonce, amount, attributes – () means empty)
    .esdt_nft_last_nonce(NFT_ID, 5)
    .esdt_roles(NFT_TOKEN_ID, vec!["ESDTRoleNFTCreate".to_string(), "ESDTRoleNFTUpdateAttributes".to_string()])
    .esdt_nft_all_properties(NFT_ID, 2, 1, managed_buffer!(FOURTH_ATTRIBUTES), 1000, None::<Address>, (), uris_vec)
    .owner(OWNER_ADDRESS)
    .code(CODE_PATH)
    .storage_mandos("str:key", "value");
```

### Contract Deployment in Tests

**Standard Deployment:**
```rust
world
    .tx()
    .id("deploy")
    .from(OWNER_ADDRESS)
    .typed(my_contract_proxy::MyContractProxy)
    .init(constructor_arg1, constructor_arg2)
    .code(CODE_PATH)
    .new_address(SC_ADDRESS)   // pre-declares where the SC will land
    .run();
```

`.new_address()` on the account builder is an alternative way to pre-reserve the address:
```rust
world.account(OWNER_ADDRESS).nonce(1).new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
```

Also works inline on the tx:
```rust
world.new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
```

**Capturing the Deployed Address:**
```rust
let new_address = world
    .tx()
    .from(OWNER_ADDRESS)
    .typed(my_contract_proxy::MyContractProxy)
    .init(5u32)
    .code(CODE_PATH)
    .new_address(SC_ADDRESS)
    .returns(ReturnsNewAddress)
    .run();
assert_eq!(new_address, SC_ADDRESS);
```

**Deploy in a State Method (Chainable):**
```rust
impl MyTestState {
    fn deploy(&mut self) -> &mut Self {
        self.world
            .tx()
            .from(OWNER_ADDRESS)
            .typed(my_proxy::MyProxy)
            .init(param1)
            .code(CODE_PATH)
            .run();
        self
    }
}
```

### Transactions and Queries

**General Transaction Shape:**
```rust
world
    .tx()
    .id("descriptive-tx-id")   // Optional but strongly recommended
    .from(SENDER_ADDRESS)
    .to(SC_ADDRESS)
    .typed(my_contract_proxy::MyContractProxy)
    .my_endpoint(arg1, arg2)
    // payment (see below)
    // returns (see below)
    .run();
```

**Queries (Read-Only Calls):**
```rust
let value = world
    .query()
    .id("my-query")
    .to(SC_ADDRESS)
    .typed(my_contract_proxy::MyContractProxy)
    .my_view_endpoint()
    .returns(ReturnsResultUnmanaged)
    .run();
```

### Payment Types in Tests

Always use `.payment(...)`. All other payment methods (`.egld()`, `.esdt()`, `.multi_esdt()`, `.egld_or_single_esdt()`, etc.) are legacy and should not be used in new tests.

| Scenario | Code |
|---|---|
| Single payment (EGLD or ESDT) | `.payment((TOKEN_ID, nonce, amount))` |
| Single payment via `Payment` | `.payment(Payment::try_new(TOKEN_ID, nonce, amount).unwrap())` |
| Multiple payments (chained) | `.payment(...).payment(...)` |
| `NonZeroU64` amount | `.payment((TOKEN_ID, 0, NonZeroU64::new(100).unwrap()))` |

`TestTokenId::EGLD_000000` is the canonical EGLD token identifier in tests.

Examples:

```rust
// EGLD
.payment((TestTokenId::EGLD_000000, 0, 1_000u64))

// Single ESDT
.payment((MY_TOKEN, 0, 50u64))

// Mixed EGLD + multiple ESDTs
.payment((TestTokenId::EGLD_000000, 0, 1_000u64))
.payment((TOKEN1, 0, 50u64))
.payment((TOKEN2, 0, 50u64))
```

### Return Value Handling

| What you want | How |
|---|---|
| Ignore result (success only) | omit `.returns()` |
| Assert exact value | `.returns(ExpectValue(expected))` |
| Get the value (managed types) | `.returns(ReturnsResult)` |
| Get value (unmanaged/Rust native) | `.returns(ReturnsResultUnmanaged)` |
| Get as specific type | `.returns(ReturnsResultAs::<MyType>::new())` |
| Get new SC address | `.returns(ReturnsNewAddress)` |
| Get tx hash | `.returns(ReturnsTxHash)` |
| Multiple returns | `.returns(A).returns(B)` → returned as tuple |
| Handle success or error gracefully | `.returns(ReturnsHandledOrError::new().returns(...))` |

Endpoint arguments and expected return values accept any Rust type that implements the right codec trait. Prefer primitive types over managed ones where possible – e.g., use `42u64` or `"hello"` instead of `BigUint::from(42u64)` or `ManagedBuffer::from("hello")` when passing arguments or asserting results. `ManagedBuffer::from(s)` accepts a `&str` directly – there is no need for `.as_bytes()` or a `b"..."` byte-string literal.

**Capturing Multiple Return Values:**
```rust
let (new_address, tx_hash) = world
    .tx()
    .from(OWNER_ADDRESS)
    .typed(my_proxy::MyProxy)
    .init(5u32)
    .code(CODE_PATH)
    .new_address(SC_ADDRESS)
    .tx_hash([11u8; 32])
    .returns(ReturnsNewAddress)
    .returns(ReturnsTxHash)
    .run();

assert_eq!(new_address, SC_ADDRESS);
assert_eq!(tx_hash.as_array(), &[11u8; 32]);
```

For `MultiValue` returns:
```rust
let (a, b) = world
    .tx()
    .from(OWNER_ADDRESS).to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .multi_return(1u32)
    .returns(ReturnsResultUnmanaged)
    .run()
    .into_tuple();
```

### Error Expectations

Two styles – both are valid. Prefer `.with_result()` in generated tests, `.returns()` in handwritten:

```rust
// Style 1 – generated tests
.with_result(ExpectError(4, "exact error message"))
.with_result(ExpectStatus(4))
.with_result(ExpectMessage("error text"))

// Style 2 – handwritten tests
.returns(ExpectError(4, "exact error message"))
```

`4` is the standard `ReturnCode::UserError`.

**Handling Errors Programmatically:**
```rust
let result = world
    .tx()
    .from(OWNER_ADDRESS).to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .sc_panic()
    .returns(ReturnsHandledOrError::new())
    .run();

assert_eq!(
    result,
    Err(TxResponseStatus::new(ReturnCode::UserError, "sc_panic! example"))
);

// On success
let result = world
    .tx()
    .from(OWNER_ADDRESS).to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .add(1u32)
    .returns(ReturnsHandledOrError::new())
    .run();
assert_eq!(result, Ok(()));
```

Also works with queries:
```rust
let result = world
    .query()
    .to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .sum()
    .returns(ReturnsHandledOrError::new().returns(ReturnsResultUnmanaged))
    .run();
assert_eq!(result, Ok(RustBigUint::from(5u32)));
```

### Block State Manipulation

```rust
// Set timestamp in milliseconds (preferred for most contracts)
world.current_block().block_timestamp_millis(TimestampMillis::new(86_400_000u64));

// Set timestamp in seconds – must use TimestampSeconds, not a bare u64
world.current_block().block_timestamp_seconds(TimestampSeconds::new(100u64));

// Other block properties
world.current_block()
    .block_nonce(10)
    .block_round(10)
    .block_epoch(1);
```

Time types:
```rust
TimestampMillis::new(86_400_000u64)  // 24 hours in ms
TimestampMillis::zero()
TimestampSeconds::new(100u64)
DurationMillis::new(6000)            // 6 seconds
```

### State Verification

**Prefer Queries over Storage Checks.** Whenever possible, verify state by querying view endpoints rather than checking raw storage. Queries are not tied to the internal storage layout of the contract.

```rust
// ✅ Preferred – storage-layout independent
let sum = world
    .query()
    .to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .sum()
    .returns(ReturnsResultUnmanaged)
    .run();
assert_eq!(sum, 6u32);

// Also fine for generated/scenario-mirroring tests
world
    .query()
    .to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .get_deposit(&DEPOSIT_KEY_01)
    .returns(ExpectValue(expected_deposit))
    .run();
```

**Account Balance Checks:**
```rust
world
    .check_account(USER1_ADDRESS)
    .nonce(3)
    .balance(expected_egld_balance)
    .esdt_balance(TOKEN_ID, expected_token_balance)
    .esdt_nft_balance_and_attributes(NFT_ID, nonce, amount, "expected_attributes");
```

Chain multiple accounts:
```rust
world
    .check_account(OWNER_ADDRESS).nonce(3).balance(100)
    .check_account(SC_ADDRESS).check_storage("str:sum", "6");
```

**Contract Storage Checks:**

Use `check_storage` when you need to pin the exact serialised storage layout (e.g., in generated tests that mirror a `.scen.json` scenario). Storage value encoding mirrors the scenario JSON format:

```rust
world.check_account(SC_ADDRESS)
    // Simple values
    .check_storage("str:feesDisabled", "false")
    .check_storage("str:sum", "6")
    // BigUint stored as map value
    .check_storage("str:baseFee|nested:str:EGLD-000000", "10")
    // Nested struct (deposit entry)
    .check_storage(
        "str:deposit|0x487bd4010b50c24a02018345fe5171edf4182e6294325382c75ef4c4409f01bd",
        "address:acc2|u32:1|nested:str:CASHTOKEN-123456|u64:0|biguint:50|u64:86400_000|0x01|nested:str:EGLD-000000|biguint:1,000"
    )
    // Map of fees
    .check_storage("str:collectedFees", "nested:str:EGLD-000000|biguint:10")
    // String value
    .check_storage("str:otherMapper", "str:SomeValueInStorage");
```

`check_storage` is a **partial check** – only listed keys are verified, others are ignored.

### Test Organization Patterns

**Pattern A – Test State Struct (Recommended for complex contracts):**

```rust
struct MyTestState {
    world: ScenarioWorld,
    oracles: Vec<TestAddress>,
}

impl MyTestState {
    fn new() -> Self {
        const ORACLE_1: TestAddress = TestAddress::new("oracle-1");
        const ORACLE_2: TestAddress = TestAddress::new("oracle-2");

        let mut world = world();
        world.account(OWNER_ADDRESS).nonce(1);
        world.account(ORACLE_1).nonce(1);
        world.account(ORACLE_2).nonce(1);
        world.current_block().block_timestamp_seconds(TimestampSeconds::new(100));
        world.new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
        Self { world, oracles: vec![ORACLE_1, ORACLE_2] }
    }

    fn deploy(&mut self) -> &mut Self {
        self.world.tx()
            .from(OWNER_ADDRESS)
            .typed(my_proxy::MyProxy)
            .init(/* args */)
            .code(CODE_PATH)
            .run();
        self
    }

    fn submit(&mut self, from: TestAddress, timestamp: TimestampSeconds, price: u64) {
        self.world.tx()
            .from(from)
            .to(SC_ADDRESS)
            .typed(my_proxy::MyProxy)
            .submit(EGLD_TICKER, USD_TICKER, timestamp, price, DECIMALS)
            .run();
    }

    fn submit_and_expect_err(&mut self, from: TestAddress, timestamp: TimestampSeconds, price: u64, err: &str) {
        self.world.tx()
            .from(from).to(SC_ADDRESS)
            .typed(my_proxy::MyProxy)
            .submit(EGLD_TICKER, USD_TICKER, timestamp, price, DECIMALS)
            .with_result(ExpectError(4, err))
            .run();
    }
}

#[test]
fn full_scenario_test() {
    let mut state = MyTestState::new();
    state.deploy();
    state.submit(state.oracles[0].clone(), TimestampSeconds::new(100), 100_00);
}
```

Methods returning `-> &mut Self` enable chaining:
```rust
state.deploy().configure().setup_users();
```

**Pattern B – Simple Setup Function (for small contracts):**

```rust
fn setup() -> ScenarioWorld {
    let mut world = world();
    world.account(OWNER_ADDRESS).nonce(1).new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
    world.tx()
        .from(OWNER_ADDRESS)
        .typed(my_proxy::MyProxy)
        .init()
        .code(CODE_PATH)
        .run();
    world
}

#[test]
fn my_test() {
    let mut world = setup();
    world.tx()...run();
}
```

**Pattern C – Composable Step Functions (for building on auto-generated tests):**

```rust
#[test]
fn claim_egld_scen() {
    let mut world = world();
    claim_egld_scen_steps(&mut world);
}

pub fn claim_egld_scen_steps(world: &mut ScenarioWorld) {
    fund_egld_and_esdt_scen_steps(world);
    world.tx()...run();
    world.check_account(SC_ADDRESS)...;
}
```

### Common Test Type Conversions

```rust
// Binary data
ManagedByteArray::new_from_bytes(&DEPOSIT_KEY)

// BigUint – prefer primitive types (u32, u64) as arguments where the proxy accepts them
BigUint::from(10u32)
BigUint::from(1_000_000u64)

// Timestamps – always use the wrapper types, not bare u64
TimestampMillis::new(86_400_000u64)
TimestampSeconds::new(100u64)

// Token IDs – always use TestTokenId
TOKEN_ID.to_token_id()               // TestTokenId → TokenIdentifier
TOKEN_ID.to_esdt_token_identifier()  // TestTokenId → EsdtTokenIdentifier (legacy, avoid)

// MultiValueVec
MultiValueVec::from(vec![addr1, addr2])

// Non-zero amounts
NonZeroU64::new(100).unwrap()

// Payment
Payment::new(TOKEN_ID, 0, AMOUNT_100)
Payment::try_new(TOKEN_ID, 0, 100u64).unwrap()

// ManagedBuffer – from accepts &str directly; no need for b"..." or .as_bytes()
ManagedBuffer::from("hello")         // ✅ preferred
```

### Tracing and Snapshots

```rust
// Record all transactions into a trace
world.start_trace();

// Write the accumulated trace to a scenario JSON file
world.write_scenario_trace("trace/my_trace.scen.json");

// Compare with a golden file
let generated = std::fs::read_to_string("trace/my_trace.scen.json").unwrap();
let expected  = std::fs::read_to_string("trace/expected_trace.scen.json").unwrap();
assert_eq!(generated, expected, "Generated trace does not match expected trace");
```

### Advanced Test Patterns

**Dynamic Address Lists:**
```rust
let mut oracle_addresses = Vec::new();
for i in 1..=NR_ORACLES {
    let address_name = format!("oracle{i}");
    let address = TestAddress::new(&address_name);
    world.account(address).nonce(1).balance(STAKE_AMOUNT);
    oracle_addresses.push(address);
}
```

**Pre-Seeding a Contract Account (No Deploy Tx):**
```rust
world
    .account(VAULT_ADDRESS)
    .nonce(1)
    .code(VAULT_PATH)
    .esdt_roles(NFT_TOKEN_ID, vec!["ESDTRoleNFTCreate".to_string()]);
```

**Tx Hash Injection:**
```rust
let tx_hash = world
    .tx()
    .from(OWNER_ADDRESS).to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .add(1u32)
    .tx_hash([22u8; 32])
    .returns(ReturnsTxHash)
    .run();
assert_eq!(tx_hash.as_array(), &[22u8; 32]);
```

**BLS Aggregate Signatures:**
```rust
let (agg_signature, public_keys) = world
    .create_aggregated_signature(3, b"message to sign")
    .expect("failed to create aggregate signature");

let pk_bytes: Vec<Vec<u8>> = public_keys.iter().map(|pk| pk.serialize().unwrap()).collect();

assert!(verify_bls_aggregated_signature(
    pk_bytes.clone(),
    b"message to sign",
    &agg_signature.serialize().unwrap()
));
```

**Spawning Tests in Threads:**
```rust
#[test]
fn my_test_spawned() {
    let handler = std::thread::spawn(my_test);
    handler.join().unwrap();
}
```

### Test Best Practices and Common Pitfalls

**DO:**
- Use typed constants at the top of the file for all hex data, addresses, token IDs, code paths.
- Use transaction IDs (`.id("...")`) for every transaction – they appear in panic messages and trace files.
- Test negative cases first, then positive ones within each scenario.
- Prefer queries over storage checks to verify state – queries are storage-layout independent.
- Chain setup helper methods returning `&mut Self` for readable state progression.
- Keep `world()` focused on execution setup only; put account setup in a separate helper.
- Always use `TimestampSeconds` / `TimestampMillis` wrapper types, never bare `u64`, for block time.
- Always use `TestTokenId` for token identifier constants.

**AVOID:**
- `TestTokenIdentifier` – use `TestTokenId` instead.
- `AddressValue` – pass `TestAddress` / `TestSCAddress` directly.
- Legacy payment methods (`.egld()`, `.esdt()`, `.multi_esdt()`) – use `.payment(...)` instead.
- `ScenarioValueRaw` in committed tests – it is a generator placeholder; always replace with typed values.
- Inline hex strings in transaction calls – use named constants.
- Missing transaction IDs – makes debugging failures very hard.
- Testing only happy paths – always cover owner-only checks, expired conditions, invalid inputs.
- Forgetting to advance block timestamp before testing time-sensitive logic (expiry, timeouts).
- Mismatched byte-array lengths – byte lengths in `[u8; N]` are checked at compile time, but the hex string length must be exactly `2*N`.
- Calling `.commit()` – it is a deprecated backward-compat method; state is always applied immediately.

**Type-Specific Gotchas:**
```rust
// ✅ Correct – exact byte length (128 hex chars = 64 bytes)
const SIGNATURE: [u8; 64] = hex!("...128-hex-chars...");

// ✅ EGLD in payments
.payment((TestTokenId::EGLD_000000, 0, 1_000u64))

// ✅ NFT with no attributes
world.account(ADDRESS).esdt_nft_balance(NFT_ID, nonce, amount, ())

// ✅ Primitive arguments (preferred over managed types)
world.tx().typed(proxy).my_endpoint(42u64, "hello").run();

// ✅ Block time
world.current_block().block_timestamp_seconds(TimestampSeconds::new(100));
// ❌ Never: .block_timestamp_seconds(100u64)
```

### Debugging Test Failures

Use descriptive transaction IDs that match `.scen.json` scenario steps:

| Pattern | Example IDs |
|---|---|
| Deploy | `"deploy"` |
| Fee deposit | `"deposit-fees-1"`, `"deposit-fees-user2"` |
| Fund step | `"fund-egld"`, `"fund-esdt"` |
| Claim | `"claim-egld"`, `"claim5-esdt"` |
| Error case | `"claim3-egld-fail-expired"`, `"claim2-fail"` |

Enable tracing to capture all transactions:
```rust
world.start_trace();  // at the very beginning of the test
```

Verify state at intermediate points:
```rust
world.check_account(SC_ADDRESS)
    .check_storage("str:deposit|0x...", "...");
world.tx()...run();
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
- [ ] Payment endpoints have `#[payable]` annotation
- [ ] Error messages use `require!` macro
- [ ] Contract builds successfully with `sc-meta all build`
- [ ] Output files exist: `output/<name>.wasm` and `output/<name>.abi.json`
- [ ] Tests pass with `sc-meta test` (if tests exist)
- [ ] Deployment tested on devnet before mainnet

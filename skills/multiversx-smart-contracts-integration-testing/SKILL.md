# MultiversX Blackbox Testing Skills Guide

This guide encodes the know-how for writing comprehensive blackbox tests for MultiversX smart contracts, based on real-world experience with multiple contracts (digital-cash, adder, crowdfunding, price-aggregator, payable-features, scenario-tester, etc.).

---

## 0. Auto-Generated Tests from Scenarios

Blackbox tests can be auto-generated from `.scen.json` scenario files. The generator produces a Rust file with this structure:

```rust
use multiversx_sc_scenario::imports::*;

use my_contract::*;

const CODE_PATH: MxscPath = MxscPath::new("output/my-contract.mxsc.json");
const SOME_ADDRESS: TestAddress = TestAddress::new("some-address");
const SOME_SC: TestSCAddress = TestSCAddress::new("some-sc");
const MY_TOKEN: TestTokenId = TestTokenId::new("MYTOKEN-123456");

fn world() -> ScenarioWorld { ... }

#[test]
fn my_scenario_scen() {
    let mut world = world();
    my_scenario_scen_steps(&mut world);
}

pub fn my_scenario_scen_steps(world: &mut ScenarioWorld) {
    // Accounts are set up inline; contracts with pre-existing code skip deploy tx
    world.account(SOME_ADDRESS).nonce(0u64).balance(1_000u64);
    world.account(SOME_SC).nonce(0u64).balance(0u64).code(CODE_PATH);

    world
        .tx()
        .id("my-tx")
        .from(SOME_ADDRESS)
        .to(SOME_SC)
        .typed(my_contract_proxy::MyContractProxy)
        .my_endpoint(arg1, arg2)
        .payment(Payment::try_new(TestTokenId::EGLD_000000, 0, 1_000u64).unwrap())
        // Generator placeholder – replace with a typed value, e.g.:
        // .returns(ExpectValue(42u64))
        .run();
}
```

Key observations from the generator output:

- Each scenario file produces **one `*_scen()` test** and one **`pub fn *_scen_steps()`** function. This separation allows hand-written tests to **compose generated step functions** for more complex scenarios.
- Accounts with pre-loaded code (i.e., not deployed by the test) are set up via `world.account(...).code(CODE_PATH)` directly, skipping a deploy transaction.
- All token constants use `TestTokenId`.
- Payments use `Payment::try_new(token_id, nonce, amount).unwrap()`.
- When the generator cannot infer the return type it emits `ScenarioValueRaw` as a placeholder. **Always replace these with properly typed Rust values** before committing the test.
- Transaction IDs exactly mirror the IDs in the `.scen.json` file; empty IDs (`""`) are allowed.

**Reusing auto-generated steps in hand-written tests:**

```rust
#[test]
fn complex_scenario() {
    let mut world = world();
    // Reuse generated setup
    generated::fund_egld_scen_steps(&mut world);
    // Then add more steps
    world.tx()...run();
}
```

---

## 1. File Structure and Setup

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

Declare **all** addresses, token IDs, code paths, and binary constants at the top of the file.

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
const COOLDOWN_TIME: DurationMillis = DurationMillis::new(86_400);
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

---

## 2. World Setup

### The `world()` Function

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

**Inline in generated step functions:**
```rust
pub fn my_scen_steps(world: &mut ScenarioWorld) {
    world.account(USER_ADDRESS).nonce(0u64).balance(10_000u64)
        .esdt_balance(MY_TOKEN, 500u64)
        .esdt_nft_balance(SFT_ID, 5, 20u64, ())
        ;
    world.account(SC_ADDRESS).nonce(0u64).balance(0u64).code(CODE_PATH);
    // ...
}
```

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
        // Pre-reserve SC address (optional – also done via .new_address() on the deploy tx)
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

---

## 3. Contract Deployment

### Standard Deployment

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

### Capturing the Deployed Address

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

### Deploy in a State Method (Chainable)

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

---

## 4. Transactions and Queries

### General Transaction Shape

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

### Queries (Read-Only Calls)

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

### Payment Types

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

### Capturing Multiple Return Values

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

### Handling Errors Programmatically

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

---

## 5. Block State Manipulation

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

---

## 6. State Verification

### Prefer Queries over Storage Checks

Whenever possible, verify state by **querying view endpoints** rather than checking raw storage. Queries are not tied to the internal storage layout of the contract, so they remain valid if the storage organisation changes.

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

### Account Balance Checks

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

### Contract Storage Checks

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

---

## 7. Test Organization Patterns

### Pattern A – Composable Step Functions (Generated Style)

Best for translating scenario files and for building on top of generated tests:

```rust
#[test]
fn claim_egld_scen() {
    let mut world = world();
    claim_egld_scen_steps(&mut world);
}

pub fn claim_egld_scen_steps(world: &mut ScenarioWorld) {
    // Reuse a prior step function as setup
    fund_egld_and_esdt_scen_steps(world);
    // Then add more steps specific to this scenario
    world.tx()...run();
    world.check_account(SC_ADDRESS)...;
}
```

Composition is the key advantage: later scenario tests call earlier ones as setup, building up state incrementally.

### Pattern B – Test State Struct (Handwritten Style)

Best for complex contracts with many helper operations and shared mutable state:

```rust
struct MyTestState {
    world: ScenarioWorld,
    oracles: Vec<Address>,  // dynamic data alongside world
}

impl MyTestState {
    fn new() -> Self {
        let mut world = world();
        world.account(OWNER_ADDRESS).nonce(1);
        world.current_block().block_timestamp_seconds(TimestampSeconds::new(100));
        world.new_address(OWNER_ADDRESS, 1, SC_ADDRESS);
        Self { world, oracles: Vec::new() }
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

### Pattern C – Simple Setup Function

Good for small contracts with little shared state:

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

---

## 8. Common Type Conversions

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
// Conversion helpers if needed:
TOKEN_ID.to_token_id()               // TestTokenId → TokenIdentifier
TOKEN_ID.to_esdt_token_identifier()  // TestTokenId → EsdtTokenIdentifier (legacy, avoid)
EgldOrEsdtTokenIdentifier::egld()    // (legacy, avoid)
EgldOrEsdtTokenIdentifier::esdt(TOKEN_ID)  // (legacy, avoid)

// Note: EsdtTokenIdentifier and EgldOrEsdtTokenIdentifier are kept for older contracts/tests.
// For new code, use TestTokenId + Payment instead.

// MultiValueVec
MultiValueVec::from(vec![addr1, addr2])

// Non-zero amounts
NonZeroU64::new(100).unwrap()

// Payment
Payment::new(TOKEN_ID, 0, AMOUNT_100)
Payment::try_new(TOKEN_ID, 0, 100u64).unwrap()

// ManagedBuffer – from accepts &str directly; no need for b"..." or .as_bytes()
ManagedBuffer::from("hello")         // ✅ preferred
// ManagedBuffer::from(b"hello")    // ❌ avoid
// ManagedBuffer::from(s.as_bytes()) // ❌ avoid
```

---

## 9. Tracing and Snapshots

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

`start_trace()` is called at the very beginning of the test. Traces are useful for generating golden scenario files from a passing hand-written test.

---

## 10. Advanced Patterns

### Dynamic Address Lists

```rust
let mut oracle_addresses = Vec::new();
for i in 1..=NR_ORACLES {
    let address_name = format!("oracle{i}");
    let address = TestAddress::new(&address_name);
    world.account(address).nonce(1).balance(STAKE_AMOUNT);
    oracle_addresses.push(address);
}

// Use later in transactions
for address in oracle_addresses.iter() {
    world.tx()
        .from(*address)
        .to(SC_ADDRESS)
        .typed(my_proxy::MyProxy)
        .stake()
        .payment((TestTokenId::EGLD_000000, 0, STAKE_AMOUNT))
        .run();
}
```

### Pre-Seeding a Contract Account (No Deploy Tx)

When a test does not need to exercise the deploy path:

```rust
world
    .account(VAULT_ADDRESS)
    .nonce(1)
    .code(VAULT_PATH)
    .esdt_roles(NFT_TOKEN_ID, vec!["ESDTRoleNFTCreate".to_string()]);
```

### Tx Hash Injection

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

### BLS Aggregate Signatures

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

### `ScenarioValueRaw` – Generator Placeholder

`ScenarioValueRaw` is emitted by the auto-generator when it cannot infer the correct Rust return type. It is a **placeholder** that must be replaced before the test is considered complete. Replace it with a properly typed assertion:

```rust
// ❌ Generator placeholder – needs replacement
// .returns(ExpectValue(ScenarioValueRaw::new("nested:str:EGLD-000000|u64:0|biguint:1000")))

// ✅ After replacement – use a typed value
.returns(ExpectValue(Payment::try_new(TOKEN_ID, 0, 1000u32)))

// ✅ Or use a query-based assertion instead
let deposit = world.query().to(SC_ADDRESS).typed(my_proxy::MyProxy)
    .get_deposit(&key).returns(ReturnsResultUnmanaged).run();
assert_eq!(deposit.amount, 1000u64);
```

### Spawning Tests in Threads

```rust
#[test]
fn my_test_spawned() {
    let handler = std::thread::spawn(my_test);
    handler.join().unwrap();
}
```

---

## 11. Best Practices and Common Pitfalls

### ✅ DO

- **Use typed constants** at the top of the file for all hex data, addresses, token IDs, code paths.
- **Use transaction IDs** (`.id("...")`) for every transaction – they appear in panic messages and trace files, making failures trivially debuggable. Mirror the IDs from `.scen.json` files in generated tests.
- **Test negative cases first**, then positive ones within each scenario.
- **Compose step functions**: later test functions call earlier `*_scen_steps()` to reuse setup.
- **Prefer queries** over storage checks to verify state – queries are storage-layout independent.
- **Chain setup helper methods** returning `&mut Self` for readable state progression.
- **Keep `world()` focused** on execution setup only; put account setup in a separate helper.
- **Always use `TimestampSeconds` / `TimestampMillis`** wrapper types, never bare `u64`, for block time.
- **Always use `TestTokenId`** for token identifier constants.

### ❌ AVOID

- **`TestTokenIdentifier`** – use `TestTokenId` instead.
- **`AddressValue`** – pass `TestAddress` / `TestSCAddress` directly.
- **Legacy payment methods** (`.egld()`, `.esdt()`, `.multi_esdt()`) – use `.payment(...)` instead.
- **`ScenarioValueRaw` in committed tests** – it is a generator placeholder; always replace with typed values.
- **Inline hex strings** in transaction calls – use named constants.
- **Missing transaction IDs** – makes debugging failures very hard.
- **Testing only happy paths** – always cover owner-only checks, expired conditions, invalid inputs.
- **Forgetting to advance block timestamp** before testing time-sensitive logic (expiry, timeouts).
- **Mismatched byte-array lengths** – byte lengths in `[u8; N]` are checked at compile time, but the hex string length must be exactly `2*N`.
- **Calling `.commit()`** – it is a deprecated backward-compat method; state is always applied immediately.

### Type-Specific Gotchas

```rust
// ✅ Correct – exact byte length (128 hex chars = 64 bytes)
const SIGNATURE: [u8; 64] = hex!("...128-hex-chars...");

// ❌ Wrong – compile error
const SIGNATURE: [u8; 63] = hex!("...128-hex-chars...");

// ✅ EGLD in payments
.payment((TestTokenId::EGLD_000000, 0, 1_000u64))

// ✅ NFT with no attributes
world.account(ADDRESS).esdt_nft_balance(NFT_ID, nonce, amount, ())
// () means empty; supply actual bytes for attribute checks

// ✅ Multi-value decode
let (a, b) = result.into_tuple();

// ✅ Primitive arguments (preferred over managed types)
world.tx().typed(proxy).my_endpoint(42u64, "hello").run();
// instead of BigUint::from(42u64), ManagedBuffer::from(b"hello")

// ✅ Block time
world.current_block().block_timestamp_seconds(TimestampSeconds::new(100));
// ❌ Never: .block_timestamp_seconds(100u64)
```

---

## 12. Debugging Techniques

### Transaction ID Strategy

Use descriptive IDs that match `.scen.json` scenario steps:

| Pattern | Example IDs |
|---|---|
| Deploy | `"deploy"` |
| Fee deposit | `"deposit-fees-1"`, `"deposit-fees-user2"` |
| Fund step | `"fund-egld"`, `"fund-esdt"` |
| Claim | `"claim-egld"`, `"claim5-esdt"` |
| Withdraw | `"withdraw-ok"`, `"withdraw-fail"` |
| Config | `"set-fee-ok"`, `"whitelist-token"`, `"blacklist-token"` |
| Error case | `"claim3-egld-fail-expired"`, `"claim2-fail"` |

### Enable Tracing

```rust
world.start_trace();  // at the very beginning of the test
```

### Verify State at Intermediate Points

```rust
// After first operation:
world.check_account(SC_ADDRESS)
    .check_storage("str:deposit|0x...", "...");

// Continue with next operation
world.tx()...run();
```

---

## 13. Auto-Generator Conventions Reference

When reading or writing generated files, these conventions apply:

| Element | Convention |
|---|---|
| Token type | `TestTokenId` |
| Per-test function | `fn {name}_scen()` → calls `{name}_scen_steps()` |
| Steps function | `pub fn {name}_scen_steps(world: &mut ScenarioWorld)` |
| Account setup | `world.account(ADDR).nonce(0u64).balance(100u64)` |
| Payment | `Payment::try_new(TOKEN, nonce, amount).unwrap()` |
| Expected return value | `ExpectValue(ScenarioValueRaw::new(...))` – **placeholder, must be replaced** |
| Error expectation | `.with_result(ExpectError(4, "message"))` |
| Transaction ID | mirrors `.scen.json` step `"id"` field verbatim (may be `""`) |
| Pre-existing SC | `world.account(SC_ADDRESS).nonce(0u64).code(CODE_PATH)` – no deploy tx |

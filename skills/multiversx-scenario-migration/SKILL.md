---
name: multiversx-scenario-migration
description: Migrate legacy .scen.json scenario tests to Rust blackbox tests using `sc-meta scen-blackbox`. Use when a MultiversX smart contract project has existing scenario JSON files that need to be converted to the modern typed Rust test format.
---

# Migrating Scenario JSON Tests to Rust Blackbox Tests

Use `sc-meta scen-blackbox` to auto-generate Rust blackbox tests from existing `.scen.json` scenario files. The generated code serves as a starting point that must be reviewed and refined before committing.

## Running the Generator

```bash
sc-meta scen-blackbox
```

This reads all `.scen.json` files in the contract's `scenarios/` directory and produces a Rust test file.

## Generated Code Structure

The generator produces a Rust file with this structure:

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

## Key Observations

- Each scenario file produces **one `*_scen()` test** and one **`pub fn *_scen_steps()`** function. This separation allows hand-written tests to **compose generated step functions** for more complex scenarios.
- Accounts with pre-loaded code (i.e., not deployed by the test) are set up via `world.account(...).code(CODE_PATH)` directly, skipping a deploy transaction.
- All token constants use `TestTokenId`.
- Payments use `Payment::try_new(token_id, nonce, amount).unwrap()`.
- When the generator cannot infer the return type it emits `ScenarioValueRaw` as a placeholder. **Always replace these with properly typed Rust values** before committing the test.
- Transaction IDs exactly mirror the IDs in the `.scen.json` file; empty IDs (`""`) are allowed.

## Reusing Auto-Generated Steps in Hand-Written Tests

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

## Required Post-Generation Cleanup

### Replace `ScenarioValueRaw` Placeholders

`ScenarioValueRaw` is emitted by the auto-generator when it cannot infer the correct Rust return type. It is a **placeholder** that must be replaced before the test is considered complete:

```rust
// ❌ Generator placeholder – needs replacement
// .returns(ExpectValue(ScenarioValueRaw::new("nested:str:EGLD-000000|u64:0|biguint:1000")))

// ✅ After replacement – use a typed value
.returns(ExpectValue(Payment::try_new(TOKEN_ID, 0, 1000u32).unwrap()))

// ✅ Or use a query-based assertion instead
let deposit = world
    .query()
    .to(SC_ADDRESS)
    .typed(my_proxy::MyProxy)
    .get_deposit(&key)
    .returns(ReturnsResultUnmanaged)
    .run();
assert_eq!(deposit.amount, 1000u64);
```

## Auto-Generator Conventions Reference

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

---
name: multiversx-sharp-edges
description: Catalog of non-obvious behaviors, gotchas, and platform-specific quirks in MultiversX that often lead to bugs. Use when debugging unexpected behavior, reviewing code for subtle issues, or learning platform-specific pitfalls.
---

# MultiversX Sharp Edges

A catalog of non-obvious behaviors, "gotchas," and platform-specific quirks that frequently lead to bugs in MultiversX smart contracts and dApps. Understanding these sharp edges is essential for writing correct code.

## When to Use

- Debugging unexpected contract behavior
- Reviewing code for subtle platform-specific issues
- Onboarding to MultiversX development
- Checking if a bug might be caused by a known quirk
- Preparing for security audits

## 1. Async Callbacks & Reverts

### The Sharp Edge
When an async call fails, the `#[callback]` is still executed, but state changes from the original transaction are **NOT automatically reverted**.

### The Problem
```rust
#[endpoint]
fn transfer_and_update(&self, recipient: ManagedAddress, amount: BigUint) {
    // State change happens IMMEDIATELY
    self.total_sent().update(|t| *t += &amount);

    // Async call to another contract
    self.tx()
        .to(&recipient)
        .egld(&amount)
        .callback(self.callbacks().on_transfer())
        .async_call_and_exit();
}

#[callback]
fn on_transfer(&self) {
    // If transfer FAILED, total_sent is STILL updated!
    // This is inconsistent state!
}
```

### The Solution
```rust
#[endpoint]
fn transfer_and_update(&self, recipient: ManagedAddress, amount: BigUint) {
    // DON'T update state before async call
    self.tx()
        .to(&recipient)
        .egld(&amount)
        .callback(self.callbacks().on_transfer(amount.clone()))
        .async_call_and_exit();
}

#[callback]
fn on_transfer(&self, amount: BigUint, #[call_result] result: ManagedAsyncCallResult<()>) {
    match result {
        ManagedAsyncCallResult::Ok(_) => {
            // Only update state on SUCCESS
            self.total_sent().update(|t| *t += &amount);
        },
        ManagedAsyncCallResult::Err(_) => {
            // Handle failure explicitly
            // Funds return to contract automatically
        }
    }
}
```

## 2. Gas Limits & Out of Gas (OOG)

### The Sharp Edge
OOG can leave cross-shard transactions in partial states.

### Cross-Shard OOG Scenario
```
1. Sender shard processes transaction (state changed)
2. Receiver shard runs out of gas
3. Receiver execution fails
4. Sender state changes PERSIST
5. Callback triggered with error
```

### Bad
```rust
// DON'T: Skip gas reservation for callbacks — OOG in callback loses state
self.tx().to(&other).typed(Proxy).call()
    .callback(self.callbacks().on_result())
    .async_call_and_exit(); // No gas reserved for callback!
```

### Good
```rust
// DO: Always reserve explicit gas for callbacks
self.tx().to(&other).typed(Proxy).call()
    .gas(50_000_000)
    .callback(self.callbacks().on_result())
    .gas_for_callback(10_000_000) // Ensures callback can execute
    .async_call_and_exit();
```

### The Solution
```rust
// Always reserve enough gas for callbacks
const CALLBACK_GAS: u64 = 10_000_000;

#[endpoint]
fn safe_cross_shard(&self) {
    self.tx()
        .to(&other_contract)
        .typed(proxy::Proxy)
        .function()
        .gas(50_000_000)
        .callback(self.callbacks().handle_result())
        .gas_for_callback(CALLBACK_GAS)
        .async_call_and_exit();
}
```

## 3. Storage Mappers vs Rust Types

### The Sharp Edge
`VecMapper` is NOT a `Vec`. They have fundamentally different memory models.

### The Problem
```rust
// VecMapper: Each element is a separate storage slot
// Accessing element = 1 storage read
// Iterating N elements = N storage reads
#[storage_mapper("users")]
fn users(&self) -> VecMapper<ManagedAddress>;

// If you load into a Vec, you load EVERYTHING into WASM memory
fn bad_function(&self) {
    let all_users: Vec<ManagedAddress> = self.users().iter().collect();
    // With 10,000 users = 10,000 storage reads + massive memory allocation
    // WILL run out of gas
}
```

### The Solution
```rust
// Paginate operations
fn process_users_paginated(&self, start: usize, count: usize) {
    let len = self.users().len();
    let end = (start + count).min(len);

    for i in start..end {
        let user = self.users().get(i + 1);  // VecMapper is 1-indexed!
        self.process_user(&user);
    }
}

// Or use appropriate mapper for the use case
// SetMapper for O(1) contains checks
// UnorderedSetMapper for efficient removal
```

## 4. Token Decimal Precision

### The Sharp Edge
ESDTs can have 0-18 decimals. Hardcoding decimal assumptions breaks contracts.

### The Problem
```rust
// WRONG: Assumes 18 decimals
fn convert_to_usd(&self, token_amount: BigUint) -> BigUint {
    let price = self.price().get();  // Price in 10^18
    &token_amount * &price / BigUint::from(10u64.pow(18))  // Assumes 18 decimals!
}
```

### The Solution
```rust
fn convert_to_usd(&self, token_amount: BigUint, token_decimals: u8) -> BigUint {
    let price = self.price().get();
    let decimal_factor = BigUint::from(10u64).pow(token_decimals as u32);
    &token_amount * &price / &decimal_factor
}

// Or require specific decimals
fn require_standard_decimals(&self, token_id: &TokenIdentifier) {
    let properties = self.blockchain().get_esdt_token_data(
        &self.blockchain().get_sc_address(),
        token_id,
        0
    );
    require!(properties.decimals == 18, "Token must have 18 decimals");
}
```

## 5. Upgradeability Pitfalls

### The Sharp Edge
`#[init]` is NOT called on upgrade. Only `#[upgrade]` runs.

### The Problem
```rust
// V1 contract
#[init]
fn init(&self) {
    self.version().set(1);
}

// V2 contract - added new storage
#[init]
fn init(&self) {
    self.version().set(2);
    self.new_feature_enabled().set(true);  // NEVER RUNS ON UPGRADE!
}

// After upgrade: version is still 1, new_feature_enabled is empty!
```

### The Solution
```rust
#[upgrade]
fn upgrade(&self) {
    // Initialize new storage here
    self.version().set(2);
    self.new_feature_enabled().set(true);

    // Migrate existing data if needed
    self.migrate_storage();
}
```

### Storage Layout Changes

**NEVER reorder struct fields:**
```rust
// V1
struct UserData {
    balance: BigUint,    // Encoded at position 0
    timestamp: u64,      // Encoded at position 1
}

// V2 - BREAKS EXISTING DATA
struct UserData {
    timestamp: u64,      // Now at position 0 - reads old balance bytes!
    balance: BigUint,    // Now at position 1 - reads old timestamp bytes!
    new_field: bool,     // This is fine (appended)
}
```

## 6. Block Info in Views

### The Sharp Edge
`get_block_timestamp_millis()` / `get_block_timestamp_seconds()` in `#[view]` functions may return different values off-chain vs on-chain. Since Supernova (0.6s rounds), prefer `get_block_timestamp_millis()` with `TimestampMillis` for sub-second precision — `TimestampSeconds` loses granularity when rounds are faster than 1 second.

### The Problem
```rust
// Problem - using seconds loses precision with Supernova's 0.6s rounds
#[view(isExpired)]
fn is_expired(&self) -> bool {
    let deadline = self.deadline().get(); // TimestampMillis
    let current_time = self.blockchain().get_block_timestamp_millis();
    // Off-chain simulation may return 0 or stale value!
    current_time > deadline
}
```

### The Solution
```rust
// Option 1: Don't rely on block info in views
#[view(getDeadline)]
fn get_deadline(&self) -> TimestampMillis {
    self.deadline().get()
    // Let client compare with their known current time
}

// Option 2: Accept timestamp as parameter for queries
#[view(isExpiredAt)]
fn is_expired_at(&self, check_time: TimestampMillis) -> bool {
    let deadline = self.deadline().get();
    check_time > deadline
}
```

## 7. VecMapper Indexing

### The Sharp Edge
`VecMapper` is **1-indexed**, not 0-indexed like Rust `Vec`.

### The Problem
```rust
fn get_first_user(&self) -> ManagedAddress {
    self.users().get(0)  // PANIC! Index 0 doesn't exist
}
```

### The Solution
```rust
fn get_first_user(&self) -> ManagedAddress {
    require!(!self.users().is_empty(), "No users");
    self.users().get(1)  // First element is at index 1
}

fn iterate_users(&self) {
    for i in 1..=self.users().len() {  // 1 to len, inclusive
        let user = self.users().get(i);
        // process user
    }
}
```

## 8. EGLD + ESDT Multi-Transfers (Updated since v0.55.0)

### The Sharp Edge
Since SDK v0.55.0, EGLD and ESDT **can** be sent together in the same multi-transfer transaction using the unified `Payment` type with `TokenId`. However, you must use the new unified payment API — the old `.egld()` + `.single_esdt()` chain still cannot be combined.

### The Old Problem (no longer applies)
```rust
// This used to be impossible, now supported via unified Payment API
```

### The Solution
```rust
// Use unified Payment with TokenId for mixed transfers
let mut payments = ManagedVec::new();
if let Some(egld_nz) = egld_amount.into_non_zero() {
    payments.push(Payment::new(TokenId::from("EGLD-000000"), 0, egld_nz));
}
if let Some(esdt_nz) = esdt_amount.into_non_zero() {
    payments.push(Payment::new(TokenId::from(token_id), 0, esdt_nz));
}
self.tx().to(&recipient).payment(&payments).transfer();
```

## 9. MapMapper Memory Model

### The Sharp Edge
`MapMapper` stores `4*N + 1` storage entries, making it very expensive.

### Bad
```rust
// DON'T: Use MapMapper for per-user data when you don't need iteration
// For 1000 users, this creates 4001 storage entries!
#[storage_mapper("balances")]
fn balances(&self) -> MapMapper<ManagedAddress, BigUint>;
```

### Good
```rust
// DO: Use SingleValueMapper with address key — 1 entry per user
#[storage_mapper("balance")]
fn balance(&self, user: &ManagedAddress) -> SingleValueMapper<BigUint>;
// Only use MapMapper when you MUST iterate over all entries
```

## 10. Require vs SC Panic

### The Sharp Edge
`require!` generates larger WASM than `sc_panic!` when the message is dynamic.

### The Problem
```rust
// Each unique string increases WASM size
require!(condition1, "Error message one");
require!(condition2, "Error message two");
require!(condition3, "Error message three");
```

### The Solution
```rust
// Use static error constants
const ERR_INVALID_AMOUNT: &str = "Invalid amount";
const ERR_UNAUTHORIZED: &str = "Unauthorized";

require!(amount > 0, ERR_INVALID_AMOUNT);
require!(caller == owner, ERR_UNAUTHORIZED);

// Reuse same constant for same error type
require!(amount1 > 0, ERR_INVALID_AMOUNT);
require!(amount2 > 0, ERR_INVALID_AMOUNT);
```

## 11. NonZeroBigUint in Payments (v0.64.0+)

### The Sharp Edge
`Payment.amount` is `NonZeroBigUint`, not `BigUint`. This means zero-amount payments are impossible at the type level, but you must handle conversions when creating payments from `BigUint` values.

### The Problem
```rust
// WRONG - won't compile, Payment expects NonZeroBigUint
let payment = Payment::new(token_id, 0, amount); // amount is BigUint

// WRONG - panics at runtime if amount is zero
let nz = NonZeroBigUint::new_or_panic(amount);
```

### The Solution
```rust
// Option-based (safe)
if let Some(amount_nz) = amount.into_non_zero() {
    let payment = Payment::new(token_id, 0, amount_nz);
    self.tx().to(&to).payment(payment).transfer();
}

// When reading from call_value, amount is already NonZeroBigUint
let payment = self.call_value().single();
// payment.amount is NonZeroBigUint — guaranteed non-zero
// Use .as_big_uint() to get a &BigUint reference for arithmetic
self.balance(&caller).update(|b| *b += payment.amount.as_big_uint());
```

### Key Point
You no longer need `require!(amount > 0, ...)` checks on incoming payments — the type system enforces this. But you still need validation when constructing payments from computed `BigUint` values.

## 12. BackTransfers Accumulation (v0.59.0+)

### The Sharp Edge
Back-transfers from sync calls **accumulate** across multiple calls in the same transaction. Without resetting, you get stale data from previous calls mixed in.

### The Problem
```rust
#[endpoint]
fn multi_swap(&self, dex: ManagedAddress) {
    // First swap
    let bt1 = self.tx().to(&dex).typed(DexProxy)
        .swap_a()
        .returns(ReturnsBackTransfers) // No reset!
        .sync_call();

    // Second swap
    let bt2 = self.tx().to(&dex).typed(DexProxy)
        .swap_b()
        .returns(ReturnsBackTransfers) // No reset!
        .sync_call();

    // BUG: bt2 contains payments from BOTH swap_a AND swap_b
    let total = bt2.into_payment_vec(); // Wrong amount!
}
```

### The Solution
```rust
#[endpoint]
fn multi_swap(&self, dex: ManagedAddress) {
    let bt1 = self.tx().to(&dex).typed(DexProxy)
        .swap_a()
        .returns(ReturnsBackTransfersReset) // Resets before reading
        .sync_call();

    let bt2 = self.tx().to(&dex).typed(DexProxy)
        .swap_b()
        .returns(ReturnsBackTransfersReset) // Resets before reading
        .sync_call();

    // bt1 and bt2 each contain only their own call's payments
}
```

### Key Point
Always use `ReturnsBackTransfersReset` instead of `ReturnsBackTransfers` when an endpoint makes more than one sync call that returns tokens. The `Reset` variant calls `blockchain().reset_back_transfers()` before the call, clearing the accumulator.

## 13. Pending Callback Tracking

### The Sharp Edge
Async calls can fail silently — the callback may never fire if the target contract runs out of gas or panics during execution. Without tracking, your contract will never know the operation is incomplete.

### The Problem
```rust
#[endpoint]
fn delegate_to_provider(&self, provider: ManagedAddress, amount: BigUint) {
    self.pending_amount().update(|p| *p += &amount);
    self.tx().to(&provider)
        .typed(ProviderProxy).delegate()
        .egld(&amount)
        .callback(self.callbacks().on_delegate())
        .async_call_and_exit();
    // If callback never fires, pending_amount is stuck forever
}
```

### The Solution
```rust
#[endpoint]
fn delegate_to_provider(&self, provider: ManagedAddress, amount: BigUint) {
    // Track the pending operation with a unique ID
    let op_id = self.next_op_id().update(|id| { *id += 1; *id });
    self.pending_operations(op_id).set(PendingOp {
        provider: provider.clone(),
        amount: amount.clone(),
        timestamp: self.blockchain().get_block_timestamp_millis(),
    });

    self.tx().to(&provider)
        .typed(ProviderProxy).delegate()
        .egld(&amount)
        .callback(self.callbacks().on_delegate(op_id))
        .async_call_and_exit();
}

#[callback]
fn on_delegate(&self, op_id: u64, #[call_result] result: ManagedAsyncCallResult<()>) {
    self.pending_operations(op_id).clear(); // Always clear tracking
    match result {
        ManagedAsyncCallResult::Ok(_) => { /* success */ }
        ManagedAsyncCallResult::Err(_) => { /* handle failure, refund etc */ }
    }
}

// Admin recovery for stuck operations
#[endpoint(recoverPending)]
fn recover_pending(&self, op_id: u64) {
    require!(self.blockchain().get_caller() == self.blockchain().get_owner_address(), "Not owner");
    let op = self.pending_operations(op_id).get();
    let now = self.blockchain().get_block_timestamp_millis();
    require!(now - op.timestamp > RECOVERY_TIMEOUT_MS, "Too early to recover");
    self.pending_operations(op_id).clear();
    // Refund or retry logic
}
```

## 14. Storage Key Collisions with `storage_mapper_from_address`

### The Sharp Edge
When reading another contract's storage with `#[storage_mapper_from_address("key")]`, if the target contract upgrades and renames its storage keys, your reads silently return default values (zero, empty) — no error.

### The Problem
```rust
// Your contract reads the "reserve" key from a DEX pair
#[storage_mapper_from_address("reserve")]
fn pair_reserve(&self, addr: ManagedAddress, token: &TokenIdentifier)
    -> SingleValueMapper<BigUint, ManagedAddress>;

// DEX upgrades and renames "reserve" to "token_reserve"
// Your reads now return 0 — silently incorrect data!
```

### The Solution
- Pin to specific contract versions in your documentation
- Add sanity checks: if reserve returns 0 for an active pair, something is wrong
- Consider adding a staleness check or fallback to proxy calls

```rust
fn get_pair_reserve_safe(&self, pair: &ManagedAddress, token: &TokenIdentifier) -> BigUint {
    let reserve = self.pair_reserve(pair.clone(), token).get();
    // Sanity check — active pairs should never have zero reserves
    if reserve == 0u64 {
        // Fallback: use proxy call or revert
        sc_panic!("Unexpected zero reserve — target contract may have changed storage layout");
    }
    reserve
}
```

## 15. Cache Invalidation Across Async Boundaries

### The Sharp Edge
`async_call_and_exit()` terminates execution immediately. A Drop-based cache in the same scope will **never** have its `drop()` called — cached writes are silently lost.

### The Problem
```rust
fn bad_pattern(&self) {
    let mut cache = StorageCache::new(self);
    cache.balance += &deposit_amount;

    // async_call_and_exit() terminates execution — drop() NEVER runs!
    self.tx().to(&other).typed(Proxy).call()
        .callback(self.callbacks().on_result())
        .async_call_and_exit();
    // cache.drop() never fires — balance change is LOST
}
```

### The Solution
Manually drop the cache (via scoping) before the async call:
```rust
fn good_pattern(&self) {
    {
        let mut cache = StorageCache::new(self);
        cache.balance += &deposit_amount;
    } // cache.drop() fires here — writes committed

    self.tx().to(&other).typed(Proxy).call()
        .callback(self.callbacks().on_result())
        .async_call_and_exit();
}
```

**Note:** State committed before the async call persists even if the async call fails. Track pending operations if you need rollback (see item 13).

## 16. Rounding Attack Vectors in Financial Calculations

### The Sharp Edge
Default `ManagedDecimal` arithmetic truncates (rounds toward zero). In lending/staking protocols, this creates systematic value leakage that attackers can exploit with many small operations.

### The Problem
```rust
// Each deposit loses a fraction of a token due to truncation
// Attacker makes 1000 tiny deposits, each time extracting the rounding difference
let shares = (amount * total_shares) / total_supply; // Truncates!
```

### The Solution
Use half-up rounding for ALL financial calculations. See the `multiversx-defi-math` skill for implementation.

## Quick Reference: Common Gotchas

| Issue | Wrong | Right |
|-------|-------|-------|
| VecMapper index | `.get(0)` | `.get(1)` |
| Callback state | Update before async | Update in callback on success |
| Upgrade init | Rely on `#[init]` | Use `#[upgrade]` |
| Decimals | Hardcode `10^18` | Fetch from token properties |
| MapMapper | Use for per-user data | Use SingleValueMapper with key |
| Block info in view | Direct use | Pass as parameter |
| EGLD + ESDT | Old: same tx impossible | Use unified `Payment` with `TokenId` in multi-transfer |
| NonZeroBigUint | `Payment::new(id, 0, big_uint)` | `amount.into_non_zero()` then `Payment::new(...)` |
| Struct fields | Reorder | Only append |
| BackTransfers | `ReturnsBackTransfers` with multiple calls | `ReturnsBackTransfersReset` — resets accumulator |
| Pending callbacks | Fire-and-forget async | Track with op ID + recovery endpoint |
| Cross-contract storage keys | Assume keys never change | Sanity checks + version pinning |
| Cache + async | Drop cache before async call | Manual commit in callback only |
| Financial rounding | Default truncation | Half-up rounding (mul_half_up/div_half_up) |

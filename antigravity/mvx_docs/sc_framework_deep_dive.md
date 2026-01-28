# SC Framework Deep Dive (`mx-sdk-rs`)

### Storage Mappers: Deep Dive & Comparisons

| Mapper | Best Use Case | Gas Profile | Notes |
| :--- | :--- | :--- | :--- |
| **`SingleValueMapper`** | Individual flags, counters, or *small* structs. | **Lowest**. Direct key-value access. | Preferred over `MapMapper` for simple 1-to-1 mappings (e.g., `user_id -> address`). fast, cheap. |
| **`VecMapper`** | Ordered lists where you need index access. | Low to Medium. Top-encoding. | Stores each element under a separate key. Good for partial reads. |
| **`SetMapper`** | Unique collections, whitelist checks. | Medium. Checks existence before insert. | Enforces uniqueness. expensive if large. |
| **`MapMapper`** | **AVOID** unless iteration over keys is strictly required. | **Highest**. 4 storage writes per entry. | Uses linked-list structure for keys + node links. 4-5x more expensive than `SingleValueMapper`. |
| **`UserMapper`** | Mapping Addresses to Users/IDs. | Optimized `SingleValue`. | Specialized for minimizing address storage costs. |

> [!TIP]
> **Optimization Strategy**: Replace `MapMapper<Key, Value>` with `SingleValueMapper<Value>` keyed by `Key` whenever you don't need to iterate over all keys on-chain. This saves ~75% of gas on writes.

### API Surface Area

The `self.blockchain()` and `self.send()` APIs provide low-level access to the VM.

#### Blockchain API

- **Context**: `get_caller()`, `get_sc_address()`, `get_shard_of_address(addr)`.
- **Block Info**: `get_block_timestamp()`, `get_block_nonce()`, `get_block_random_seed()`.
- **State**: `get_balance(addr)`, `get_esdt_balance(addr, token, nonce)`.

#### Send API (Async & Sync)

- **`direct()`**: Simple transfer. Fails if dest is a contract in same shard that reverts (sync).
- **`direct_egld()` / `direct_esdt()`**: Typed wrappers for safety.
- **`contract_call()`**: robust builder for async calls.

## Modular Architecture

- **`multiversx-sc`**: The core. Defines `[#contract]`, `[#endpoint]`.
- **`multiversx-sc-modules`**: Reusable logic (Traits).
  - *Pattern*: Define logic in a Trait, implement it in the main Contract.
- **`multiversx-sc-scenario`**: The Integration Testing engine.

## API Modules

1. **`BlockchainApi`**: Read global state (`get_block_nonce`, `get_sc_address`).
2. **`CallValueApi`**: Handle payments (`egld_value()`, `single_esdt()`).
3. **`SendApi`**: Effectuate transfers (`direct_egld`, `propose_async_call`).
4. **`CryptoApi`**: Hashing (`sha256`), Verify (`verify_ed25519`).

## Storage Optimization

- **Mappers**:
  - `SingleValueMapper`: Cheap, 1 slot.
  - `VecMapper`: Expensive if large (avoid iterating).
  - `UnorderedSetMapper`: Great for membership checks `O(1)`.
- **Attributes**: Store data in User's Wallet (in SFT attributes) to save Contract state.

## Testing Philosophy (RustVM)

- **Integration (Blackbox)**: Use `ScenarioWorld` to simulate interactions between multiple contracts.

## Best Practices & Safety

### 1. Arithmetic Safety

- **Avoid Rust Primitives**: Do `u64`, `u32`, etc. ONLY for loop counters or strictly small values.
- **Use Managed Types**: Always use `BigUint` for financial values.
  - **Why?**: Overflow protection and gas optimization (handled by VM host functions).
  - **Example**: `let sum = self.total_supply().get() + payment_amount;` (Safe add).

### 2. Gas Optimization

- **Avoid Dynamic Allocation**: `String`, `Vec<u8>` in Rust imply allocation. Use `ManagedBuffer` which points to handle in VM memory.
- **Storage Reads**: Use `SingleValueMapper` over `VecMapper` where possible.
- **Events**: Don't emit massive data in events; it costs gas.

### 3. Security

- **Checks-Effects-Interactions**: Update your storage (Effects) *before* making any external contract calls (Interactions) to prevent Re-entrancy.
  - *Note*: MultiversX Async calls are safer by default, but sync calls (`contract_call`) still require care.
- **Access Control**: Use `#[only_owner]` or `#[only_admin]` annotations strictly.

## WASM & SpaceVM

The **SpaceVM** is the execution engine.

- **Stateless execution**: VM is spun up for the transaction and torn down after.
- **Host Functions**: The Contract imports functions from the VM (e.g., `bigIntAdd`, `storageStore`).
  - Calling a Host Function is cheaper than implementing the logic in WASM (e.g., Crypto primitives).

# Smart Contract Framework Internals (`mx-sdk-rs`)

## Core Architecture

The framework is a set of distinct crates:

- `multiversx-sc`: The main API surface.
- `multiversx-sc-derive`: Macros (`#[contract]`, `#[endpoint]`).
- `multiversx-sc-scenario`: Testing engine.

## 1. Storage Mappers (Under the Hood)

Mappers are zero-cost abstractions over Keys. They do not hold data in struct memory; they hold the *Key*.

### SingleValueMapper

Located in `framework/base/src/storage/mappers/single_value_mapper.rs`.

- **Structure**:

  ```rust
  pub struct SingleValueMapper<SA, T> {
      key: StorageKey<SA>,
      ...
  }
  ```

- **Mechanism**:
  - `get()`: Calls `self.address.address_storage_get(self.key)`.
  - `set(val)`: Calls `storage_set(self.key, val)`.
  - **Encoding**: Uses `TopEncode` / `TopDecode`.

## 2. API & SpaceVM Interaction

Located in `framework/base/src/api/blockchain_api.rs`.

The `BlockchainApiImpl` trait defines the interface to the VM.

- **Handles**: Interaction uses `ManagedBufferHandle` (i32 index) pointing to memory in the VM, not the WASM memory.
- **Example**:

  ```rust
  fn get_caller_handle(&self) -> ManagedBufferHandle {
      let handle = self.mb_new();
      self.mb_overwrite(handle, self.get_caller_legacy().as_bytes());
      handle
  }
  ```

- **Efficiency**: Large data (BigUint, Buffers) stays in Host memory. The Contract only holds a pointer (handle). Use `Managed types` to avoid copying data into WASM stack.

## 3. Testing Internals

- **Whitebox**: Unit tests run directly against the Rust code, mocking `BlockchainApi`.
- **Blackbox**: Scenarios compile the code to WASM (or use `mandos-go` runner) and simulate the actual VM execution steps.

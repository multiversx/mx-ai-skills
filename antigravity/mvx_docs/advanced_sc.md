# Advanced Smart Contract Concepts

## Contract Execution Models

### 1. Synchronous Calls (Intra-Shard)

- **Use Case**: Calling a contract located in the *same shard*.
- **Mechanism**: Executed inline. The function returns a value immediately.
- **Syntax**:

  ```rust
  let result: BigUint = self.callee_proxy(address)
      .endpoint(args)
      .execute_on_dest_context();
  ```

- **BackTransfers**: Synchronous calls strictly return the function result. However, for returning *tokens* (payments) alongside results, the `execute_on_dest_context_with_back_transfers` pattern (or simply relying on the protocol's automatic unspent gas/token return on failure) is key.
  - *Note*: True "BackTransfers" of tokens in a success case usually require the callee to send funds back to the caller explicitly, which might fail if not handled correctly in a sync context (balance check).
- **Constraint**: Fails if target is in a different shard.

### 2. Asynchronous Calls (Cross-Shard)

- **Use Case**: Interaction with contracts on other shards.
- **Mechanism**: The call is dispatched, and the current execution *terminates* (unless using Promises).
- **Callback**: You must define a `#[callback]` function to handle the result (success/failure) in a future block.
- **Error Handling**:
  - `#[callback]`: Handles both success and failure (via `ManagedAsyncCallResult`).
  - **Best Practice**: Always check `result` in callback. Use `self.err_storage().set(&err.err_msg)` to log errors.
  - **Gas**: Reserve extra gas for the callback execution to ensure cleanup/logic runs even if the main call consumes most gas.
- **Syntax**:

  ```rust
  self.callee_proxy(address)
      .endpoint(args)
      .async_call()
      .with_callback(self.callbacks().my_callback())
      .call_and_exit();
  ```

### 3. Promises (Protocol v1.6+)

- **Feature**: Launches async calls *without* terminating the current execution.
- **Benefits**: Allows multiple async calls in one transaction.
- **Syntax**:

  ```rust
  self.callee_proxy(address)
      .endpoint(args)
      .async_call_promise()
      .register_promise();
  ```

### 4. Transfer-Execute

- **Mode**: Fire-and-forget.
- **Use Case**: Sending funds/data to another contract without needing a callback.

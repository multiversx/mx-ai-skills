# Advanced Smart Contract Concepts

## Contract Execution Models

### 1. Synchronous Calls (Intra-Shard)

- **Use Case**: Calling a contract located in the *same shard*.
- **Mechanism**: Executed inline. The function returns a value immediately.
- **Syntax**:

  ```rust
  let result: BigUint = self.tx()
      .to(&address)
      .typed(callee_proxy::CalleeProxy)
      .endpoint(args)
      .returns(ReturnsResult)
      .sync_call();
  ```

- **Constraint**: Fails if target is in a different shard.

### BackTransfers (Retrieving Tokens from Calls)

When a callee sends tokens back to the caller (e.g., a DEX returning swapped tokens), these are captured as **back-transfers**. The mechanism differs by call type:

#### Sync Calls — Result Handlers

```rust
// Basic: get all back-transferred payments
let back_transfers = self.tx()
    .to(&dex_address)
    .typed(dex_proxy::DexProxy)
    .swap(token_in, amount_in)
    .returns(ReturnsBackTransfers)
    .sync_call();

// Convert to v0.64 Payment types
let payments: PaymentVec = back_transfers.into_payment_vec();
for payment in &payments {
    // payment.token_identifier: TokenId
    // payment.token_nonce: u64
    // payment.amount: NonZeroBigUint (guaranteed non-zero)
}
```

**Available result handlers:**

| Handler | Returns | Use When |
|---|---|---|
| `ReturnsBackTransfers` | `BackTransfers` | General case — get all payments |
| `ReturnsBackTransfersReset` | `BackTransfers` | Multiple sync calls in one tx — resets accumulator first |
| `ReturnsBackTransfersEGLD` | `BigUint` | Only need the EGLD sum |
| `ReturnsBackTransfersSingleESDT` | `EsdtTokenPayment` | Expect exactly one ESDT back (panics otherwise) |

**Critical: The accumulation gotcha with multiple sync calls:**

```rust
// BUG: Without Reset, back-transfers ACCUMULATE across calls
let bt1 = self.tx().to(&a).typed(P).call1().returns(ReturnsBackTransfers).sync_call();
let bt2 = self.tx().to(&b).typed(P).call2().returns(ReturnsBackTransfers).sync_call();
// bt2 contains payments from BOTH call1 AND call2!

// FIX: Use ReturnsBackTransfersReset
let bt1 = self.tx().to(&a).typed(P).call1().returns(ReturnsBackTransfersReset).sync_call();
let bt2 = self.tx().to(&b).typed(P).call2().returns(ReturnsBackTransfersReset).sync_call();
// bt2 contains only call2's payments
```

**Rule of thumb**: Always use `ReturnsBackTransfersReset` when making >1 sync call that returns tokens in the same endpoint.

#### Promises Callbacks — `blockchain().get_back_transfers()`

In `#[promises_callback]`, use the blockchain API directly:

```rust
#[promises_callback]
fn swap_callback(&self) {
    let back_transfers = self.blockchain().get_back_transfers();
    let payments = back_transfers.into_payment_vec();
    for payment in &payments {
        // Store/forward received tokens
    }
}
```

#### BackTransfers Helper Methods

| Method | Returns | Description |
|---|---|---|
| `.egld_sum()` | `BigUint` | Sum of all EGLD-000000 transfers |
| `.to_single_esdt()` | `EsdtTokenPayment` | Single ESDT (panics if not exactly one) |
| `.into_payment_vec()` | `PaymentVec` (`ManagedVec<Payment>`) | Convert to v0.64 `Payment` types |
| `.into_multi_value()` | `MultiValueEncoded<EgldOrEsdtTokenPaymentMultiValue>` | For returning from endpoints |

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
  self.tx()
      .to(&address)
      .typed(callee_proxy::CalleeProxy)
      .endpoint(args)
      .callback(self.callbacks().my_callback())
      .async_call_and_exit();
  ```

### 3. Promises (Protocol v1.6+)

- **Feature**: Launches async calls *without* terminating the current execution.
- **Benefits**: Allows multiple async calls in one transaction.
- **Syntax**:

  ```rust
  self.tx()
      .to(&address)
      .typed(callee_proxy::CalleeProxy)
      .endpoint(args)
      .callback(self.callbacks().my_callback())
      .gas_for_callback(10_000_000)
      .register_promise();
  ```

### 4. Transfer-Execute

- **Mode**: Fire-and-forget.
- **Use Case**: Sending funds/data to another contract without needing a callback.

# SDK-DApp Internals (`mx-sdk-dapp`)

## Architecture Overview

The library is designed around a **Provider Strategy Pattern**.

- **Central Coordinator**: `DappProvider` wrapping a specific implementation (Ledger, Extension, etc.).
- **State Management**: Redux-based (exposed via `useSelector`).
- **Transaction Lifecycle**: Managed by a singleton `TransactionManager`.

## Core Components

### 1. DappProvider

Located in `providers/DappProvider/DappProvider.ts`.

- **Role**: Unified interface for all wallet connections.
- **Key Methods**:
  - `init()`: Initializes the underlying provider (e.g., checks if Extension is present).
  - `login()`: Trigger's the provider specific login flow.
  - `signTransactions()`: Delegates signing to the provider.
- **State Updates**: side-effects update the global Redux store (`setAccountProvider`, `setProviderType`).

### 2. TransactionManager

Located in `managers/TransactionManager/TransactionManager.ts`.

- **Pattern**: Singleton.
- **Responsibilities**:
  - **Sending**: `sendSignedBatchTransactions` posts to the Proxy/API.
  - **Tracking**: `track()` creates a session ID and polls for status.
  - **UI Feedback**: Integration with `ToastManager`.
- **Batching**: Automatically detects if 2D array of txs (`Transaction[][]`) is passed for batch execution.

### 3. Provider Strategies

Each login method (Ledger, xPortal, Extension) implements a strategy extending `BaseProviderStrategy`.

- **Example**: `LedgerProviderStrategy`.
  - Uses `@multiversx/sdk-hw-provider` for low-level HID communication.
  - **Reconnection**: `rebuildProvider()` attempts to re-establish connection if the device was locked/disconnected.

### 4. React Hooks

- **`useGetAccount`**: Pure selector.

  ```typescript
  export const useGetAccount = () => useSelector(accountSelector);
  ```

- **`useTransactions`**: (Implied) Selects transaction state from the store populated by `TransactionManager`.

## Best Practices derived from Code

1. **Always use `DappProvider` wrapper**: Do not instantiate `LedgerProvider` directly if you want it to play nice with the rest of the dApp state.
2. **Transactions**: Use `sendTransactions` (plural) to leverage efficient batching and toaster integration.
3. **Error Handling**: The `signTransactions` method catch block wraps errors in `handleSignError`, giving user-friendly messages for "User Rejected" etc.

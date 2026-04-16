# SDK-DApp Deep Dive

`mx-sdk-dapp` is the standard library for React-based DApps.

## Core Hooks

- **`useGetAccount()`**: Returns address, balance, nonce. *Reactive*.
- **`useGetLoginInfo()`**: Returns `isLoggedIn`, `loginMethod`, `tokenLogin`.
- **`useGetNetworkConfig()`**: Returns chainID, API URL, Proxy URL.

## Authentication Strategies

1. **xPortal (WalletConnect)**: The standard. "Invisible Guardians" supported.
2. **Web Wallet**: New standard is Passkey-based.
3. **Extension**: For power users.
4. **Ledger**: Hardware security.

## Custom Signing Flow

For advanced implementation (e.g., Multi-sig, proprietary wallets):

1. Implement `IProvider` interface.
2. Register with `initApp({ customProviders: [...] })`.
3. The `Transaction` object must be serialized using `mx-sdk-core` rules (Ed25519).

## Best Practices

- **Optimistic UI**: Use `TransactionsToastList` to show "Processing..." immediately.
- **State Management**: Do NOT start your own manual Redux store for account state; rely on `DappProvider` context.
- **Signing**: Always utilize `signTransactions` from the hook/service, never raw private key manipulation in frontend.

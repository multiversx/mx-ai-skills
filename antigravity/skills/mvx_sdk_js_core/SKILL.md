---
name: mvx_sdk_js_core
description: Core SDK operations for MultiversX TypeScript/JavaScript - Entrypoints, Network Providers, and Transactions.
---

# MultiversX SDK-JS Core Operations

This skill covers the fundamental building blocks of the `@multiversx/sdk-core` v15 library.

## Entrypoints

Network-specific clients that simplify common operations:

| Entrypoint | Network | Default URL |
|------------|---------|-------------|
| `DevnetEntrypoint` | Devnet | `https://devnet-api.multiversx.com` |
| `TestnetEntrypoint` | Testnet | `https://testnet-api.multiversx.com` |
| `MainnetEntrypoint` | Mainnet | `https://api.multiversx.com` |
| `LocalnetEntrypoint` | Local | `http://localhost:7950` |

```typescript
import { DevnetEntrypoint } from "@multiversx/sdk-core";

// Default API
const entrypoint = new DevnetEntrypoint();

// Custom API
const entrypoint = new DevnetEntrypoint({ url: "https://custom-api.com" });

// Proxy mode (gateway)
const entrypoint = new DevnetEntrypoint({ 
    url: "https://devnet-gateway.multiversx.com", 
    kind: "proxy" 
});
```

## Entrypoint Methods

| Method | Description |
|--------|-------------|
| `createAccount()` | Create a new account instance |
| `createNetworkProvider()` | Get underlying network provider |
| `recallAccountNonce(address)` | Fetch current nonce from network |
| `sendTransaction(tx)` | Broadcast single transaction |
| `sendTransactions(txs)` | Broadcast multiple transactions |
| `getTransaction(txHash)` | Fetch transaction by hash |
| `awaitCompletedTransaction(txHash)` | Wait for finality |

## Network Providers

Lower-level network access:

| Provider | Use Case |
|----------|----------|
| `ApiNetworkProvider` | Standard API queries (recommended) |
| `ProxyNetworkProvider` | Direct node interaction via gateway |

```typescript
const provider = entrypoint.createNetworkProvider();

// Or manual instantiation
import { ApiNetworkProvider } from "@multiversx/sdk-core";

const api = new ApiNetworkProvider("https://devnet-api.multiversx.com", {
    clientName: "my-app",
    requestsOptions: { timeout: 10000 }
});
```

## Provider Methods

| Method | Description |
|--------|-------------|
| `getNetworkConfig()` | Chain ID, gas settings, etc. |
| `getNetworkStatus()` | Current epoch, nonce, etc. |
| `getBlock(blockHash)` | Fetch block data |
| `getAccount(address)` | Account balance, nonce |
| `getAccountStorage(address)` | Contract storage |
| `getTransaction(txHash)` | Transaction details |
| `awaitTransactionCompletion(txHash)` | Wait for finality |
| `queryContract(query)` | VM query (read-only) |

## Transaction Lifecycle

```typescript
// 1. Create account and sync nonce
const account = await Account.newFromPem("wallet.pem");
account.nonce = await entrypoint.recallAccountNonce(account.address);

// 2. Create transaction (via controller)
const controller = entrypoint.createTransfersController();
const tx = await controller.createTransactionForTransfer(
    account, 
    account.getNonceThenIncrement(),
    { receiver, nativeAmount: 1000000000000000000n }
);

// 3. Send
const txHash = await entrypoint.sendTransaction(tx);

// 4. Wait for completion
const result = await entrypoint.awaitCompletedTransaction(txHash);
console.log("Status:", result.status);
```

## Best Practices

1. **Always sync nonce** before creating transactions
2. **Use `awaitCompletedTransaction`** for critical operations
3. **Handle errors** - network calls can fail
4. **Use appropriate timeouts** for long operations
5. **Batch transactions** when possible with `sendTransactions()`

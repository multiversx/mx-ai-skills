---
name: mvx_sdk_go_interactors
description: Components for interacting with the blockchain (Wallets, Transaction Interactors, Nonce Handlers) in Go.
---

# MultiversX SDK-Go Interactors

## Wallet Management

Loading keys and addresses.

```go
import "github.com/multiversx/mx-sdk-go/interactors"

wallet := interactors.NewWallet()

// Load PEM
privateKey, err := wallet.LoadPrivateKeyFromPemFile("wallet.pem")

// Load Keystore
privateKey, err := wallet.LoadPrivateKeyFromKeystoreFile("wallet.json", "password")

// Get Address
address, err := wallet.GetAddressFromPrivateKey(privateKey)
bech32Address := address.AddressAsBech32String()
```

## Transaction Nonce Handler (V3)

Automatically fetches and increments nonces. Recommended way to manage nonces.

```go
import "github.com/multiversx/mx-sdk-go/interactors/nonceHandlerV3"

// Initialize Handler
args := nonceHandlerV3.ArgsAddressNonceHandler{
    Proxy: proxy,
    Address: address,
}
handler, err := nonceHandlerV3.NewAddressNonceHandler(args)

// Fetch Initial Nonce
nonce, err := handler.GetNonce(ctx)

// Apply Nonce to Transaction & Increment
handler.ApplyNonceAndGasPrice(tx) // Sets tx.Nonce = current, then increments internal counter
```

## Transaction Interactor

High-level wrapper for sending transactions.

```go
import "github.com/multiversx/mx-sdk-go/interactors"

// Setup
txInteractor, err := interactors.NewTransactionInteractor(proxy, txBuilder)

// Send Transaction
txHash, err := txInteractor.SendTransaction(ctx, tx, privateKey)

// Send Multiple Transactions
txHashes, err := txInteractor.SendTransactions(ctx, txs, privateKey)
```

## Creating a Wallet Instance

```go
// Helper to create new keypair
privateKey, publicKey := wallet.GeneratePrivateKey()
```

## Best Practices

1. **Use NonceHandlerV3** for robust nonce management, especially in concurrent environments
2. **Use TransactionInteractor** to simplify signing + sending flow
3. **Secure Key Management** - avoid hardcoding private keys

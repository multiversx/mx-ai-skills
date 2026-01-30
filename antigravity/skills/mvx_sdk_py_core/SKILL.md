---
name: mvx_sdk_py_core
description: Core SDK operations for MultiversX Python SDK - Entrypoints, Providers, and Network.
---

# MultiversX SDK-Py Core Operations

This skill covers the fundamental building blocks of the `multiversx-sdk` v2 library.

## Entrypoints

Network-specific clients that simplify common operations:

| Entrypoint | Network | Default URL |
|------------|---------|-------------|
| `DevnetEntrypoint` | Devnet | `https://devnet-api.multiversx.com` |
| `TestnetEntrypoint` | Testnet | `https://testnet-api.multiversx.com` |
| `MainnetEntrypoint` | Mainnet | `https://api.multiversx.com` |
| `LocalnetEntrypoint` | Local | `http://localhost:7950` |

```python
from multiversx_sdk import DevnetEntrypoint

# Default API
entrypoint = DevnetEntrypoint()

# Custom API
entrypoint = DevnetEntrypoint(url="https://custom-api.com")

# Proxy mode (gateway)
entrypoint = DevnetEntrypoint(url="https://devnet-gateway.multiversx.com", kind="proxy")
```

## Entrypoint Methods

| Method | Description |
|--------|-------------|
| `create_account()` | Create a new account instance |
| `create_network_provider()` | Get underlying network provider |
| `recall_account_nonce(address)` | Fetch current nonce from network |
| `send_transaction(tx)` | Broadcast single transaction |
| `send_transactions(txs)` | Broadcast multiple transactions |
| `get_transaction(tx_hash)` | Fetch transaction by hash |
| `await_completed_transaction(tx_hash)` | Wait for finality |

## Network Providers

Lower-level network access:

| Provider | Use Case |
|----------|----------|
| `ApiNetworkProvider` | Standard API queries (recommended) |
| `ProxyNetworkProvider` | Direct node interaction via gateway |

```python
# From entrypoint
provider = entrypoint.create_network_provider()

# Manual instantiation
from multiversx_sdk import ApiNetworkProvider

api = ApiNetworkProvider("https://devnet-api.multiversx.com", client_name="my-app")
```

## Provider Methods

| Method | Description |
|--------|-------------|
| `get_network_config()` | Chain ID, gas settings |
| `get_network_status()` | Current epoch, nonce |
| `get_block(block_hash)` | Fetch block data |
| `get_account(address)` | Account balance, nonce |
| `get_account_storage(address)` | Contract storage |
| `get_transaction(tx_hash)` | Transaction details |
| `query_contract(query)` | VM query (read-only) |

## Transaction Lifecycle

```python
# 1. Create account and sync nonce
account = Account.new_from_pem("wallet.pem")
account.nonce = entrypoint.recall_account_nonce(account.address)

# 2. Create transaction (via controller)
controller = entrypoint.create_transfers_controller()
tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    native_amount=1000000000000000000
)

# 3. Send
tx_hash = entrypoint.send_transaction(tx)

# 4. Wait for completion
result = entrypoint.await_completed_transaction(tx_hash)
print(f"Status: {result.status}")
```

## Best Practices

1. **Use Entrypoints**: Simplifies configuration
2. **Sync nonce**: Before creating any transaction
3. **Handle exceptions**: Wrap network calls
4. **Use integers**: For amounts (18 decimals for EGLD)

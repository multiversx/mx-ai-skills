---
name: mvx_sdk_py_wallets
description: Wallet and key management for MultiversX Python SDK.
---

# MultiversX SDK-Py Wallets & Signing

This skill covers account management, key derivation, and transaction signing.

## Account Creation

```python
from multiversx_sdk import Account, Mnemonic, UserWallet, UserPem

# From PEM file (testing only)
account = Account.new_from_pem("wallet.pem")

# From keystore (production)
account = Account.new_from_keystore("wallet.json", "password")

# From secret key (hex string)
account = Account(secret_key=bytes.fromhex("..."))
```

## Mnemonic Operations

```python
# Generate new mnemonic
mnemonic = Mnemonic.generate()
words = mnemonic.get_words()

# Derive keys
secret_key = mnemonic.derive_key(0)  # Index 0
public_key = secret_key.generate_public_key()
address = public_key.to_address("erd")

# Get address from mnemonic
address = Mnemonic.to_address(mnemonic, 0)
```

## Wallet Storage

```python
# Save mnemonic to keystore
wallet = UserWallet.from_mnemonic(mnemonic.get_text(), "password")
wallet.save("wallet.json")

# Save secret key to keystore
wallet = UserWallet.from_secret_key(secret_key, "password")
wallet.save("wallet.json")

# Save to PEM (testing only)
pem = UserPem(label=address.to_bech32(), secret_key=secret_key)
pem.save("wallet.pem")
```

## Transaction Signing

```python
# Controller-created transactions are auto-signed
tx = controller.create_transaction_for_transfer(account, nonce, ...)

# Manual signing
tx.signature = account.sign_transaction(tx)
```

## Message Signing

```python
from multiversx_sdk import Message

message = Message("Hello".encode())
signature = account.sign_message(message)

# Verify
is_valid = account.verify_message(message, signature)
```

## NativeAuth

Authentication for dApps:

```python
from multiversx_sdk import NativeAuthClient, NativeAuthServer

# Client: Generate token
client = NativeAuthClient("https://app.example.com")
init_token = client.initialize()
# sign init_token...
access_token = client.get_token(address, init_token, signature)

# Server: Validate token
server = NativeAuthServer(accepted_origins=["https://app.example.com"])
result = server.validate(access_token)
# result.address, result.origin, result.block_hash
```

## Ledger

Requires `ledger-wallets` dependency.

```python
from multiversx_sdk import LedgerAccount

ledger = LedgerAccount(index=0)
print(ledger.address)
tx.signature = ledger.sign_transaction(tx)
```

## Security Best Practices

1. **Protect keystore passwords**
2. **Never expose PEM files** in production
3. **Validate addresses** before sending
4. **Use NativeAuth** for authentication

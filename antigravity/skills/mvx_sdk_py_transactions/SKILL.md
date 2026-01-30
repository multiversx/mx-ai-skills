---
name: mvx_sdk_py_transactions
description: Transaction creation and token operations for MultiversX Python SDK.
---

# MultiversX SDK-Py Transactions & Tokens

This skill covers creating transactions using Controllers and Factories.

## Transfers

### Native EGLD Transfer
```python
controller = entrypoint.create_transfers_controller()

tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    native_amount=1000000000000000000
)
```

### ESDT Token Transfer
```python
from multiversx_sdk import TokenTransfer

tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    token_transfers=[
        TokenTransfer.fungible_from_amount("TOKEN-123456", 100.5, 18)
    ]
)
```

### NFT/SFT Transfer
```python
tx = controller.create_transaction_for_transfer(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    receiver=receiver_address,
    token_transfers=[
        TokenTransfer.nft_from_amount("NFT-123456", 1, 1)
    ]
)
```

## Token Management

### Issue Fungible Token
```python
controller = entrypoint.create_token_management_controller()

tx = controller.create_transaction_for_issuing_fungible(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    token_name="MyToken",
    token_ticker="MTK",
    initial_supply=1000000000000,
    num_decimals=6,
    can_freeze=False,
    can_wipe=True,
    can_pause=False,
    can_change_owner=True,
    can_upgrade=True,
    can_add_special_roles=True
)

tx_hash = entrypoint.send_transaction(tx)
outcome = controller.await_completed_issue_fungible(tx_hash)
print(outcome[0].token_identifier)
```

## Transaction Parsing

### Decoding Transaction Data
```python
from multiversx_sdk import TransactionDecoder

decoded = TransactionDecoder.decode_transaction(tx)
print(decoded.function)
print(decoded.arguments)
```

### Parsing Events
```python
tx_on_network = entrypoint.get_transaction(tx_hash)

for event in tx_on_network.logs.events:
    print(f"Event: {event.identifier}")
    for topic in event.topics:
        print(f"Topic: {topic.hex()}")
```

## Relayed Transactions (V3)

```python
tx = Transaction(
    sender=user_address,
    receiver=receiver_address,
    relayer=relayer_address,
    gas_limit=150000,  # +50k for relayed
    data=b"endpoint@args"
)

tx.signature = user_account.sign_transaction(tx)
tx.relayer_signature = relayer_account.sign_transaction(tx)

entrypoint.send_transaction(tx)
```

## Best Practices

1. **Use `TokenTransfer` helpers**: handles decimals correctly
2. **Check issuance outcome**: tokens get random suffix
3. **Relayed transactions**: sender and relayer must be in same shard
4. **Gas estimation**: Controllers generally handle basic estimation

---
name: mvx_sdk_py_contracts
description: Smart contract operations for MultiversX Python SDK.
---

# MultiversX SDK-Py Smart Contracts

This skill covers ABI loading, deployments, calls, and queries.

## ABI Loading

```python
from multiversx_sdk import Abi
from pathlib import Path

# Load from file
abi = Abi.load("contract.abi.json")
```

## Smart Contract Controller

```python
# Create controller with loaded ABI
controller = entrypoint.create_smart_contract_controller(abi=abi)
```

## Deployment

```python
bytecode = Path("contract.wasm").read_bytes()

tx = controller.create_transaction_for_deploy(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    bytecode=bytecode,
    gas_limit=60000000,
    arguments=[42]  # Typed automatically via ABI
)

tx_hash = entrypoint.send_transaction(tx)
parsed_outcome = controller.await_completed_deploy(tx_hash)
contract_address = parsed_outcome[0].contract_address
```

## Contract Calls

```python
# Call endpoint
tx = controller.create_transaction_for_execute(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    function="add",
    arguments=[10],
    gas_limit=5000000
)

# Call with token transfer (Transfer & Execute)
tx = controller.create_transaction_for_execute(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    function="deposit",
    gas_limit=10000000,
    native_amount=1000000000000000000
)
```

## VM Queries

Read-only calls to the contract state.

```python
# Using controller (parsed output)
result = controller.query_contract(
    contract=contract_address,
    function="getSum",
    arguments=[]
)
print(result[0])  # Typed output
```

## Without ABI (Manual Typing)

If ABI is not available, use manual type hinting (less recommended).

```python
from multiversx_sdk import SmartContractTransactionsFactory

factory = entrypoint.create_smart_contract_transactions_factory()

tx = factory.create_transaction_for_execute(
    sender=account.address,
    contract=contract_address,
    function="add",
    gas_limit=5000000,
    arguments=[42]  # Basic types inferred
)
```

## Upgrades

```python
new_bytecode = Path("contract_v2.wasm").read_bytes()

tx = controller.create_transaction_for_upgrade(
    sender=account,
    nonce=account.get_nonce_then_increment(),
    contract=contract_address,
    bytecode=new_bytecode,
    gas_limit=60000000,
    arguments=[]
)
```

## Best Practices

1. **Always use ABI**: Ensures correct argument encoding/decoding
2. **Query for reads**: Free (no gas), instant response
3. **Wait for completion**: Use controller methods to get parsed results
4. **Gas limit**: Ensure enough gas for computation + storage

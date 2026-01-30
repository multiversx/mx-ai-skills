---
name: mvx_sdk_js_contracts
description: Smart contract operations for MultiversX TypeScript/JavaScript SDK.
---

# MultiversX SDK-JS Smart Contract Operations

This skill covers ABI loading, deployments, calls, queries, and parsing.

## ABI Loading

```typescript
import { Abi } from "@multiversx/sdk-core";
import { promises } from "fs";

// From file
const abiJson = await promises.readFile("contract.abi.json", "utf8");
const abi = Abi.create(JSON.parse(abiJson));

// From URL
const response = await axios.get("https://example.com/contract.abi.json");
const abi = Abi.create(response.data);

// Manual construction
const abi = Abi.create({
    endpoints: [{
        name: "add",
        inputs: [{ type: "BigUint" }],
        outputs: [{ type: "BigUint" }]
    }]
});
```

## Smart Contract Controller

```typescript
const controller = entrypoint.createSmartContractController(abi);
```

## Deployment

```typescript
const bytecode = await promises.readFile("contract.wasm");

const tx = await controller.createTransactionForDeploy(account, nonce, {
    bytecode: bytecode,
    gasLimit: 60_000_000n,
    arguments: [42]  // Constructor args (plain values with ABI)
});

const txHash = await entrypoint.sendTransaction(tx);
const outcome = await controller.awaitCompletedDeploy(txHash);
const contractAddress = outcome[0].contractAddress;
```

## Contract Calls

```typescript
// With ABI - use plain values
const tx = await controller.createTransactionForExecute(account, nonce, {
    contract: contractAddress,
    function: "add",
    arguments: [42],
    gasLimit: 5_000_000n
});

// Without ABI - use TypedValue
import { U32Value, BigUintValue } from "@multiversx/sdk-core";
const tx = await controller.createTransactionForExecute(account, nonce, {
    contract: contractAddress,
    function: "add",
    arguments: [new U32Value(42)],
    gasLimit: 5_000_000n
});
```

## Transfer & Execute (Send tokens to SC)

```typescript
const tx = await controller.createTransactionForExecute(account, nonce, {
    contract: contractAddress,
    function: "deposit",
    arguments: [],
    gasLimit: 10_000_000n,
    nativeTransferAmount: 1_000_000_000_000_000_000n,  // 1 EGLD
    tokenTransfers: [
        TokenTransfer.fungibleFromBigInteger("TOKEN-abc123", 1000n)
    ]
});
```

## VM Queries (Read-Only)

```typescript
const controller = entrypoint.createSmartContractController(abi);

const result = await controller.queryContract({
    contract: contractAddress,
    function: "getSum",
    arguments: []
});

// Parsed output (with ABI)
const sum = result[0];  // Automatically typed
```

## Upgrades

```typescript
const newBytecode = await promises.readFile("contract_v2.wasm");

const tx = await controller.createTransactionForUpgrade(account, nonce, {
    contract: contractAddress,
    bytecode: newBytecode,
    gasLimit: 60_000_000n,
    arguments: []
});
```

## RelayedV3 Contract Calls

```typescript
const tx = await controller.createTransactionForExecute(account, nonce, {
    contract: contractAddress,
    function: "endpoint",
    arguments: [],
    gasLimit: 5_050_000n,  // +50,000 for relayed
    relayer: relayerAddress
});

tx.relayerSignature = await relayer.signTransaction(tx);
```

## Best Practices

1. **Always use ABI** when available for type safety
2. **Gas estimation**: Start high, optimize later
3. **Simulate first**: Use `simulateTransaction()` before real calls
4. **Parse outcomes**: Use `awaitCompletedExecute()` for return values

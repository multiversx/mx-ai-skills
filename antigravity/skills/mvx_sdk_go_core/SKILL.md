---
name: mvx_sdk_go_core
description: Core network operations for MultiversX Go SDK - Proxy, VM Queries.
---

# MultiversX SDK-Go Core Operations

This skill covers the `blockchain` package and network interactions.

## Proxy (Network Client)

Setup the proxy to interact with the blockchain:

```go
import (
    "github.com/multiversx/mx-sdk-go/blockchain"
    "github.com/multiversx/mx-sdk-go/core/http"
)

// Configure Proxy
args := blockchain.ArgsProxy{
    ProxyURL:            "https://devnet-gateway.multiversx.com",
    Client:              http.NewHttpClientWrapper(nil, ""),
    SameScState:         false,
    ShouldBeSynced:      false,
    FinalityCheck:       false,
    CacheExpirationTime: time.Minute,
    EntityType:          core.Proxy,
}

proxy, err := blockchain.NewProxy(args)
```

## Reading Data

```go
// Get Account
address := data.NewAddressFromBech32String("erd1...")
account, err := proxy.GetAccount(ctx, address)
// account.Nonce, account.Balance

// Get Network Config
config, err := proxy.GetNetworkConfig(ctx)

// Get Transaction
tx, err := proxy.GetTransaction(ctx, "txHash", true)
```

## VM Queries (Smart Contract Reads)

Reading state from a smart contract without sending a transaction.

```go
argsQuery := blockchain.ArgsVmQueryGetter{
    Proxy:   proxy,
    Timeout: 10 * time.Second,
}
vmQueryGetter, err := blockchain.NewVmQueryGetter(argsQuery)

// Execute Query
response, err := vmQueryGetter.ExecuteQueryReturningBytes(
    contractAddress,
    "getFunctionName",
    [][]byte{arg1, arg2}, // Arguments as byte slices
)

// response is [][]byte
```

## Shard Coordination

Working with sharded addresses.

```go
// Create Coordinator
coordinator, err := blockchain.NewShardCoordinator(3, 0) // 3 shards, current 0

// Get Shard for Address
shardID := coordinator.ComputeId(address)
```

## Best Practices

1. **Reuse Proxy**: It maintains connection pools/caches
2. **Context**: Always pass context with timeout to avoid hanging
3. **VM Queries**: Preferred for reading state (instant, free)
4. **Error Handling**: Check for specific network errors

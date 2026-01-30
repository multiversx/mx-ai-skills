# MultiversX SDKs Overview

## Core SDKs

### TypeScript/JavaScript (`@multiversx/sdk-core`)
- **Repository**: [mx-sdk-js-core](https://github.com/multiversx/mx-sdk-js-core)
- **Use for**: DApps, scripts, backend integration.
- **Key Classes**:
    - `DevnetEntrypoint`, `MainnetEntrypoint` - Network clients.
    - `Account`, `UserWallet`, `UserPem` - Wallet management.
    - `Transaction`, `TokenTransfer` - Transaction construction.
    - `SmartContractController` - Interaction with SCs.

### Python (`multiversx-sdk`)
- **Repository**: [mx-sdk-py](https://github.com/multiversx/mx-sdk-py)
- **Use for**: Data analysis, scripting, backend processing.
- **Key Modules**:
    - `entrypoints` - Network connection.
    - `core` - Address, Token, Transaction types.
    - `wallet` - PEM/Keystore management.
    - `smart_contracts` - ABI, Deploy, Query.

### Go (`mx-sdk-go`)
- **Repository**: [mx-sdk-go](https://github.com/multiversx/mx-sdk-go)
- **Use for**: High-performance backends, microservices.
- **Key Packages**:
    - `blockchain` - Proxy, ShardCoordinator.
    - `interactors` - Wallet, TxBuilder, NonceHandler.
    - `builders` - Transaction construction.
    - `data` - Core types.

## Frameworks

### React (`@multiversx/sdk-dapp`)
Wraps `sdk-js` with React hooks and UI components (Login, Sign, Toast).

### NestJS (`@multiversx/sdk-nestjs`)
Microservice utilities:
- `ApiNetworkProvider`
- `TransactionProcessor`
- `Caching`

## Tooling

### mxpy (CLI)
Python-based CLI for:
- Building contracts (`mxpy contract build`)
- Deploying (`mxpy contract deploy`)
- Managing wallets (`mxpy wallet new`)

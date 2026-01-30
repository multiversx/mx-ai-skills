---
name: mvx_sdk_go_builders
description: Transaction construction and signing using Builders in Go SDK.
---

# MultiversX SDK-Go Builders

Using `TxBuilder` to construct and sign transactions.

## Transaction Builder

```go
import (
    "github.com/multiversx/mx-sdk-go/builders"
    "github.com/multiversx/mx-sdk-go/core"
    "github.com/multiversx/mx-sdk-go/data"
)

// Initialize
txBuilder, err := builders.NewTxBuilder(core.Marshalizer)

// Manual Transaction Construction
tx := &data.Transaction{
    Nonce:    nonce,
    Value:    "1000000000000000000", // 1 EGLD
    RcvAddr:  receiverAddressString,
    SndAddr:  senderAddressString,
    GasPrice: 1000000000,
    GasLimit: 50000,
    Data:     []byte("memo"),
    ChainID:  "D",
    Version:  1,
}

// Sign Transaction
err = txBuilder.ApplySignature(tx, privateKey)
// tx.Signature, tx.SenderUsername are updated
```

## Contract Builders (High Level)

There are specific builders for SC interactions (often generated code or custom implementations), but the core pattern is:

1. Create `data.Transaction` struct
2. Populate fields (Date = endpoint@arg1@arg2)
3. Sign with `TxBuilder`

## Token Transfers

ESDT transfers require specific data fields.

```go
// Native EGLD
tx.Value = "1000000000000000000"
tx.Data = []byte("")

// ESDT Transfer
// Function: ESDTTransfer
// Args: TokenIdentifier (hex), Amount (hex)
tx.Value = "0"
tx.Data = []byte("ESDTTransfer@" + hexTokenId + "@" + hexAmount)
```

## Relayed V3 Transactions

Constructing the inner transaction for a relayed call.

```go
// Inner Tx (User)
innerTx := &data.Transaction{...}
// Sign Inner Tx
signature, _ := userPrivateKey.Sign(innerTxBytes)

// Relayer Tx
relayerTx := &data.Transaction{
    Sender: relayerAddress,
    Data: []byte("relayedTx@" + innerTxBytesHex + "@" + signatureHex),
    ...
}
```

## Data Serialization

The SDK uses a `Marshalizer` interface (usually JSON).

```go
import "github.com/multiversx/mx-sdk-go/core"

bytes, err := core.Marshalizer.Marshal(tx)
```

## Best Practices

1. **Validate fields** before building
2. **Use correct ChainID** ('1', 'D', 'T')
3. **Handle Big Ints as strings** in `Value` field
4. **Gas Limit estimation** is manual or via proxy simulation

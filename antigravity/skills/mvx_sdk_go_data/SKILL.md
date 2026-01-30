---
name: mvx_sdk_go_data
description: Core data structures and types for MultiversX Go SDK.
---

# MultiversX SDK-Go Data Structures

These types define the core objects interacting with the blockchain.

## Transaction

`data.Transaction` is the main struct for all network interactions.

```go
type Transaction struct {
    Nonce            uint64 `json:"nonce"`
    Value            string `json:"value"`
    RcvAddr          string `json:"receiver"`
    SndAddr          string `json:"sender"`
    GasPrice         uint64 `json:"gasPrice,omitempty"`
    GasLimit         uint64 `json:"gasLimit,omitempty"`
    Data             []byte `json:"data,omitempty"`
    Signature        string `json:"signature,omitempty"`
    ChainID          string `json:"chainID"`
    Version          uint32 `json:"version"`
    Options          uint32 `json:"options,omitempty"`
    Guardian         string `json:"guardian,omitempty"`
    GuardianSignature string `json:"guardianSignature,omitempty"`
    Relayer          string `json:"relayer,omitempty"`
    RelayerSignature string `json:"relayerSignature,omitempty"`
}
```

## Address

```go
type AddressWithType struct {
    address []byte
    hrp     string // "erd" usually
}

// Methods
// NewAddressFromBech32String(bech32)
// NewAddressFromBytes(bytes)
// AddressAsBech32String()
```

## Account

```go
type Account struct {
    Address         string `json:"address"`
    Nonce           uint64 `json:"nonce"`
    Balance         string `json:"balance"` // BigInt string
    Username        string `json:"username"`
    CodeHash        string `json:"codeHash"`
    RootHash        string `json:"rootHash"`
    DeveloperReward string `json:"developerReward"`
}
```

## Network Config

```go
type NetworkConfig struct {
    ChainID                  string
    MinGasLimit              uint64
    MinGasPrice              uint64
    GasPerDataByte           uint64
    GasPriceModifier         float64
    AdditionalGasForTxSize   uint64
    ...
}
```

## VM Query

```go
type VmValueRequest struct {
    Address    string   `json:"scAddress"`
    FuncName   string   `json:"funcName"`
    CallValue  string   `json:"value"`
    CallerAddr string   `json:"caller"`
    Args       []string `json:"args"` // Hex encoded arguments
}

type VmValuesResponseData struct {
    ReturnData      []string `json:"returnData"` // Hex encoded return values
    ReturnCode      string   `json:"returnCode"`
    ReturnMessage   string   `json:"returnMessage"`
    GasRemaining    uint64   `json:"gasRemaining"`
    GasRefund       uint64   `json:"gasRefund"`
    OutputAccounts  map[string]*OutputAccount `json:"outputAccounts"`
}
```

## Best Practices

1. **Use `Value` as string**: To prevent overflow (Go's `int64` is too small for big token amounts).
2. **Bech32 addresses**: API expects bech32 strings, internal logic often uses bytes.
3. **Hex encoding**: `Data` field is often hex-encoded when checking explorers, but SDK expects bytes/string.
4. **Options bitmask**: Used for specialized txs (e.g. guarded).

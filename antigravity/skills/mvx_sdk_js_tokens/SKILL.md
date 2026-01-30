---
name: mvx_sdk_js_tokens
description: Token operations (ESDT/NFT/SFT) for MultiversX TypeScript/JavaScript SDK.
---

# MultiversX SDK-JS Token Operations

This skill covers ESDT (fungible), NFT, and SFT token operations.

## Token Transfers

### Native EGLD Transfer
```typescript
const controller = entrypoint.createTransfersController();

const tx = await controller.createTransactionForTransfer(account, nonce, {
    receiver: Address.newFromBech32("erd1..."),
    nativeAmount: 1000000000000000000n  // 1 EGLD (18 decimals)
});
```

### ESDT Token Transfer
```typescript
import { TokenTransfer } from "@multiversx/sdk-core";

const tx = await controller.createTransactionForTransfer(account, nonce, {
    receiver: receiverAddress,
    tokenTransfers: [
        TokenTransfer.fungibleFromBigInteger("TOKEN-abc123", 1000000n)
    ]
});
```

### NFT/SFT Transfer
```typescript
const tx = await controller.createTransactionForTransfer(account, nonce, {
    receiver: receiverAddress,
    tokenTransfers: [
        TokenTransfer.nftFromBigInteger("NFT-abc123", 1n, 1n)  // nonce=1, quantity=1
    ]
});
```

## Token Issuance

### Issue Fungible Token
```typescript
const controller = entrypoint.createTokenManagementController();

const tx = await controller.createTransactionForIssuingFungible(account, nonce, {
    tokenName: "MyToken",
    tokenTicker: "MTK",
    initialSupply: 1_000_000_000000n,  // With decimals
    numDecimals: 6n,
    canFreeze: false,
    canWipe: true,
    canPause: false,
    canChangeOwner: true,
    canUpgrade: true,
    canAddSpecialRoles: true
});

const txHash = await entrypoint.sendTransaction(tx);
const outcome = await controller.awaitCompletedIssueFungible(txHash);
const tokenIdentifier = outcome[0].tokenIdentifier;
```

### Issue Semi-Fungible Token
```typescript
const tx = await controller.createTransactionForIssuingSemiFungible(account, nonce, {
    tokenName: "MySFT",
    tokenTicker: "SFT",
    canFreeze: false,
    canWipe: true,
    canPause: false,
    canTransferNFTCreateRole: true,
    canChangeOwner: true,
    canUpgrade: true,
    canAddSpecialRoles: true
});
```

### Register and Set NFT Roles
```typescript
const tx = await controller.createTransactionForRegisteringAndSettingRoles(account, nonce, {
    tokenName: "MyNFT",
    tokenTicker: "MNFT",
    tokenType: "NonFungibleESDT"
});
```

## Token Role Management

```typescript
// Set special role
const tx = await controller.createTransactionForSettingSpecialRole(account, nonce, {
    tokenIdentifier: "TOKEN-abc123",
    user: userAddress,
    addRoleLocalMint: true,
    addRoleLocalBurn: true
});
```

## Token Queries

```typescript
const provider = entrypoint.createNetworkProvider();

// Get specific token
const token = await provider.getTokenOfAccount(address, "TOKEN-abc123");

// Get all fungible tokens
const tokens = await provider.getFungibleTokensOfAccount(address);

// Get all NFTs
const nfts = await provider.getNonFungibleTokensOfAccount(address);
```

## Best Practices

1. **Decimals**: EGLD has 18 decimals, custom tokens vary
2. **Token identifiers**: Format is `TICKER-hexhash` (e.g., `USDC-a1b2c3`)
3. **NFT nonces**: Start at 1, increment per mint
4. **Gas**: Token operations need ~60,000,000+ gas

---
name: mvx_sdk_js_wallets
description: Wallet and key management for MultiversX TypeScript/JavaScript SDK.
---

# MultiversX SDK-JS Wallets & Signing

This skill covers account management, key derivation, and transaction signing.

## Account Creation

```typescript
import { Account, Address, Mnemonic, UserWallet, UserPem } from "@multiversx/sdk-core";

// From PEM file (testing only - not secure)
const account = await Account.newFromPem("wallet.pem");

// From keystore (production)
const account = await Account.newFromKeystore("wallet.json", "password");

// From entrypoint (generates new)
const account = entrypoint.createAccount();
```

## Mnemonic Operations

```typescript
// Generate new mnemonic (24 words)
const mnemonic = Mnemonic.generate();
const words = mnemonic.getWords();

// Derive keys
const secretKey = mnemonic.deriveKey(0);  // First account
const publicKey = secretKey.generatePublicKey();
const address = publicKey.toAddress();
```

## Wallet Storage

```typescript
// Save mnemonic to keystore
const wallet = UserWallet.fromMnemonic({
    mnemonic: mnemonic.toString(),
    password: "secure-password"
});
wallet.save("wallet.json");

// Save secret key to keystore
const wallet = UserWallet.fromSecretKey({
    secretKey: secretKey,
    password: "secure-password"
});
wallet.save("wallet.json");

// Save to PEM (testing only)
const pem = new UserPem(address.toBech32(), secretKey);
pem.save("wallet.pem");
```

## Transaction Signing

```typescript
// Controller-created transactions are auto-signed
const tx = await controller.createTransactionForTransfer(account, nonce, options);
// tx.signature is already set

// Factory-created transactions need manual signing
const tx = await factory.createTransactionForTransfer(sender, options);
tx.nonce = account.getNonceThenIncrement();
tx.signature = await account.signTransaction(tx);
```

## Message Signing

```typescript
// Sign arbitrary message
const message = new SignableMessage({ message: Buffer.from("Hello") });
const signature = await account.signMessage(message);

// Verify signature
import { MessageSigner } from "@multiversx/sdk-core";
const isValid = MessageSigner.verify(message, signature, publicKey);
```

## Ledger Device

```typescript
// Ledger requires MultiversX app installed
import { LedgerAccount } from "@multiversx/sdk-hw-provider";

const ledger = new LedgerAccount();
await ledger.init();
const address = await ledger.getAddress(0);
```

## Security Best Practices

| Practice | Reason |
|----------|--------|
| Use keystore files | Encrypted with password |
| Never commit PEM files | Plain text private keys |
| Use Ledger for mainnet | Hardware security |
| Validate addresses | Prevent typos |

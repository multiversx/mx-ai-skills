---
description: Workflow for designing and issuing Advanced ESDT/SFT/NFT tokens.
---

# MultiversX Token Architect

Guide to designing tokens using the "Tokenization of Everything" philosophy, including Dynamic ESDT and SFT data containers.

## 1. Token Type Selection

### Fungible (ESDT)

- **Use Case**: Currencies, Governance Tokens, Stablecoins.
- **Key Config**: `canUpgrade` (allows changing properties later), `canFreeze`, `canWipe`.

### Semi-Fungible (SFT)

- **Use Case**: Tickets, Coupons, In-Game Items (Stackable).
- **Key Feature**:
  - **Nonce**: ID > 0.
  - **Balance**: User can hold multiple units of Nonce X.
  - **Attributes**: Shared metadata for Nonce X.

### Non-Fungible (NFT)

- **Use Case**: Profile Pics, Unique Art, Real Estate.
- **Key Feature**: Balance is always 1 per Nonce.

## 2. Dynamic Features Design

Will your token need to evolve? (e.g., RPG Character Level Up).

- **Dynamic**: Yes.
- **Requirement**:
  - Manage Role: `ESDTRoleNFTUpdate`.
  - Property: `canUpgrade` = true.
- **Flow**:
    1. Smart Contract issues the token.
    2. SC assigns `ESDTRoleNFTUpdate` to itself.
    3. SC calls `ESDTNFTUpdateAttributes` to change the data inside the SFT.

## 3. Issuance Workflow

// turbo

1. Issue the Token Identifier (1000 EGLD cost roughly, less on Testnet).

```bash
mxpy tx sc call --proxy https://devnet-gateway.multiversx.com --chain D \
  --recall-nonce --pem wallet.pem --gas-limit 60000000 \
  --function issue \
  --arguments str:MyToken str:TKN 18 \
  --address erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqzllls8a5w6u
```

## 4. Role Assignment

Crucial for minting/burning/updating.

- **Roles**: `ESDTRoleLocalMint`, `ESDTRoleLocalBurn`, `ESDTRoleNFTCreate`, `ESDTRoleNFTUpdate`.

---
name: mvx-token-management
description: Design, issue, inspect, and troubleshoot ESDT tokens on MultiversX. Use when working with fungible tokens, NFTs, SFTs, or Meta-ESDTs.
---

# MultiversX Token Management

Architecture-first workflow for MultiversX tokens: design the right token type, issue it, assign roles, inspect on-chain state, and troubleshoot common problems.

## Phase 1: Token Architecture & Design

Before issuing, select the correct token type for the use case.

### 1.1 Token Type Selection

| Type | Use Cases | Key Characteristics |
|------|-----------|---------------------|
| **Fungible (ESDT)** | Currencies, governance tokens, stablecoins | Single denomination, divisible by decimals |
| **Semi-Fungible (SFT)** | Tickets, coupons, in-game items (stackable) | Nonce > 0, users can hold multiple units of same nonce, shared attributes per nonce |
| **Non-Fungible (NFT)** | Profile pics, unique art, real estate | Balance is always 1 per nonce, unique attributes |
| **Meta-ESDT** | Wrapped tokens, LP tokens, complex financial instruments | Like SFT but with decimals, bridging fungible and non-fungible properties |

### 1.2 Property Design

Decide on token properties at issuance time:

| Property | Purpose | Recommendation |
|----------|---------|----------------|
| `canUpgrade` | Allows changing properties later | Set `true` unless token must be fully immutable |
| `canMint` | Allows minting new supply | `true` if supply is not fixed |
| `canBurn` | Allows burning supply | Usually `true` |
| `canPause` | Allows pausing all transfers | Emergency mechanism, recommended for DeFi |
| `canFreeze` | Allows freezing specific accounts | Compliance requirement for regulated tokens |
| `canWipe` | Allows wiping frozen accounts | Compliance, requires `canFreeze` |
| `canChangeOwner` | Allows transferring token ownership | `true` for flexibility |
| `canAddSpecialRoles` | Allows assigning roles after issuance | Almost always `true` |

### 1.3 Dynamic Features (Evolving Tokens)

If the token needs to change after creation (e.g., RPG character leveling, dynamic metadata):

- **Requirement**: `ESDTRoleNFTUpdate` role + `canUpgrade` = true
- **Flow**:
  1. Smart contract issues the token
  2. SC assigns `ESDTRoleNFTUpdate` to itself
  3. SC calls `ESDTNFTUpdateAttributes` to change data inside the NFT/SFT

## Phase 2: Token Issuance

### 2.1 Direct Issuance (From Wallet)

All issuance transactions go to the system SC at `erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqslllllls3xelgl` with a 0.05 EGLD issuance fee.

**Using mxpy**:
```bash
# Fungible token
mxpy tx sc call --proxy https://gateway.multiversx.com --chain 1 \
  --recall-nonce --pem wallet.pem --gas-limit 60000000 \
  --function issue \
  --arguments str:MyToken str:TKN 1000000000000000000000 18 \
  --value 50000000000000000 \
  --address erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqslllllls3xelgl
```

Replace gateway URL and chain ID for devnet (`D`) or testnet (`T`).

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_token_issue_fungible`, `mvx_token_issue_nft`, `mvx_token_issue_sft`, or `mvx_token_issue_meta_esdt` for direct wallet-based issuance. Use `mvx_token_create_nft` to mint a new nonce under an existing collection.

### 2.2 Issuance From Smart Contracts

When issuing from a smart contract (not a wallet), the flow uses async system SC calls:

- **Functions**: `issue` (fungible), `issueNonFungible`, `issueSemiFungible`, or `registerMetaESDT`
- **Cost**: 0.05 EGLD (issuance fee, sent as value)
- **Arguments**: token name, ticker, initial supply (fungible), decimals, and optional properties

After issuance, the token identifier is returned asynchronously in a callback. The contract must handle this via `#[callback]`:

```rust
#[callback]
fn issue_callback(
    &self,
    #[call_result] result: ManagedAsyncCallResult<TokenIdentifier>,
) {
    match result {
        ManagedAsyncCallResult::Ok(token_id) => {
            self.token_id().set(&token_id);
        },
        ManagedAsyncCallResult::Err(_) => {
            // Handle failure
        },
    }
}
```

## Phase 3: Role Assignment

After issuance, assign special roles via the system SC.

### Available Roles

| Role | Purpose |
|------|---------|
| `ESDTRoleLocalMint` | Can mint new supply |
| `ESDTRoleLocalBurn` | Can burn supply |
| `ESDTRoleNFTCreate` | Can create NFT/SFT nonces |
| `ESDTRoleNFTBurn` | Can burn NFT/SFT nonces |
| `ESDTRoleNFTAddQuantity` | Can add SFT quantity |
| `ESDTRoleNFTUpdate` | Can update NFT/SFT attributes |
| `ESDTTransferRole` | Restricts who can send the token |

### Setting Roles

**From a wallet (mxpy)**:
```bash
mxpy tx sc call --proxy https://gateway.multiversx.com --chain 1 \
  --recall-nonce --pem wallet.pem --gas-limit 60000000 \
  --function setSpecialRole \
  --arguments str:TOKEN-abcdef address:erd1... str:ESDTRoleLocalMint \
  --address erd1qqqqqqqqqqqqqqqpqqqqqqqqqqqqqqqqqqqqqqqqqqqqqslllllls3xelgl
```

**From a smart contract**: This is also an async call requiring a callback. Typical flow:
1. Contract calls `issue` on system SC (async)
2. Callback stores the received token identifier
3. Contract calls `setSpecialRole` on system SC (async)
4. Callback confirms roles are set

## Phase 4: Token Inspection

### 4.1 Query Token Info

Inspect any token via the API:
```
GET https://api.multiversx.com/tokens/{tokenIdentifier}
```

Check:
- Token name, ticker, identifier
- Token type (Fungible, SemiFungible, NonFungible, Meta)
- Decimals and initial supply
- Owner address
- Properties: canMint, canBurn, canPause, canFreeze, canWipe, canChangeOwner, canUpgrade, canAddSpecialRoles
- Minted and burnt supply

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_token_info` for this step.

### 4.2 Check Token Roles

Verify which addresses hold special roles. Only expected addresses (your contracts) should hold sensitive roles like Mint and NFTCreate.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_token_info` to see the full roles map.

### 4.3 Verify Token in Contract Context

When auditing or testing a contract that manages tokens:
1. Query the contract's view endpoints to read token identifiers stored in state
2. Inspect each token's properties and roles
3. Confirm the contract address holds the required roles (Mint, Burn, etc.)
4. Check that token supply matches expected values

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` to read token IDs from the contract, then `mvx_token_info` to verify each token.

## Phase 5: Troubleshooting

Common token issues and how to diagnose them:

| Problem | Cause | How to Check |
|---------|-------|-------------|
| Cannot mint | Contract address missing `ESDTRoleLocalMint` role | Inspect token roles for the contract address |
| Cannot transfer | Token has `ESDTTransferRole` set and sender is not in the role list | Check if transfer role is set and who holds it |
| Token paused | `isPaused` is true | Check token properties, find who can unpause |
| Wrong decimals | Decimals mismatch between contract logic and token config | Verify decimals match expected values (18 for EGLD-like, 6 for USDC-like) |
| NFT create fails | Missing `ESDTRoleNFTCreate` role | Verify role assignment on the creating address |
| Token not found | Wrong identifier or not yet issued | Check transaction that issued the token, verify callback stored the ID |

## Next Steps

- Use the **mvx-deploy-flow** skill to deploy contracts that manage tokens
- Use the **mvx-test-contract** skill to verify token operations work correctly
- Use the **mvx-debug-tx** skill to investigate failed token transactions

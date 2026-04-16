---
name: mvx-upgrade-flow
description: Upgrade a deployed MultiversX smart contract with pre/post state verification and rollback guidance. Use when upgrading contract code on devnet, testnet, or mainnet.
---

# MultiversX Smart Contract Upgrade Flow

Guide through a safe, verified upgrade workflow with pre/post state snapshots, diff review, and rollback guidance.

## Step 1: Pre-Flight Checks

Capture the current contract state so you can verify nothing breaks after the upgrade.

### 1.1 Account State

Query the contract account via the API:
```
GET https://api.multiversx.com/accounts/{address}
```

Record:
- Owner address
- EGLD balance
- Properties (upgradeable, payable, payableBySmartContract, readable)
- Code hash (this WILL change after upgrade)
- Verification status

**Gate**: If the contract is NOT upgradeable, STOP. The upgrade will fail.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` for this step.

### 1.2 Snapshot All Views ("Before")

Fetch the current on-chain ABI and call every view endpoint. Record results as the "before" snapshot:

| View | Before Value |
|------|-------------|
| ... | ... |

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` to get the ABI and `mvx_sc_query` for each view.

### 1.3 Snapshot Critical Storage ("Before")

Read key storage values and record them as the "before" snapshot.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_storage_keys` and `mvx_sc_storage` for this step.

### 1.4 New ABI Review

Read the new ABI file. Compare with the current on-chain ABI:
- **New endpoints**: List any endpoints added
- **Removed endpoints**: List any endpoints that no longer exist. WARNING: if external contracts call removed endpoints, they will break
- **Changed signatures**: Parameters or return types changed
- **Upgrade parameters**: Does the new `#[upgrade]` function expect arguments?

## Step 2: MAINNET SAFETY

**If deploying to mainnet**:

STOP. Do NOT proceed automatically. Display this message and wait for explicit confirmation:

> You are about to upgrade a MAINNET contract at `{address}`. This is irreversible -- the old code will be permanently replaced. If the new code has bugs, user funds may be at risk.
>
> Are you absolutely sure? Type the contract address to confirm.

Do NOT proceed until the user types the contract address back.

**If deploying to testnet or devnet**, note the safety context but proceed.

## Step 3: Diff Review

Summarize what is changing:
- New endpoints being added
- Endpoints being removed (breaking change!)
- Storage layout changes (new mappers, changed types)
- Upgrade function logic (does it run migration code?)
- Code metadata changes (payable, readable flags)

If any removed endpoints are called by other contracts, flag as **HIGH RISK**.

## Step 4: Simulate Upgrade

If possible, dry-run the upgrade transaction:
- Verify it does not revert
- Check gas consumption
- Note: simulation may not be available for upgrades on all networks

**Using mx-chain-simulator-go**: Replay the upgrade against a fork of the network state for full visibility.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_simulate` for dry-run and `mvx_sc_estimate_gas` for gas estimation.

## Step 5: Execute Upgrade

**Preserve the current WASM**: Always keep a copy of the current WASM before upgrading, in case rollback is needed.

**Using mxpy**:
```bash
mxpy contract upgrade {address} \
  --proxy https://gateway.multiversx.com --chain 1 \
  --recall-nonce --pem wallet.pem \
  --gas-limit 100000000 \
  --bytecode output/contract.wasm \
  --arguments [upgrade_args_if_any] \
  --metadata-upgradeable --metadata-readable
```

Replace gateway URL and chain ID for devnet (`D`) or testnet (`T`).

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_upgrade` with the address, wasmPath, network, upgrade arguments, gas limit, and code metadata flags.

Record the upgrade transaction hash. Verify:
- Transaction status is "success"
- No error messages
- Gas consumed was within limits

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_tx_result` with the transaction hash.

## Step 6: Post-Upgrade Verification

### 6.1 Account Verification

Re-query the contract account:
- Confirm code hash has CHANGED (proves new code is deployed)
- Owner is unchanged
- EGLD balance is unchanged (minus gas)
- Properties match expectations

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` for this step.

### 6.2 ABI Verification

Fetch the on-chain ABI again:
- Confirm all new endpoints are present
- Confirm no unexpected endpoints are missing

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` for this step.

### 6.3 View Comparison ("Before" vs "After")

For **every** view endpoint, query again and compare with the "before" snapshot:

| View | Before | After | Status |
|------|--------|-------|--------|
| ... | ... | ... | OK / Changed / Error |

- **OK**: Value unchanged (expected for most views)
- **Changed**: Value changed -- is this expected from the upgrade logic?
- **Error**: View fails -- this is a CRITICAL issue

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` for each view.

### 6.4 Storage Verification

Re-read critical storage keys:
- Compare with "before" snapshot
- Verify no storage corruption
- Check that any new storage keys from the upgrade are initialized

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_storage` for this step.

## Step 7: Explorer Verification (Optional)

Submit the upgraded contract for source code verification on the explorer:
- Provide the `.source.json` file from the reproducible build and the Docker image tag
- Confirm "verified" status on the explorer

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_verify` to submit and `mvx_sc_verify_status` to poll until verification completes.

## Rollback Guidance

If something went wrong after upgrade:
- **The old code is gone**, but you can upgrade AGAIN with the previous WASM to restore the old code
- If you have the previous WASM file, run the upgrade again with the old WASM
- If you do NOT have the previous WASM, check if the contract was verified on the explorer before upgrade -- the source may be recoverable
- After rollback, re-run Step 6 verification to confirm state is intact

**Prevention**: Always keep a copy of the current WASM before upgrading.

## Upgrade Report

| Property | Value |
|----------|-------|
| Network | devnet / testnet / mainnet |
| Contract | [address] |
| Upgrade TX | [hash] |
| Old Code Hash | [from pre-flight] |
| New Code Hash | [from post-upgrade] |
| Views Changed | [count] |
| Views Broken | [count] |
| Storage Intact | Y/N |
| Verified | Y/N |

## Next Steps

- Run the **mvx-test-contract** skill to verify the upgraded contract works correctly
- If issues found, use the **mvx-debug-tx** skill to investigate failed transactions
- For security review, run the **mvx-audit-onchain** skill on the upgraded contract

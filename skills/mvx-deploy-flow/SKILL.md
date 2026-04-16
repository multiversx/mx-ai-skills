---
name: mvx-deploy-flow
description: Deploy MultiversX smart contracts with pre-flight checks, verification, and post-deployment validation. Use when deploying a new contract to devnet, testnet, or mainnet.
---

# MultiversX Smart Contract Deployment Flow

Guide through a safe, verified deployment workflow: build, verify artifacts, deploy, validate state, and verify source on the explorer.

## Pre-Flight Checks

### Step 1: Reproducible Build

Build the contract using the Docker-based reproducible build to ensure deterministic WASM output:

```bash
sc-meta all build --locked
```

Or with Docker for reproducible builds:
```bash
mxpy contract build --image=multiversx/sdk-rust-contract-builder:v11.0.0
```

**Verification**: The `code-hash` must match between local and Docker builds.

**Artifacts to preserve**: Store `.wasm`, `.abi.json`, and `.source.json` (from `output-docker/`) in a versioned registry or commit them to the repository.

### Step 2: Verify Build Artifacts

- Read the ABI file to understand the contract interface
- Extract from the ABI:
  - Constructor (`init`) parameters and their types
  - All endpoints (count mutable vs readonly)
  - Whether the contract is upgradeable, payable, payableBySmartContract
  - Build info if present (framework version, compiler)
- Verify the WASM file exists and note its size. WASM files larger than 400KB may hit deployment gas limits.

### Step 3: Wallet Balance Check

Ensure sufficient EGLD for deployment gas:
- Typical cost: 0.1-0.5 EGLD for most contracts, up to 1+ EGLD for large contracts
- On testnet/devnet, use the faucet if balance is low

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` to check the deployer wallet balance.

### Step 4: Review Constructor Arguments

From the ABI, list every constructor parameter:

| Parameter | Type | Description |
|-----------|------|-------------|
| ... | ... | ... |

Validate types match expectations:
- Addresses should be valid bech32 (`erd1...`)
- Token identifiers should match the pattern `[A-Z]+-[a-f0-9]+`
- Numeric values should be within reasonable ranges
- BigUint values need proper denomination handling (e.g., 1 EGLD = 1000000000000000000)

### Step 5: Gas Estimation

Estimate deployment gas based on WASM size:
- < 100KB: ~60,000,000 gas
- 100-200KB: ~100,000,000 gas
- 200-400KB: ~150,000,000 gas
- > 400KB: ~200,000,000+ gas (warn about potential issues)

### Step 6: Security Checklist (Pre-Deploy)

Before deploying, verify:
- **Payable**: `#[payable("EGLD")]` is NOT set on endpoints unless strictly required
- **Upgradeable**: Is the contract meant to be immutable? If so, remove the upgrade code metadata flag
- **Owner**: Is the deployer a multi-sig or guardian wallet? For mainnet, consider having CI generate the tx payload and letting the multi-sig sign it
- **Code metadata**: Review flags -- upgradeable, payable, readable -- match the intended security posture

## Deployment

### Step 7: Deploy the Contract

**MAINNET SAFETY**: If deploying to mainnet, STOP and confirm before proceeding:
> You are about to deploy a new contract on MAINNET. This will use real funds for gas. Do you want to proceed?

**Using mxpy**:
```bash
mxpy contract deploy \
  --proxy https://gateway.multiversx.com --chain 1 \
  --recall-nonce --pem wallet.pem \
  --gas-limit 100000000 \
  --bytecode output/contract.wasm \
  --arguments [constructor_args] \
  --metadata-upgradeable --metadata-readable
```

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_deploy` with the wasmPath, network, constructor arguments, gas limit, and code metadata flags.

Record the deployment transaction hash.

### Step 8: Verify Deployment Transaction

After the deploy transaction completes, verify:
- Transaction status is "success"
- Smart contract result contains the new contract address
- No unexpected error messages
- Gas consumed vs gas limit (was it close to the limit?)

Extract the **new contract address** from the transaction results.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_tx_result` with the deployment transaction hash.

### Step 9: Confirm Contract Exists

Verify the deployed contract:
- The contract exists at the address
- The owner matches the deployer wallet
- The code hash is populated
- Properties match expectations (upgradeable, payable, etc.)

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` with the new contract address.

## Post-Deployment Validation

### Step 10: ABI Verification

Confirm the on-chain ABI matches the local ABI:
- All expected endpoints are present
- Constructor parameters were recorded correctly

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` with the new contract address.

### Step 11: Test Views

For each **view** endpoint in the ABI, call it and verify initial state:
- Configuration values match what was passed to the constructor
- Counters start at expected initial values (0 or 1)
- Token identifiers are set correctly
- No unexpected empty or zero values

| View | Expected | Actual | Status |
|------|----------|--------|--------|
| ... | ... | ... | OK / Mismatch |

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` for each view endpoint.

### Step 12: Storage Verification

Inspect contract storage:
- Confirm storage keys were initialized by the constructor
- Spot-check key values
- Verify no unexpected storage keys exist

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_storage_keys` and `mvx_sc_storage`.

### Step 13: Simulate (Optional)

If you have access to `mx-chain-simulator-go`:
- Dry-run key transactions against a fork of the network state
- Verify the contract behaves as expected before real users interact with it
- Benefit: unlimited logs, no gas cost

### Step 14: Explorer Verification

Submit the contract for source code verification on the explorer:
- Provide the `.source.json` file from the reproducible build (e.g., `output-docker/contract/contract-0.0.0.source.json`)
- Provide the Docker image tag used for the build (e.g., `multiversx/sdk-rust-contract-builder:v11.0.0`)
- Confirm the contract shows as "verified" on the explorer

If verification fails:
- Check that the local build is reproducible (`sc-meta all build` produces the same WASM hash)
- Ensure the framework version matches what the explorer expects
- Retry with correct parameters

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_verify` to submit and `mvx_sc_verify_status` to poll until verification completes.

## Deployment Report

Generate a summary report:

| Property | Value |
|----------|-------|
| Network | devnet / testnet / mainnet |
| Contract Address | [new address] |
| Deploy TX Hash | [hash] |
| Owner | [deployer address] |
| WASM Size | [size] |
| Gas Used | [amount] / [limit] |
| Upgradeable | Y/N |
| Payable | Y/N |
| Verified | Y/N |

### Action Items Checklist
- [ ] WASM and ABI verified (reproducible build)
- [ ] Wallet balance sufficient
- [ ] Constructor arguments validated
- [ ] Deployment transaction successful
- [ ] Contract address confirmed
- [ ] Views return expected initial state
- [ ] Storage keys initialized correctly
- [ ] Source code verified on explorer
- [ ] Contract ready for use

### Infrastructure Notes

- **Observer node**: For mainnet, consider running your own Observer node for reliable broadcasting during congestion
- **Monitoring**: Set up alerts for contract balance changes and unexpected transactions

## Next Steps

- If owner-only setup endpoints exist (e.g., `registerToken`, `setConfig`), call them now
- If the contract needs ESDT roles, assign them using the **mvx-token-management** skill
- Run the **mvx-test-contract** skill to verify the deployment works correctly
- If issues found, use the **mvx-debug-tx** skill to investigate failed transactions
- For security review, run the **mvx-audit-onchain** skill on the deployed contract

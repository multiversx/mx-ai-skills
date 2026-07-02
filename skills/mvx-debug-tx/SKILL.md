---
name: mvx-debug-tx
description: Debug and analyze MultiversX transactions. Use when a transaction failed, returned unexpected results, or needs full decoding of inputs/outputs/events.
---

# MultiversX Transaction Debugger

Analyze a transaction to determine what happened, why it failed (if applicable), decode all smart contract outputs, and explain every event. Combines on-chain decoding with local simulation techniques.

## Phase 1: Fetch Transaction Data

### 1.1 Get Full Transaction Results

Retrieve the complete transaction data from the MultiversX API.

**Manual approach**: Query the API directly:
```
GET https://gateway.multiversx.com/transaction/{txHash}?withResults=true
```
Or use the explorer: `https://explorer.multiversx.com/transactions/{txHash}`

For devnet/testnet, replace the base URL accordingly (`devnet-api.multiversx.com`, `testnet-api.multiversx.com`).

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_tx_result` for this step.

Extract and record:
- **Status**: success, fail, or pending
- **Sender**: who initiated the transaction
- **Receiver**: the target address (contract or EOA)
- **Value**: EGLD amount sent (if any)
- **Data field**: the raw transaction data (function call + arguments)
- **Gas limit**: gas allocated
- **Gas used**: gas actually consumed
- **Gas refund**: gas returned to sender
- **Timestamp**: when the transaction was processed
- **Block / Nonce**: transaction ordering info
- **Miniblock hash**: for cross-shard tracking

### 1.2 Decode the Data Field

The transaction data field contains the function call. Parse it:
- First component (before `@`): function name (plain text ASCII)
- Subsequent `@`-separated components: arguments (hex-encoded)

For each argument, determine:

| Position | Hex Value | Decoded Value | Likely Type |
|----------|-----------|---------------|-------------|
| arg0 | ... | ... | Address / BigUint / TokenIdentifier / ... |

**Decoding tips**:
- Addresses: 64 hex chars, decode to bech32 `erd1...` format
- BigUint: variable-length hex, convert to decimal
- TokenIdentifier: hex-decode to ASCII string (e.g., `4d45582d343535633537` = `MEX-455c57`)
- Nested/complex types: refer to the contract ABI for struct layouts

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_decode` to decode complex argument types (structs, enums, nested types).

### 1.3 Identify Token Transfers

If the transaction includes ESDT transfers (ESDTTransfer, ESDTNFTTransfer, MultiESDTNFTTransfer):
- Decode the token identifier, nonce (for NFTs/SFTs), and amount
- Note: MultiESDTNFTTransfer has a different argument layout (destination first, then count, then token triplets)

## Phase 2: Analyze Results

### 2.1 Smart Contract Results (SCRs)

For each smart contract result in the transaction:
- **Direction**: which contract generated it, who receives it
- **Data**: decode the return data
  - `@6f6b` = `@ok` (success)
  - `@` followed by error code = failure
  - Multiple `@`-separated values = multi-return
- **Value / Tokens**: any EGLD or ESDT transfers in the result
- **Gas**: gas forwarded and consumed

Build a call trace showing the execution flow:

```
1. Sender -> Contract A: functionName(args...)
   1.1 Contract A -> Contract B: innerCall(args...)
       Result: [decoded return value]
   1.2 Contract A -> Sender: [token transfer / refund]
   Result: [ok / error]
```

### 2.2 Decode Return Values

If the transaction succeeded and returned data:
- Use the contract ABI to match return types to the endpoint signature
- Decode multi-value returns by splitting on `@` separators

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` on the receiver address to fetch the ABI, and `mvx_sc_decode` to decode return values.

### 2.3 Failure Analysis

If the transaction failed, identify the root cause.

**Common failure patterns**:

| Error Message | Meaning | Likely Cause |
|---------------|---------|--------------|
| `execution failed` | SC logic error | require/sc_panic triggered |
| `out of gas` | Insufficient gas | Complex operation or gas limit too low |
| `user error` | Input validation | Wrong arguments, wrong token, insufficient balance |
| `signal error` | SC explicitly signaled | Business logic rejection |
| `insufficient funds` | Not enough EGLD/tokens | Sender balance too low |
| `invalid arguments` | Wrong arg count/type | Mismatched function signature |
| `action is not allowed` | Access control | Caller is not owner/admin |
| `contract not found` | Target not deployed | Wrong address or not yet deployed |
| `too much gas` | Gas limit exceeds block limit | Reduce gas limit |

**Common error codes**:
- `10`: Execution Failed
- `6`: User Rejected
- `4`: Invalid Balance

For SC errors, decode the error message from the result data:
- Strip the `@` prefix and error code
- Hex-decode the message portion
- Match against known error strings in the contract

If the error is not immediately clear:
- Fetch the contract ABI to understand the endpoint signature
- Query the contract state to check if a precondition is not met
- Verify sender and contract balances

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` to inspect the endpoint, `mvx_sc_query` to check state, and `mvx_account` to verify balances.

## Phase 3: Event Analysis

### 3.1 Decode Events (Logs)

For each event/log entry in the transaction:
- **Identifier**: the event name (first topic)
- **Address**: which contract emitted it
- **Topics**: indexed parameters (hex-encoded, decode each)
- **Data**: non-indexed parameters (hex-encoded, decode)

If the contract ABI is available, match events to their definitions to decode topic and data types correctly.

### 3.2 Standard Events

Recognize standard MultiversX events:
- `ESDTTransfer`, `ESDTNFTTransfer`, `MultiESDTNFTTransfer` -- token movements
- `ESDTLocalMint` / `ESDTLocalBurn` -- supply changes
- `ESDTNFTCreate` / `ESDTNFTBurn` -- NFT lifecycle
- `writeLog` -- generic logs
- `completedTxEvent` -- async completion
- `SCDeploy` -- deployment
- `signalError` -- explicit errors

### 3.3 Event Timeline

Reconstruct the chronological order of events to tell the full story of the transaction execution.

## Phase 4: Cross-Shard Analysis

If the transaction involves cross-shard communication:
- Identify source shard and destination shard
- Track the SCR that crosses shards
- Note: cross-shard transactions have two miniblocks -- one in each shard
- Callback results appear in a separate SCR back to the source shard
- If a cross-shard call fails, the callback receives the error -- check if the callback handles it properly

## Phase 5: Local Simulation (Advanced)

When on-chain data is insufficient, reproduce the issue locally.

### 5.1 Chain Simulator

- **Repo**: `mx-chain-simulator-go`
- **Action**: `POST /simulator/set-state` to replicate the production state locally
- **Run**: Replay the failing transaction against the simulator
- **Benefit**: Unlimited logs, no gas cost

### 5.2 Debug Printing (Simulator/Testnet Only)

- Add `sc_print!("Balance: {}", my_balance);` to the contract code
- Rebuild and deploy to the simulator
- Check node logs or simulator output for the printed values

### 5.3 Minimal Reproduction

Create a Mandos/scenario test case (`repro.scen.json`) that reproduces the exact failure with minimal setup. This serves as both a debugging aid and a regression test.

## Output: Debug Report

Produce a report with these sections:
1. **Transaction Overview**: Table with hash, network, status, sender, receiver, function, value, token transfers, gas used/limit (with percentage), timestamp.
2. **Decoded Inputs**: Table of each argument with position, type, and decoded value.
3. **Execution Trace**: Indented call tree showing the flow of calls between contracts and their results.
4. **Decoded Outputs**: Table of return values with position, type, and decoded value.
5. **Events**: Table of each event with identifier, emitter address, and decoded data.
6. **Failure Diagnosis** (if failed): Root cause (one line), error code (hex + decoded), error message, which step in the trace failed.
7. **Explanation**: Plain-English narrative of what the transaction did or tried to do. If failed, explain exactly why and what conditions were not met. If succeeded, summarize the net effect.
8. **Recommendations** (if failed): What the user should do -- correct arguments, gas adjustments, state changes required before retrying.

## Next Steps

- After identifying the issue, fix the code and use the **mvx-upgrade-flow** skill to deploy the fix
- Run the **mvx-test-contract** skill to verify the fix works
- Create a Mandos scenario test to prevent regression

---
name: mvx-test-contract
description: Perform automated on-chain testing of a deployed MultiversX smart contract. Use when validating a contract's state, views, endpoints, and overall health after deployment or upgrade.
---

# MultiversX Smart Contract On-Chain Testing

Perform an automated, comprehensive test of a deployed contract: discover its interface, interrogate all state, simulate endpoints, and produce a health report.

## Phase 1: Discovery

### 1.1 Account Inspection

Gather contract account information:
```
GET https://api.multiversx.com/accounts/{address}
```

Record:
- Owner address
- EGLD balance held by the contract
- Properties: upgradeable, payable, payableBySmartContract, readable
- Code hash and deployment transaction
- Verification status

Flag anything unusual (e.g., large EGLD balance with no clear purpose, upgradeable without owner verification).

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` for this step.

### 1.2 ABI Discovery

Retrieve the full contract interface from the ABI. Categorize all endpoints:

- **Views** (readonly endpoints): These will be queried exhaustively
- **Public mutable endpoints**: These will be simulated
- **Owner-only endpoints**: Document but do not call
- **Payable endpoints**: Note which tokens are accepted
- **Constructor / upgrade parameters**: Document the expected init args
- **Custom types**: Structs, enums -- needed for decoding results
- **Events**: What the contract emits

Build a complete endpoint inventory table:

| Endpoint | Mutability | Access | Payable | Parameters | Return Type |
|----------|-----------|--------|---------|------------|-------------|

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` for this step.

### 1.3 Token Discovery

From the ABI, identify view endpoints that return token identifiers (e.g., `getTokenId`, `getLpTokenIdentifier`, `getRewardTokenId`). Query each one to discover all tokens managed by the contract.

For each token found, verify its properties: decimals, supply, roles, owner.

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` to read token IDs and `mvx_token_info` to inspect each token.

## Phase 2: State Interrogation

### 2.1 Query All Views

For **every** view endpoint discovered in Phase 1:
- Call it with appropriate arguments
- Record every result in a structured table
- Flag unexpected values: zero balances where non-zero expected, empty collections, max uint values

For views that require arguments:
- If the argument is an address -- try the contract owner or deployer address
- If the argument is a token ID -- use tokens discovered in Phase 1.3
- If the argument is a number -- try 0, 1, or values found in other view results
- If arguments cannot be determined, document the view as "untested -- requires: [arg types]"

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` for each view.

### 2.2 Storage Key Enumeration

List all storage keys, then for each important or interesting key:
- Read the raw value
- Cross-reference with view results -- storage values should be consistent with what views report
- Look for storage keys that no view exposes (hidden state)

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_storage_keys` to list keys and `mvx_sc_storage` to read values.

### 2.3 State Consistency Checks

Verify internal consistency:
- Do token balances (from account) match what views report as reserves/holdings?
- Are configuration values within reasonable ranges (fees < 100%, ratios > 0)?
- Is the contract in the expected operational state (paused/active)?
- Do counters, indices, and epoch values make sense relative to the current network epoch?

## Phase 3: Endpoint Simulation

### 3.1 Public Endpoint Simulation

For each **public, non-owner** mutable endpoint:
- Perform a dry-run (simulation) or describe expected behavior based on ABI analysis
- Document what each endpoint does based on its name, parameters, and the state observed
- Identify endpoints that could be called by anyone and assess risk:
  - Can a user drain funds?
  - Can a user manipulate state to benefit themselves?
  - Can a user cause a denial of service?

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_simulate` for dry-run simulation.

### 3.2 Edge Case Analysis

For critical public endpoints, reason about:
- **Zero-value inputs**: What happens with amount = 0?
- **Self-referencing**: Sender == receiver, tokenA == tokenB
- **Overflow**: Maximum BigUint values
- **Empty collections**: What if a required collection is empty?
- **Re-entrancy**: Does the endpoint make external calls before updating state?

### 3.3 Gas Analysis

For key operations, estimate gas costs:
- Endpoints that iterate over storage collections
- Endpoints that make multiple cross-contract calls
- Endpoints with unbounded loops
- Are any endpoints unusually expensive? (potential DoS vector)

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_estimate_gas` for gas estimates.

## Phase 4: Cross-Contract Analysis

If the ABI or storage reveals references to other contract addresses:
- Inspect each dependency contract's account
- Verify: same owner? Verified? Upgradeable?
- Are dependencies still active and funded?

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` to inspect each dependency contract.

## Phase 5: Test Report

Generate a comprehensive report:

### Contract Overview
| Property | Value |
|----------|-------|
| Address | [address] |
| Network | [network] |
| Owner | [owner] |
| Balance | [EGLD held] |
| Verified | Y/N |
| Upgradeable | Y/N |
| Payable | Y/N |
| Endpoints | N mutable, M views |
| Tokens Managed | [list] |

### View Results Summary
| View | Result | Status |
|------|--------|--------|
| ... | ... | OK / Unexpected / Error |

### Storage Analysis
| Key | Value | Matches View | Notes |
|-----|-------|-------------|-------|

### Endpoint Risk Assessment
| Endpoint | Risk Level | Reason |
|----------|-----------|--------|

### Issues Found

For each issue:
```
### Issue #N: [Title]
Severity: [Critical / High / Medium / Low / Info]
Category: [State Inconsistency / Access Control / Economic / Configuration]
Evidence: [Which step and result revealed this]
Impact: [What could go wrong]
Recommendation: [Suggested action]
```

### Health Score
```
State Consistency: [score /10]
Access Control: [score /10]
Economic Safety: [score /10]
Overall Health: [score /10]
```

### Checklist
- [ ] Account properties inspected
- [ ] Full ABI retrieved and analyzed
- [ ] All views queried
- [ ] Storage keys enumerated
- [ ] Key storage values read
- [ ] State consistency verified
- [ ] Public endpoints analyzed for risk
- [ ] Edge cases considered
- [ ] Cross-contract dependencies checked
- [ ] Report generated with evidence

## Next Steps

- If issues found, use the **mvx-debug-tx** skill to investigate specific transactions
- For deeper analysis, run the **mvx-audit-onchain** skill
- If source code is available, complement with the **mvx-sc-audit** skill for code-level analysis

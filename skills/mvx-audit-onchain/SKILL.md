---
name: mvx-audit-onchain
description: Perform a comprehensive on-chain security audit of a deployed MultiversX smart contract. Use when reviewing a contract's security posture, permissions, state, and economic safety without source code.
---

# MultiversX On-Chain Smart Contract Audit

Perform a comprehensive security audit of a deployed contract using only on-chain data: account properties, ABI, storage, views, and simulation.

## Phase 1: Reconnaissance

### 1.1 Account Properties

Gather contract account information via the API:
```
GET https://api.multiversx.com/accounts/{address}
```

Record:
- Owner address
- Properties: upgradeable, payable, payableBySmartContract, readable
- Balance (EGLD held by contract)
- Deploy date and tx hash
- Verification status
- Developer reward accumulated

**Flags**:
- Upgradeable + large balance = owner trust assumption
- Payable without clear reason = potential fund trap
- Unverified = no public source, higher risk

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` for this step.

### 1.2 ABI Inspection

Retrieve the contract ABI and document:
- All endpoints with mutability (mutable vs readonly)
- Access control annotations (`onlyOwner` visible in ABI)
- Payable endpoints and which tokens they accept (`*` = any token)
- Constructor and upgrade parameters
- Custom types (structs, enums)
- Events

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_abi` for this step.

### 1.3 Permission Matrix

For each endpoint, classify:

| Endpoint | Access | Payable | Risk |
|----------|--------|---------|------|
| ... | public/owner/admin | yes/no | Critical/High/Medium/Low |

**Risk classification**:
- **Critical**: Public + payable + state-changing
- **High**: Public + state-changing (no payment)
- **Medium**: Owner-only + state-changing
- **Low**: Readonly views

### 1.4 Token Discovery

Identify tokens managed by the contract:
- Query endpoints like `getTokenId`, `getLpTokenIdentifier`, `getRewardTokens`, `getBaseToken`
- For each token found, check its decimals, supply, roles, and owner

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` to read token IDs and `mvx_token_info` to inspect each token.

## Phase 2: State Analysis

### 2.1 Query All Views

For **every** view endpoint in the ABI:
- Call it with appropriate arguments
- Record the result
- Flag unexpected values: zeros where non-zero expected, max values, empty strings

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_query` for each view.

### 2.2 Storage Inspection

List all storage keys, then for important keys:
- Read the raw value
- Decode any complex storage values from hex format
- Cross-reference with view results (should match)
- Look for keys that views don't expose (hidden state)

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_storage_keys` to list keys, `mvx_sc_storage` to read values, and `mvx_sc_decode` to decode complex hex values.

### 2.3 State Consistency Checks

- Do token balances match what views report?
- Are configuration values within reasonable ranges?
- Is the contract paused/active as expected?
- Do counters/indices make sense?

## Phase 3: Vulnerability Analysis

### 3.1 Access Control

From the ABI:
- **Missing onlyOwner**: Endpoints that modify state but lack access control
- **Public payable**: What tokens can anyone send? Is token identity validated?
- **Dangerous public endpoints**: Can anyone call pause, set fees, change config?

### 3.2 Endpoint Analysis

For each **public mutable** endpoint:
- What does it do? (infer from name, parameters, return type)
- What tokens does it accept?
- Can it be abused? (send wrong tokens, zero amounts, self-referencing addresses)

### 3.3 Economic Analysis (DeFi Contracts)

If the contract manages liquidity, rewards, or tokens:
- **Reserves**: Query pool reserves, check if balanced
- **Rates**: Query exchange rates, fee percentages
- **Supply consistency**: Total minted == total distributed + remaining?
- **Fee extraction**: Are fees accumulating correctly?

### 3.4 Property-Based Risks

- **Upgradeable**: Owner can change any logic at any time (inherent trust)
- **Not payable but holds tokens**: How do tokens enter? (ESDT transfers don't need payable)
- **Verified vs unverified**: Unverified = no public source, higher risk

## Phase 4: Simulation

### 4.1 Simulate Public Endpoints

For each public (non-owner) endpoint, perform a dry-run:
- Call with zero/default arguments -- should fail gracefully, not panic
- Call with edge case values -- max BigUint, empty strings, zero address
- Document which succeed and which fail (and error messages)

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_simulate` for dry-run simulation.

### 4.2 Gas Analysis

For key operations, estimate gas costs:
- Are any endpoints unusually expensive? (potential DoS vector)
- Do gas costs scale with any user-controlled parameter?

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_sc_estimate_gas` for gas estimates.

## Phase 5: Cross-Contract Analysis

If the contract interacts with other contracts (visible from ABI parameters accepting addresses):
- Identify dependency contracts
- Check their properties: same owner? Verified? Upgradeable?
- Are dependencies still active and funded?

**MCP shortcut**: If a MultiversX MCP server is available (multiversx-sc-mcp or multiversx-mcp-server), use `mvx_account` to inspect each dependency contract.

## Output Format

### Executive Summary
3-5 sentences: what was audited, overall risk level, most critical findings.

### Contract Overview
| Property | Value |
|----------|-------|
| Address | ... |
| Owner | ... |
| Verified | ... |
| Upgradeable | ... |
| Deployed | ... |
| Balance | ... |
| Endpoints | N mutable, M views |

### Permission Matrix
| Endpoint | Access | Payable | Risk | Notes |
|----------|--------|---------|------|-------|

### State Summary
Key view results in a table.

### Findings

For each finding:
```
### [SEVERITY] Finding #N: Title

**Category**: Access Control / Economic / State / Gas
**Evidence**: Which step/query revealed this (include the actual result)
**Impact**: What could go wrong
**Recommendation**: How to fix
```

### Risk Assessment
```
Overall: [Safe / Low Risk / Medium Risk / High Risk / Critical]
Access Control: [score /10]
Economic Safety: [score /10]
State Consistency: [score /10]
```

### Checklist
- [ ] Account properties checked
- [ ] Full ABI inspected
- [ ] Permission matrix built
- [ ] All views queried
- [ ] Storage keys inspected
- [ ] State consistency verified
- [ ] Public endpoints simulated
- [ ] Gas costs checked
- [ ] Cross-contract dependencies analyzed
- [ ] Findings documented with evidence

**If you found zero issues, you missed something. Re-check.**

## Next Steps

- If the contract source code is available (verified or local), also run the **mvx-sc-audit** skill for full vulnerability analysis with code-level patterns
- Use the **mvx-test-contract** skill to systematically test all endpoints
- For suspicious transactions, use the **mvx-debug-tx** skill

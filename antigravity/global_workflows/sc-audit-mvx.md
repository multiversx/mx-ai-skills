---
description: security-audit-mvx
---

MultiversX Security Audit Workflow
Use this workflow to conduct a thorough security assessment of MultiversX smart contracts, focusing on protocol-specific vulnerabilities (ESDT, Async Calls) and general DeFi risks.

Phase 1: Context & Scope Analysis
Architecture Review:
Identify all contract interactions and Cross-Shard vs Intra-Shard dependencies.
Map out the Async Call Graph: Which contracts call which, and where are the callbacks?
ESDT Tokenomics Check:
List all ESDT tokens managed by the contract.
Identify roles: ESDTRoleLocalMint, ESDTRoleLocalBurn, ESDTRoleNFTCreate.
Verify who holds the TokenIdentifier (is it hardcoded, stored, or passed via arguments?).
Phase 2: Vulnerability Analysis (MultiversX Specific)
// turbo

Async Callback Analysis:
Reentrancy Check: Does the callback update state after triggering another async call? (Violation of Checks-Effects-Interactions).
Callback Logic: Is the callback named [callback] or custom? Are generic callbacks handling payments correctly?
Payment Validation: Does the callback verify #[payment_amount] and #[payment_token]?
ESDT Safety:
Mint/Burn Risks: Are mint/burn limits enforced? Can an attacker inflate supply within a localized shard?
Transfer Guardrails: Are direct_send or multi_transfer used safely?
Storage & Gas:
Mapper Loop DOS: Are VecMapper or SetMapper iterated over in part of a view or endpoint? (Potential Gas DOS).
Cold/Hot Storage: Are we reading/writing to the same key multiple times redundantly?
Phase 3: Vulnerability Analysis (DeFi Standard)
Math Safety:
Verify usage of BigUint / BigInt for all financial math.
Zero Checks: Are denominators checked for zero?
Rounding Errors: Are divisions performed last?
Access Control:
Verify #[only_owner] or custom #[only_role] annotations on administrative endpoints.
Check #[payable("*")] vs non-payable endpoints.
Flash Loan Resistance:
Does the contract rely on spot prices from a DEX in the same block? (Oracle manipulation risk).
Phase 4: Automated Scanning & Verification
// turbo

Static Analysis: Run cargo check and cargo clippy -- -D warnings.
Mandos verification: Ensure all .scen.json files pass.
Rust Scenario Tests: Execute cargo test to validate complex flows in the ScenarioWorld.
Phase 5: Reporting & Remediation
Draft Findings:
Classify as Critical/High/Medium/Low based on Impact x Likelihood.
Example Critical: "Async Callback Reentrancy allows double-spending of ESDT."
Remediation Plan:
For Async Reentrancy: Suggest locking the contract or using the "Checks-Effects-Interactions" pattern strictly.
For Gas DOS: Suggest pagination or off-chain indexing.
Phase 6: Final Verification
Re-run all tests after fixes.
Verify that mxpy contract build produces a reproducible WASM binary.

MultiversX DeFi Audit Workflow
Use this workflow specifically for DeFi protocols (DEX, Lending, Yield Farming) where financial logic, composability, and ExecuteOnDestContext risks are paramount.

Phase 1: Composability & Interaction Analysis
Synchronous Interaction Risk:
Identify all usage of execute_on_dest_context or sync_call.
Reentrancy Check: Can the target contract call back into the caller?
Context Leaks: Are we sending unintended funds or permissions to the destination context?
Flash Loan Vector Analysis:
Does any endpoint rely on the current balance of the contract to calculate exchange rates or collateral health?
Mitigation: Verify usage of internal accounting variables (virtual_reserves) vs. blockchain().get_sc_balance().
Phase 2: Financial Logic & Tokenomics
// turbo

Math Precision Audit:
Rounding Direction: Are user-favored operations (withdrawals) rounding DOWN and protocol-favored operations (deposits) rounding UP?
Order of Operations: Verify (amount * rate) / precision (Multiplication BEFORE Division).
Decimal Mismatch: Check mixing of 6-decimal (USDC) and 18-decimal (EGLD) tokens.
ESDT Handling:
Payment Validation: Does every payable endpoint verify token_identifier and nonce (for SFTs)?
Dust accumulation: Is there a mechanism to sweep dust or prevent it from locking the contract?
Phase 3: Oracle & Price Feed Safety
Price Source Verification:
Are we using a pure on-chain AMM spot price? (Vulnerable to Flash Loans).
Recommendation: Use a TWAP (Time-Weighted Average Price) or a trusted off-chain Oracle (with delegation/aggregator check).
Stale Data Check:
Doesland the oracle read perform a check for timestamp or round_id availability?
Phase 4: Access Control & Governance
Privileged Roles:
Map all #[only_owner] and #[only_role] endpoints.
Timelock Check: Are critical parameter changes (fees, collateral factors) behind a Timelock?
Pause Mechanism:
Is there a pause endpoint to halt borrowing/swapping in emergencies?
Phase 5: Advanced Simulation (DeFi Specific)
Economic Fuzzing (Invariant Testing):
Define invariants: k = x * y (AMM), total_supply == total_collateral (Lending).
Use ScenarioWorld to run thousands of random trades/interactions to break invariants.
Flash Loan Simulation:
Create a test scenario where an attacker borrows 1M EGLD and interacts with the protocol.
Verify that the protocol's solvency remains intact.
Phase 6: Final Report Generation
Compile findings with a specific focus on Economic Risk vs Technical Risk.
Suggest mitigation strategies specific to MultiversX (e.g., "Use Async Callbacks for cross-shard price updates").
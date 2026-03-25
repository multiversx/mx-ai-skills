---
description: Audit MultiversX smart contracts (Rust). Vulnerability patterns, ESDT safety, async callbacks, DeFi analysis, storage lifecycle, test verification.
---
# MultiversX Smart Contract Auditor

You **hunt for vulnerabilities** in smart contracts. If you find nothing, you missed something.

## Phase 1: Reconnaissance

Before grepping, understand the system.

1. **Map the system**: Use `audit_context` to identify core logic, value flows, access controls, external dependencies, and cross-contract calls.
2. **Spec compliance**: If a specification exists, use `spec_compliance` to verify the SC implements it correctly. Flag any deviations.
3. **Enumerate entry points**: Use `mvx_entry_points` to inventory all `#[endpoint]`, `#[view]`, `#[payable]`, `#[init]`, `#[upgrade]`, `#[callback]` functions.
   - Tag every `#[payable]` endpoint. Is `call_value()` checked?
   - Classify risk: Critical (payable/state-changing), High (admin), Low (read-only).
   - **Prioritize Phases 2-3 by risk classification** — analyze Critical endpoints first.
4. **ESDT tokenomics**: List all ESDT tokens managed by the contract.
   - Identify roles: `ESDTRoleLocalMint`, `ESDTRoleLocalBurn`, `ESDTRoleNFTCreate`.
   - Is the `TokenIdentifier` hardcoded, stored, or passed via arguments?
5. **Async call graph**: Map which contracts call which, and where the callbacks are.
   - Cross-shard vs intra-shard dependencies.
6. **Scope detection**: Determine which conditional phases apply:
   - **Is this an upgrade?** → Phase 2 (Differential Review) applies. Detect: `#[upgrade]` present, or PR/diff provided.
   - **Is this DeFi?** → Phase 4 applies. Detect: token swaps, liquidity pools, lending, yield, price calculations, or oracle usage.
   - **Is this a multi-contract project?** → Repeat Phases 3-5 per contract, then analyze cross-contract interactions.

## Phase 2: Differential Review (If Upgrade)

**Do this BEFORE deep analysis** — it scopes what to focus on.

Use `diff_review` to check:
- Storage layout compatibility (field reordering = memory corruption).
- New mappers initialized in `#[upgrade]` (not `#[init]`).
- Removed mappers cleaned in `#[upgrade]`.
- Regression risks from changes.
- Focus subsequent phases on changed code paths.

## Phase 3: Analysis

### Bug Patterns (A-M)

| ID | Pattern | What It Catches | How to Find |
|----|---------|-----------------|-------------|
| A | Incomplete Balance | Available calc missing storage | Multiple storage mappers, only one subtracted |
| B | Single-Period Update | Multi-period gap loses data | `current - 1` without loop |
| C | Unprotected Removal | Critical items removable | `swap_remove` without base token check |
| D | Removal Orphans Current | Current period stuck | Remove without clear accumulated |
| E | Permissionless Special | Energy inflation | `#[payable("*")]` accepts XMEX without auth |
| F | Storage Not Cleaned | Old key not in upgrade | **Check git history for removed mappers** |
| G | Unbounded Collection | Gas DoS | No max size + iteration |
| H | Dead Code | Unnecessary surface | `add_X` never used after init |
| I | Early Period Blocking | First N periods blocked | `if current_week < X` returns zero |
| J | Removal Orphans Multi-Period | K periods inaccessible | Removal affects claimable window |
| **K** | **Sync Call Reentrancy** | Reentrant call via sync | `sync_call` / `execute_on_dest_context` without CEI |
| **L** | **Unverified Async Returns** | Silent failure on cross-contract call | Callback ignores error or missing callback entirely |
| **M** | **Re-initialization** | Init callable post-deploy | `#[init]` not protected or callable again |

### Don't Be Fooled — Bug Pattern Code Examples

These patterns LOOK intentional but are often BUGS. Check each against the code.

**Pattern A — Only subtracts from one storage location:**
```rust
// BUG: Only checks accumulated_fees, not total_rewards_for_week
fn get_available_amount(&self, token_id: &TokenIdentifier) -> BigUint {
    let balance = self.blockchain().get_sc_balance(token_id);
    for week in start..end {
        balance -= self.accumulated_fees(week, token_id).get();
        // MISSING: balance -= amount from total_rewards_for_week(week)
    }
    balance
}
```
**CRITICAL** — causes fund drain or user claim failures.

**Pattern B — Single week update after multi-week gap:**
```rust
// BUG: Only updates current_week - 1, loses intermediate weeks
fn accumulate_for_previous_week(&self) {
    self.accumulated_fees(current_week - 1, &token).update(...);
    // MISSING: loop for weeks between last_update and current_week
}
```
**Permanent token loss** for skipped weeks.

**Pattern C — No validation on token removal:**
```rust
// BUG: Allows removing critical tokens
fn remove_reward_tokens(&self, tokens: MultiValueEncoded<TokenIdentifier>) {
    for token in tokens {
        self.reward_tokens().swap_remove(&token);
        // MISSING: require!(token != base_token && token != locked_token)
    }
}
```
**Permanently locks user rewards.**

**Pattern D — Removal doesn't clear in-flight state:**
```rust
// BUG: Removing item but not clearing associated pending state
fn remove_token(&self, token: TokenIdentifier) {
    self.token_whitelist().swap_remove(&token);
    // MISSING: What about accumulated_fees for this token in current period?
    // Those become stuck - can't be claimed (not in whitelist)
    // and can't be withdrawn (still reserved)
}
```
**Orphans tokens for the current time period.**

**Pattern E — Permissionless deposit of special tokens:**
```rust
// BUG: Anyone can deposit XMEX without energy deduction
#[payable("*")]
fn deposit_swap_fees(&self) {
    let payment = self.call_value().single_esdt();
    // MISSING: If payment is XMEX, verify caller is authorized (e.g., Token Unstake SC)
    // Otherwise sender's energy isn't deducted but recipients gain energy
}
```
**Breaks energy conservation** — total energy inflates.

**Pattern F — Storage key removed from code but not cleaned (CONTEXT-DEPENDENT):**
```rust
// NOTE: On MultiversX, empty upgrade() is the STANDARD pattern.
// Orphaned storage keys from removed mappers are just wasted space —
// the framework never reads them unless code references them.
//
// This is ONLY a real issue when:
// 1. The old storage key name is reused by a NEW mapper with different semantics
// 2. A storage format change requires data migration (e.g., struct fields added/removed)
// 3. The old data is actively read by remaining code via raw storage access
//
// Do NOT flag empty upgrade() as a vulnerability by default.
// Do NOT flag orphaned storage keys as a vulnerability unless collision/migration is needed.
```
**Only check git history** when there is reason to believe key name collision or data migration is needed.

**Pattern G — Unbounded collection without size limit:**
```rust
// BUG: Collection can grow indefinitely
#[only_owner]
fn add_to_whitelist(&self, items: MultiValueEncoded<Item>) {
    for item in items {
        self.whitelist().insert(item);
        // MISSING: require!(self.whitelist().len() < MAX_SIZE)
    }
}
// Later, iteration over whitelist can exceed gas limits
```
**Any collection iterated in user-facing functions needs a max size.**

**Pattern H — Endpoint exists but planned for removal:**
```rust
// BUG: Unnecessary endpoint increasing attack surface
#[only_owner]
#[endpoint(addRewardTokens)]
fn add_reward_tokens(&self, tokens: MultiValueEncoded<TokenIdentifier>) {
    // Is it actually needed after initial setup?
    // Is there a plan to remove all tokens except base tokens?
}
```
**Check**: Is this endpoint needed? Look for matching remove without add being used.

**Pattern I — Returns zero for early weeks (OFTEN MISSED):**
```rust
// BUG: No swapping/claiming possible for first N weeks
fn get_available(&self) -> BigUint {
    if current_week < THRESHOLD {
        return BigUint::zero(); // Blocks functionality for first N weeks!
    }
    // ... rest of calculation
}
```
**CHECK**: Is this intentional? Does it block admin functions like swap_token_to_base_token?
**SEARCH FOR**: `if current_week < ` or `if current_epoch < ` patterns.

**Pattern J — Removal orphans multi-period state (OFTEN MISSED):**
```rust
// BUG: Removal makes claimable periods inaccessible
fn remove_token(&self, token: TokenIdentifier) {
    self.reward_tokens().swap_remove(&token);
    // accumulated_fees for CURRENT period is stuck (Pattern D)
    // BUT ALSO: What about the NEXT K periods where users can still claim?
    // If claim window is 4 weeks, users lose access to 4 weeks of rewards!
}
```
**For any removal, calculate**: How many time periods of data become inaccessible?

### Search Commands

```bash
# Pattern A: Balance calculations
grep -r "available\|get_sc_balance" --include="*.rs"
grep -r "accumulated\|total_rewards" --include="*.rs"

# Pattern B: Time gaps
grep -r "current_week.*-.*1\|last_update" --include="*.rs"

# Pattern C/D/J: Removals
grep -r "swap_remove\|remove.*token" --include="*.rs" | grep -v test

# Pattern E: Payable
grep -r "#\[payable" --include="*.rs"

# Pattern F: CRITICAL - Git history for removed storage
git log -p --all -S "storage_mapper" -- "*.rs" | grep -E "^-.*storage_mapper"
grep -A 100 "fn upgrade" src/*.rs | grep -E "clear|take|StorageKey"

# Pattern G: Collections
grep -r "VecMapper\|SetMapper" --include="*.rs"
grep -r "for .* in .*\.iter()" --include="*.rs"

# Pattern H: Dead code
grep -r "add_.*token\|remove_.*token" --include="*.rs"
grep -r "TODO.*remove\|deprecated\|unnecessary" --include="*.rs"

# Pattern I: Early period blocking
grep -r "current_week <\|current_epoch <\|< USER_MAX" --include="*.rs"

# Pattern K: Sync call reentrancy
grep -r "sync_call\|execute_on_dest_context" --include="*.rs"

# Pattern L: Missing/empty callbacks
grep -r "#\[callback\]" --include="*.rs"
grep -r "async_call\|\.register_promise" --include="*.rs" | grep -v callback

# Pattern M: Re-initialization
grep -r "#\[init\]" --include="*.rs"
```

### Automated Static Analysis

Use `mvx_static_analysis` (Semgrep + Clippy):
- Floating point usage in financial math.
- Unsafe arithmetic (`+`, `-`, `*` without `checked_` or `BigUint`).
- Panic inducers (`unwrap`, `expect` in non-test code).
- Missing access control on state-changing endpoints.

Run `cargo check` and `cargo clippy -- -D warnings`.

### MultiversX-Specific Vulnerabilities

Use `mvx_sharp_edges` as reference. Check each area once:

**Async Callbacks & Cross-Contract**:
- State changes are NOT auto-reverted on callback failure. Is Checks-Effects-Interactions followed?
- Does the callback verify payment amount and token?
- Callback named `[callback]` or custom? Generic callbacks handling payments correctly?
- Cross-shard OOG: what happens when gas runs out on destination shard?
- Return values from async calls — are they checked or silently ignored?

**ESDT Safety**:
- Mint/burn limits enforced? Can an attacker inflate supply within a shard?
- `direct_send` / `multi_transfer` used safely?
- Every payable endpoint verifies `token_identifier` and `nonce` (for SFTs)?
- Does the contract handle all transfer types (single ESDT, multi-ESDT, NFT)? What happens on unexpected transfer type?

**Storage & Gas**:
- VecMapper/SetMapper iterated in an endpoint or view? (Gas DoS)
- Same storage key read/written multiple times redundantly? (Cold/hot storage)
- VecMapper is 1-indexed with separate storage slots — not a Rust Vec.
- MapMapper: each entry = separate storage key (overhead).

**Access Control**:
- `#[only_owner]` or `#[only_role]` on all admin endpoints?
- `#[payable]` vs non-payable: correct separation?
- Pause mechanism available for emergencies?
- Can `#[init]` be called again after deployment?

**Math Safety**:
- `BigUint` / `BigInt` for all financial math.
- Zero denominator checks.
- Multiplication BEFORE division: `(amount * rate) / precision`.
- Rounding direction: withdrawals round DOWN, deposits round UP.
- Decimal mismatch: mixed 6-decimal (USDC) and 18-decimal (EGLD) tokens.

### Critical Path Verification (MANDATORY)

**YOU MUST COMPLETE THIS SECTION — DO NOT SKIP**

For reward/fee distribution contracts, trace these flows end-to-end:

**1. Token Deposit → Storage → Claim Flow**:
```
Q: When tokens are deposited, where do they go in storage?
Q: When the first claim happens for a week, does storage move/transform?
Q: Does the "available balance" calculation account for BOTH locations?
```

**2. Balance Calculation Audit**:
- Find ALL functions that calculate "available" or "claimable" amounts.
- For EACH function, verify it subtracts ALL reserved/committed amounts.
- **Specific check**: If there's `accumulated_fees` AND `total_rewards_for_week`, does the available calculation subtract BOTH?

**3. Token Removal Flow**:
- What happens to tokens in various storage locations when a reward token is removed?
- Can removal orphan committed rewards?
- Can base tokens (MEX/XMEX) be removed?
- **What happens to claims for the NEXT N periods after removal?**

**4. Time Gap Scenarios**:
- What if no interaction for multiple weeks?
- Does each missed week get handled, or only the most recent?
- What about week 0 / first weeks edge cases?

**5. Permissionless Endpoint Abuse**:
- For each public endpoint, what tokens can be sent?
- If XMEX can be sent, is the sender's energy properly deducted?
- Can someone inflate totals by depositing and claiming?

**6. Early Period Blocking**:
- Do any functions return early/zero for the first N periods?
- Is this blocking intentional or does it break functionality?
- Can users/admins work around it?

### Cross-Cutting Checks (G1-G8)

Use as a final sweep — each should already be covered above. Verify nothing was missed:

| Check | Key Question |
|-------|--------------|
| G1: Admin Cascading | What breaks N periods later? Quantify: K periods. |
| G2: Storage Lifecycle | Is removed mapper cleaned? **Must check git history.** |
| G3: Unbounded Collections | Max size + iteration? |
| G4: Dead Code | Is add_X used after init? Check for add/remove pairs. |
| G5: Time-Delayed | Admin action during validity? Early period blocking? |
| G6: State Transitions | All dependent calcs updated? |
| G7: Async Callbacks | All async/callback issues addressed? |
| G8: Math Safety | All math issues addressed? |

## Phase 4: DeFi-Specific Analysis

**Applies when**: Token swaps, liquidity pools, lending, yield farming, price calculations, or oracle usage detected.

**Composability**:
- Identify all `sync_call()` (or legacy `execute_on_dest_context`) usage.
- Can the target contract call back into the caller? Context leaks?

**Flash Loan Resistance**:
- Does any endpoint rely on `get_sc_balance()` for exchange rates or collateral health?
- Mitigation: internal accounting (`virtual_reserves`) instead of live balance.

**Oracle & Price Feed Safety**:
- On-chain AMM spot price = manipulable via flash loans.
- Check for TWAP or off-chain oracle with staleness/round_id checks.

**Governance**:
- Are critical parameter changes (fees, collateral factors) behind a timelock?
- Is there a pause endpoint to halt operations in emergencies?

**Dust Accumulation**:
- Mechanism to sweep dust or prevent it from locking the contract?

**Invariant Testing**:
- Define protocol invariants: `k = x * y` (AMM), `total_supply == total_collateral` (Lending).
- Use ScenarioWorld to run thousands of random interactions to break invariants.
- Simulate flash loan attack: borrow 1M EGLD, interact with protocol, verify solvency.

## Phase 5: Dynamic Verification

Static analysis is not enough. Run the code.

### Test Execution
1. Run `cargo test` — document pass/fail/skip counts.
2. Run Mandos scenarios (`.scen.json`) — verify all pass.
3. Run `sc-meta test-gen` — ensure scenarios are up to date.
4. If `mx-chain-simulator-go` tests exist, run them. If not, score -2 on test quality.

### Build Verification
- Run `sc-meta all build` and verify the WASM binary is reproducible.
- Compare output hash against any previously deployed version.

### Test Quality Assessment
Use `mvx_testing_handbook` to evaluate:
- **Unit test coverage** — percentage and gaps.
- **Integration tests** — are mocks realistic?
- **Access control tests** — every `#[only_owner]` / `#[only_role]` endpoint tested for unauthorized access?
- **Money flow tests** — 100% Mandos coverage for all payment endpoints?

### Property Testing
Use `mvx_property_testing` to find edge cases:
- Define invariants (conservation, monotonicity, bounds, idempotency).
- Run `proptest` / `cargo-fuzz` against core logic.
- Generate Mandos scenarios from any failures found.

## Phase 6: Post-Discovery

### Variant Analysis
After finding any vulnerability, use `variant_analysis` to:
- Abstract the bug to a general pattern.
- Search for all instances across the codebase.
- Create Semgrep rules using `mvx_semgrep_creator` for CI/CD prevention.

### Fix Verification
When fixes are proposed, use `fix_verification` to:
- Reproduce the original bug with a test.
- Verify the fix makes the exploit test fail.
- Run regression suite to confirm no side effects.

## Severity Calibration

### Owner Trust on MultiversX
On MultiversX, the contract owner can ALWAYS deploy a contract upgrade and bypass any logic. This means `#[only_owner]` endpoints do NOT represent a new trust boundary — the owner already has unlimited power. Do NOT classify owner-accessible functionality as Critical or High severity. Owner-only endpoints are at most Low/Informational observations about operational convenience.

### Critical (Funds at immediate risk)
ALL must be true:
- [ ] Direct, exploitable path to fund loss or theft
- [ ] No admin action required to exploit (owner endpoints are NOT critical — see above)
- [ ] Can write a working proof-of-concept test
- [ ] Impact is significant (>1% of TVL or affects all users)

**Examples**: Balance calculation missing reserved amounts, missing access control on withdrawals, reentrancy double-spend.

### High/Major (Significant impact, exploitable)
MOST must be true:
- [ ] Clear exploit path exists
- [ ] Requires specific conditions but achievable
- [ ] Impact affects protocol economics or user funds

**Examples**: Energy double-counting, gas DoS blocking critical functions, slippage attacks, admin action corrupts user-claimable state for multiple periods.

### Medium (Limited impact or requires privileged access)
- [ ] Issue is real but impact is contained
- [ ] May require admin/owner action to trigger
- [ ] Workaround exists

**Examples**: Admin can accidentally remove critical tokens, first N weeks blocked, storage not cleaned on upgrade (future collision risk), token removal orphans rewards for claimable window.

### Low (Code quality, minor issues)
Style issues, typos, inefficient patterns with no security impact. Missing tests (unless for critical paths). Dead code that doesn't affect security.

## False Positive Reduction

**Before reporting ANY Critical/High finding, verify:**

1. [ ] **Exploitability**: Can an attacker actually trigger this?
2. [ ] **Proof of Concept**: Can you write a failing test?
3. [ ] **Real-world Impact**: What's the actual damage?
4. [ ] **Attack Vector**: Who can exploit? (anyone / user / admin)
5. [ ] **Existing Mitigations**: Are there other checks preventing this?

### Common False Positive Patterns to AVOID:
- CEI violations that aren't actually exploitable (no reentrancy vector in MultiversX).
- "Missing slippage protection" when caller provides min_amount.
- Access control concerns for legitimately public functions.
- "Unbounded iteration" on lists that are practically bounded by design.
- **Owner-level access control as a vulnerability**: `#[only_owner]` endpoints do NOT introduce a new trust boundary. Only flag if an owner endpoint can cause *unintended* harm (e.g., accidentally breaking invariants), and classify as Low/Informational, not Critical/High.
- **Empty upgrade() as a vulnerability**: Empty `fn upgrade(&self) {}` is the STANDARD MultiversX pattern. Do NOT flag unless there is a concrete issue (storage format change requiring migration, or storage key name collision with new mappers).
- **Block nonce / timestamp subtraction as underflow risk**: When the stored value always originates from `self.blockchain().get_block_nonce()` or `get_block_timestamp()`, subtraction `current - stored` cannot underflow because the blockchain is monotonically increasing. Only flag if there is an admin setter or any code path that writes an arbitrary (potentially future) value to that storage.
- **Unreachable precondition findings**: Before assigning severity, trace preconditions back through actual callers. If the trigger condition is unreachable from any public entry point, the finding is a false positive.

### DO NOT Dismiss:
- Balance calculations that miss storage locations.
- Token flows where energy/amounts aren't conserved.
- Admin functions that can break invariants.
- Storage cleanup claims without verification.
- Early period blocking without verification it's intentional.
- Removal effects on multi-period claimable state.

## Output Format

### Audit Scope
```
Contract: [name]
Commit: [hash]
Files: [list]
Excluded: [list, if any]
Conditional Phases: [Upgrade: Y/N] [DeFi: Y/N] [Multi-contract: Y/N]
```

### Executive Summary
3-5 sentences: what was audited, overall risk level, most critical findings, and key recommendation.

### Risk Rating
```
Overall: [Safe / Low Risk / Medium Risk / High Risk / Critical]
```

### Pattern Results (A-M)
```
A (Incomplete Balance): [FOUND/CLEAN]
B (Single-Period Update): [FOUND/CLEAN]
C (Unprotected Removal): [FOUND/CLEAN]
D (Removal Orphans Current): [FOUND/CLEAN]
E (Permissionless Special): [FOUND/CLEAN]
F (Storage Not Cleaned): [FOUND/CLEAN] - git history checked: [Y/N]
G (Unbounded Collection): [FOUND/CLEAN]
H (Dead Code): [FOUND/CLEAN]
I (Early Period Blocking): [FOUND/CLEAN]
J (Removal Orphans Multi-Period): [FOUND/CLEAN] - K=[periods]
K (Sync Call Reentrancy): [FOUND/CLEAN]
L (Unverified Async Returns): [FOUND/CLEAN]
M (Re-initialization): [FOUND/CLEAN]
```

### Critical Path Verification Results
**MANDATORY SECTION** — Show your work:
```
Token Flow Analysis:
- Deposit location: [where tokens go]
- Claim location: [where tokens come from]
- Available calculation: [function name]
  - Subtracts accumulated_fees: YES/NO
  - Subtracts total_rewards_for_week: YES/NO
  - VERDICT: CORRECT / BUG FOUND

Early Period Analysis:
- Functions with early returns: [list]
- First N periods blocked: [YES/NO, which functions]
- Intentional or bug: [assessment]

Removal Impact Analysis:
- Claim window: [N periods]
- If token removed, inaccessible data: [N periods x affected users]
```

### Cross-Cutting Check Results
```
G1 (Admin Cascading): [checked] - removal impact: [N periods]
G2 (Storage Lifecycle): current=[list], removed from git=[list], cleaned=[Y/N per mapper]
G3 (Unbounded Collections): [collection]: max=[X or NONE], iterated=[Y/N]
G4 (Dead Code): add_X without usage=[list or none]
G5 (Time-Delayed): early period blocking=[list functions, intentional Y/N]
G6 (State Transitions): [transition]: source cleared=[Y/N], calcs updated=[Y/N]
G7 (Async Callbacks): [checked] - issues=[list or none]
G8 (Math Safety): [checked] - issues=[list or none]
```

### Test Quality Score (1-10)
```
Unit Tests: [coverage %] - [pass/fail/skip]
Integration: [realistic mocks: Y/N]
Access Control Tests: [/10] - negative tests for admin functions
Edge Case Coverage: [/10] - time gaps, week 0, empty states
Economic Invariant Tests: [/10] - balance/energy conservation
System (Chain Sim): [available: Y/N] - [pass/fail/skip]
Build: [reproducible WASM: Y/N]
Overall Score: [1-10]
```

### Vulnerability Matrix
```
| # | Title | Severity | Confidence | Category | Remediation | PoC |
|---|-------|----------|------------|----------|-------------|-----|
| 1 | ...   | Critical | High       | Funds at risk | [recommended fix] | Y/N |
| 2 | ...   | High     | Medium     | DoS / Gas    | [recommended fix] | Y/N |
| 3 | ...   | Medium   | High       | Inefficiency | [recommended fix] | Y/N |
| 4 | ...   | Low      | High       | Style        | [recommended fix] | Y/N |
```

### Finding Detail Template
For each Critical/High finding:
```
### Finding #[N]: [Title]

**Severity**: [Critical/High]
**Confidence**: [High/Medium/Low]
**Category**: [Balance-Calculation | Access-Control | Time-Logic | Economic | Gas-DoS | Upgrade | State-Transition | Admin-Safety | Dead-Code | Early-Period]
**Location**: [file:line]

**Root Cause**: One sentence explaining WHY this happens.

**Description**: What is the vulnerability and why it matters.

**Impact**: What an attacker can achieve. Quantify if possible (e.g., "drain X tokens").

**Proof of Concept**:
[Code or test scenario demonstrating the exploit]

**Recommendation**: How to fix it.

**Status**: [Open / Fixed / Acknowledged]
```

### Verification Evidence
```
- "Ran [N] tests. [X] Passed. [Y] Skipped."
- "WASM build reproducible: [Y/N]."
- "Verified fix for Issue #N using fix_verification."
- "Variant analysis found [N] additional instances."
```

### Audit Checklist
```
Reconnaissance:
- [ ] System mapped (audit_context)
- [ ] Spec compliance checked (if spec exists)
- [ ] Entry points inventoried (mvx_entry_points)
- [ ] ESDT roles and tokens documented
- [ ] Async call graph mapped
- [ ] Conditional phases determined

Upgrade (if applicable):
- [ ] Storage layout compatible (diff_review)
- [ ] Initialization in #[upgrade] verified
- [ ] Removed mappers cleaned
- [ ] Analysis scoped to changed code paths

Analysis:
- [ ] Patterns A-M searched (with code examples verified)
- [ ] G1-G8 cross-cutting sweep completed
- [ ] Automated analysis run (mvx_static_analysis)
- [ ] Platform sharp edges reviewed (mvx_sharp_edges)
- [ ] Git history checked for removed storage

Critical Path Verification:
- [ ] Token flow traced end-to-end
- [ ] Balance calculations verified (all storage locations)
- [ ] Token removal flow analyzed
- [ ] Time gap scenarios checked
- [ ] Permissionless endpoint abuse checked
- [ ] Early period blocking analyzed

DeFi (if applicable):
- [ ] Composability risks checked
- [ ] Flash loan resistance verified
- [ ] Oracle safety verified
- [ ] Governance (timelock, pause) verified
- [ ] Invariant testing run

Time & Removal:
- [ ] Early period blocking analyzed
- [ ] Functions affected: [list]
- [ ] Removal impact quantified: [K periods x users]

Dynamic Verification:
- [ ] cargo test executed: [pass/fail/skip]
- [ ] Mandos scenarios verified
- [ ] Chain simulator tested (if available)
- [ ] Property testing run
- [ ] WASM build reproducible

Post-Discovery:
- [ ] Variant analysis for each finding
- [ ] Fix verification for each remediation
- [ ] Critical/High findings have PoC
- [ ] Findings classified as Technical vs Economic risk
```

## Quality Gates

1. [ ] Phase 1 reconnaissance completed
2. [ ] Differential review completed (if upgrade)
3. [ ] Patterns A-M searched (manual + automated + code examples checked)
4. [ ] G1-G8 cross-cutting sweep completed
5. [ ] MVX-specific vulnerabilities analyzed (async, ESDT, storage, access, math)
6. [ ] Critical path verification completed (token flows, balance calcs, removals)
7. [ ] DeFi-specific checks completed (if applicable)
8. [ ] Tests executed and quality scored
9. [ ] WASM build verified reproducible
10. [ ] Git history checked for removed storage
11. [ ] Early period blocking analyzed
12. [ ] Removal impact quantified
13. [ ] Unnecessary endpoints identified
14. [ ] Variant analysis run for each finding
15. [ ] Critical/High have PoC
16. [ ] False positive checklist applied to every finding
17. [ ] Not dismissed without proof

**Zero major issues in DeFi = rare. Re-check.**

---
description: Comprehensive Multi-Agent Workflow for MultiversX Smart Contract System Testing
---

# MultiversX Chain Simulator Interactor Test Generation Workflow

You are a senior QA expert and specialized Prompt Engineer. Your objective is to orchestrate a reliable, multi-phased process to analyze a MultiversX smart contract, prepare its interactor, design high-coverage scenarios, and generate validated Rust Chain Simulator tests.

## 0. Workflow State Manifest
Before starting, initialize your "Current Session State." **MANDATORY**: Re-print the updated manifest after completing every Phase to maintain context and prevent drift.

- **Target Contract:** [Path]
- **Analyzed Endpoints/Storage:** [Count/List]
- **Implemented Scenarios:** [Count]
- **Verification Status:** [Pending/Passed/Failed]
- **Last Error:** [Description]
- **WASM Hash:** [Hash (if built)]

---

## 1. Phase 0: Discovery & Analysis (The Analyst Role)
**Objective:** Create a high-fidelity business logic schema including storage layout.

### Steps:
1. **Asset Discovery:**
   - Use `find_by_name` to locate `#[multiversx_sc::contract]` in `src/`.
   - Use `grep_search` to identify ALL `#[endpoint]`, `#[view]`, `#[init]`, `#[event]`, `#[storage_mapper]`, and `#[multiversx_sc::module]` declarations. 
   - **MANDATORY**: Search recursively through all modules (`mod.rs`) and sub-directories to ensure 100% logic coverage.
2. **Metadata Extraction:**
   - Profile parameter types, return values, and `#[payable("*")]` status.
   - Capture ALL business constraints (e.g., `require!`, `sc_panic!`) and their exact error messages.
3. **Caching:** 
   - Check `.agent/cache/contract_schema.json`. Use it if valid; otherwise, overwrite it with the new analysis.

### Verification Gate 0: Analysis Summary
Output a table following this schema (Example provided):
| Type | Name | Inputs/Fields | Output | Payable | Error Messages / Logic |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Endpoint | deposit | amount: BigUint | () | Yes | "insufficient funds", "amount must be > 0" |
| Storage | UserBalance | Address | SingleValueMapper<BigUint> | No | tracks user deposits |

---

## 2. Phase 1: Infrastructure Setup (The Developer Role)
**Objective:** Ensure a compilable, lint-free interactor environment with correct toolchains.

### Steps:
1. **Toolchain & Workspace Check:**
   - Verify `mxpy --version` and `sc-meta --version`.
   - **MANDATORY**: Ensure the simulator is initialized: `sc-meta cs start`. (Run `sc-meta cs install` first if image is missing).
   - **MANDATORY**: Ensure `rust-toolchain.toml` in the `interactor` matches the root/contract toolchain to avoid compilation errors.
   - Ensure the interactor is a member in the root `Cargo.toml` `[workspace]` section.
2. **WASM Preparation:**
   - Run `mxpy contract build` or `sc-meta all build` to ensure the contract WASM and **ABI** are available in the `output/` or `bin/` directory. Snippets REQUIRE the ABI.
3. **Snippet Generation:**
   - **RULE**: Run `sc-meta all snippets`. **NEVER** hallucinate or manually create interactor methods. 
   - If a method is missing: 1. Verify ABI exists. 2. Re-build. 3. Re-run snippets.
4. **Dependency Audit:**
   - Ensure `interactor/Cargo.toml` includes:
     - `features = ["chain-simulator-tests"]` for `multiversx-sc-snippets`.
     - `tokio` (with `full` features).
     - Relative path dependency to contract: `path = "../.."`.

### Verification Gate 1: Compilation Check
- Run `cargo check --features chain-simulator-tests` from the `interactor` directory.
- **Constraint**: If check fails, use `ls -R` to verify relative paths. Fix paths/dependencies before proceeding.

---

## 3. Phase 2: Scenario Design (The Architect Role)
**Objective:** Plan deterministic, observable state transitions.

### Guidance:
- **Deterministic Identities:** Use `alice.pem`, `bob.pem`, etc. (Imported from SDK).
- **Initialization First:** Every scenario must start with an `init` or `upgrade` flow.
- **Assertion Density Requirement:** 100% of state-changing transactions MUST be followed by a View function query to verify the change (Query-After-Tx Pattern).
- **Comprehensive Coverage:** 
    - **Happy Paths:** Standard usage.
    - **Edge Cases:** Zero amounts, max/boundary values, time-locked interactions.
    - **Negative Testing:** Unauthorized callers and invalid state transitions (must assert specific error messages).

### Verification Gate 2: Logic Flow
Output a sequence for each test case:
`Action (Role) -> Parameters -> Expected Result -> Verification Query -> Expected State`

---

## 4. Phase 3: Implementation (The Developer Role)
**Objective:** Generate "Chain Simulator Only" Rust tests.

### Implementation Rules:
- **Test Harness:** Use `#[tokio::test]` and `ContractInteractor::new(Config::chain_simulator_config())`.
- **Wallet Handling:** 
    - Import test wallets directly from the Rust SDK (`multiversx-sc-snippets` or `rs-sdk` utilities).
    - **Note:** Chain Simulator automatically funds wallets registered via the interactor. Use `register_wallet` for all test actors.
- **Constants:** Define `const GAS_LIMIT: u64 = 100_000_000;` at the top. Use `.gas_limit(GAS_LIMIT)` on all transactions.
- **Naming:** Suffix all test functions with `_cs`.
- **Patterns:**
    ```rust
    // 1. Transaction with Registration
    interactor.register_wallet(alice_pem).await;
    interactor.deposit(100).from(&alice).gas_limit(GAS_LIMIT).run_cs().await;
    
    // 2. Immediate Verification
    let bal = interactor.get_balance(&alice).check_value(100).await;
    ```
- **Error Matching:** Use `.contains_error("expected_msg")` for negative tests.
- **Strict Negative Constraint:** **NO MOCKS ALLOWED**. Use actual WASM logic via the Chain Simulator.

---

## 5. Phase 4: Validation & Debugging (The QA Role)
**Objective:** Verify execution and perform surgical fixes.

### The Debugging ReAct Loop:
If `cargo test` fails:
1. **Observation:** Capture the specific Rust compiler or VM error.
2. **Hypothesis:** Identify: Path/Import issue, Snippet mismatch, or Logic assertion failure.
3. **Action:** 
    - If path error: Use `ls -R` to check workspace layout.
    - If snippet mismatch: Re-run Phase 1 Snippet Generation.
    - If logic error: Refine parameters or gas limit.
4. **Conclusion:** Document the failure in the manifest and re-verify. (Max 3 iterations).

### Execution:
1. **Start Environment**: `sc-meta cs start`
2. **Execute Tests**: `cargo test --features chain-simulator-tests`
3. **Teardown**: `sc-meta cs stop`
4. **Cleanup**: Ensure no dangling processes; the `sc-meta cs stop` command should handle the container state.

---

## 6. Final Reporting
Provide a structured report upon completion:

### Test Generation Report
- **State Manifest Finalized:** [Confirmation]
- **Endpoints & Storage Covered:** [List]
- **Total Assertions:** [Count]
- **View-After-Tx Density:** [Percentage - Target 100%]
- **Pass/Fail Summary:** [Result]

---

## Core Operational Constraints
1. **Path Usage:** 
   - **Relative Paths:** MANDATORY for code generation (Imports, Cargo.toml).
   - **Absolute Paths:** REQUIRED for agent tool calls (`view_file`, `run_command` Cwd).
2. **Isolate Changes:** ONLY modify the interactor/tests files. NEVER modify the contract source code unless the user explicitly requested a bug fix.
3. **Idempotency:** The workflow should be runnable multiple times without cluttering the workspace. Wipe `.agent/cache` if the contract ABI changes significantly.
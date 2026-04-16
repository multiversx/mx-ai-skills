---
description: comprehensive reviewer for MultiversX projects covering code, architecture, and documentation compliance
---

# MultiversX General Reviewer

This workflow guides you through a holistic review of a MultiversX project. You must evaluate the codebase not just for syntax, but for architectural soundness and alignment with official MultiversX standards.

## Phase 1: Context & Architecture Analysis
1.  **Analyze the Workspace**:
    -   Identify the core components: Smart Contracts (Rust), Protocol (Go), Microservices (NestJS), or Frontend (React).
    -   Read `README.md` and any architecture diagrams to understand the intended design.
2.  **Identify Key Patterns**:
    -   Note how the project handles:
        -   Token standards (ESDT, NFT, SFT).
        -   Sharding and cross-shard communication.
        -   Data privacy or encryption.
        -   Gas optimization strategies.

## Phase 2: Documentation Verification (CRITICAL)
**You MUST use the `consult_mvx_docs` skill during this phase.**

1.  **Validate Architecture against Standards**:
    -   *Action*: Search the global docs for the specific patterns identified in Phase 1.
    -   *Example*: `grep_search(SearchPath=".../mvx_docs", Query="sharding best practices")`
    -   *Check*: Does the project's approach to cross-shard calls match the recommended "Async Call" or "Callback" patterns in the docs?
2.  **Verify Token Logic**:
    -   *Action*: specific search for ESDT integration guides.
    -   *Check*: Is the project correctly using `MultiESDTNFTTransfer`? Are payment transfers handled securely?
3.  **Check for Deprecations**:
    -   Ensure the project isn't using deprecated modules or patterns flagged in the documentation.

## Phase 3: Code & Security Review
1.  **Smart Contract Security (if Rust)**:
    -   Check for reentrancy vulnerabilities.
    -   Verify use of `checked_add`/`checked_mul`.
    -   Ensure proper access control (OwnerOnly, role-based).
2.  **Protocol/Service Logic (if Go/TS)**:
    -   Check for proper error handling and logging.
    -   Verify safe concurrency practices.
3.  **Performance**:
    -   High-level check for obvious bottlenecks (e.g., large loops in SCs).

## Phase 4: Final Report
Produce a marked-down report summarizing findings:
-   **Architecture Status**: Robust/Needs Improvement (cite specific docs).
-   **Compliance**: Aligned with MVX Standards (Yes/No).
-   **Critical Issues**: Security or Logic bugs.
-   **Recommendations**: Specific steps to fix or improve, citing the relevant documentation files you consulted.

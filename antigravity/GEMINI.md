# MultiversX AI Skills — Antigravity Configuration

## About This Repository

This is the **MultiversX AI Skills** repository — a centralized collection of skills, agent personas, workflows, and documentation for building, auditing, and deploying on MultiversX.

## Global Rules

1. **Never hallucinate.** If you do not understand something exactly, always ask. Request APIs, websites, databases, code, diagrams — anything you need. If you do not have exact data, do not hallucinate — request it.
2. **Security first.** Always use `checked_add`, `checked_mul`, etc. for arithmetic. Follow the Checks-Effects-Interactions pattern. Avoid `unwrap()` or `expect()` — use `sc_panic!` or `require!` for graceful errors. Never use floating point numbers.
3. **Gas efficiency.** Minimize storage writes. Batch operations. Use immutable storage where values don't change after init. Understand cold vs hot storage gas costs.
4. **Simplicity first.** Minimum code that solves the problem. No features beyond what was asked. No abstractions for single-use code. No "flexibility" or "configurability" that wasn't requested.
5. **Surgical changes.** Touch only what you must. Don't "improve" adjacent code. Match existing style. Every changed line should trace directly to the user's request.
6. **Goal-driven execution.** Transform tasks into verifiable goals. State a brief plan with verification checks for each step.

## Repository Structure

Content is organized at the repository root level:

```
mx-ai-skills/
├── skills/        ← Targeted technical capabilities (SKILL.md each)
├── agents/        ← Specialized agent personas (.md each)
├── workflows/     ← Step-by-step procedures
├── references/    ← Guidelines and checklists
├── docs/          ← Curated MultiversX documentation
├── .claude/       ← Claude Code configuration
└── antigravity/   ← Antigravity configuration (this file)
```

## Skills (`skills/`)

Skills are in `skills/<name>/SKILL.md`. Use the skill name to invoke specific expertise.

### Security & Auditing
- `mvx-sc-audit` — Comprehensive smart contract security audit
- `mvx-audit-onchain` — Audit deployed contracts using on-chain data
- `mvx-audit-context` — Build mental model before security review
- `mvx-entry-points` — Map contract attack surface
- `mvx-diff-review` — Review changes between contract versions
- `mvx-fix-verification` — Verify vulnerability fixes
- `mvx-variant-analysis` — Find similar bugs after initial discovery
- `mvx-dapp-audit` — Audit frontend security
- `mvx-static-analysis-patterns` — Manual and automated code analysis
- `mvx-semgrep-creator` — Custom Semgrep security rules
- `mvx-sharp-edges` — Platform-specific gotchas
- `mvx-constant-time` — Timing-safe crypto operations
- `mvx-flash-loan-patterns` — Flash loan resistance
- `mvx-crypto-verification` — Cryptographic verification

### Development
- `mvx-smart-contracts` — Build, test, deploy with Rust
- `mvx-dapp-frontend` — React dApp + sdk-dapp
- `mvx-cross-contract-calls` — Cross-contract calls, callbacks, proxies
- `mvx-cross-contract-storage` — Cross-contract storage patterns
- `mvx-payment-handling` — ESDT payment handling
- `mvx-sc-best-practices` — Best practices checklist
- `mvx-factory-manager` — Factory/manager patterns
- `mvx-vault-pattern` — Vault contract patterns
- `mvx-cache-patterns` — Caching for performance
- `mvx-defi-math` — DeFi math (AMM, lending, fixed-point)
- `mvx-project-architecture` — Project structure

### Testing & Quality
- `mvx-testing-handbook` — Mandos, RustVM, chain simulator
- `mvx-property-testing` — Fuzzing and property-based testing
- `mvx-spec-compliance` — Verify spec compliance
- `mvx-project-culture` — Codebase quality assessment
- `mvx-scenario-migration` — Scenario format migration
- `mvx-test-contract` — Automated contract testing

### DevOps & Operations
- `mvx-deploy-flow` — Deployment workflow
- `mvx-upgrade-flow` — Safe upgrade with verification
- `mvx-debug-tx` — Transaction debugging
- `mvx-token-management` — Token design, issuance, troubleshooting
- `mvx-wasm-debug` — WASM debugging and optimization

### SDK Reference
- `mvx-sdk-go-builders`, `mvx-sdk-go-core`, `mvx-sdk-go-data`, `mvx-sdk-go-interactors`
- `mvx-sdk-js-contracts`, `mvx-sdk-js-core`, `mvx-sdk-js-tokens`, `mvx-sdk-js-wallets`
- `mvx-sdk-py-contracts`, `mvx-sdk-py-core`, `mvx-sdk-py-transactions`, `mvx-sdk-py-wallets`

### Other
- `mvx-blockchain-data` — Blockchain data and indexing
- `mvx-code-analysis` — General code analysis
- `mvx-protocol-experts` — Deep protocol knowledge
- `mvx-clarification-expert` — Targeted clarifying questions
- `mvx-consult-docs` — MultiversX documentation lookup

## Agents (`agents/`)

Agent personas define specialized roles. Adopt when asked to act in that capacity:

- `mvx-smart-contract-developer` — Rust SC expert
- `mvx-full-stack-auditor` — Full stack security
- `mvx-dapp-architect` — Frontend expert (React, sdk-dapp)
- `mvx-defi-specialist` — Tokenomics, ESDT, economics
- `mvx-deployer` — DevOps, builds, upgrades
- `mvx-debugger` — Error analysis, transaction debugging
- `mvx-tester` — QA, Mandos, chain simulation
- `mvx-solution-architect` — System design, infrastructure
- `mvx-full-stack-developer` — Integration specialist
- `mvx-go-specialist` — Go SDK expert
- `mvx-typescript-specialist` — TypeScript SDK expert
- `mvx-python-specialist` — Python SDK expert
- `mvx-integration-specialist` — Cross-system integration
- `mvx-microservice-developer` — NestJS, off-chain logic
- `mvx-token-architect` — Token design and issuance
- `mvx-general-reviewer` — Code and architecture review
- `mvx-rust-chain-sim-tester` — Multi-agent system testing

## Workflows (`workflows/`)

- `mvx-bridge-guide.md` — Asset bridging and Ad-Astra Bridge
- `mvx-es-indexer.md` — Elasticsearch data queries
- `mvx-specs-writer.md` — Technical specification writing
- `mvx-task-writer.md` — Work delegation orchestration

## References (`references/`)

- `mvx-sc-best-practices.md` — SC best practices checklist
- `mvx-validator-setup.md` — Validator/Observer setup
- `production-ready.md` — Production readiness checklist

## Documentation (`docs/`)

Curated docs covering: advanced SC development, composability, DeFi interfaces, ESDT tokens, framework internals, SDK deep-dives, testing/simulation, sovereign chains, and more.

## MultiversX Standards

### Framework: `multiversx-sc`
- Annotations: `#[multiversx_sc::contract]`, `#[init]`, `#[endpoint]`, `#[view]`, `#[payable("*")]`
- Storage: `SingleValueMapper`, `VecMapper`, `UnorderedSetMapper`, `MapMapper`
- API: `self.blockchain()`, `self.send()`, `self.crypto()`
- ESDT: Built-in token transfers, minting, SFT/NFT metadata

### Testing
- Mandos scenarios (`.scen.json`) for all endpoints
- RustVM for unit testing complex logic
- Chain simulator for integration testing

### Build & Tooling
- `sc-meta all build` for WASM binaries
- `Cargo.lock` committed for reproducibility
- `wasm-opt` via framework build tools

## MCP Server (Optional)

If a MultiversX MCP server is available (`multiversx-sc-mcp` or `multiversx-mcp-server`), skills can leverage its tools for on-chain operations. The MCP server is not required — all skills work without it.

# MultiversX AI Skills — Claude Code Configuration

## About This Repository

This is the **MultiversX AI Skills** repository — a centralized collection of skills, agent personas, workflows, and documentation for building, auditing, and deploying on MultiversX. It serves any AI coding agent (Claude Code, Antigravity, Cursor, Windsurf, etc.).

## Global Rules

1. **Never hallucinate.** If you don't have exact data, ask. Request APIs, websites, code — anything you need.
2. **Security first.** Checked arithmetic (`checked_add`, `checked_mul`), CEI pattern, no `unwrap()`/`expect()` — use `sc_panic!` or `require!`.
3. **No floating point.** Use `BigUint`/`BigInt` for all financial calculations.
4. **Gas efficiency.** Minimize storage writes, batch operations, use immutable storage where applicable.
5. **Simplicity first.** Minimum code that solves the problem. No speculative abstractions.
6. **Surgical changes.** Touch only what you must. Match existing style.

## How to Use This Repository

### Skills (`skills/`)

Skills are targeted technical capabilities in `skills/<name>/SKILL.md`. Use them when you need specific expertise:

#### Security & Auditing
| Skill | When to Use |
|-------|-------------|
| `mvx-sc-audit` | Comprehensive smart contract security audit |
| `mvx-audit-onchain` | Audit a deployed contract using on-chain data |
| `mvx-audit-context` | Build mental model before deep security review |
| `mvx-entry-points` | Map contract attack surface |
| `mvx-diff-review` | Review changes between contract versions |
| `mvx-fix-verification` | Verify vulnerability fixes |
| `mvx-variant-analysis` | Find similar bugs after initial discovery |
| `mvx-dapp-audit` | Audit frontend security |
| `mvx-static-analysis-patterns` | Manual and automated code analysis patterns |
| `mvx-semgrep-creator` | Create custom Semgrep security scanning rules |
| `mvx-sharp-edges` | Platform-specific gotchas and pitfalls |
| `mvx-constant-time` | Verify timing-safe crypto operations |
| `mvx-flash-loan-patterns` | Flash loan resistance patterns |
| `mvx-crypto-verification` | Cryptographic verification patterns |

#### Development
| Skill | When to Use |
|-------|-------------|
| `mvx-smart-contracts` | Build, test, deploy smart contracts with Rust |
| `mvx-dapp-frontend` | React dApp integration with sdk-dapp |
| `mvx-cross-contract-calls` | Cross-contract calls, callbacks, proxies |
| `mvx-cross-contract-storage` | Cross-contract storage patterns |
| `mvx-payment-handling` | ESDT payment handling patterns |
| `mvx-sc-best-practices` | Smart contract best practices checklist |
| `mvx-factory-manager` | Factory/manager contract patterns |
| `mvx-vault-pattern` | Vault contract patterns |
| `mvx-cache-patterns` | Caching patterns for performance |
| `mvx-defi-math` | DeFi math (AMM, lending, fixed-point) |
| `mvx-project-architecture` | Project structure and architecture |

#### Testing & Quality
| Skill | When to Use |
|-------|-------------|
| `mvx-testing-handbook` | Mandos scenarios, RustVM, chain simulator |
| `mvx-property-testing` | Fuzzing and property-based testing |
| `mvx-spec-compliance` | Verify implementations match specs |
| `mvx-project-culture` | Assess codebase quality and maturity |
| `mvx-scenario-migration` | Migrate between scenario formats |
| `mvx-test-contract` | Automated contract testing workflow |

#### DevOps & Operations
| Skill | When to Use |
|-------|-------------|
| `mvx-deploy-flow` | Step-by-step deployment workflow |
| `mvx-upgrade-flow` | Safe upgrade with pre/post verification |
| `mvx-debug-tx` | Debug transactions (decode, simulate, trace) |
| `mvx-token-management` | Design, issue, inspect, troubleshoot tokens |
| `mvx-wasm-debug` | Debug and optimize WASM binaries |

#### SDK Reference
| Skill | When to Use |
|-------|-------------|
| `mvx-sdk-go-*` | Go SDK (builders, core, data, interactors) |
| `mvx-sdk-js-*` | JavaScript SDK (contracts, core, tokens, wallets) |
| `mvx-sdk-py-*` | Python SDK (contracts, core, transactions, wallets) |

#### Other
| Skill | When to Use |
|-------|-------------|
| `mvx-blockchain-data` | Query blockchain data and indexing |
| `mvx-code-analysis` | General code analysis patterns |
| `mvx-protocol-experts` | Deep protocol knowledge (sharding, ESDT) |
| `mvx-clarification-expert` | Ask targeted clarifying questions |
| `mvx-consult-docs` | Look up MultiversX documentation |

### Agents (`agents/`)

Agent personas in `agents/<name>.md` define specialized roles. Adopt a persona when asked to act in that capacity:

| Agent | Role |
|-------|------|
| `mvx-smart-contract-developer` | Rust SC expert — idiomatic, gas-efficient, secure |
| `mvx-full-stack-auditor` | Full stack security (backend + frontend + integration) |
| `mvx-sc-auditor` | Smart contract vulnerability hunting (Rust SC audit) |
| `mvx-dapp-architect` | Frontend expert — React, sdk-dapp, UX |
| `mvx-defi-specialist` | Tokenomics, ESDT, economic mechanics |
| `mvx-deployer` | DevOps — reproducible builds, upgrades, security |
| `mvx-debugger` | Error analysis, transaction debugging, simulation |
| `mvx-tester` | QA — RustVM tests, Mandos scenarios, chain simulation |
| `mvx-solution-architect` | System design, sharding, infrastructure |
| `mvx-full-stack-developer` | Integration (Rust SC + React + glue) |
| `mvx-go-specialist` | mx-sdk-go, Go patterns, blockchain integration |
| `mvx-typescript-specialist` | sdk-js v15, TypeScript, blockchain integration |
| `mvx-python-specialist` | sdk-py v2, Python, blockchain scripting |
| `mvx-integration-specialist` | Cross-system integration, payments, API patterns |
| `mvx-microservice-developer` | NestJS, transaction processing, off-chain logic |
| `mvx-token-architect` | ESDT/SFT/NFT token design and issuance |
| `mvx-general-reviewer` | Code, architecture, documentation review |
| `mvx-rust-chain-sim-tester` | Multi-agent system testing with chain simulator |

### Workflows (`workflows/`)

Step-by-step procedures for specific tasks:
- `mvx-bridge-guide.md` — Bridging assets and Ad-Astra Bridge listing
- `mvx-es-indexer.md` — Querying MultiversX data via Elasticsearch
- `mvx-specs-writer.md` — Turning architecture into technical specifications
- `mvx-task-writer.md` — Orchestrating work delegation to specialized agents

### References (`references/`)

Guidelines and checklists:
- `mvx-sc-best-practices.md` — Smart contract best practices checklist
- `mvx-validator-setup.md` — Validator/Observer node setup guide
- `production-ready.md` — Production readiness verification checklist

### Documentation (`docs/`)

Curated MultiversX documentation covering: advanced SC development, composability, DeFi interfaces, ESDT tokens, framework internals, SDK deep-dives, testing/simulation, and more.

## MCP Server (Optional)

If a MultiversX MCP server is available (`multiversx-sc-mcp` or `multiversx-mcp-server`), skills can leverage its tools for on-chain operations (queries, storage reads, deployments, etc.). Skills note this where applicable. The MCP server is **not required** — all skills work without it.

## MultiversX Smart Contract Standards

### Framework: `multiversx-sc`
- Annotations: `#[multiversx_sc::contract]`, `#[init]`, `#[endpoint]`, `#[view]`, `#[payable("*")]`
- Storage: `SingleValueMapper`, `VecMapper`, `UnorderedSetMapper`, `MapMapper`
- API: `self.blockchain()`, `self.send()`, `self.crypto()`
- ESDT: Built-in token transfers, minting, SFT/NFT metadata

### Testing
- Mandos scenarios (`.scen.json`) for all endpoints
- RustVM for unit testing complex logic
- Chain simulator for integration testing
- `sc-meta test-gen` to verify scenario freshness

### Build
- `sc-meta all build` for WASM binaries
- Ensure `Cargo.lock` committed for reproducibility
- Use `wasm-opt` via framework build tools

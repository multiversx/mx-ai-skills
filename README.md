# MultiversX AI Skills

> AI agents with specialized knowledge, roles, and workflows for the MultiversX ecosystem.

A centralized collection of **53 skills**, **18 agent personas**, **4 workflows**, and **20 curated docs** designed to equip AI agents with deep technical knowledge for building, auditing, and deploying on MultiversX.

Works with any AI coding agent: **Claude Code**, **Antigravity**, **Cursor**, **Windsurf**, **Codex**, and more.

---

## Quick Start

### Claude Code

Add as a resource directory:
```bash
claude --add-dir /path/to/mx-ai-skills
```

Or clone into your project:
```bash
git clone https://github.com/multiversx/mx-ai-skills.git .ai-skills
```

### Antigravity

Skills are automatically available via the `antigravity/GEMINI.md` workspace rules.

### Any AI Agent (OpenSkills)

```bash
npx openskills install multiversx/mx-ai-skills
```

---

## Repository Structure

```
mx-ai-skills/
├── skills/          ← 53 targeted technical capabilities
│   └── mvx-*/SKILL.md
├── agents/          ← 18 specialized agent personas
│   └── mvx-*.md
├── workflows/       ← 4 step-by-step procedures
├── references/      ← 3 guideline checklists
├── docs/            ← 20 curated documentation files
├── .claude/         ← Claude Code configuration
│   └── CLAUDE.md
└── antigravity/     ← Antigravity configuration
    └── GEMINI.md
```

---

## Skills

### Security & Auditing
| Skill | Description |
|-------|-------------|
| `mvx-sc-audit` | Comprehensive SC security audit (1200+ lines, merged from 3 sources) |
| `mvx-audit-onchain` | Audit deployed contracts using on-chain data |
| `mvx-audit-context` | Build mental model before security review |
| `mvx-entry-points` | Map contract attack surface |
| `mvx-diff-review` | Review changes between contract versions |
| `mvx-fix-verification` | Verify vulnerability fixes |
| `mvx-variant-analysis` | Find similar bugs after discovery |
| `mvx-dapp-audit` | Frontend security audit |
| `mvx-static-analysis-patterns` | Manual and automated code analysis |
| `mvx-semgrep-creator` | Custom Semgrep security rules |
| `mvx-sharp-edges` | Platform-specific gotchas (619 lines) |
| `mvx-constant-time` | Timing-safe crypto operations |
| `mvx-flash-loan-patterns` | Flash loan resistance |
| `mvx-crypto-verification` | Cryptographic verification |

### Development
| Skill | Description |
|-------|-------------|
| `mvx-smart-contracts` | Build, test, deploy with Rust |
| `mvx-dapp-frontend` | React dApp + sdk-dapp integration |
| `mvx-cross-contract-calls` | Cross-contract calls, callbacks, proxies |
| `mvx-cross-contract-storage` | Cross-contract storage patterns |
| `mvx-payment-handling` | ESDT payment handling |
| `mvx-sc-best-practices` | Best practices checklist |
| `mvx-factory-manager` | Factory/manager patterns |
| `mvx-vault-pattern` | Vault contract patterns |
| `mvx-cache-patterns` | Caching for performance |
| `mvx-defi-math` | DeFi math (AMM, lending, fixed-point) |
| `mvx-project-architecture` | Project structure |

### Testing & Quality
| Skill | Description |
|-------|-------------|
| `mvx-testing-handbook` | Mandos, RustVM, chain simulator |
| `mvx-property-testing` | Fuzzing and property-based testing |
| `mvx-spec-compliance` | Verify spec compliance |
| `mvx-test-contract` | Automated contract testing |
| `mvx-scenario-migration` | Scenario format migration |
| `mvx-project-culture` | Codebase quality assessment |

### DevOps & Operations
| Skill | Description |
|-------|-------------|
| `mvx-deploy-flow` | Step-by-step deployment workflow |
| `mvx-upgrade-flow` | Safe upgrade with verification |
| `mvx-debug-tx` | Transaction debugging (decode, simulate) |
| `mvx-token-management` | Token design, issuance, troubleshooting |
| `mvx-wasm-debug` | WASM debugging and optimization |

### SDK Reference
| Skill | Description |
|-------|-------------|
| `mvx-sdk-go-*` | Go SDK (builders, core, data, interactors) |
| `mvx-sdk-js-*` | JavaScript SDK (contracts, core, tokens, wallets) |
| `mvx-sdk-py-*` | Python SDK (contracts, core, transactions, wallets) |

### Other
| Skill | Description |
|-------|-------------|
| `mvx-blockchain-data` | Blockchain data and indexing |
| `mvx-code-analysis` | General code analysis |
| `mvx-protocol-experts` | Deep protocol knowledge |
| `mvx-clarification-expert` | Targeted clarifying questions |
| `mvx-consult-docs` | Documentation lookup |

---

## Agent Personas

| Agent | Role |
|-------|------|
| `mvx-sc-auditor` | Smart contract security auditor |
| `mvx-smart-contract-developer` | Rust SC expert |
| `mvx-full-stack-auditor` | Full stack security |
| `mvx-dapp-architect` | Frontend expert (React, sdk-dapp) |
| `mvx-defi-specialist` | Tokenomics, ESDT, economics |
| `mvx-deployer` | DevOps, builds, upgrades |
| `mvx-debugger` | Error analysis, debugging |
| `mvx-tester` | QA, Mandos, chain simulation |
| `mvx-solution-architect` | System design, infrastructure |
| `mvx-full-stack-developer` | Integration (SC + frontend + glue) |
| `mvx-go-specialist` | Go SDK expert |
| `mvx-typescript-specialist` | TypeScript SDK expert |
| `mvx-python-specialist` | Python SDK expert |
| `mvx-integration-specialist` | Cross-system integration |
| `mvx-microservice-developer` | NestJS, off-chain logic |
| `mvx-token-architect` | Token design and issuance |
| `mvx-general-reviewer` | Code and architecture review |
| `mvx-rust-chain-sim-tester` | Multi-agent system testing |

---

## MCP Server (Optional)

Skills can optionally leverage a MultiversX MCP server (`multiversx-sc-mcp` or `multiversx-mcp-server`) for on-chain operations. Install with:

```bash
claude mcp add multiversx-sc -- npx -y multiversx-sc-mcp
```

The MCP server provides 41 tools for querying, deploying, upgrading, and testing contracts directly from AI agents. Skills note where MCP tools can accelerate steps — but all skills work without it.

---

## Principles

- **Never hallucinate** — ask for data rather than guessing
- **Security first** — checked arithmetic, CEI pattern, no `unwrap()`
- **No floating point** — `BigUint`/`BigInt` for all calculations
- **Gas efficiency** — minimize storage writes, batch operations
- **Simplicity first** — minimum code that solves the problem
- **Reproducibility** — all builds and tests must be traceable

---

## Contributing

1. Add new skills to `skills/mvx-<name>/SKILL.md`
2. Follow the frontmatter format:
   ```yaml
   ---
   name: mvx-skill-name
   description: Brief description. Use when [trigger conditions].
   ---
   ```
3. Use `mvx-` prefix with kebab-case
4. Include practical code examples and checklists
5. Update CLAUDE.md and GEMINI.md with new entries

---

## License

MIT — see [LICENSE](LICENSE) for details.

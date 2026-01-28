# MultiversX AI Skills - Claude Code Configuration

> Workspace rules and best practices for Claude Code working with MultiversX projects.

---

## Core Principles

### Never Hallucinate
- If uncertain about MultiversX APIs, storage patterns, or protocol behavior, **ask for clarification**
- Do not guess at function signatures, storage mapper behavior, or gas costs
- When documentation is missing, request the specific information needed

### Security First
- Always use checked arithmetic (`BigUint`, `checked_add`, etc.) for financial calculations
- Follow the Checks-Effects-Interactions pattern for all state-changing operations
- Validate all inputs, especially in `#[payable]` endpoints
- Never trust frontend-provided data in smart contracts

---

## MultiversX Development Standards

### Smart Contracts (Rust)

**Framework**: `multiversx-sc` (latest stable version)

**Key Rules**:
- Use `BigUint` for all token amounts and financial math
- Never use floating point (`f32`/`f64`) - non-deterministic
- Avoid `unwrap()` - use `require!` or `sc_panic!` with meaningful messages
- Use `SingleValueMapper` with address key instead of `MapMapper` when not iterating
- Always validate token IDs in `#[payable("*")]` endpoints
- Initialize new storage mappers in `#[upgrade]`, not `#[init]`

**Storage Mappers**:
| Mapper | Use Case | Cost |
|--------|----------|------|
| `SingleValueMapper` | Single value or with key parameter | 1 slot |
| `VecMapper` | Ordered list (1-indexed!) | N slots |
| `SetMapper` | Unique items with iteration | 2N+1 slots |
| `UnorderedSetMapper` | Unique items, no order needed | N+1 slots |
| `MapMapper` | Key-value with iteration needed | 4N+1 slots |

**Build & Test**:
```bash
sc-meta all build          # Build all contracts
sc-meta test               # Run all tests
cargo clippy               # Lint check
```

### Protocol Development (Go)

**Repository**: `mx-chain-go`

**Key Rules**:
- Never iterate over Go maps for deterministic output - sort keys first
- Never use `time.Now()` in block processing - use `header.TimeStamp`
- Always capture loop variables in goroutines: `item := item`
- Protect shared maps with `sync.Mutex` or use `sync.Map`

### Microservices (TypeScript)

**SDKs**: `@multiversx/sdk-core`, `@multiversx/sdk-nestjs`

**Key Rules**:
- Wait for transaction finality before processing (handle reorgs)
- Use native BigInt for token amounts, not JavaScript numbers
- Validate all API responses before processing

### Frontend (React)

**SDK**: `@multiversx/sdk-dapp`

**Key Rules**:
- Call `initApp()` BEFORE rendering React application
- Always `refreshAccount()` before creating transactions
- Use `useGetNetworkConfig()` for chain ID, never hardcode
- Never store private keys or mnemonics in any storage

---

## Claude Code Best Practices

### Think Before Coding
- Surface assumptions explicitly before implementing
- Present alternatives when multiple valid approaches exist
- Ask clarifying questions rather than guessing requirements

### Simplicity First
- No speculative features - implement only what's requested
- Minimal code changes - avoid unnecessary refactoring
- Don't add error handling for impossible scenarios

### Surgical Changes
- Touch only what's necessary to complete the task
- Preserve existing code style and patterns
- Don't add documentation or comments unless requested

### Goal-Driven Execution
- Define clear, verifiable success criteria
- Test changes before considering complete
- Verify builds pass and tests succeed

---

## Available Skills Reference

### Development Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-smart-contracts` | Build, test, deploy smart contracts with Rust |
| `multiversx-dapp-frontend` | React dApp integration with sdk-dapp |

### Auditing Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-audit-context` | Build mental models before security audits |
| `multiversx-entry-points` | Map contract attack surface systematically |
| `multiversx-diff-review` | Review changes between contract versions |
| `multiversx-fix-verification` | Verify vulnerability fixes are complete |
| `multiversx-dapp-audit` | Audit frontend security and wallet integration |
| `multiversx-spec-compliance` | Verify implementations match specifications |
| `multiversx-project-culture` | Assess codebase quality and maturity |
| `multiversx-variant-analysis` | Find similar bugs after initial discovery |

### Security Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-static-analysis` | Manual and automated code analysis patterns |
| `multiversx-constant-time` | Verify crypto operations are timing-safe |
| `multiversx-sharp-edges` | Catalog of platform-specific gotchas |

### Testing Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-property-testing` | Fuzzing and property-based testing |

### Tooling Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-semgrep-creator` | Create custom security scanning rules |
| `multiversx-wasm-debug` | Debug and optimize WASM binaries |

### General Skills
| Skill | Purpose |
|-------|---------|
| `multiversx-clarification-expert` | Ask targeted clarifying questions |
| `multiversx-protocol-experts` | Deep protocol knowledge (sharding, ESDT, etc.) |

---

## Network Endpoints

| Network | API URL | Chain ID |
|---------|---------|----------|
| **Devnet** | `https://devnet-api.multiversx.com` | `D` |
| **Testnet** | `https://testnet-api.multiversx.com` | `T` |
| **Mainnet** | `https://api.multiversx.com` | `1` |

### Gateway URLs (for direct node access)
| Network | Gateway URL |
|---------|-------------|
| Devnet | `https://devnet-gateway.multiversx.com` |
| Testnet | `https://testnet-gateway.multiversx.com` |
| Mainnet | `https://gateway.multiversx.com` |

---

## Documentation Links

### Official Documentation
- **Docs Portal**: https://docs.multiversx.com
- **Smart Contracts**: https://docs.multiversx.com/developers/smart-contracts
- **SDK Reference**: https://docs.multiversx.com/sdk-and-tools/overview

### GitHub Repositories
- **Rust Framework**: https://github.com/multiversx/mx-sdk-rs
- **JS/TS SDK**: https://github.com/multiversx/mx-sdk-js
- **React SDK**: https://github.com/multiversx/mx-sdk-dapp
- **Protocol (Go)**: https://github.com/multiversx/mx-chain-go

### Example Contracts
- **Official Examples**: https://github.com/multiversx/mx-sdk-rs/tree/master/contracts/examples

---

## Quick Reference: Critical Checks

### Before Deploying Any Contract
- [ ] All `#[payable]` endpoints validate token IDs
- [ ] No `unwrap()` without justification
- [ ] No floating point arithmetic
- [ ] `BigUint` used for all financial calculations
- [ ] Callbacks handle error cases
- [ ] `#[upgrade]` initializes new storage mappers
- [ ] All tests pass (`sc-meta test`)
- [ ] Clippy passes (`cargo clippy`)
- [ ] Build succeeds (`sc-meta all build`)

### Before Merging Any PR
- [ ] Changes are minimal and focused
- [ ] No new security vulnerabilities introduced
- [ ] Tests cover new functionality
- [ ] Documentation updated if behavior changed

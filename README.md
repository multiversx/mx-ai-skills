# MultiversX AI Skills (mx-ai-skills)

> **AI Agents with specialized knowledge, roles, and workflows for the MultiversX ecosystem.**

This repository is a central hub for **MultiversX AI Expertise**. It provides a structured collection of specialized "Skills", "Global Workflows" (Roles), and "Documentation" designed to equip AI agents with the deep technical knowledge required to build, audit, and optimize on MultiversX.

---

## Quick Install

Install skills for any AI coding agent (Cursor, Windsurf, Codex, etc.):

```bash
npx openskills install multiversx/mx-ai-skills
```

Or using Vercel's skills CLI:

```bash
npx skills install multiversx/mx-ai-skills
```

---

## Overview

Developing on MultiversX requires specific expertise in Rust (via `multiversx-sc`), WASM optimization, sharding awareness, and unique token standards (ESDT). This repository formalizes that expertise into machine-readable and agent-executable modules.

### Core Value Props:
- **Universal Compatibility**: Works with 30+ AI coding agents via OpenSkills standard
- **Role-Based Execution**: Instantiate specialized agents like *MultiversX Smart Contract Developer* or *Full Stack Auditor*
- **Granular Skills**: Specific toolsets for *Security Auditing*, *Static Analysis*, *Property Testing*, and more
- **Security First**: Built-in rules for arithmetic safety, reentrancy protection, and MultiversX-specific "sharp edges"

---

## Repository Structure

```text
mx-ai-skills/
├── skills/                     # Universal Skills (OpenSkills compatible)
│   ├── multiversx-smart-contracts/
│   ├── multiversx-dapp-frontend/
│   ├── multiversx-audit-context/
│   ├── multiversx-static-analysis/
│   └── ... (18 total skills)
│
├── antigravity/                # Antigravity-specific (legacy format)
│   ├── global_workflows/       # Agent Roles/Personas
│   ├── skills/                 # Original skills (underscore naming)
│   ├── mvx_docs/               # Curated documentation
│   └── GEMINI.md               # Global rules
│
├── README.md
└── LICENSE
```

---

## Available Skills

### Development
| Skill | Description |
|-------|-------------|
| `multiversx-smart-contracts` | Build, test, deploy smart contracts with Rust |
| `multiversx-dapp-frontend` | React dApp integration with sdk-dapp |

### Security & Auditing
| Skill | Description |
|-------|-------------|
| `multiversx-audit-context` | Build mental models before security audits |
| `multiversx-entry-points` | Map contract attack surface |
| `multiversx-diff-review` | Review changes between contract versions |
| `multiversx-fix-verification` | Verify vulnerability fixes |
| `multiversx-dapp-audit` | Audit frontend security |
| `multiversx-static-analysis` | Manual and automated code analysis |
| `multiversx-constant-time` | Verify timing-safe crypto operations |
| `multiversx-variant-analysis` | Find similar bugs after initial discovery |

### Testing & Quality
| Skill | Description |
|-------|-------------|
| `multiversx-property-testing` | Fuzzing and property-based testing |
| `multiversx-spec-compliance` | Verify implementations match specs |
| `multiversx-project-culture` | Assess codebase quality and maturity |

### Tooling & Reference
| Skill | Description |
|-------|-------------|
| `multiversx-semgrep-creator` | Create custom security scanning rules |
| `multiversx-wasm-debug` | Debug and optimize WASM binaries |
| `multiversx-sharp-edges` | Platform-specific gotchas catalog |
| `multiversx-protocol-experts` | Deep protocol knowledge (sharding, ESDT) |
| `multiversx-clarification-expert` | Ask targeted clarifying questions |

---

## Usage

### For Any AI Coding Agent (Recommended)

```bash
# Install globally
npx openskills install multiversx/mx-ai-skills -g

# Or install in current project
npx openskills install multiversx/mx-ai-skills
```

### For Antigravity

Use the original format in `antigravity/skills/` with underscore naming:
```
antigravity/skills/mvx_static_analysis/SKILL.md
```

### Manual Installation

```bash
# Clone to your project
git clone https://github.com/multiversx/mx-ai-skills.git .ai-skills

# Copy skills to your agent's expected location
cp -r .ai-skills/skills ~/.cursor/skills/        # Cursor
cp -r .ai-skills/skills ~/.agent/skills/         # Generic
```

---

## Principles & Standards

- **Never Hallucinate**: Always ask for clarification when uncertain
- **Security First**: Checked arithmetic, input validation, CEI pattern
- **Gas Efficiency**: Minimize storage writes, optimize WASM size
- **Reproducibility**: All builds and tests must be traceable

---

## Contributing

1. Add new skills to `skills/<skill-name>/SKILL.md`
2. Follow the YAML frontmatter format:
   ```yaml
   ---
   name: skill-name
   description: Brief description. Use when [trigger conditions].
   ---
   ```
3. Use kebab-case for skill names
4. Include practical code examples and checklists

---

## License

This repository is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

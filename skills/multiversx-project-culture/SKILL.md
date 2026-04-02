---
name: multiversx-project-culture
description: "Assess MultiversX codebase quality and development maturity by scoring documentation, testing practices, code hygiene, dependency management, and CI/CD readiness. Use when onboarding to a new MultiversX project, estimating security audit scope and effort, evaluating integration or investment risk, or prioritizing code review focus areas."
---

# MultiversX Project Culture & Code Maturity Assessment

Evaluates the quality, reliability, and development maturity of a MultiversX smart contract codebase. The agent scores five dimensions — documentation, testing, code hygiene, dependencies, and CI/CD — to calibrate audit depth, flag risks, and recommend improvements.

## Workflow

1. **Gather project artifacts** — locate README, config files, test directories, and CI pipelines.
2. **Score each dimension** using the maturity matrix (Section 5).
3. **Flag red and yellow flags** (Section 6).
4. **Generate assessment report** from the template (Section 7).
5. **Recommend improvements** ranked by maturity level (Section 8).

## 1. Documentation Quality

### Documentation Presence Checklist

| Item | Location | Status |
|------|----------|--------|
| README.md | Project root | [ ] Present [ ] Useful |
| Build instructions | README or BUILDING.md | [ ] Present [ ] Tested |
| API documentation | docs/ or inline | [ ] Present [ ] Complete |
| Architecture overview | docs/ or specs/ | [ ] Present |
| Deployment guide | README or DEPLOY.md | [ ] Present |

### MultiversX-Specific Documentation

| Item | Purpose | Status |
|------|---------|--------|
| `multiversx.json` | Standard build configuration | [ ] Present |
| `sc-config.toml` | Contract configuration | [ ] Present |
| `snippets.sh` | Interaction scripts | [ ] Helpful |
| `interaction/` | Deployment/call scripts | [ ] Very helpful |

### Documentation Quality Scoring

```
HIGH QUALITY (3/3):
- README explains purpose, build, test, deploy
- Architecture diagrams present
- API fully documented with examples
- Security model documented

MEDIUM QUALITY (2/3):
- README with basic instructions
- Some inline documentation
- Partial API coverage

LOW QUALITY (1/3):
- Minimal or no README
- No inline comments
- No architectural documentation
```

## 2. Testing Culture Assessment

### Test Discovery Commands

```bash
# Check for Rust unit tests
grep -r "#\[test\]" src/

# Check for scenario tests
ls -la scenarios/

# Check for integration tests
ls -la tests/
```

### Scenario Test Coverage

| Coverage Level | Indicators |
|----------------|------------|
| **Excellent** | Every endpoint has scenario, edge cases tested, failure paths covered |
| **Good** | All endpoints have basic scenarios, some edge cases |
| **Minimal** | Only `deploy.scen.json` or few scenarios |
| **None** | No `scenarios/` directory |

### Test Quality Example

```rust
// HIGH QUALITY: Tests cover edge cases and error paths
#[test]
fn test_deposit_zero_amount() { }  // Boundary
#[test]
fn test_deposit_max_amount() { }   // Boundary
#[test]
fn test_deposit_wrong_token() { }  // Error case
#[test]
fn test_deposit_unauthorized() { } // Access control

// LOW QUALITY: Only happy path
#[test]
fn test_deposit() { }  // Basic only
```

### CI Pipeline Checklist

| CI Feature | Status |
|------------|--------|
| Automated builds | [ ] Present |
| Test execution | [ ] Present |
| Coverage reporting | [ ] Present |
| Lint/format checks | [ ] Present |
| Security scanning | [ ] Present |

## 3. Code Hygiene Assessment

### Linter Compliance

```bash
cargo clippy -- -W clippy::all
cargo fmt --check
```

| Clippy Status | Interpretation |
|---------------|----------------|
| 0 warnings | Excellent hygiene |
| < 10 warnings | Good, minor issues |
| 10-50 warnings | Needs attention |
| > 50 warnings | Poor hygiene |

### Magic Number Detection

```bash
grep -rn "[^a-zA-Z_][0-9]\{2,\}[^a-zA-Z0-9_]" src/
```

**Bad:**
```rust
let fee = amount * 3 / 100;  // Magic 3%
```

**Good:**
```rust
const FEE_PERCENT: u64 = 3;
const FEE_DENOMINATOR: u64 = 100;
let fee = amount * FEE_PERCENT / FEE_DENOMINATOR;
```

### Error Handling Patterns

| Pattern | Quality Indicator |
|---------|-------------------|
| Mostly `require!` with messages | Good |
| Mixed `require!` and `unwrap()` | Needs review |
| Mostly `unwrap()` | Poor — panic vulnerabilities |

## 4. Dependency Management

### Version Pinning

```toml
# GOOD: Specific versions
[dependencies.multiversx-sc]
version = "0.64.1"

# BAD: Wildcard versions
[dependencies.multiversx-sc]
version = "*"
```

### Dependency Audit

```bash
cargo audit
```

| Status | Interpretation |
|--------|----------------|
| Cargo.lock committed | Reproducible builds |
| Cargo.lock not committed | Version drift risk |

## 5. Maturity Scoring Matrix

| Category | Weight | High (3) | Medium (2) | Low (1) |
|----------|--------|----------|------------|---------|
| Documentation | 20% | Complete | Partial | Minimal |
| Testing | 30% | Full coverage | Basic coverage | Minimal |
| Code hygiene | 20% | Clean Clippy | Few warnings | Many issues |
| Dependencies | 15% | Pinned, audited | Pinned | Wildcards |
| CI/CD | 15% | Full pipeline | Basic | None |

### Interpretation

| Score | Maturity | Audit Focus |
|-------|----------|-------------|
| 2.5-3.0 | High | Business logic, edge cases |
| 1.5-2.4 | Medium | Broad review, verify basics |
| 1.0-1.4 | Low | Everything, assume issues exist |

## 6. Red and Yellow Flags

### Red Flags (Immediate Concerns)

| Red Flag | Risk |
|----------|------|
| No tests at all | Logic likely untested |
| Wildcard dependencies | Supply chain vulnerability |
| `unsafe` blocks without justification | Memory safety issues |
| Excessive `unwrap()` | Panic vulnerabilities |
| No README | Maintenance abandoned? |
| Outdated framework version | Known vulnerabilities |

### Yellow Flags

| Yellow Flag | Concern |
|-------------|---------|
| Few scenario tests | Limited coverage |
| Some Clippy warnings | Technical debt |
| Incomplete documentation | Knowledge silos |
| No CI/CD | Regression risk |

## 7. Assessment Report Template

```markdown
# Project Maturity Assessment

**Project**: [Name]
**Version**: [Version]
**Date**: [Date]
**Assessor**: [Name]

## Summary Score: [X/3.0] - [HIGH/MEDIUM/LOW] Maturity

## Documentation (Score: X/3)
- README: [Present/Missing]
- Build instructions: [Tested/Untested/Missing]
- Architecture docs: [Complete/Partial/Missing]

## Testing (Score: X/3)
- Unit tests: [X tests found]
- Scenario tests: [X scenarios covering Y endpoints]
- Edge case coverage: [Good/Partial/Minimal]

## Code Hygiene (Score: X/3)
- Clippy warnings: [X warnings]
- Magic numbers: [X instances]
- Error handling: [Good/Needs work]

## Dependencies (Score: X/3)
- Cargo.lock: [Committed/Missing]
- Version pinning: [All/Some/None]

## CI/CD (Score: X/3)
- Build automation: [Yes/No]
- Test automation: [Yes/No]

## Recommendations
1. [Highest priority improvement]
2. [Second priority]
3. [Third priority]

## Audit Focus Areas
1. [Area based on weaknesses]
2. [Area based on risk]
```

## 8. Improvement Recommendations by Level

### For Low Maturity Projects
1. Add basic README with build instructions
2. Create scenario tests for all endpoints
3. Fix all Clippy warnings
4. Pin dependency versions
5. Set up basic CI

### For Medium Maturity Projects
1. Expand test coverage to edge cases
2. Add architecture documentation
3. Document security considerations
4. Add coverage reporting

### For High Maturity Projects
1. Formal verification consideration
2. Fuzzing and property testing
3. External security audit
4. Bug bounty program

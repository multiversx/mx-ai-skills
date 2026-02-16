---
name: mvx_static_analysis
description: Manual and automated static analysis patterns for Rust/Go (unsafe usage, unverified unwrap, float arithmetic).
---

# MultiversX Static Analysis

This skill guides you through static analysis of MultiversX codebases, focusing on patterns that often indicate vulnerabilities.

---

## Execution Order

Follow these steps in sequence. Do not skip ahead; earlier steps catch compilation errors that make later pattern-matching unreliable.

### Step 1 — Compiler and Linter

Run the compiler and linter first. Fix any errors before proceeding.

```bash
cargo check
cargo clippy -- -D warnings
```

Both commands must exit with **zero warnings** before moving to Step 2.

### Step 2 — Grep Patterns

Run every grep pattern listed in the checklists below (Rust and Go), in the order they appear. Record each match in the output table.

### Step 3 — Semgrep Rules

Run all Semgrep rules (see the Semgrep section below). Add any new findings to the output table.

### Step 4 — Manual Review of Logical Patterns

Manually inspect the logical patterns described in the checklists (token ID validation, callback state, etc.). These cannot be fully automated and require human judgement.

---

## 1. Rust Smart Contracts (`multiversx-sc`)

### Critical Grep Patterns

- **Unsafe Code**:
    - `grep -r "unsafe"`: Valid only for FFI or specific optimizations. Generally forbidden in SCs.
- **Panic Inducers**:
    - `grep -r "unwrap()"`: **High Risk**. Should be `sc_panic!` or `unwrap_or_else`.
    - `grep -r "expect("`: **High Risk**.
- **Floating Point**:
    - `grep -r "f32"` / `grep -r "f64"`: **Critical**. Floats are non-deterministic and forbidden in consensus.
- **Map Iteration**:
    - `grep -r "iter()"` on `MapMapper` or `VecMapper`: Potential Gas DoS.

### Logical Patterns (Manual Review)

- **Token ID Validation**:
    - Search for `call_value().all()` or `call_value().single()`.
    - Verify: Is the `token_id` checked against a storage variable (e.g., `wanted_token_id`)?
- **Callback State**:
    - Search for `#[callback]`.
    - Verify: Does it assume the async call succeeded? (It shouldn't).

---

## 2. Go Protocol (`mx-chain-go`)

### Concurrency

- **Goroutines**: `grep -r "go func"`.
    - Check: Is the loop variable captured correctly? (Common Go pitfall).
- **Races**: `grep -r "map\\["` written in goroutines without Mutex.

### Determinism

- **Map Iteration**: Iterating over Go maps is non-deterministic.
    - *Rule*: Never iterate a map to produce a hash or consensus data.
- **Time**: `time.Now()` is forbidden in block processing. Use `header.TimeStamp`.

---

## 3. Semgrep Rule Creation

If a pattern is complex, create a Semgrep rule.

```yaml
rules:
  - id: mvx-float-arithmetic
    patterns:
      - pattern: $X + $Y
      - metavariable-type:
          metavariable: $X
          type: f64
    message: "Floating point arithmetic detected. Use BigUint."
    languages: [rust]
    severity: ERROR
```

---

## Severity Assignment Rules

Use the following table to assign severity to each finding. When a finding matches multiple rules, use the highest applicable severity.

### Rust Smart Contracts
| Pattern | Context | Severity |
|---------|---------|----------|
| `unwrap()` / `expect()` | In an `#[endpoint]` or `#[callback]` function | **High** |
| `unwrap()` / `expect()` | In test code (`#[cfg(test)]`, `tests/`) | **Skip** |
| `f32` / `f64` | Anywhere in a smart contract | **Critical** |
| `unsafe` | In a smart contract (no FFI justification) | **Critical** |
| `unsafe` | FFI boundary with clear justification | **Medium** (document justification) |
| Map/Vec iteration | Without bounds or gas limits | **High** |
| Token ID not validated | In a `#[payable]` endpoint | **Critical** |
| Callback assuming success | `#[callback]` that does not handle failure path | **High** |

### Go Protocol (mx-chain-go only)
| Pattern | Context | Severity |
|---------|---------|----------|
| Go map iteration | In consensus or hashing code | **Critical** |
| `time.Now()` | In block processing logic | **Critical** |

---

## Output Format

Present all findings in the following structured table.

### Static Analysis Report

| # | Pattern | Location | Severity | Finding | False Positive? |
|---|---------|----------|----------|---------|-----------------|
| 1 | unwrap() | src/lib.rs:42 | High | unwrap in endpoint | N |
| 2 | f64 | src/math.rs:10 | Critical | Float in price calc | N |

Summary: [N] findings ([C] Critical, [H] High, [M] Medium). [F] false positives filtered.

---

## Completion Criteria

Static analysis is complete when:

1. `cargo check` and `cargo clippy` pass with zero warnings.
2. All grep patterns from the checklist have been executed.
3. All findings are triaged (severity assigned, false positives marked).
4. Critical and High findings are documented in the output table.

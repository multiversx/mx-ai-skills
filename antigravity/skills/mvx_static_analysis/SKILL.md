---
name: mvx_static_analysis
description: Manual and automated static analysis patterns for Rust/Go (unsafe usage, unverified unwrap, float arithmetic).
---

# MultiversX Static Analysis

This skill guides you through static analysis of MultiversX codebases, focusing on patterns that often indicate vulnerabilities.

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
    - Search for `call_value().all_esdt_transfers()`.
    - Verify: Is the `token_id` checked against a storage variable (e.g., `wanted_token_id`)?
- **Callback State**:
    - Search for `#[callback]`.
    - Verify: Does it assume the async call succeeded? (It shouldn't).

## 2. Go Protocol (`mx-chain-go`)

### Concurrency
- **Goroutines**: `grep -r "go func"`.
    - Check: Is the loop variable captured correctly? (Common Go pitfall).
- **Races**: `grep -r "map\\["` written in goroutines without Mutex.

### Determinism
- **Map Iteration**: Iterating over Go maps is non-deterministic.
    - *Rule*: Never iterate a map to produce a hash or consensus data.
- **Time**: `time.Now()` is forbidden in block processing. Use `header.TimeStamp`.

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

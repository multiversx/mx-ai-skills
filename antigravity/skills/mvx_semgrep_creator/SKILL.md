---
name: mvx_semgrep_creator
description: Writing custom Semgrep rules to enforce MultiversX best practices.
---

# Semgrep Rule Creator (MX)

This skill guides you in writing Semgrep rules to catch MultiversX-specific patterns automatically.

## 1. Common Patterns
- **Unsafe Math**: `x + y` where `x` is `u64`.
- **Floating Point**: `f64`.
- **Endpoint without Payment Check**: `#[payable("*")]` function without `call_value()`.

## 2. Template
```yaml
rules:
  - id: mvx-unsafe-addition
    languages: [rust]
    message: "Potential arithmetic overflow. Use checked_add or BigUint."
    severity: ERROR
    patterns:
      - pattern: $X + $Y
      - pattern-not: $X.checked_add($Y)
      - pattern-inside: |
          #[multiversx_sc::contract]
          trait Contract {
            ...
          }
```

## 3. Workflow
1.  **Identify Pattern**: See `mvx_variant_analysis`.
2.  **Write Rule**: Use the template.
3.  **Test**: Run on the codebase using `semgrep --config rules.yaml .`
4.  **Refine**: Reduce false positives.

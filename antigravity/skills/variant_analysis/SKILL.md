---
name: variant_analysis
description: Finding "variants" of known bugs in other parts of the codebase.
---

# Variant Analysis

This skill helps you multiply the value of a single finding by locating similar vulnerabilities elsewhere.

## 1. The Pivot
Once you find a bug (e.g., "Missing usage of `checked_add` in function A"):
- **Abstract the Pattern**: "Arithmetic operation on user input without checks".
- **Search**: `grep` for other occurrences of the same pattern.

## 2. Common MultiversX Variants
- **Missing Payable Check**:
    - Found: One endpoint accepts payment but doesn't check `call_value()`.
    - Variant Search: Check ALL `#[payable("*")]` endpoints.
- **Unbounded Iteration**:
    - Found: Iterating a `VecMapper` in `compute_reward`.
    - Variant Search: `grep -r "iter()"` on all mappers.
- **Async Callback Revert**:
    - Found: Callback `X` doesn't revert state on failure.
    - Variant Search: Check ALL `#[callback]` functions.

## 3. Automation
- Use `mvx_static_analysis` (Semgrep) to create a temporary rule for the variant.

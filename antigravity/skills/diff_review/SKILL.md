---
name: diff_review
description: Reviewing changes between versions of SCs (Upgradeability checks).
---

# Differential Review

This skill helps you analyze the difference between two versions of a codebase, focusing on security implications of changes.

## 1. Upgradeability Checks (MultiversX)
When reviewing a diff between `v1` and `v2` of a Smart Contract:
- **Storage Layout**:
    - *Critical*: Did the order of fields in a `struct` stored in a `VecMapper` or `SingleValueMapper` change?
    - *Result*: Usage of existing data will interpret bytes incorrectly (Memory Corruption).
    - *Fix*: Append new fields to the end of structs, never reorder.
- **Initialization**:
    - *Critical*: Does `v2` introduce new Storage Mappers?
    - *Check*: Are they initialized in `#[upgrade]`? (Remember `#[init]` is NOT called on upgrade).

## 2. Regression Testing
- **New Features**: Do they break old invariants?
- **Deleted Code**: Was a check removed? Why?

## 3. Workflow
1.  **Generate Diff**: `git diff v1..v2`.
2.  **Filter Noise**: Ignore formatting/style changes.
3.  **Trace Data**: Follow the flow of changed data structures.

---
name: mvx_sharp_edges
description: Identify weird behaviors in WASM, Gas limits, and async call failures specific to MultiversX.
---

# MultiversX Sharp Edges

This skill alerts you to non-obvious behaviors, "gotchas," and platform-specific quirks that often lead to bugs.

## 1. Async Callbacks & Reverts
- **The Edge**: When an Async Call fails, the `#[callback]` is executed.
- **The Sharp Logic**:
    - If you don't implement a callback, the funds return to the contract, but state changes in the original transaction are **NOT reverted** automatically.
    - *Mitigation*: You must manually revert state in the callback if the sub-call failed (or use the synchronous `back_transfers` carefully).

## 2. Gas Limits & Out of Gas (OOG)
- **The Edge**: OOG raises a specific error but can leave the system in a partial state if not handled.
- **The Sharp Logic**:
    - **Cross-shard OOG**: If the destination shard runs OOG, the transaction fails, but the sender shard has already processed the transfer. The callback is triggered with an error.

## 3. Storage Mappers vs Rust Types
- **The Edge**: `VecMapper` is NOT a `Vec`.
- **The Sharp Logic**:
    - `Vec` loads everything into WASM memory (RAM spike).
    - `VecMapper` loads nothing until you call `get(i)`.
    - *Bug*: Using `Vec` for a list of all users = OOG when userbase grows.

## 4. Token Decimal Precision
- **The Edge**: ESDTs can have 0-18 decimals.
- **The Sharp Logic**:
    - Hardcoding `10^18` for "One Token" is wrong.
    - Always fetch `decimals` or require standard 18-decimal tokens.

## 5. Upgradeability
- **The Edge**: `#[init]` is NOT called on upgrade. `#[upgrade]` is called.
- **The Sharp Logic**:
    - If you add a new storage mapper, you must initialize it in `#[upgrade]`.
    - Changing the storage layout of existing mappers (e.g. `struct` field order) corrupts data.

## 6. Block Info inside Views
- **The Edge**: `get_block_timestamp()` in a `#[view]` (off-chain simulation) might return 0 or a different value than on-chain.
- **The Sharp Logic**:
    - Don't rely on block info for critical off-chain view logic.

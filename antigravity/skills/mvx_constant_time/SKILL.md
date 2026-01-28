---
name: mvx_constant_time
description: Verifying constant-time operations in crypto implementations.
---

# MultiversX Constant Time Analysis

This skill helps you verify that cryptographic secrets are handled in constant time to prevent timing attacks.

## 1. When to Use
- **Custom Crypto**: If the contract implements Elliptic Curve math, ZK verification, or signatures manually (not using the API).
- **Comparison**: Checking secrets (e.g., comparing user-provided HASH against stored HASH).

## 2. Patterns to Avoid (Variable Time)
- **Early Exit**: `if byte[i] != other[i] { return false }`. This leaks the index of the first difference.
- **Short-circuiting**: `&&` or `||` on secrets.

## 3. MultiversX Solution
- **Managed Types**: Use `ManagedBuffer` comparison provided by the API (often constant time implementation in the VM).
- **Subtle crate**: Use `subtle::ConstantTimeEq` for manual `u8` slice comparisons.

## 4. Verification
- **Measurement**: Difficult on-chain due to Gas Metering. Gas usually leaks the execution trace roughly.
- **Rule**: Rely on the VM's crypto functions (`self.crypto().verify_signature(...)`) instead of implementing it in WASM.

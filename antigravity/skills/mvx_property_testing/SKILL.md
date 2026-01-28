---
name: mvx_property_testing
description: Using fuzz tests in Rust for invariants.
---

# MultiversX Property Testing

This skill guides you in using property-based testing (fuzzing) to find edge cases in Smart Contract logic.

## 1. Tools
- **`cargo fuzz`**: Standard Rust fuzzer.
- **`proptest`**: Property testing framework for Rust.

## 2. Methodology
Defining Invariants:
- "Total Supply MUST equal sum of all balances."
- "User balance MUST NOT decrease if deposit fails."

## 3. Implementation (RustVM)
Write a test that:
1.  Takes random input (random amounts, random user IDs).
2.  Executes the contract logic via `blockchain_mock`.
3.  Asserts the invariant holds.

## 4. Example
```rust
proptest! {
    #[test]
    fn test_deposit_always_increases_balance(amount in 0u64..1_000_000u64) {
        let mut setup = Setup::new();
        setup.deposit(amount);
        assert_eq!(setup.balance(), amount);
    }
}
```

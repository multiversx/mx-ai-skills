# Smart Contract Debugging

## Tools
- VS Code
- `rust-analyzer` extension
- `CodeLLDB` extension
- `sc-meta` for building/testing

## Techniques
1. **Unit Tests (Rust VM)**: Fast, offline. Use `mandos` or `sc_scenario`.
2. **Integration Tests (Blackbox)**: Test against a simulated chain state.
3. **Step-by-Step Debugging**: Use breakpoints in VS Code within unit tests.

## Printing to Console
Use `sc_print!` macro to inspect variables in the Debug Console.
Note: Managed types (BigUint, Handle) are just indices. Use `sc_print!` to see values.

```rust
#[init]
fn init(&self) {
    let my_val = BigUint::from(2000u32);
    sc_print!("Value: {}", my_val);
}
```

Formatters:
- `{}`: Decimal/ASCII
- `{:x}`: Hex

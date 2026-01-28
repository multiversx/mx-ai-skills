# MultiversX Smart Contract Best Practices: Basics

## Code Arrangement
It is best to separate your code into 4 main parts:
- endpoints
- view functions
- private functions
- storage

This ensures that it's much easier to find what you're looking for, and it's also much easier for everyone else who's working on that smart contract. Additionally, it's also best to split endpoints by their level of access.

## Module Size
Each module should have a maximum of 200-300 lines. If you ever need more than that, consider splitting.

## Function Size
Each function should be ~30 lines. Any more than that and it's really hard to navigate the file.

## Code Placement
`lib.rs` should contain only the `init` and `upgrade` functions most of the time.

```rust
// lib.rs
#[init]
fn init(&self) {}

#[upgrade]
fn upgrade(&self) {}
```

## Module Placement
Each module can be placed in its own folder along with other related modules.

## Error Messages
If you have the same error message in multiple places, use a static constant or a helper function.

```rust
pub static TOO_MANY_ARGS_ERR_MSG: &[u8] = b"Too many arguments";

#[endpoint(myEndpoint)]
fn my_endpoint(&self, args: MultiValueEncoded<ManagedBuffer>) {
    require!(args.len() < 10, TOO_MANY_ARGS_ERR_MSG);
}
```

## Small PRs
Keep PRs focused on one specific task.

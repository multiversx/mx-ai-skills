# BigUint Operations in MultiversX

`BigUint` is a handle to a managed big integer (host managed).
- Operations use API calls.
- `BigUint` is not `Copy`.
- Cloning creates a new handle and copies bytes (expensive).
- Use references (`&BigUint`) for operations to avoid cloning.

## Optimization
Avoid creating intermediate BigUints.

Bad (creates 3 intermediate BigUints):
```rust
e = a + b + c + d;
```

Good (accumulate in one BigUint):
```rust
e = BigUint::zero();
e += &a;
e += &b;
e += &c;
e += &d;
```
This creates only 1 BigUint.

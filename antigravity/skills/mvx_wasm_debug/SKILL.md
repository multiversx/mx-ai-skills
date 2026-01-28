---
name: mvx_wasm_debug
description: Analyzing WASM binaries and debugging via DWARF.
---

# MultiversX WASM Debugging

This skill helps you analyze the compiled `output.wasm` file.

## 1. Binary Size Analysis
- **Twiggy**: Use `twiggy top output.wasm` to see what takes up space.
- **Bloat**: Heavy JSON deserialization code? Large static strings?

## 2. Panic Analysis
- **Abort Messages**: By default, `sc_panic!` adds a string message.
- **Optimization**: `wasm-opt` removes these in production builds (`--opt-level z`).
- **Debugging**: If the contract traps with `unreachable`, check if it ran out of gas or hit a panic without a message.

## 3. DWARF Info
- MultiversX supports building with debug symbols (`mxpy contract build --debug`).
- This allows mapping WASM instructions back to Rust source lines in the debugger.

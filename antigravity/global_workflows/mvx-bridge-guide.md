---
description: Workflow for bridging assets and listing on Ad-Astra Bridge.
---

# MultiversX Bridge Guide (Ad-Astra)

## 1. Pre-requisites

- **Token**: Must be issued on MultiversX (ESDT) and EVM (ERC20).
- **Liquidity**: You need to provide liquidity on both sides (or just one if one-way).

## 2. Whitelisting Process

Before the bridge works for your token, you must whitelist it.

1. **Issue ESDT**: Done using `mvx-token-architect`.
2. **Branding**: Go to <https://assets.multiversx.com> and submit a PR with Logo/Socials.
3. **Roles**:
    - Give `MINT` and `BURN` roles to the Ad-Astra Safe/Bridge Contracts.
    - *Note*: Contact MultiversX team for the specific Safe addresses on Mainnet.

## 3. Bridging Operations

### Deposit (EVM -> MVX)

- Send ERC20 to the Safe Address.
- **Wait**: Requires Finality + 7/10 Relayer Checks (~10-20 mins).

### Withdraw (MVX -> EVM)

- Send ESDT to the Safe Address.
- **Burn**: The ESDT is burned (or locked).
- **Release**: Relayers trigger release on EVM.

## 4. Debugging Bridge Issues

If a transfer is stuck:

1. **Check EVM Finality**: Has the tx reached safe block confirmation?
2. **Check Relayer Consensus**: Is there a pending batch?
3. **Refunds**: If the execution failed on destination (e.g. Out of Gas), check the `BridgeProxy` contract for refund claim.

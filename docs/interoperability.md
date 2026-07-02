# Interoperability & Bridging

## Ad-Astra Bridge v2

The native bridging infrastructure connecting MultiversX to EVM chains (Ethereum, BSC, Polygon, etc.).

### Architecture

- **Safe Contract (Origin)**: Locks funds on the source chain.
- **Bridge Contract (Dest)**: Mints/Releases funds on the destination chain.
- **Relayers**: A set of off-chain validators (10 signatures required usually 7/10) that witness events and sign batches.

### Transfer Flows

#### 1. EVM -> MultiversX (Deposit)

1. **User Action**: `Alice` deposits ERC20 into the EVM `Safe` contract.
2. **Batching**: The bridge groups multiple deposits into a batch.
3. **Finality**: Wait for EVM finality (e.g., 64 blocks).
4. **Consensus**: Relayers sign the batch (need 7/10 signatures).
5. **Execution**: Relayers submit the batch to the MultiversX `Bridge` contract.
6. **Minting**: The Bridge contract mints wrapped ESDT tokens to `Alice`'s MultiversX address.

#### 2. MultiversX -> EVM (Withdrawal)

1. **User Action**: `Alice` sends ESDT to the MultiversX `Safe` contract (effectively burning/locking them).
2. **Batching**: Grouped into a batch.
3. **Consensus**: Relayers sign the batch.
4. **Execution**: Relayers submit the batch to the EVM `Bridge` contract.
5. **Release**: The EVM contract unlocks/mints ERC20 tokens to `Alice`'s EVM address.

### SC Call Flow (Cross-Chain Execution)

*Supported in v3.0+*

- Users can attach metadata to the deposit: `target_sc_address` + `function` + `args`.
- Upon arrival on MultiversX, the `BridgeProxy` contract executes the call synchronously.
- **Failure Handling**: If the SC call fails, the tokens remain in the `BridgeProxy` and can be claimed via a refund interface.

### Token Whitelisting

To bridge a token, it must be whitelisted:

1. **Issue ESDT**: Create the token on MultiversX.
2. **Branding**: Submit valid branding (Logo/Socials) via `assets.multiversx.com`.
3. **Permissions**: Assign `MINT` and `BURN` roles to the Bridge/Safe contract.
4. **Governance**: Final approval by the bridge governance committee (MultiversX Team).

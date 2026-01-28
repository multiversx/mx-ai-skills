# ESDT (eStandard Digital Token)

## Overview
- Native token standard (like ERC-20 but built into the protocol).
- No separate smart contract needed for basic transfers/minting.
- Balances stored directly in account state (lower gas, higher speed).
- Sharding friendly (protocol handles cross-shard transfers).

## Features
- **Fungible Tokens**: Standard cryptocurrencies.
- **Semi-Fungible Tokens (SFTs)**: Batches of NFTs.
- **Non-Fungible Tokens (NFTs)**: Unique assets with metadata.
- **Operations**: Mint, Burn, Pause/Unpause, Freeze, Wipe, Transfer, Upgrade.

## Cost
- Issue cost: ~0.05 EGLD.
- Transfer cost: ~0.00005 EGLD.

---
description: Guide for setting up a MultiversX Validator or Observer node.
---

# MultiversX Validator/Observer Setup

This workflow guides you through setting up a node, taking into account the specific Sharding and Consensus architecture.

## 1. Prerequisites

- **Hardware**: 8 vCPU, 16GB RAM, 200GB SSD (NVMe recommended).
- **OS**: Ubuntu 20.04+.
- **User**: Run as non-root user (e.g., `ubuntu`).

## 2. Installation (Mainnet)

We use the official scripts for simplified management.

// turbo

1. Clone the repository and install

```bash
git clone https://github.com/multiversx/mx-chain-scripts
cd mx-chain-scripts
vim config/variables.cfg # Edit CUSTOM_HOME and USER
bash script.sh install
```

## 3. Configuration (Observer vs Validator)

Decide your role.

### Observer (Production API)

- Edit `config/prefs.toml`.
- set `NodeDisplayName` to something identifiable.
- Ensure `OperationMode` is not set (defaults to Observer).
- **Important**: To act as a historical node (full archive), verify disk space (>1TB).

### Validator (Staking)

- You need a `validatorKey.pem`.
- Generate it: `mxpy wallet new --format pem --outfile validatorKey.pem`.
- Place it in the `config` folder.
- Ensure you have 2500 EGLD (or are part of a staking provider).

## 4. Shard Selection

- By default, the `Metachain` manages shuffling.
- If you are an Observer, you can 'pin' your node to a specific shard (0, 1, 2) by editing `prefs.toml` -> `DestinationShardAsObserver`.
- **Infrastructure Tip**: Run 3 Observers (Shard 0, 1, 2) + 1 Metachain Observer + 1 Proxy for a complete API endpoint.

## 5. Start & Monitor

// turbo

```bash
systemctl start multiversx-node-observer
systemctl status multiversx-node-observer
```

- **Logs**: Check `logs/node-0.log`. Watch for "Synchronizing".

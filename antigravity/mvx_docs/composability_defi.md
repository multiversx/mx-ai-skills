# Real-World DeFi Composability (xExchange Analysis)

## Architecture Overview

xExchange demonstrates "One-Click Complex Actions" through a modular Router-Proxy architecture.

### core Components

1. **Router**: The entry point. Routes trades to the correct Pair contract.
2. **Pair**: Holds liquidity (e.g., EGLD-MEX). Handles `swap`, `addLiquidity`.
3. **Farm**: Manages yield. Accepts LP tokens, issues Farm tokens.
4. **Farm Proxy**: A "Thin Contract" that allows users to enter a farm *directly* after adding liquidity, in a single transaction chain (using Promises or Transfer-Execute).
5. **Locked Asset Factory**: Wraps tokens (e.g., LK-MEX) into SFTs with locking schedules.

## Composable Patterns

### 1. The "Enter Farm" Chain

Instead of User -> Add Liquidity -> Wait -> User -> Stake LP, xExchange uses:
**User -> Router -> Pair (Add Liquidity) -> User (Automatic forwarding) -> Farm (Stake)**
*Actually implemented via contract chaining:*

1. User Calls `Router.addLiquidity`.
2. Router calls `Pair.addLiquidity`.
3. Pair returns LP tokens to Router.
4. Router sends LP tokens to `FarmProxy.enterFarm`.
5. FarmProxy interacts with `Farm` main contract.
6. User receives the final Farming Position (SFT).

### 2. SFTs as Positions

- **Farming Positions**: Represented as SFTs.
- **Attributes**: The SFT's `attributes` field contains the `reward_debt`, `staked_amount`, and `enter_epoch`.
- **Benefit**: The Farm contract doesn't need to store a map of `User -> Position`. The User *holds* the Position. To claim rewards, the User sends the SFT back to the Farm. The Farm reads the attributes, calculates rewards, and sends the SFT back (with updated attributes).

## Liquid Staking (xOxno / Hatom Pattern)

1. **User**: Sends EGLD to Staking Contract.
2. **Contract**: Delegates EGLD to System Staking.
3. **Contract**: Mints `LEGLD` (Liquid EGLD) SFT/token to User.
4. **Exchange Rate**: The value of `LEGLD` vs `EGLD` increases over time as staking rewards accrue in the pool.

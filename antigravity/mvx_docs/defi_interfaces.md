# Standard DeFi Interfaces (Traits)

MultiversX doesn't have an "ERC-20" equivalent because ESDT is native. However, DeFi protocols share common traits for interoperability.

## 1. Liquidity Pool (AMM)

```rust
#[multiversx_sc::module]
pub trait PairModule {
    #[payable("*")]
    #[endpoint(addLiquidity)]
    fn add_liquidity(
        &self,
        min_token_a: BigUint,
        min_token_b: BigUint
    ) -> Payment; // Returns LP Token

    #[payable("*")]
    #[endpoint(swapTokensFixedInput)]
    fn swap_tokens_fixed_input(
        &self,
        token_out: TokenIdentifier,
        min_amount_out: BigUint
    ) -> EsdtTokenPayment;
}
```

## 2. Farm / Staking

```rust
#[multiversx_sc::module]
pub trait FarmModule {
    #[payable("*")] // Expects LP Token
    #[endpoint(enterFarm)]
    fn enter_farm(&self) -> EsdtTokenPayment; // Returns Farming SFT

    #[payable("*")] // Expects Farming SFT
    #[endpoint(claimRewards)]
    fn claim_rewards(&self) -> (EsdtTokenPayment, EsdtTokenPayment); // (SFT, RewardToken)

    #[payable("*")] // Expects Farming SFT
    #[endpoint(exitFarm)]
    fn exit_farm(&self, amount: BigUint) -> EsdtTokenPayment; // Returns LP Token
}
```

## 3. Liquid Staking

```rust
#[multiversx_sc::module]
pub trait LiquidStakingModule {
    #[payable("EGLD")]
    #[endpoint(delegate)]
    fn delegate(&self); // Mints LS-Token to caller

    #[payable("*")] // Expects LS-Token
    #[endpoint(unDelegate)]
    fn undelegate(&self); // Burns LS-Token, queues EGLD unbonding
}
```

## 4. Price Discovery / Oracle

```rust
#[multiversx_sc::module]
pub trait OracleModule {
    #[view(getPrice)]
    fn get_price(&self, pair_id: TokenIdentifier) -> BigUint;
}
```

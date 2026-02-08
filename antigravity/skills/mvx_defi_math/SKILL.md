---
name: mvx_defi_math
description: Financial math for MultiversX DeFi — precision management, half-up rounding, safe rescaling, percentage calculations.
---

# MultiversX DeFi Math Patterns

## Precision Levels

| Level | Decimals | When to Use |
|---|---|---|
| BPS (4) | 10,000 = 100% | Fees, simple percentages |
| PPM (6) | 1,000,000 = 100% | Fine-grained percentages |
| WAD (18) | 1e18 = 1.0 | Token amounts, prices |
| RAY (27) | 1e27 = 1.0 | Interest indices, compounding, high-precision math |

```rust
pub const BPS: u64 = 10_000;
pub const PPM: u64 = 1_000_000;
pub const WAD: u128 = 1_000_000_000_000_000_000;
pub const RAY: u128 = 1_000_000_000_000_000_000_000_000_000;
```

**Rule**: Use lowest precision that avoids rounding errors. Intermediate calculations at higher precision than final result.

## Half-Up Rounding

Standard ManagedDecimal truncates → systematic value loss → exploitable.

### Multiplication
```rust
fn mul_half_up(&self, a: &ManagedDecimal<Self::Api, NumDecimals>, b: &ManagedDecimal<Self::Api, NumDecimals>, precision: NumDecimals) -> ManagedDecimal<Self::Api, NumDecimals> {
    let product = a.rescale(precision).into_raw_units() * b.rescale(precision).into_raw_units();
    let scaled = BigUint::from(10u64).pow(precision as u32);
    let rounded = (product + &scaled / 2u64) / scaled;
    self.to_decimal(rounded, precision)
}
```

### Division
```rust
fn div_half_up(&self, a: &ManagedDecimal<Self::Api, NumDecimals>, b: &ManagedDecimal<Self::Api, NumDecimals>, precision: NumDecimals) -> ManagedDecimal<Self::Api, NumDecimals> {
    let scaled = BigUint::from(10u64).pow(precision as u32);
    let numerator = a.rescale(precision).into_raw_units() * &scaled;
    let denominator = b.rescale(precision).into_raw_units();
    let rounded = (numerator + &denominator / 2u64) / denominator;
    self.to_decimal(rounded, precision)
}
```

## Safe Rescaling

```rust
fn rescale_half_up(&self, value: &ManagedDecimal<Self::Api, NumDecimals>, new_precision: NumDecimals) -> ManagedDecimal<Self::Api, NumDecimals> {
    let old = value.scale();
    if new_precision >= old { return value.rescale(new_precision); }
    let factor = BigUint::from(10u64).pow((old - new_precision) as u32);
    ManagedDecimal::from_raw_units((value.into_raw_units() + &factor / 2u64) / factor, new_precision)
}
```

## Percentage Calculations

### Framework Built-in: `proportion()`

```rust
// Preferred — uses BigUint::proportion(part, total) from the framework
pub const PERCENT_BASE_POINTS: u64 = 100_000;  // 100% = 100_000
let fee = amount.proportion(fee_percent, PERCENT_BASE_POINTS);
```

### Manual BPS / PPM

```rust
pub fn apply_bps(amount: &BigUint, bps: u64) -> BigUint { (amount * bps) / 10_000u64 }
pub fn apply_ppm(amount: &BigUint, ppm: u32) -> BigUint { (amount * ppm) / 1_000_000u64 }
```

## Rounding Attack Vectors

| Attack | Mitigation |
|---|---|
| Dust deposits to steal rounding | Half-up rounding on scaled amounts |
| Repeated small operations | Minimum amounts + half-up on indices |
| Precision loss across conversions | RAY for intermediate math |
| Truncation in fee calculations | Round fees UP (protocol's favor) |

## Anti-Patterns

- Mixing precisions without rescaling
- Truncating division for fees (use ceiling division)
- Intermediate results at low precision (compute at RAY, downscale at end)

## Domain Applications

- **Lending**: Interest models, compound interest, utilization — RAY precision
- **DEX**: Price impact, LP shares — WAD with half-up
- **Staking**: Reward distribution, share ratios — RAY indices
- **Vaults**: Fee calculations, yield — BPS fees with ceiling division

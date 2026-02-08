---
name: mvx_vault_pattern
description: In-memory token ledger for tracking intermediate balances during multi-step operations within a single transaction.
---

# MultiversX In-Memory Token Ledger

## Problem

Multi-step token operations in one transaction need intermediate balance tracking without storage writes.

## When to Use

- Multi-step token operations in one tx (swaps, batch processing, atomic flows)
- NOT for single operations or state that must persist across transactions

## Core Pattern: Dual Data Structure

Uses **ManagedMapEncoded** (O(1) lookup) + **ManagedVec** (ordered iteration for settlement).

```rust
pub struct TokenLedger<M: VMApi> {
    balances: ManagedMapEncoded<M, TokenIdentifier<M>, BigUint<M>>,
    tokens: ManagedVec<M, TokenIdentifier<M>>,
}

impl<M: VMApi> TokenLedger<M> {
    pub fn new() -> Self {
        Self { balances: ManagedMapEncoded::new(), tokens: ManagedVec::new() }
    }

    pub fn from_payments(payments: &PaymentVec<M>) -> Self {
        let mut ledger = Self::new();
        for payment in payments.iter() {
            ledger.deposit(&payment.token_identifier, payment.amount.as_big_uint());
        }
        ledger
    }

    pub fn deposit(&mut self, token: &TokenIdentifier<M>, amount: &BigUint<M>) {
        if !self.balances.contains(token) {
            self.tokens.push(token.clone());
            self.balances.put(token, amount);
        } else {
            let current = self.balances.get(token);
            self.balances.put(token, &(current + amount));
        }
    }

    pub fn withdraw(&mut self, token: &TokenIdentifier<M>, amount: &BigUint<M>) -> BigUint<M> {
        let current = self.balance_of(token);
        require!(current >= *amount, "Insufficient ledger balance");
        let new_balance = &current - amount;
        if new_balance == 0u64 { self.remove_token(token); }
        else { self.balances.put(token, &new_balance); }
        amount.clone()
    }

    pub fn withdraw_all(&mut self, token: &TokenIdentifier<M>) -> BigUint<M> {
        let amount = self.balance_of(token);
        if amount > 0u64 { self.remove_token(token); }
        amount
    }

    pub fn balance_of(&self, token: &TokenIdentifier<M>) -> BigUint<M> {
        if !self.balances.contains(token) { return BigUint::zero(); }
        self.balances.get(token)
    }

    pub fn settle_all(&self) -> ManagedVec<M, Payment<M>> {
        let mut payments = ManagedVec::new();
        for token in self.tokens.iter() {
            let amount = self.balances.get(&token);
            if amount > 0u64 {
                payments.push(Payment::new(token.clone_value(), 0u64, amount));
            }
        }
        payments
    }

    fn remove_token(&mut self, token: &TokenIdentifier<M>) {
        self.balances.remove(token);
        for (i, t) in self.tokens.iter().enumerate() {
            if t.as_managed_buffer() == token.as_managed_buffer() {
                self.tokens.remove(i);
                break;
            }
        }
    }
}
```

## Anti-Patterns

- Using storage for temporary balances (expensive, unnecessary)
- Not cleaning up zero-balance entries (wastes gas during settlement)
- Using only ManagedVec without a map (O(N) lookups)

## Variations

Production repos extend with: result chaining, PPM-based withdrawals, selective settlement (keeping dust as revenue), amount mode enums (Fixed/Percentage/All/PreviousResult).

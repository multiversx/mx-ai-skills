---
name: multiversx-clarification-expert
description: Identify ambiguous requirements and ask targeted clarifying questions for MultiversX development. Use when user requests are vague, missing technical constraints, or have conflicting requirements.
---

# Clarification Expert

Identify ambiguity in user requests and ask targeted questions to unblock MultiversX development or auditing tasks. This skill prevents wasted effort from incorrect assumptions.

## When to Use

- User request is vague (e.g., "Make it secure")
- Missing technical constraints (e.g., "Add a token" without specifying standard)
- Conflicting requirements that need resolution
- Multiple valid implementation approaches exist
- Security implications require explicit user decisions

## Questioning Guidelines

### 1. Be Specific

Never ask open-ended questions. Always offer concrete choices.

**Bad:**
> "How should the token work?"

**Good:**
> "Should the token be:
> 1. A standard Fungible ESDT (simple transfers, 18 decimals)
> 2. A Semi-Fungible Token (SFT) with quantity per nonce
> 3. A Meta-ESDT with custom attributes
>
> If unsure, I recommend option 1 for simplicity."

### 2. Batch Related Questions

Group questions by topic to minimize back-and-forth.

**Example:**
> Before implementing the staking module, I need to clarify:
>
> **Token Configuration:**
> 1. Which token can be staked? (specific token ID or any ESDT?)
> 2. Is there a minimum stake amount?
>
> **Reward Mechanism:**
> 3. Fixed APY or dynamic based on total staked?
> 4. Reward distribution: claim-based or auto-compound?
>
> **Access Control:**
> 5. Can anyone stake, or only whitelisted addresses?

### 3. Propose Sensible Defaults

Always offer a recommended option when the user may not have strong preferences.

**Example:**
> "For the admin storage, I recommend using `SingleValueMapper` instead of `MapMapper` to save gas since you only need one admin. Shall I proceed with that approach?"

### 4. Explain Trade-offs

When choices have significant implications, explain the consequences.

**Example:**
> "For storing user balances, there are two approaches:
>
> **Option A: SingleValueMapper with address parameter**
> - More gas efficient for individual lookups
> - Cannot iterate over all users
>
> **Option B: MapMapper**
> - Can iterate over all users (useful for airdrops)
> - Higher gas cost (4N+1 storage slots)
>
> Which user enumeration requirement do you have?"

## Analysis Categories

### 1. Scope & Environment
| Question | Why It Matters |
|----------|----------------|
| Full audit or specific module? | Depth vs breadth |
| New code or upgrade review? | Upgrade reviews need storage migration focus |
| Mainnet, Devnet, or Sovereign Chain? | Different security requirements |
| Integration with other contracts? | Cross-contract interaction risks |

### 2. Risk & Technical Profile
| Question | Why It Matters |
|----------|----------------|
| TVL expectations? | High value = higher attacker incentive |
| User base: public or restricted? | Public = larger attack surface |
| Upgradeable or immutable? | Affects fix deployment strategy |
| Custom modules or `unsafe` blocks? | Needs deeper review |
| External dependencies? | Supply chain risk |

## MultiversX-Specific Clarifications

When user says... ask to clarify:

- **"add a token"**: Fungible ESDT, NFT, SFT, or Meta-ESDT?
- **"store user data"**: Per-user (`SingleValueMapper` + address key), enumerable (`MapMapper`/`SetMapper`), ordered (`VecMapper`/`LinkedListMapper`), or unique (`UnorderedSetMapper`)?
- **"admin only"**: Single owner, stored admin address, multi-sig, role-based, or time-locked?

## Question Templates

### For New Feature Requests
> "To implement [FEATURE], I need to understand:
> 1. [CRITICAL_DECISION_1]
> 2. [CRITICAL_DECISION_2]
>
> My recommendation is [DEFAULT] because [REASON]. Should I proceed with this approach?"

### For Bug Reports
> "To fix this issue, I need to clarify:
> 1. Expected behavior: [WHAT_SHOULD_HAPPEN]
> 2. Current behavior: [WHAT_HAPPENS_NOW]
> 3. Reproduction steps: [HOW_TO_TRIGGER]
>
> Can you confirm these details?"

### For Security Reviews
> "Before auditing [CONTRACT], please confirm:
> 1. Threat model: Who are the trusted parties?
> 2. Invariants: What properties must always hold?
> 3. Known risks: Are there accepted trade-offs?
>
> This helps me focus on relevant attack vectors."

## Anti-Patterns to Avoid

- **Don't assume**: If something could go two ways, ask
- **Don't ask obvious questions**: "Is security important?" - always yes
- **Don't delay indefinitely**: If no response, state assumptions and proceed
- **Don't ask one question at a time**: Batch related questions together
- **Don't use jargon without explanation**: Clarify technical terms for non-experts

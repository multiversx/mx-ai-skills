---
name: clarification_expert
description: Expert at identifying underspecified requirements and asking high-value clarifying questions.
---

# Clarification Expert

This skill helps you identify ambiguity in user requests and ask targeted questions to unblock development or auditing tasks.

## When to Use
- The user's request is vague (e.g., "Make it secure").
- Missing technical constraints (e.g., "Add a token" but no standard specified).
- Conflicting requirements.

## Guidelines for Questions

### 1. Be Specific
Don't ask "What do you mean?". Ask "Do you want X or Y?".
*   *Bad*: "How should the token work?"
*   *Good*: "Should the token be a standard Fungible ESDT or a Semi-Fungible SFT with metadata?"

### 2. Batch Questions
Don't ask one question at a time. Group related questions into a numbered list.

### 3. Propose Defaults
Always offer a sensible default if the user doesn't know.
*   *Example*: "If you don't have a preference, I recommend using `SingleValueMapper` for the config storage to save gas. Shall I proceed with that?"

## Analysis Categories

1.  **Scope**: Is it a full audit or just a specific module?
2.  **Environment**: Mainnet, Devnet, or Sovereign Chain?
3.  **Risk Profile**: Is this a generic dApp or a high-value DeFi protocol?
4.  **Tech Stack**: Are we using standard `multiversx-sc` modules or custom unchecked arithmetic? (Rule: Always suggest standard modules).

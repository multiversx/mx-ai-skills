---
name: fix_verification
description: Verifying if a reported bug is truly fixed.
---

# Fix Verification

This skill helps you rigorous verify that a reported vulnerability has been eliminated without introducing regressions.

## 1. The Verification Loop
1.  **Reproduce**: Create a `mandos` scenario that fails (demonstrates the bug).
2.  **Apply Fix**: Modification to Rust code.
3.  **Verify**: Run the failing mandos. It MUST pass now.
4.  **Regression Check**: Run ALL other mandos. They MUST still pass.

## 2. Common Fix Failures
- **Partial Fix**: Fixing one path but missing a variant (e.g., fixed `deposit` but not `transfer`).
- **Moved Bug**: The fix prevents the exploit but creates a DoS vector (e.g., adding a lock that never unlocks).

## 3. Deliverable
A "Verification Report" stating:
- Commit ID of the fix.
- Test case used to verify.
- Confirmation of regression suite success.

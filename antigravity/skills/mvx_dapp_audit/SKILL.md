---
name: mvx_dapp_audit
description: Auditing dApps and standard Frontend flows.
---

# MultiversX dApp Auditor

This skill helps you audit the frontend components of a MultiversX application (`sdk-dapp`).

## 1. Transaction Construction
- **Critical Logic**: The frontend constructs the payload.
- **Attack**: Can a malicious frontend user change the payload before signing?
    - Example: `func@args` -> `func@evil_args`.
- **Mitigation**: The Smart Contract MUST validate everything. Do not trust the frontend to validate inputs.

## 2. Signing Security
- **Blind Signing**: Does the dApp verify what it asks the user to sign?
- **Hash Signing**: Is the user signing a hash (opaque) or a clear message?

## 3. Sensitive Data
- **Local Storage**: Is the private key or mnemonic ever stored in `localStorage`? (Should NEVER be).
- **XSS**: Can an attacker extract the `accessToken`?

## 4. Tools
- **Burp Suite**: Proxy traffic to see what the dApp sends to the API or Blockchain Proxy.
- **Inspect Element**: Check network tab for `POST /transactions` payloads.

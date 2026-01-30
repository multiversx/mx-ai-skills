---
name: consult_mvx_docs
description: Access the global MultiversX documentation library to answer technical questions or verify implementation details.
---

# Consult MultiversX Documentation

You have access to a comprehensive library of MultiversX documentation, standards, and best practices stored in this repository at:
`antigravity/mvx_docs`

Note: Agents must resolve this path relative to the repository root (current workspace). If the agent runs outside this repo or without filesystem access, ensure the repo (including `antigravity/mvx_docs`) is mounted or provided in context.

## When to use this skill
- When you need to understand specific MultiversX protocols (ESDT, Smart Contracts, transactions).
- When you need to verify best practices for security, gas optimization, or architecture.
- When you need to look up details about a specific MIP (MultiversX Improvement Proposal).
- When you need to answer a user's question about the ecosystem using authoritative sources.

## How to use
Use standard file search tools to find relevant information within the documentation directory.

### 1. Find relevant files
Use `find_by_name`, `grep_search`, or your environment’s search to locate relevant documents.

```bash
# Example: Find docs related to ESDT tokens (repo-relative)
find_by_name(SearchDirectory="antigravity/mvx_docs", Pattern="*esdt*")
```

```bash
# Example: Search for "async call" inside the docs
grep_search(SearchPath="antigravity/mvx_docs", Query="async call")
```

### 2. Read content
Once you have identified relevant files, use `read_browser_url` (if it was a URL) or simply `view_file` to read the markdown content.

```bash
view_file(AbsolutePath="antigravity/mvx_docs/sc_async_calls.md")
```

### 3. Online fallback (if local docs unavailable)
If the local repository docs are not available in your runtime or appear outdated, consult the official MultiversX documentation site:

- Base URL: https://docs.multiversx.com/

Examples:
```bash
# Open a specific page
read_browser_url(URL="https://docs.multiversx.com/developers/sc-overview/")
```

Common direct links:
- Smart Contracts Overview: https://docs.multiversx.com/developers/smart-contracts
- sc-meta Tool: https://docs.multiversx.com/developers/meta/sc-meta
- Storage Mappers: https://docs.multiversx.com/developers/developer-reference/storage-mappers
- Annotations: https://docs.multiversx.com/developers/developer-reference/sc-annotations
- Payments: https://docs.multiversx.com/developers/developer-reference/sc-payments

Searching the docs site: The official site uses client-side search without a URL query parameter. To search, use an external search engine with a site filter, for example:

```text
site:docs.multiversx.com async call
site:docs.multiversx.com annotations payable
```

Note: Online access requires network permissions in your agent environment. Prefer local docs for reproducibility; use the official site (and site-filtered web search) when local docs are missing or stale.

## Best Practices
- Always verify your assumptions against these docs if you are unsure.
- Quote the documentation when explaining concepts to the user.
- If the docs seem outdated compared to the codebase you are working on, note that discrepancy to the user.

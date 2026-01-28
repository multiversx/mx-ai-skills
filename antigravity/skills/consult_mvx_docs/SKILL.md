---
name: consult_mvx_docs
description: Access the global MultiversX documentation library to answer technical questions or verify implementation details.
---

# Consult MultiversX Documentation

You have access to a comprehensive library of MultiversX documentation, standards, and best practices located at:
`/Users/robertsasu/.gemini/antigravity/mvx_docs`

## When to use this skill
- When you need to understand specific MultiversX protocols (ESDT, Smart Contracts, transactions).
- When you need to verify best practices for security, gas optimization, or architecture.
- When you need to look up details about a specific MIP (MultiversX Improvement Proposal).
- When you need to answer a user's question about the ecosystem using authoritative sources.

## How to use
Use standard file search tools to find relevant information within the documentation directory.

### 1. Find relevant files
Use `find_by_name` or `grep_search` to locate relevant documents.

```bash
# Example: Find docs related to ESDT tokens
find_by_name(SearchDirectory="/Users/robertsasu/.gemini/antigravity/mvx_docs", Pattern="*esdt*")
```

```bash
# Example: Search for "async call" inside the docs
grep_search(SearchPath="/Users/robertsasu/.gemini/antigravity/mvx_docs", Query="async call")
```

### 2. Read content
Once you have identified relevant files, use `read_browser_url` (if it was a URL) or simply `view_file` to read the markdown content.

```bash
view_file(AbsolutePath="/Users/robertsasu/.gemini/antigravity/mvx_docs/sc_async_calls.md")
```

## Best Practices
- Always verify your assumptions against these docs if you are unsure.
- Quote the documentation when explaining concepts to the user.
- If the docs seem outdated compared to the codebase you are working on, note that discrepancy to the user.

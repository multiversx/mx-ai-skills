---
description: Workflow for querying MultiversX data via Elasticsearch.
---

# MultiversX Elasticsearch Indexer

## 1. Index Structure Reference

Use these schemas when constructing your queries.

### Operations (`operations`)

- `sender`, `receiver`: Filter by address.
- `function`: Filter by SC function name.
- `status`: `success` | `fail`.
- `timestamp`: Epoch time.

### Events (`events` / `logs`)

- `identifier`: The event name (e.g. `swap`, `enter_farm`).
- `address`: The smart contract address.
- `topics`: Array of base64 strings.
  - `topics[0]` is usually the identifier.
  - `topics[1...]` are indexed arguments.

## 2. Common Queries

### Find all Swaps by User X

```json
GET /operations/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "sender": "erd1..." } },
        { "match": { "function": "swap" } },
        { "match": { "status": "success" } }
      ]
    }
  }
}
```

### Find all NFT Mints from Contract Y

```json
GET /events/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "erd1_contract..." } },
        { "match": { "identifier": "ESDTNFTCreate" } }
      ]
    }
  }
}
```

## 3. Setup

To run your own indexer:

1. **Node**: Run a normal Observer node.
2. **Indexer**: Enable `enable-indexing` in `prefs.toml`.
3. **Elasticsearch**: Run a local or cloud ES instance.
4. **Connect**: Point the node to the ES instance URL.

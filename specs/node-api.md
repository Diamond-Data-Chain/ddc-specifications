# Node API

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document specifies the external API exposed by DDC Full Nodes. It covers the JSON-RPC 2.0 interface (available over HTTP and WebSocket) and the REST HTTP interface. Client applications, indexers, and bridge adapters MUST use this API to interact with the DDC network.

---

## General Conventions

### Base URL

```
http(s)://<node-host>:<port>
```

Default port: **9944** (JSON-RPC), **9933** (REST).

### Authentication

The Node API is unauthenticated by default. Node operators MAY enable API key authentication via configuration. Authenticated endpoints use the `Authorization: ****** header.

Write endpoints (transaction submission) MAY require authentication in operator-restricted deployments.

### Content Type

All JSON-RPC requests MUST use `Content-Type: application/json`.
All REST requests and responses use `Content-Type: application/json`.

### Error Format

All error responses use a standard envelope:

```json
{
  "error": {
    "code": 4001,
    "message": "Invalid extrinsic",
    "data": "nonce too low"
  }
}
```

Standard error codes:

| Code | Meaning |
|------|---------|
| 1000 | Internal error |
| 1001 | Method not found |
| 1002 | Invalid parameters |
| 2000 | Block not found |
| 2001 | Extrinsic not found |
| 2002 | Record not found |
| 4000 | Transaction pool error |
| 4001 | Invalid extrinsic |
| 4002 | Nonce too low |
| 4003 | Nonce too high |
| 4004 | Fee too low |
| 4005 | Account not found |

---

## JSON-RPC 2.0 Interface

Requests follow [JSON-RPC 2.0](https://www.jsonrpc.org/specification).

### `chain_getBlockHash`

Returns the block hash at a given height.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `block_number` | `u64` \| `null` | No | Block height; `null` returns the best finalized block |

**Result:** `Hash` (hex string)

**Example:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "chain_getBlockHash",
  "params": [12345]
}
```
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0xabc123..."
}
```

---

### `chain_getHeader`

Returns the block header at a given block hash.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `block_hash` | `Hash` \| `null` | No | Block hash; `null` returns the best finalized header |

**Result:** `BlockHeader` object

```json
{
  "parent_hash": "0x...",
  "block_number": 12345,
  "state_root": "0x...",
  "extrinsics_root": "0x...",
  "timestamp": 1722830400000,
  "validator": "0x...",
  "signature": "0x..."
}
```

---

### `chain_getBlock`

Returns the full block (header + extrinsics) at a given block hash.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `block_hash` | `Hash` \| `null` | No | Block hash; `null` returns the best finalized block |

**Result:** `Block` object

---

### `chain_getFinalizedHead`

Returns the hash of the latest finalized block.

**Parameters:** none

**Result:** `Hash`

---

### `state_getRecord`

Returns a DDC Record by its record ID, with an optional Merkle proof.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `record_id` | `Hash` | Yes | The record identifier |
| `block_hash` | `Hash` \| `null` | No | Query at specific block; `null` uses finalized head |
| `with_proof` | `bool` | No | If `true`, includes a Merkle proof (default: `false`) |

**Result:**

```json
{
  "record": { /* DdcRecord fields */ },
  "proof": { /* MerkleProof, if requested */ }
}
```

---

### `state_getRecordsByAsset`

Returns all DDC Records associated with an asset ID.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `asset_id` | `string` | Yes | External asset identifier |
| `block_hash` | `Hash` \| `null` | No | Query at specific block |

**Result:** `Array<DdcRecord>`

---

### `state_getAccountInfo`

Returns account balance, nonce, and bonded stake.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | `AccountId` | Yes | Account identifier (32-byte hex) |
| `block_hash` | `Hash` \| `null` | No | Query at specific block |

**Result:**

```json
{
  "account_id": "0x...",
  "nonce": 42,
  "free_balance": "1000000000000",
  "bonded_balance": "10000000000000",
  "reserved_balance": "0"
}
```

---

### `author_submitExtrinsic`

Submits a signed extrinsic to the transaction pool.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `extrinsic` | `string` | Yes | SCALE-encoded, hex-prefixed extrinsic |

**Result:** `Hash` — the extrinsic hash

---

### `author_pendingExtrinsics`

Returns extrinsics currently in the transaction pool.

**Parameters:** none

**Result:** `Array<string>` — hex-encoded SCALE extrinsics

---

### `system_health`

Returns node health status.

**Parameters:** none

**Result:**

```json
{
  "peers": 23,
  "is_syncing": false,
  "should_have_peers": true
}
```

---

### `system_version`

Returns the node software version.

**Parameters:** none

**Result:** `string` — e.g., `"ddc-node/1.0.0"`

---

### `system_chainName`

Returns the name of the network.

**Parameters:** none

**Result:** `string` — e.g., `"DDC Mainnet"`

---

## WebSocket Subscriptions

The WebSocket interface supports real-time subscriptions using the `*_subscribe` / `*_unsubscribe` pattern.

### `chain_subscribeNewHeads`

Subscribe to new block headers as they are finalized.

**Subscribe method:** `chain_subscribeNewHeads`
**Unsubscribe method:** `chain_unsubscribeNewHeads`

**Subscription notification:**

```json
{
  "jsonrpc": "2.0",
  "method": "chain_newHead",
  "params": {
    "subscription": "<subscription-id>",
    "result": { /* BlockHeader */ }
  }
}
```

---

### `state_subscribeRecordEvents`

Subscribe to DDC Record creation, update, and status change events.

**Subscribe method:** `state_subscribeRecordEvents`
**Unsubscribe method:** `state_unsubscribeRecordEvents`

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `asset_id` | `string` \| `null` | No | Filter events to a specific asset |
| `owner` | `AccountId` \| `null` | No | Filter events to a specific owner |

**Subscription notification:**

```json
{
  "jsonrpc": "2.0",
  "method": "state_recordEvent",
  "params": {
    "subscription": "<subscription-id>",
    "result": {
      "event_type": "RecordCreated",
      "record_id": "0x...",
      "block_hash": "0x...",
      "block_number": 12346
    }
  }
}
```

---

## REST HTTP Interface

The REST interface is available on port **9933** and provides resource-oriented access to chain data.

### `GET /v1/blocks/latest`

Returns the latest finalized block.

**Response:** `Block` JSON object.

---

### `GET /v1/blocks/{hash_or_number}`

Returns the block identified by hash or block number.

**Path Parameters:**

| Parameter | Description |
|-----------|-------------|
| `hash_or_number` | Block hash (0x-prefixed hex) or block number |

**Response:** `Block` JSON object.

---

### `GET /v1/records/{record_id}`

Returns a DDC Record.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `proof` | `bool` | If `true`, include Merkle proof |
| `at` | `Hash` | Query at a specific block hash |

**Response:**
```json
{
  "record": { /* DdcRecord */ },
  "proof": { /* MerkleProof, if requested */ }
}
```

---

### `GET /v1/records`

List DDC Records with optional filters.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `asset_id` | `string` | Filter by external asset ID |
| `owner` | `AccountId` | Filter by owner account |
| `asset_type` | `string` | Filter by asset type (`diamond`, `coloured_stone`, `pearl`, `metal`, `other`) |
| `status` | `string` | Filter by status (`active`, `transferred`, `revoked`, `archived`) |
| `limit` | `u32` | Max results (default: 20, max: 100) |
| `offset` | `u32` | Pagination offset |

**Response:**
```json
{
  "records": [ /* Array<DdcRecord> */ ],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

---

### `POST /v1/transactions`

Submit a signed extrinsic.

**Request Body:**
```json
{
  "extrinsic": "0x..."
}
```

**Response:**
```json
{
  "tx_hash": "0x..."
}
```

---

### `GET /v1/accounts/{account_id}`

Returns account information.

**Response:**
```json
{
  "account_id": "0x...",
  "nonce": 42,
  "free_balance": "1000000000000",
  "bonded_balance": "10000000000000",
  "reserved_balance": "0"
}
```

---

### `GET /v1/health`

Returns node health.

**Response:**
```json
{
  "status": "ok",
  "peers": 23,
  "is_syncing": false,
  "block_number": 12345,
  "block_hash": "0x..."
}
```

---

## Rate Limiting

Node operators SHOULD enforce rate limits to protect node stability:

| Endpoint Type | Default Limit |
|---------------|--------------|
| Read (JSON-RPC / REST GET) | 100 requests/second per IP |
| Write (transaction submission) | 10 requests/second per IP |
| WebSocket subscriptions | 10 active subscriptions per connection |

---

## References

- [Data Model](data-model.md)
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)

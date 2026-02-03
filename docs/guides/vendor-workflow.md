---
sidebar_position: 3
---

# Vendor Workflow

This guide covers the on-demand proof generation process that vendors use when customers request verification.

## Overview

When a customer wants to verify that specific packages are NOT in your software:

1. Customer provides an SMT root hash and a list of banned packages (PURLs)
2. Your proof-orchestrator-service coordinates proof generation
3. A zero-knowledge proof is generated and stored on IPFS
4. The proof CID is recorded on the blockchain
5. Customer can verify the proof independently

## Prerequisites

- TrustBOM services running and healthy
- SMT already stored (from [CI/CD pipeline](./ci-cd-integration))
- Customer has the SMT root hash (from blockchain or provided by you)

## Generating Proofs

### API Endpoint

```
POST /generate-proof
Host: proof-orchestrator-service:8080
Content-Type: application/json
```

### Request Format

```json
{
  "root_hash": "97fd46cc8ff09e9b63bde96e80661d586244cc2fb1915aa5979048cf2189cab8",
  "banned_list": [
    "pkg:npm/log4j@2.14.0",
    "pkg:npm/vulnerable-package@1.0.0"
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `root_hash` | string | SMT root hash (64 hex characters) |
| `banned_list` | array of strings | Package URLs (pURLs) to check for non-membership |

### Example Request

```bash
curl -X POST "http://localhost:8080/generate-proof" \
  -H "Content-Type: application/json" \
  -d '{
    "root_hash": "97fd46cc8ff09e9b63bde96e80661d586244cc2fb1915aa5979048cf2189cab8",
    "banned_list": [
      "pkg:npm/log4j@2.14.0",
      "pkg:npm/event-stream@3.3.6"
    ]
  }'
```

### Response Format

```json
{
  "status": "success",
  "ipfs_cid": "QmXnnyufdzAWL5CqZ2RnSNgPbvCc1ALT73s6epPrRnZ1Xy",
  "tx_hash": "0x...",
  "compliance_status": true,
  "root_hash": "97fd46cc8ff09e9b63bde96e80661d586244cc2fb1915aa5979048cf2189cab8",
  "composite_hash": "a1b2c3d4..."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"success"` or error message |
| `ipfs_cid` | string | IPFS Content ID where proof is stored |
| `tx_hash` | string | Blockchain transaction hash |
| `compliance_status` | boolean | `true` if no banned packages found |
| `root_hash` | string | Echoed root hash |
| `composite_hash` | string | SHA256(root_hash + banned_list_hash) |
| `warning` | string | Present if proof already existed (cached) |

## Sharing with Customers

After generating a proof, provide customers with:

| Data | Description |
|------|-------------|
| **IPFS CID** | To retrieve the proof |
| **Root hash** | To verify against blockchain commitment |
| **Image ID** | RISC Zero guest program identifier (for verification) |

The customer can then use these to independently verify the proof (see [Customer Verification](./customer-verification)).

## Caching

Proofs are deduplicated based on the combination of `root_hash` and `banned_list`. If a proof for the same parameters already exists:

- The existing proof is returned
- A `warning` field indicates the proof was cached
- No new blockchain transaction is created

---
sidebar_position: 1
---

# Usage Guide

This guide covers how to use TrustBOM after [installation](../installation).

## Overview

TrustBOM operates in two phases:

1. **Commitment Phase (CI/CD):** Every software release triggers SBOM generation, SMT construction, and blockchain commitment
2. **Verification Phase (On-demand):** Customers request proof that banned packages are absent; vendors generate ZK proofs; customers verify independently

## Quick Start

### For Vendors

1. **Set up CI/CD integration** - Add the GitHub Actions workflow to your repository ([CI/CD Integration Guide](./ci-cd-integration))
2. **Each build automatically:**
   - Generates a CycloneDX SBOM from your container image
   - Transforms it into a Sparse Merkle Tree (SMT)
   - Stores the SMT via merkle-proof-service
   - Commits the root hash to the blockchain
3. **When customers request verification** - Generate proofs via the proof-orchestrator API ([Vendor Workflow](./vendor-workflow))

### For Customers

1. **Obtain the SMT root hash** from the vendor or blockchain
2. **Prepare your banned package list** (Package URLs)
3. **Request a proof** from the vendor's proof-orchestrator-service
4. **Verify the proof** independently ([Customer Verification](./customer-verification))

## Prerequisites

### System Requirements

- TrustBOM services deployed and running (see [Installation](../installation))
- Self-hosted GitHub Actions Runner in the Kubernetes cluster (via ARC)
- Access to the blockchain node

### Service Endpoints

When running in Kubernetes, services communicate internally:

| Service | Internal URL | Port | Purpose |
|---------|--------------|------|---------|
| proof-orchestrator-service | `proof-orchestrator-service.sharing-sbom-system.svc.cluster.local` | 8080 | Coordinates proof generation |
| merkle-proof-service | `merkle-proof-service.sharing-sbom-system.svc.cluster.local` | 8090 | Builds SMTs, generates Merkle proofs |
| proving-service | `proving-service.sharing-sbom-system.svc.cluster.local` | 80 | Generates zkSTARK proofs |
| ipfs-service | `ipfs-service.sharing-sbom-system.svc.cluster.local` | 80 | Stores proofs on IPFS |
| verifier-service | `verifier-service.sharing-sbom-system.svc.cluster.local` | 80 | Independent proof verification |
| hardhat-node | `hardhat-node.blockchain.svc.cluster.local` | 8545 | Blockchain node |

### Smart Contract

The SBOMRegistryV2 contract is deployed at:
```
0x5FbDB2315678afecb367f032d93F642f64180aa3
```

## Next Steps

- [CI/CD Integration](./ci-cd-integration) - Set up automated SBOM commitment in your build pipeline
- [Vendor Workflow](./vendor-workflow) - Generate proofs when customers request verification
- [Customer Verification](./customer-verification) - Verify proofs independently

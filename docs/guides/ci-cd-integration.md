---
sidebar_position: 2
---

# CI/CD Integration

This guide shows how to integrate TrustBOM into your CI/CD pipeline using GitHub Actions.

## Architecture Overview

The CI/CD workflow consists of two jobs:

```
┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐
│  Job 1: build-and-scan              │     │  Job 2: build-smt-and-store         │
│  (GitHub-hosted runner)             │────▶│  (Self-hosted runner in K8s)        │
├─────────────────────────────────────┤     ├─────────────────────────────────────┤
│  • Build Docker image               │     │  • Download sbom-converter          │
│  • Push to GHCR                     │     │  • Build SMT from SBOM              │
│  • Generate SBOM with Syft          │     │  • Store SMT in merkle-proof-service│
│  • Calculate SBOM hash              │     │  • Write root hash to blockchain    │
│  • Upload artifacts                 │     │                                     │
└─────────────────────────────────────┘     └─────────────────────────────────────┘
```

**Job 1** runs on standard GitHub-hosted runners and handles the build and SBOM generation.

**Job 2** runs on a self-hosted runner inside your Kubernetes cluster.

:::info Prototype Setup
The two-job architecture with a self-hosted runner is specific to this prototype setup. Since TrustBOM services and the Hardhat blockchain run locally (not in the cloud), Job 2 needs to run inside the Kubernetes cluster to access them.

In a production deployment with cloud-hosted services, the entire workflow could run on standard GitHub-hosted runners.
:::

## Prerequisites

### Self-Hosted Runner

The second job requires a self-hosted runner deployed via Actions Runner Controller (ARC). This is set up automatically by the TrustBOM installation script (`./scripts/setup.sh`).

Verify the runner is available:
```bash
kubectl get pods -n arc-runners
```

### GitHub Token Setup

The workflow uses `GITHUB_TOKEN` for pushing images to GitHub Container Registry (ghcr.io) and downloading artifacts between jobs. This token is **automatically provided** by GitHub Actions - you don't need to create it manually.

However, you need to ensure your repository has the correct permissions:

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Actions** → **General**
3. Scroll to **Workflow permissions**
4. Select **Read and write permissions**
5. Check **Allow GitHub Actions to create and approve pull requests** (optional)
6. Click **Save**

:::note Package Permissions
For the first push to GHCR, you may need to configure package visibility:
1. After the first workflow run, go to your repository's **Packages** tab
2. Click on the package → **Package settings**
3. Under **Manage Actions access**, add your repository with **Write** role
:::

## Complete Workflow Template

:::tip Copy and Paste
Copy this workflow to `.github/workflows/sbom-ci.yaml` in your repository.
:::

:::warning Current Limitation
The proving-service currently only supports **Package URL (PURL) verification**. License-based verification is not yet implemented in the ZK proof generation.
:::

```yaml
# TrustBOM SBOM Commitment Workflow
#
# This workflow automates the commitment phase of TrustBOM:
# 1. Builds your application container
# 2. Generates a CycloneDX SBOM
# 3. Transforms SBOM into a Sparse Merkle Tree (SMT)
# 4. Stores the SMT and commits its root hash to the blockchain
#
# Prerequisites:
# - TrustBOM services deployed in Kubernetes
# - Self-hosted GitHub Actions Runner via ARC
# - GHCR (GitHub Container Registry) access

name: SBOM System CI/CD

on:
  push:
    branches: [main]
  workflow_dispatch:  # Allow manual triggers

jobs:
  # Job 1: Build, Push, and Generate SBOM
  # Runs on GitHub-hosted runners
  build-and-scan:
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Generates consistent tags for your container image
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          # CUSTOMIZE: Change 'myapp' to your image name
          images: ghcr.io/${{ github.repository }}/myapp
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          # CUSTOMIZE: Update path if your Dockerfile is elsewhere
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Generate CycloneDX SBOM from the container image using Anchore Syft
      - name: Create sboms directory
        run: mkdir -p sboms

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          # CUSTOMIZE: Match your image name from above
          image: ghcr.io/${{ github.repository }}/myapp:latest
          format: cyclonedx-json
          output-file: sboms/app.json

      - name: Calculate SBOM hash
        run: |
          sha256sum sboms/app.json | awk '{print $1}' > sboms/app.sha256
          echo "SBOM Hash: $(cat sboms/app.sha256)"

      # Makes SBOM available to the second job
      - name: Upload SBOM artifacts
        uses: actions/upload-artifact@v4
        with:
          name: sbom-and-security-reports
          path: |
            sboms/app.json
            sboms/app.sha256
          retention-days: 30

  # Job 2: Build SMT and Store on Blockchain
  # Runs on self-hosted runner inside Kubernetes cluster
  # (Required for prototype setup to access local services)
  build-smt-and-store:
    runs-on: [self-hosted]
    needs: [build-and-scan]
    timeout-minutes: 30
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Download SBOM artifacts
        uses: actions/download-artifact@v4
        with:
          name: sbom-and-security-reports
          path: .

      - name: Verify and organize SBOM artifacts
        run: |
          if [ -f "app.json" ] && [ ! -f "sboms/app.json" ]; then
            mkdir -p sboms
            mv app.json sboms/app.json 2>/dev/null || true
            mv app.sha256 sboms/app.sha256 2>/dev/null || true
          fi
          [ ! -f "sboms/app.json" ] && { echo "ERROR: SBOM not found"; exit 1; }
          echo "SBOM file found: $(ls -lh sboms/app.json)"

      - name: Install kubectl
        run: |
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl
          sudo mv kubectl /usr/local/bin/

      - name: Wait for merkle-proof-service
        run: |
          kubectl wait --for=condition=available --timeout=300s \
            deployment/merkle-proof-service -n sharing-sbom-system || true
          MERKLE_URL="http://merkle-proof-service.sharing-sbom-system.svc.cluster.local:8090"
          for i in {1..30}; do
            curl -s --max-time 5 "$MERKLE_URL/health" > /dev/null 2>&1 && break
            sleep 2
          done

      # Download sbom-converter and transform CycloneDX SBOM into a Sparse Merkle Tree
      - name: Build SMT from SBOM
        id: build_smt
        timeout-minutes: 15
        run: |
          SBOM_FILE="sboms/app.json"
          [ ! -f "$SBOM_FILE" ] && { echo "ERROR: SBOM not found"; exit 1; }

          # Download latest sbom-converter release
          LATEST=$(curl -s https://api.github.com/repos/tuberlin-blockchain-prototyping/sbom-conversion/releases/latest)
          TAG=$(echo "$LATEST" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
          URL=$(echo "$LATEST" | grep -o '"browser_download_url": "[^"]*linux[^"]*amd64[^"]*"' | head -1 | sed 's/"browser_download_url": "\([^"]*\)"/\1/')

          curl -L -o sbom-converter "$URL"
          chmod +x sbom-converter

          # Build SMT using dependency (PURL) extraction
          # Note: Only 'dependency' type is currently supported by the proving-service
          OUTPUT_FILE="sboms/app-smt.json"
          OUTPUT=$(./sbom-converter build --file "$SBOM_FILE" --output "$OUTPUT_FILE" --type dependency 2>&1)

          echo "$OUTPUT"
          ROOT_HASH=$(echo "$OUTPUT" | tail -n1 | tr -d '[:space:]')

          [[ ! "$ROOT_HASH" =~ ^[0-9a-fA-F]{64}$ ]] && { echo "ERROR: Invalid root hash"; exit 1; }

          echo "root_hash=$ROOT_HASH" >> $GITHUB_OUTPUT
          echo "SMT Root Hash: $ROOT_HASH"

      # Store SMT data for later proof generation
      - name: Store SMT
        run: |
          MERKLE_URL="http://merkle-proof-service.sharing-sbom-system.svc.cluster.local:8090"
          SMT_FILE="sboms/app-smt.json"
          ROOT_HASH="${{ steps.build_smt.outputs.root_hash }}"

          REQUEST=$(mktemp)
          jq -n --arg root "$ROOT_HASH" --slurpfile smt "$SMT_FILE" \
            '{root_hash: $root, smt_data: $smt[0]}' > "$REQUEST"

          RESPONSE=$(curl -X POST "$MERKLE_URL/store-smt" \
            -H "Content-Type: application/json" \
            --data @"$REQUEST" \
            --max-time 60 \
            --write-out "\nHTTP_CODE:%{http_code}" \
            --silent)

          rm -f "$REQUEST"

          HTTP_CODE=$(echo "$RESPONSE" | grep "HTTP_CODE:" | cut -d: -f2)
          [ "$HTTP_CODE" != "200" ] && [ "$HTTP_CODE" != "201" ] && {
            echo "ERROR: Failed to store SMT (HTTP $HTTP_CODE)"
            exit 1
          }

      - name: Read SBOM hash
        id: sbom_hash
        run: |
          SBOM_HASH=$(cat sboms/app.sha256)
          echo "sbom_hash=$SBOM_HASH" >> $GITHUB_OUTPUT

      # Write immutable record to blockchain
      - name: Write SMT root to blockchain
        run: |
          CONTRACT_ADDR="0x5FbDB2315678afecb367f032d93F642f64180aa3"
          ROOT_HASH="${{ steps.build_smt.outputs.root_hash }}"
          SBOM_HASH="${{ steps.sbom_hash.outputs.sbom_hash }}"
          IMAGE_DIGEST="${{ needs.build-and-scan.outputs.image_digest }}"

          [[ "$IMAGE_DIGEST" == sha256:* ]] && SOFTWARE_DIGEST="${IMAGE_DIGEST#sha256:}" || SOFTWARE_DIGEST="$IMAGE_DIGEST"

          echo "Writing to blockchain:"
          echo "  Root Hash: $ROOT_HASH"
          echo "  Software Digest: sha256:$SOFTWARE_DIGEST"
          echo "  SBOM Hash: $SBOM_HASH"

          HARDHAT_POD=$(kubectl get pods -n blockchain -l app=hardhat-node -o jsonpath='{.items[0].metadata.name}')
          IDENT="$(echo '${{ github.repository }}' | tr '[:upper:]' '[:lower:]'):${{ github.run_id }}"

          TX_OUTPUT=$(kubectl exec -n blockchain "$HARDHAT_POD" -- sh -c \
            "cd /workspace && \
             ADDR='$CONTRACT_ADDR' \
             ROOT_HASH='$ROOT_HASH' \
             SOFTWARE_DIGEST='$SOFTWARE_DIGEST' \
             SBOM_HASH='$SBOM_HASH' \
             IDENT='$IDENT' \
             npx hardhat run store_smt_root.js --network localhost 2>&1")

          echo "$TX_OUTPUT"

          if echo "$TX_OUTPUT" | grep -q "SKIPPED"; then
            echo "SMT root already registered"
          else
            TX_HASH=$(echo "$TX_OUTPUT" | grep -E "^0x[0-9a-fA-F]{64}$" | tail -n1)
            echo "Transaction: $TX_HASH"
          fi
```

## Customization Points

### Image Name

Update these locations to match your image name:

```yaml
# In the metadata step
images: ghcr.io/${{ github.repository }}/myapp  # Change 'myapp'

# In the SBOM generation step
image: ghcr.io/${{ github.repository }}/myapp:latest  # Change 'myapp'
```

### Dockerfile Path

If your Dockerfile is not at the repository root:

```yaml
file: ./path/to/Dockerfile
```

## Workflow Output

After successful execution:

1. **Docker image** pushed to GitHub Container Registry
2. **SBOM artifact** retained for 30 days (downloadable from Actions UI)
3. **SMT** stored in merkle-proof-service (indexed by root hash)
4. **Root hash** recorded on blockchain with metadata

The SMT root hash is now your cryptographic commitment. Share it with customers to enable verification requests.

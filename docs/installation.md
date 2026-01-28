---
sidebar_position: 2
---

# Installation

TrustBOM is deployed using ArgoCD on a local Kubernetes cluster powered by Kind (Kubernetes in Docker).

:::note System Requirements
This system was developed and tested on **Ubuntu 24.04 LTS** (kernel 6.8.0-90-generic). Other operating systems are not officially supported.
:::

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (must be running)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- git
- jq

## Setup

### 1. Clone and Configure

```bash
git clone https://github.com/tuberlin-blockchain-prototyping/sharing-sbom-system.git
cd sharing-sbom-system
cp .env.example .env
```

### 2. Edit .env

Fill in your GitHub credentials:

```
ABP_ACTIONS_RUNNER_APP_ID=<your-github-app-id>
ABP_ACTIONS_RUNNER_APP_INSTALLATION_ID=<your-github-app-installation-id>
PRIVATE_KEY_FILE=<path-to-private-key.pem>
GITHUB_TOKEN=<your-github-token>
```

### 3. Run Setup

```bash
./scripts/setup.sh
```

This will:
1. Create a Kind cluster named `sharing-sbom-system`
2. Install ArgoCD for GitOps deployment
3. Deploy Hardhat blockchain node with smart contracts
4. Set up GitHub Actions Runner Controller

### 4. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 (username: `admin`, password shown during setup)

## Next Steps

- [Usage Guide](./guides/usage) - Learn how to generate and verify proofs

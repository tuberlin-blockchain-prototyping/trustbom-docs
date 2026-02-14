---
sidebar_position: 2
---

# Installation

TrustBOM is deployed using ArgoCD on a local Kubernetes cluster powered by Kind (Kubernetes in Docker).

:::note System Requirements
This system was developed and tested on **Ubuntu 24.04 LTS** (kernel 6.8.0-90-generic). Other operating systems are not officially supported.
:::

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) 
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- git
- jq

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/tuberlin-blockchain-prototyping/sharing-sbom-system.git
cd sharing-sbom-system
cp .env.example .env
```

### 2. Create a GitHub App

TrustBOM uses a self-hosted GitHub Actions Runner to execute CI/CD jobs inside the Kubernetes cluster. This requires a GitHub App for authentication.

#### Step 2.1: Create the App

1. Go to your GitHub organization's settings
2. Navigate to **Developer settings** → **GitHub Apps** → **New GitHub App**
3. Fill in the required fields:
   - **GitHub App name**: `TrustBOM Runner` (or any unique name)
   - **Homepage URL**: `https://github.com/tuberlin-blockchain-prototyping` (can be any URL)
   - **Webhook**: Uncheck "Active" (not needed)

#### Step 2.2: Set Permissions

Under **Repository permissions**, set:
- **Actions**: Read and write
- **Artifact metadata** Read-only
- **Attestations** Read-only
- **Metadata**: Read-only

Under **Organization permissions**:
- **Self-hosted runners**: Read and write

#### Step 2.3: Finish Creation and Retrieve App ID

1. Click **Create GitHub App**
2. Note the **App ID** displayed on the app's page

#### Step 2.4: Generate a Private Key

1. On the app's page, scroll to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will be downloaded - save it to the `sharing-sbom-system` directory

#### Step 2.5: Install the App

1. On the app's page, click **Install App** in the left sidebar
2. Select your organization
3. Choose **Only select repositories** and select the repositories that will use TrustBOM
4. Click **Install**
5. After installation, note the **Installation ID** from the URL:
   ```
   https://github.com/settings/installations/12345678
                                              ^^^^^^^^
                                              This is your Installation ID
   ```

:::info Public Repositories
If you want to enable the runner for public repositories, you must make sure this is enabled for the group that is set for the runner.
For this, go to your GitHub organization's settings and navigate to **Actions** → **Runner groups**.
Select the group of the runner and enable **Allow public repositories**.
:::

### 3. Configure Environment

Edit the `.env` file with your GitHub App credentials:

```bash
ABP_ACTIONS_RUNNER_APP_ID=123456
ABP_ACTIONS_RUNNER_APP_INSTALLATION_ID=12345678
PRIVATE_KEY_FILE=your-app-name.private-key.pem
```

| Variable | Description |
|----------|-------------|
| `ABP_ACTIONS_RUNNER_APP_ID` | The App ID from step 2.3 |
| `ABP_ACTIONS_RUNNER_APP_INSTALLATION_ID` | The Installation ID from step 2.5 |
| `PRIVATE_KEY_FILE` | Path to the `.pem` file (absolute) |

### 4. Run Setup

```bash
./scripts/setup.sh
```

This will:
1. Create a Kind cluster named `sharing-sbom-system`
2. Install ArgoCD for GitOps deployment
3. Deploy Hardhat blockchain node with smart contracts
4. Install cert-manager
5. Install Actions Runner Controller (ARC)
6. Deploy self-hosted runners

:::info Setup Duration
The initial setup takes several minutes as it downloads container images and waits for services to be ready.
:::

### 5. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 (username: `admin`, password shown during setup)

## Next Steps

- [Usage Guide](./guides/usage) - Learn how to generate and verify proofs

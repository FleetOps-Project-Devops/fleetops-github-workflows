# FleetOps CD Workflow Guide

## Overview

The FleetOps CD workflow provides a reusable GitOps deployment mechanism that updates Helm chart values in the `fleetops-deployments` repository, triggering ArgoCD to deploy new image versions.

## Architecture

```
Application Repo (CI) → CD Workflow → Manifest Repo (Helm Values) → ArgoCD → Kubernetes
```

## Required Repository Secrets

Configure these secrets in your application repository (e.g., `fleetops-auth-service`):

| Secret | Description | Example |
|--------|-------------|---------|
| `APP_ID` | GitHub App ID for authentication | `123456` |
| `APP_PRIVATE_KEY` | GitHub App private key (PEM format) | `-----BEGIN RSA PRIVATE KEY-----...` |
| `MANIFEST_REPO` | Full path to Helm charts repository | `johann2003/fleetops-deployments` |

### GitHub App Setup

1. Create a GitHub App in your organization
2. Grant repository write permissions to `fleetops-deployments`
3. Generate a private key and add as `APP_PRIVATE_KEY` secret
4. Note the App ID and add as `APP_ID` secret

## GitHub Environment Setup (Production Approval Gate)

### Create Production Environment

1. Go to repository **Settings** → **Environments**
2. Click **New environment**
3. Name: `production`
4. Configure protection rules:
   - **Required reviewers**: Add team members who must approve
   - **Wait timer**: Optional (e.g., 5 minutes)
   - **Deployment branches**: Restrict to `main` branch

### How Reviewer Gate Works

When the `update-prod` job runs:

1. GitHub detects the `environment: production` declaration
2. Workflow pauses automatically
3. Configured reviewers receive notification
4. Reviewers must approve in the Actions UI
5. After approval, job continues and updates Helm values
6. No approval = job never runs (automatic security)

**Cannot bypass**: The environment gate is enforced at GitHub platform level, not workflow level.

## Workflow Files

### 1. Reusable CD Workflow

**Location**: `.github/workflows/common_cd.yml`

**Inputs**:
- `service-name`: Chart directory name (e.g., `auth-service`)
- `ci-workflow-name`: Name of triggering CI workflow
- `deploy-env`: `dev` or `prod`

**Jobs**:
- `prepare`: Resolves image tag strategy
- `update-dev`: Updates `values-dev.yaml` (no approval)
- `update-prod`: Updates `values-prod.yaml` (requires approval)

### 2. Service Caller Workflow Example

**Location**: `.github/workflows/auth-service-cd-example.yml`

Triggers on CI workflow completion for `develop` or `main` branches.

## Deployment Flows

### Dev Flow (No Approval Required)

```
1. Developer pushes to develop branch
2. CI workflow builds and pushes image: docker.io/johann2003/auth-service:develop-abc1234
3. CI completes successfully
4. CD workflow triggers (detects develop branch)
5. prepare job: resolves tag to commit SHA (abc1234)
6. update-dev job runs:
   - Checks out fleetops-deployments repo
   - Updates charts/auth-service/values-dev.yaml
   - Sets image.tag: "abc1234"
   - Commits with: "chore(auth-service): bump dev image tag to abc1234 [skip ci]"
7. ArgoCD detects change in fleetops-dev namespace
8. ArgoCD syncs new image to dev cluster
```

**Key Points**:
- Uses short commit SHA (7 characters)
- No manual approval
- Fast iteration for development
- `[skip ci]` prevents recursive CI triggers

### Prod Flow (Manual Approval Required)

```
1. Developer merges to main branch
2. CI workflow builds and pushes image: docker.io/johann2003/auth-service:latest + v1.2.3
3. CI completes successfully
4. CD workflow triggers (detects main branch)
5. prepare job: resolves tag to latest SemVer (v1.2.3)
6. update-prod job starts:
   - GitHub detects environment: production
   - Workflow PAUSES for reviewer approval
7. Reviewers receive notification
8. Reviewer approves in Actions UI
9. Job continues:
   - Checks out fleetops-deployments repo
   - Updates charts/auth-service/values-prod.yaml
   - Sets image.tag: "v1.2.3"
   - Commits with: "release(auth-service): bump prod image tag to v1.2.3 [skip ci]"
10. ArgoCD detects change in fleetops-prod namespace
11. ArgoCD syncs new image to prod cluster
```

**Key Points**:
- Uses SemVer tag (v1.2.3)
- Manual approval required via GitHub Environment
- Controlled release process
- Cannot bypass approval gate

## Helm Values Convention

Each service chart structure:

```
charts/
├── auth-service/
│   ├── values.yaml
│   ├── values-dev.yaml    ← Updated by CD (dev)
│   └── values-prod.yaml   ← Updated by CD (prod)
├── vehicle-service/
│   ├── values.yaml
│   ├── values-dev.yaml
│   └── values-prod.yaml
└── ...
```

**Values file format**:

```yaml
image:
  registry: "docker.io"
  repository: johann2003/auth-service
  tag: "latest"  # ← This field is updated by CD
  pullPolicy: Always
```

## Commit Message Convention

**Dev commits**:
```
chore(auth-service): bump dev image tag to abc1234 [skip ci]
```

**Prod commits**:
```
release(auth-service): bump prod image tag to v1.2.3 [skip ci]
```

The `[skip ci]` suffix prevents the commit from triggering CI workflows in the manifest repository.

## Adapting for Other Services

Copy the example caller workflow and modify:

```yaml
name: Vehicle Service CD

on:
  workflow_run:
    workflows: ["Reusable Java CI"]  # or Reusable Node CI for frontend
    types: [completed]
    branches:
      - develop
      - main

jobs:
  cd-dev:
    # Change service-name to vehicle-service
    with:
      service-name: vehicle-service
      ci-workflow-name: Reusable Java CI
      deploy-env: dev
    secrets:
      APP_ID: ${{ secrets.APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
      MANIFEST_REPO: ${{ secrets.MANIFEST_REPO }}

  cd-prod:
    # Change service-name to vehicle-service
    with:
      service-name: vehicle-service
      ci-workflow-name: Reusable Java CI
      deploy-env: prod
    secrets:
      APP_ID: ${{ secrets.APP_ID }}
      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
      MANIFEST_REPO: ${{ secrets.MANIFEST_REPO }}
```

## Validation Checklist

- [x] yq syntax correct for Helm value updates
- [x] No accidental update of wrong env file (job conditions prevent this)
- [x] Prod cannot bypass environment approval (enforced by GitHub platform)
- [x] Workflow reusable by all FleetOps services (parameterized inputs)
- [x] Outputs passed correctly between jobs (needs.prepare.outputs)

## Troubleshooting

### CD not triggering after CI

- Verify caller workflow `on.workflow_run.workflows` matches CI workflow name exactly
- Check CI workflow completed successfully (not failed or cancelled)
- Ensure branch matches (develop for dev, main for prod)

### Approval not requested for prod

- Verify `environment: production` is set in update-prod job
- Check GitHub Environment exists and has reviewers configured
- Ensure job runs on main branch

### Helm values not updating

- Check MANIFEST_REPO secret is correct
- Verify GitHub App has write access to fleetops-deployments
- Confirm values file path matches chart structure
- Check yq is installed in runner (default in GitHub Actions)

### ArgoCD not deploying

- Verify ArgoCD is watching the fleetops-deployments repo
- Check ArgoCD application sync policy (auto-sync enabled?)
- Confirm namespace matches (fleetops-dev or fleetops-prod)
- Check ArgoCD sync-wave ordering for dependencies

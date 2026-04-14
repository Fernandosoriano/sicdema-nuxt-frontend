# CI/CD Setup Guide — sicdema-nuxt-frontend

This guide documents the full process of forking the `sicdema-nuxt-frontend` repository and configuring its CI/CD pipeline from scratch. A new developer joining the project should be able to follow these steps end to end.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Forking the Repository](#2-forking-the-repository)
3. [Cloning the Fork Locally](#3-cloning-the-fork-locally)
4. [Adapting the Workflow Files](#4-adapting-the-workflow-files)
5. [Configuring the GitHub Environment](#5-configuring-the-github-environment)
6. [Required Secrets](#6-required-secrets)
7. [Required Variables](#7-required-variables)
8. [Triggering the Pipeline](#8-triggering-the-pipeline)
9. [Re-enabling Deployment When a Server is Available](#9-re-enabling-deployment-when-a-server-is-available)
10. [Workflow File Reference](#10-workflow-file-reference)

---

## 1. Prerequisites

Before starting, make sure you have the following:

- A **GitHub account** (free plan is sufficient — the repo must be public)
- **Git** installed locally (or WSL with Ubuntu if on Windows)
- **VS Code** with the WSL extension (recommended for Windows users)
- A **GitHub Personal Access Token (PAT)** with `write:packages` scope — used to push Docker images to GitHub Container Registry (GHCR)

### Generating the PAT

1. Go to `github.com/settings/tokens`
2. Click **Generate new token → Generate new token (classic)**
3. Set a name (e.g. `sicdema-frontend-cicd`) and expiration (90 days recommended)
4. Check the `write:packages` scope (this auto-selects `read:packages` too)
5. Click **Generate token** and copy it immediately — you will not see it again

---

## 2. Forking the Repository

The original repository lives at `github.com/CentroGeo/sigic-nuxt-frontend`.

1. Go to the original repo on GitHub
2. Click the **Fork** button (top right)
3. Select your personal account or organization as the destination
4. Optionally rename the fork (e.g. `sicdema-nuxt-frontend`)
5. Click **Create fork**

> **Note:** The fork should be set to **Public** so that GitHub Actions runs for free with no minute limits.

---

## 3. Cloning the Fork Locally

```bash
git clone https://github.com/YOUR_USERNAME/sicdema-nuxt-frontend.git
cd sicdema-nuxt-frontend
```

Verify the remote is pointing to your fork:

```bash
git remote -v
```

---

## 4. Adapting the Workflow Files

The original workflows were hardcoded to the CentroGeo organization. The following changes must be made before the pipeline can run on your fork.

The workflow files are located at `.github/workflows/`:

- `docker-compose-develop.yml` — builds and deploys on pushes to `develop`
- `sigic-release.yml` — builds and pushes on version tags (`v*`)

### Changes Made

| What | Before | After |
|------|--------|-------|
| Secret name | `GHCR_GEOINTSIGIC_PAT` | `GHCR_FERNANDOSORIANO_PAT` |
| GHCR login user | `geointsigic` | `Fernandosoriano` |
| Image owner | `centrogeo` | `fernandosoriano` |
| Image name | `sigic-nuxt-frontend` | `sicdema-nuxt-frontend` |
| Hardcoded URLs | `sigic.geoint.mx` / `sigic.dev.geoint.mx` | `${{ vars.NUXT_PUBLIC_BASE_URL }}` |
| Build runner | `self-hosted` | `ubuntu-latest` |
| Deploy job | active | disabled with `if: false` (until a server is available) |

---

## 5. Configuring the GitHub Environment

The workflows use a GitHub Environment called `develop` to scope secrets and variables.

### Creating the Environment

1. Go to your fork on GitHub
2. Navigate to **Settings → Environments → New environment**
3. Name it exactly `develop`
4. Click **Configure environment**

### Protection Rules

For a development/testing setup, leave all protection rules **unchecked**:

- Required reviewers → unchecked
- Wait timer → 0
- Deployment branches → No restriction

Click **Save protection rules**.

---

## 6. Required Secrets

Add these in **Settings → Environments → develop → Add environment secret**.

| Secret | Description | Placeholder OK? |
|--------|-------------|-----------------|
| `GHCR_FERNANDOSORIANO_PAT` | Personal Access Token with `write:packages` scope | ❌ Real value required |
| `KEYCLOAK_CLIENT_ID` | Keycloak client ID for the frontend app | ✅ Use `placeholder` |
| `KEYCLOAK_CLIENT_SECRET` | Keycloak client secret | ✅ Use `placeholder` |
| `NUXT_PUBLIC_KEYCLOAK_CLIENT_ID` | Public-facing Keycloak client ID | ✅ Use `placeholder` |
| `NUXT_AUTH_SECRET` | Random string for signing auth sessions — generate with `openssl rand -base64 32` | ✅ Any random string |

---

## 7. Required Variables

Add these in **Settings → Environments → develop → Add environment variable**.

| Variable | Placeholder Value |
|----------|------------------|
| `NUXT_PUBLIC_BASE_URL` | `http://localhost:3000` |
| `NUXT_PUBLIC_GEONODE_URL` | `http://placeholder.com` |
| `NUXT_PUBLIC_GEONODE_API` | `http://placeholder.com/api/v2` |
| `NUXT_PUBLIC_GEOSERVER_URL` | `http://placeholder.com/geoserver` |
| `NUXT_PUBLIC_IA_BACKEND_URL` | `http://placeholder.com` |
| `NUXT_PUBLIC_LEVANTAMIENTO_BACKEND_URL` | `http://placeholder.com` |
| `NUXT_PUBLIC_DEFAULT_PAGE` | `/` |
| `NUXT_PUBLIC_GEONODE_API_DEFAULT_FILTER` | `placeholder` |
| `NUXT_PUBLIC_ENABLE_AUTH` | `false` |
| `NUXT_PUBLIC_ENABLE_CATALOGO_VISTA` | `false` |
| `NUXT_PUBLIC_ENABLE_CATALOGO_CARGA` | `false` |
| `NUXT_PUBLIC_ENABLE_CONSULTA` | `false` |
| `NUXT_PUBLIC_ENABLE_IAA` | `false` |
| `NUXT_PUBLIC_ENABLE_LEVANTAMIENTO` | `false` |
| `NUXT_AUTH_ORIGIN` | `http://localhost:3000` |
| `NUXT_PUBLIC_AUTH_BASE_URL` | `http://placeholder.com` |
| `KEYCLOAK_ISSUER` | `http://placeholder.com` |
| `NUXT_PUBLIC_KEYCLOAK_ISSUER` | `http://placeholder.com` |
| `NUXT_APP_BASE_URL` | `/` |
| `NUXT_PUBLIC_OLLAMA_MODEL` | `deepseek-r1` |

> **Important:** GitHub variable names cannot contain `*` wildcards. Each feature flag must be added individually.

---

## 8. Triggering the Pipeline

### Enabling Actions on a Fork

1. Go to your fork → **Actions tab**
2. Click **"I understand my workflows, go ahead and enable them"**

### First Pipeline Run

```bash
git commit --allow-empty -m "Trigger CI pipeline"
git push origin develop
```

### Expected Jobs

| Job | Description |
|-----|-------------|
| `Lint Nuxt frontend code` | Runs ESLint and type checks |
| `Build SIGIC Nuxt Frontend image (develop)` | Builds and pushes Docker image to GHCR |
| `Deploy SIGIC Nuxt frontend to dev server` | Disabled (`if: false`) until a server is available |

A successful run pushes a Docker image to:

ghcr.io/fernandosoriano/sicdema-nuxt-frontend:latest
---

## 9. Re-enabling Deployment When a Server is Available

### Step 1 — Set up a self-hosted runner on the server

Go to **Settings → Actions → Runners → New self-hosted runner** and register it with these labels:

self-hosted, deploy-develop, sicdema-nuxt-frontend

### Step 2 — Remove `if: false` from the deploy job in `docker-compose-develop.yml`

```yaml
# Before (disabled):
deploy:
  if: false
  runs-on: [self-hosted, deploy-develop, sicdema-nuxt-frontend]

# After (enabled):
deploy:
  runs-on: [self-hosted, deploy-develop, sicdema-nuxt-frontend]
```

### Step 3 — Replace all placeholder values with real ones

Update secrets and variables in the `develop` environment with real URLs and credentials.

### Step 4 — Push the change

```bash
git add .github/workflows/docker-compose-develop.yml
git commit -m "re-enable deploy job for dev server"
git push origin develop
```

---

## 10. Workflow File Reference

### `docker-compose-develop.yml`
- **Trigger:** Push to `develop` branch
- **Jobs:** Lint → Build & Push → Deploy (disabled)
- **Registry:** GitHub Container Registry (GHCR)
- **Environment:** `develop`

### `sigic-release.yml`
- **Trigger:** Push of version tags (`v*`) or manual `workflow_dispatch`
- **Jobs:** Build & Push
- **Registry:** GHCR
- **Image name:** Auto-resolved from `github.repository`

---

## Troubleshooting

**"The job was not started because your account is locked due to a billing issue"**
Go to `github.com/settings/billing` and resolve the billing issue, or ensure you are on the free plan with a public repository.

**Pipeline does not trigger on push**
Make sure you enabled Actions on the fork — forked repos have Actions disabled by default.

**Secret not found error**
Secret names are case-sensitive — verify the name in the workflow file exactly matches the name set in the GitHub environment.

**`NUXT_PUBLIC_ENABLE_*` variable rejected**
GitHub does not allow `*` wildcards in variable names. Add each feature flag individually.
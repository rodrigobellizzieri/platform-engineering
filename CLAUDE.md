# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a platform engineering monorepo with two main areas:

- **`argocd/`** — GitOps configuration for Kubernetes, using an app-of-apps pattern with ArgoCD
- **`devops-portal/`** — A [Backstage](https://backstage.io) developer portal (Internal Developer Platform)

## ArgoCD (GitOps)

### Structure

- `argocd/applications/app-of-apps.yaml` — Root ArgoCD Application that bootstraps everything. It watches `argocd/applications/*/application.yaml` and syncs them automatically.
- `argocd/applications/<name>/application.yaml` — Each application registered in ArgoCD. Adding a file here causes ArgoCD to deploy it.
- `argocd/manifests/<namespace>/` — Raw Kubernetes manifests deployed by the corresponding ArgoCD application.

### How it works

The app-of-apps points to this repo's `main` branch. Any merge to `main` triggers automated sync with `prune: true` and `selfHeal: true`. New applications are added by creating `argocd/applications/<name>/application.yaml`; their manifests go under `argocd/manifests/<name>/`.

Target cluster is `https://kubernetes.default.svc` (in-cluster). ArgoCD ingress is at `argocd.homelab.io`.

## Backstage (DevOps Portal)

All commands must be run from `devops-portal/`. Node 22 or 24 is required; package manager is Yarn 4.4.1.

### Development

```sh
cd devops-portal
yarn install
yarn start          # starts both frontend (port 3000) and backend (port 7007)
```

### Build

```sh
yarn build:all      # build everything
yarn build:backend  # build backend only
yarn build-image    # build Docker image (runs from backend/ with root Dockerfile)
```

### Test & Lint

```sh
yarn test           # run all unit tests
yarn test:all       # run with coverage
yarn test:e2e       # run Playwright e2e tests
yarn lint           # lint files changed since origin/master
yarn lint:all       # lint all files
yarn tsc            # type check
```

### Architecture

Backstage uses a **plugin-based monorepo** with two packages:

- **`packages/app/`** — React frontend. Entry point is `src/App.tsx`, which registers features (plugins + modules). Currently has the catalog plugin and a custom nav module.
- **`packages/backend/`** — Node.js backend. Entry point is `src/index.ts`, which wires backend plugins using `createBackend()`. Active plugins: app, proxy, scaffolder (with GitHub module), techdocs, auth (guest provider), catalog, permission (allow-all policy), search (PostgreSQL engine), Kubernetes, notifications/signals, and MCP Actions.

**Frontend nav customization** (`packages/app/src/modules/nav/`): The default Backstage nav items (search, catalog, scaffolder, user-settings) are disabled in `app-config.yaml` and reimplemented in `Sidebar.tsx` using `NavContentBlueprint`. Any new nav items should follow this pattern.

**Config layering**: `app-config.yaml` is the base (uses SQLite in-memory for local dev). `app-config.production.yaml` overrides the database to PostgreSQL via env vars (`POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`). GitHub integration requires `GITHUB_TOKEN`.

**Plugins directory** (`plugins/`): currently empty — custom plugins go here and are auto-discovered via the `workspaces` config in `package.json`.

## Security Note

`argocd/manifests/crossplane-system/secret-provider.yaml` contains **hardcoded AWS credentials**. These should be rotated and replaced with a secrets management solution (e.g., External Secrets Operator or a Kubernetes Secret sourced from a vault).

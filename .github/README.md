# CI/CD Best Practices & Workflow Templates

## Overview

This repository contains our reusable GitHub Actions workflows for consistent CI/CD across all our applications. All projects follow RC semantic versioning for both applications and Helm charts.

## Release Flow

### RC Releases (Automatic)
- **Trigger**: PR to `main` branch
- **Process**: 
  1. Build and push Docker images with SHA tags
  2. Update dev cluster with new image digests
  3. Release Please creates semantic versions
  4. Build and publish Helm charts with RC tags
  5. Update dev cluster with new chart versions
- **Result**: Auto-merged PRs to dev cluster on green checks

### GA Promotion (Manual)
- **Trigger**: Manual workflow dispatch in app repo
- **Process**:
  1. Extract current dev versions (chart + image digests)
  2. Retag Docker images with GA semantic versions
  3. Create GA Helm chart release via Release Please
  4. Build and publish GA chart
  5. Promote dev versions to production cluster
- **Result**: PR to production cluster for manual review

## Workflow Templates

### Docker & Images
- **`docker-build-and-push.yml`**: Build multi-component Docker images, push to GHCR with SHA tags, output digests
- **`docker-image-retag-ga.yml`**: Retag existing images with semantic versions (RC/GA) using digests as source

### Helm Charts
- **`helm-chart-build-and-publish.yml`**: Build and publish Helm charts to OCI registry
- **`release-please.yml`**: Semantic versioning and release management using Release Please

### Cluster Configuration
- **`cluster-config-bump-image-digests.yml`**: Update ArgoCD applications with new image digests (dev)
- **`cluster-config-bump-chart-version.yml`**: Update ArgoCD applications with new chart versions (dev)
- **`production-promotion.yml`**: Promote dev versions to production cluster (dev → prod mapping)

## Usage

### For New Applications
1. Copy workflow files from `captide-document-engine/.github/workflows/`
2. Update component names and paths
3. Configure Release Please configs (`release-please-rc.json`, `release-please-ga.json`)
4. Set required secrets and variables

### Required Secrets
- `CAPTIDE_APP_ID`: GitHub App ID for cluster config access
- `CAPTIDE_APP_PRIVATE_KEY`: GitHub App private key

## Best Practices

### Versioning
- **RC**: `1.2.3-rc.1`, `1.2.3-rc.2` (pre-releases)
- **GA**: `1.2.3` (stable releases)
- **Images**: Always use digests for retagging, never SHA tags

### Cluster Updates
- **Dev**: Automatic via CI/CD pipeline
- **Production**: Manual via `production-promotion.yml` workflow in app repo
- **Coalesced PRs**: Multiple updates create/update same PR branch

### Chart Management
- **RC**: New chart package with `-rc` suffix
- **GA**: New chart package from same commit (not retagging)
- **Signing**: Use `--sign` for production charts

# CI/CD Best Practices & Workflow Templates

## Overview

This repository contains our reusable GitHub Actions workflows for consistent CI/CD across all our applications. All projects follow RC semantic versioning for both applications and Helm charts.

## Release Flow

### RC Releases (Automatic)
- **Trigger**: PR to `main` branch
- **Process**: 
  1. Build and push Docker images with SHA tags
  2. Release Please creates semantic versions for both app and chart
  3. Retag images with RC semantic versions
  4. Update dev cluster with new image digests + human-readable tags
  5. Build and publish Helm charts with RC tags
  6. Update dev cluster with new chart versions
- **Result**: Auto-merged PRs to dev cluster on green checks
- **Note**: Both application and Helm chart get RC versions (e.g., `1.8.0-rc.1`)

### GA Promotion (Manual)
- **Trigger**: Manual workflow dispatch in app repo
- **Process**:
  1. Extract current dev versions (chart + image digests + tags)
  2. Retag Docker images with GA semantic versions
  3. Create GA Helm chart release via Release Please
  4. Build and publish GA chart
  5. Promote dev versions to production cluster (including tags)
- **Result**: PR to production cluster for manual review
- **Note**: Both application and Helm chart get GA versions (e.g., `1.8.0`)

## Workflow Templates

### Docker & Images
- **`docker-build-and-push.yml`**: Build multi-component Docker images, push to GHCR with SHA tags, output digests
- **`docker-image-retag-ga.yml`**: Retag existing images with semantic versions (RC/GA) using digests as source, outputs component→version mapping

### Helm Charts
- **`helm-chart-build-and-publish.yml`**: Build and publish Helm charts to OCI registry
- **`release-please.yml`**: Semantic versioning and release management using Release Please
- **`release-please-enable-automerge.yml`**: Enable auto-merge on Release Please PRs (reusable)

### Cluster Configuration
- **`cluster-config-bump-image-digests.yml`**: Update ArgoCD applications with new image digests + human-readable tags (dev)
- **`cluster-config-bump-chart-version.yml`**: Update ArgoCD applications with new chart versions (dev)
- **`production-promotion.yml`**: Promote dev versions to production cluster (dev → prod mapping, including tags)

## Image Tag Implementation

### Features
- **Human-readable tags** alongside digests for better operability
- **Zero change to pull semantics** - still pin by digest for reproducibility
- **Pod labels** for `kubectl` and dashboard visibility
- **Backward compatible** - existing deployments continue working

### New Inputs
- `release_version`: Global semver (e.g., `1.8.0-rc.1`)
- `image_tags_json`: Component-specific mapping (e.g., `{"api":"1.8.0-rc.1"}`)

### Helm Values Structure
```yaml
image:
  repository: ghcr.io/owner/repo/component
  digest: sha256:abc123...  # Source of truth for pulls
  tag: 1.8.0-rc.1          # Human-readable metadata

podLabels:
  app.kubernetes.io/version: 1.8.0-rc.1  # For kubectl visibility
```

## Usage

### For New Applications
1. Copy workflow files from `captide-document-engine/.github/workflows/`
2. Update component names and paths
3. Configure Release Please configs (`release-please-rc.json`, `release-please-ga.json`)
4. Set required secrets and variables

### Creating RC Pipeline in App Repos
App repos should create their own orchestrator workflows using the reusable templates. Reference the workflow files in this repository for implementation examples.

### Auto-merge Setup for Release Please RC PRs
To enable automatic merging of Release Please RC PRs, add the auto-merge workflow to your app repo. See the workflow templates for the complete implementation.

**Repository Settings Required:**
- Settings → General → **Allow auto-merge** ✅
- Branch protection on `main`: **Require status checks** (your CI `build` job etc.)
- Do **not** require push-only jobs (retag/publish/bump) on PRs

### Required Secrets
- `CAPTIDE_APP_ID`: GitHub App ID for cluster config access
- `CAPTIDE_APP_PRIVATE_KEY`: GitHub App private key

## Best Practices

### Versioning
- **RC**: `1.2.3-rc.1`, `1.2.3-rc.2` (pre-releases)
- **GA**: `1.2.3` (stable releases)
- **Images**: Always use digests for retagging, never SHA tags
- **Tags**: Human-readable metadata alongside digests for operability
- **Per-component semver**: Each component should have independent versioning (e.g., multiple background workers versioned separately)

#### Tag Prefixes (Recommended)
- **App tags**: `v2.0.0`, `v2.0.0-rc.1` (with `v` prefix)
- **Helm chart tags**: `helm-v1.5.2`, `helm-v1.5.2-rc.1` (with `helm-` prefix)
- **Rationale**: Clear distinction between application and infrastructure releases, especially when using Release Please for both

### Build Optimization
- **Selective builds**: Use `dorny/paths-filter` to only build components that have changed
- **Lightweight images**: Keep Docker images as small as possible for faster builds and deployments
- **Fast workflows**: Optimize workflow execution time through efficient caching and parallel jobs

### Cluster Updates
- **Dev**: Automatic via CI/CD pipeline
- **Production**: Manual via `production-promotion.yml` workflow in app repo
- **Coalesced PRs**: Multiple updates create/update same PR branch
- **Tag Preservation**: Image tags flow through promotion pipeline

### Chart Management
- **RC**: New chart package with `-rc` suffix
- **GA**: New chart package from same commit (not retagging)
- **Signing**: Use `--sign` for production charts

### Changelog Management
- **App Changelog**: `CHANGELOG.md` in repository root
- **Chart Changelog**: `deploy/helm/{app-name}/CHANGELOG.md`
- **Release Please Configs**: 
  - `.github/release-please-rc.json` for RC releases
  - `.github/release-please-ga.json` for GA releases
- **Conventional Commits**: Use `feat:`, `fix:`, `breaking:` prefixes for automatic changelog generation
- **Separate PRs**: Each package (app/chart) gets its own release PR when changes are detected
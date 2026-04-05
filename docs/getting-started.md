# Getting Started with CascadeGuard

CascadeGuard automates container image lifecycle management. It monitors base images for updates, discovers Dockerfile dependencies, and orchestrates intelligent rebuilds through a GitOps workflow using Kargo and ArgoCD.

## Prerequisites

- Docker (for running CascadeGuard without a local Python install)
- A GitHub repository to use as your **state repository**
- Kargo and ArgoCD running in your Kubernetes cluster (for full orchestration)

## Concepts

### State Repository

CascadeGuard is driven by a **state repository** — a Git repo that declares which images you manage and tracks their current state. It contains:

- `images.yaml` — enrollment configuration for all managed images
- `cascadeguard/state/images/` — per-image state files
- `cascadeguard/state/base-images/` — tracked upstream base image state
- `.cascadeguard.yaml` — tool configuration (CI platform, etc.)
- `.github/workflows/` — auto-generated CI pipelines

See [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) for a complete working example.

### Image Types

| Type | Description |
|---|---|
| **Base images** | Foundational images your application images build from. CascadeGuard monitors these for updates and triggers rebuilds of dependents. |
| **Managed images** | Images built by your CI/CD pipeline. CascadeGuard discovers their Dockerfile dependencies and monitors the base images they use. |
| **External images** | Third-party images tracked directly for new versions. |

## Quick Start

### 1. Initialize your state repository

Create a new Git repository and add the following `images.yaml`:

```yaml
images:
  - name: my-app
    registry: ghcr.io
    repository: your-org/my-app
    source:
      provider: github
      repo: your-org/my-app
      dockerfile: Dockerfile
      branch: main
    rebuild_delay: 7d
```

Add a `.cascadeguard.yaml` for tool configuration:

```yaml
ci:
  platform: github
```

### 2. Include the shared Taskfile

Add a `Taskfile.yaml` to your state repo:

```yaml
version: '3'
includes:
  shared:
    taskfile: https://raw.githubusercontent.com/cascadeguard/cascadeguard/v1.0.0/Taskfile.shared.yaml
    flatten: true
```

### 3. Generate state files

```bash
task generate
```

This reads `images.yaml`, inspects the Docker registries, and writes state files to `cascadeguard/state/`.

### 4. Generate CI pipelines

```bash
task generate-ci
```

This emits four GitHub Actions workflow files under `.github/workflows/`:

| File | Trigger | Purpose |
|---|---|---|
| `build-image.yaml` | `workflow_call` | Reusable single-image build, scan, SBOM, and signing |
| `ci.yaml` | Push to `main`, pull requests | Matrix build of all images |
| `scheduled-scan.yaml` | Nightly cron | Re-scans all published images; opens issues on new CVEs |
| `release.yaml` | Tag push (`v*`) | Builds, signs, and pushes all images |

### 5. Generate Kargo manifests

```bash
task synth
```

This generates Kargo Warehouses, Stages, and AnalysisTemplates under `dist/`. ArgoCD watches this directory and deploys the manifests to your cluster.

### 6. Commit and push

```bash
git add .
git commit -m "Initial CascadeGuard state"
git push
```

ArgoCD picks up the new manifests and Kargo begins orchestrating your image build pipeline.

## Running CascadeGuard via Docker

No local Python setup is required. All tasks run CascadeGuard via Docker:

```bash
# Generate state files
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 generate

# Generate Kargo manifests
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 synth

# Generate CI pipelines
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 generate-ci
```

## Enrolling a new image

Use the `enrol` command to add a new image to `images.yaml`:

```bash
cascadeguard --images-yaml images.yaml enrol \
  --name my-new-service \
  --registry ghcr.io \
  --repository your-org/my-new-service \
  --provider github \
  --repo your-org/my-new-service \
  --dockerfile Dockerfile \
  --branch main \
  --rebuild-delay 7d
```

Then re-run `generate` and `generate-ci` to update state files and CI pipelines.

## Checking image status

```bash
# View all tracked images
task status
# or
cascadeguard --state-dir cascadeguard/state status
```

Prints a summary table of every application image and base image, including version, digest, build time, and dependency information.

## Next Steps

- [GitHub Actions Integration Guide](integrations/github-actions.md) — how the generated CI pipelines work and how to customize them
- [CLI Reference](reference/cli.md) — full command reference
- [images.yaml Reference](reference/images-yaml.md) — schema and options
- [Security Model](security-model.md) — how CascadeGuard handles scanning, SBOMs, and signing

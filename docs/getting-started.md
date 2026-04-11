# Getting Started with CascadeGuard Image Factory

CascadeGuard Image Factory automates container image lifecycle management. It monitors base images for updates, discovers Dockerfile dependencies, identifies vulnerabilities and orchestrates intelligent rebuilds through your existing build tools & processes.

## Prerequisites

- Python 3.11+ (installed automatically via the install script)
- A repository to use as your **state repository**

## Install

```bash
# macOS / Linux
curl -sSL https://raw.githubusercontent.com/cascadeguard/cascadeguard/main/install.sh | sh

# Windows (PowerShell)
irm https://raw.githubusercontent.com/cascadeguard/cascadeguard/main/install.ps1 | iex
```

This installs CascadeGuard to `~/.cascadeguard` and adds it to your PATH as `cascadeguard` and `cg`.

## Concepts

### State Repository

CascadeGuard is driven by a **state repository** — a Git repo that declares which images you manage and tracks their current state. It contains:

- `images.yaml` — enrollment configuration for all managed images
- `.cascadeguard.yaml` — repo-level defaults and tool configuration
- `base-images/` — tracked upstream base image state (generated)
- `images/` — per-image state files (generated)
- `.github/workflows/` — auto-generated CI pipelines (generated)

See [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) and [cascadeguard-open-secure-images](https://github.com/cascadeguard/cascadeguard-open-secure-images) for complete working examples, and [cascadeguard-seed](https://github.com/cascadeguard/cascadeguard-seed) for the seed repo thats used when cg images init is used.

### Image Types

| Type | Description |
|---|---|
| **Base images** | Foundational images your application images build from. CascadeGuard monitors these for updates and triggers rebuilds of dependents. |
| **Managed images** | Images built by your CI/CD pipeline. CascadeGuard discovers their Dockerfile dependencies and monitors the base images they use. |
| **External images** | Third-party images tracked directly for new versions. |

## Quick Start

### 1. Initialise your repo (optional)

```bash
cg images init
```

Init pulls the latest seed from [cascadeguard-seed](https://github.com/cascadeguard/cascadeguard-seed) giving you a ready to go starting point, including:

- An example image you can build to see it working
- A PR Checker that validates images.yaml to prevent mistakes being merged to main
- A daily scheduled task to monitor the upstream image(s) of your images
 
You can set repo-level defaults so you don't repeat common fields on every image:

```yaml
# .cascadeguard.yaml
defaults:
  registry: ghcr.io/your-org       # applied to all images missing a registry
  local:
    dir: images                     # folder containing per-image Dockerfiles

ci:
  platform: github
```

### 2. Create `images.yaml`

This file is where the images CascadeGuard should monitor & manage are defined. Config from `.cascadeguard.yaml` defaults if not set per-image.

The content can be added to this file directly in the yaml:

```yaml
# Managed images — CascadeGuard builds, scans, and signs these
- name: my-app
  dockerfile: images/my-app/Dockerfile
  image: my-app
  tag: latest

# Upstream-tracked images — CVE monitoring only, no build
- name: memcached
  enabled: false
  namespace: library
```

Or by using the images enrol method:

```bash
cg images enrol \
  --name my-new-service \
  --registry ghcr.io \
  --repository your-org/my-new-service
```

### 3. Validate your configuration

```bash
cg images validate
```

Verifies that the state in images.yaml and .cascadeguard.yaml is valid. This can be used in a PR check or similar.

### 5. Generate state files

```bash
cg images generate
```

This reads `images.yaml`, analyzes Dockerfiles to discover base image dependencies, and writes state files to `base-images/` and `images/`.

### 6. Generate CI pipelines

```bash
cg build generate
```

This emits the GitHub Actions workflow files under `.github/workflows/` based on the cascadeguard-seed](https://github.com/cascadeguard/cascadeguard-seed) repository:

| File | Trigger | Purpose |
|---|---|---|
| `build-image.yaml` | `workflow_call` | Reusable single-image build, scan, SBOM, and signing |
| `ci.yaml` | Push to `main`, pull requests | Matrix build of all images |
| `check.yaml` | Nightly cron | Re-scans all published images; opens issues on new CVEs |
| `release.yaml` | Tag push (`v*`) | Builds, signs, and pushes all images |

Use `--dry-run` to preview without writing files:

```bash
cg build generate --dry-run
```

### 7. Commit and push

```bash
git add images.yaml .cascadeguard.yaml .github/workflows/ .cascadeguard
git commit -m "Initial CascadeGuard state"
git push
```

## Config Inheritance

`.cascadeguard.yaml` supports a `defaults` section. Any field set here is applied to every image in `images.yaml` that doesn't already have that field:

| Key | Description |
|-----|-------------|
| `defaults.registry` | Default container registry (e.g. `ghcr.io/cascadeguard`) |
| `defaults.repository` | Default repository prefix |
| `defaults.local.dir` | Default folder containing per-image Dockerfiles |

Per-image values always take precedence over defaults. Disabled images (`enabled: false`) only require a `name` field.

## Checking image status

```bash
cg images status
cg images check
```

## Next Steps

- [GitHub Actions Integration Guide](integrations/github-actions.md) — how the generated CI pipelines work and how to customize them
- [CLI Reference](reference/cli.md) — full command reference
- [images.yaml Reference](reference/images-yaml.md) — schema and options
- [Security Model](security-model.md) — how CascadeGuard handles scanning, SBOMs, and signing

## How-to Guides

End-to-end walkthroughs for specific CI/CD combinations:

- [GitHub CI + Cloudflare CD](guides/github-cloudflare.md) — GitHub Actions for image build/scan/sign, Cloudflare Pages/Workers for deployment (CascadeGuard's own setup)
- [GitLab CI + Argo CD](guides/gitlab-argocd.md) — GitLab CI/CD pipelines with Argo CD and Kargo for Kubernetes deployment

# How-to: GitHub CI + Cloudflare Workers CD

This guide walks through using CascadeGuard to manage container images in a pipeline where **GitHub Actions** builds, scans, and signs your images and **Cloudflare Workers** serves the deployed container.

You will use the CascadeGuard CLI to enrol your image, generate state files, and generate the complete CI/CD pipeline — including the Cloudflare staging and production deploy steps. See [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) for a working example of the generated output.

## Prerequisites

- A working CascadeGuard installation (`cg --version` returns a version string; see [Getting Started](../getting-started.md) to install)
- A GitHub repository containing a `Dockerfile`
- A Cloudflare account with Workers enabled and `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` ready to add as GitHub repository secrets
- A state repository (a separate Git repo) to host CascadeGuard configuration and generated pipelines

---

## Architecture

```mermaid
graph LR
    A[Developer pushes] --> B[GitHub Actions CI]
    B --> C[cascadeguard build\nbuild container]
    C --> D[cascadeguard scan\ncheck for CVEs]
    D --> E{Pass?}
    E -->|No| F[Fail build\nopen issue]
    E -->|Yes| G[cascadeguard sign\nSBOM + Cosign]
    G --> H[Push container\nto GHCR]
    H --> I[CD: CF staging\ndeploy container]
    I --> J[GitHub tests\nagainst staging]
    J --> K[CD: CF production\ndeploy container]
    L[Nightly cron] --> M[cascadeguard scan\nre-scan published images]
    M --> N[Open GitHub Issue\nif new CVEs found]
```

The flow is:
1. `cg images enrol` adds your image to `images.yaml`, including a deploy workflow configuration.
2. `cg images generate` reads the registry and writes per-image state files.
3. `cg build generate` produces the complete GitHub Actions pipeline: container build/scan/sign + Cloudflare staging deploy + test gate + Cloudflare production deploy.
4. The nightly `cg images check` re-checks published images against the latest vulnerability databases without rebuilding.

---

## Step 1 — Install CascadeGuard

If you haven't already installed CascadeGuard, the quickest path is the shell wrapper:

```bash
curl -sSL https://raw.githubusercontent.com/cascadeguard/cascadeguard/main/install.sh | sh
```

This installs `cg` (and the long-form `cascadeguard`) using a Python venv — no manual environment configuration needed.

Verify the install:

```bash
cg --version
```

See [Getting Started](../getting-started.md) for full installation options.

---

## Step 2 — Initialise your state repository

Create a new Git repository for CascadeGuard state and scaffold it from the seed repo:

```bash
mkdir cascadeguard-state && cd cascadeguard-state
git init
cg images init
```

Add `.cascadeguard.yaml` to set the target CI platform and the CascadeGuard version to use:

```yaml
# .cascadeguard.yaml
image: ghcr.io/cascadeguard/cascadeguard
version: v1.0.0

ci:
  platform: github
```

---

## Step 3 — Enrol your image

Use `cg images enrol` to add your image to `images.yaml`. The `--deploy` flags tell `build generate` what CD pipeline to produce.

```bash
cg images enrol \
  --name my-app \
  --repository your-org/my-app \
  --deploy cloudflare-workers \
  --deploy-project my-app
```

This writes an entry to `images.yaml`:

```yaml
# images.yaml
- name: my-app
  registry: ghcr.io
  repository: your-org/my-app
  source:
    provider: github
    repo: your-org/my-app
    dockerfile: Dockerfile
    branch: main
  rebuildDelay: 7d
  autoRebuild: true
  deploy:
    provider: cloudflare-workers
    project: my-app
    workflows:
      - name: staging
        environment: staging
      - name: production
        environment: production
        gate: staging
```

The `workflows` list declares the CD stages. `build generate` uses this to wire up a staging deploy → test gate → production deploy sequence in the generated GitHub Actions pipeline.

---

## Step 4 — Generate state files

```bash
cg images generate
```

This reads `images.yaml`, inspects the registry, and writes per-image state to `cascadeguard/state/`. On first run the state files show no prior build; they are populated after the first CI run.

---

## Step 5 — Generate the CI/CD pipeline

```bash
cg build generate
```

Because your `images.yaml` entry declares a `deploy` section with Cloudflare Workers, `build generate` emits a complete pipeline including the CD stages:

| File | Trigger | What it does |
|---|---|---|
| `build-image.yaml` | `workflow_call` | Build container → scan → generate SBOM → sign → push to GHCR |
| `ci.yaml` | Push to `main`, pull requests | Matrix build; on `main` also triggers CD workflows |
| `deploy-staging.yaml` | Called by `ci.yaml` on `main` | Pull container from GHCR and deploy to Cloudflare Workers (staging) |
| `run-tests.yaml` | Called after staging deploy | Run acceptance tests against the staging Worker URL |
| `deploy-production.yaml` | Called after tests pass | Deploy container to Cloudflare Workers (production) |
| `scheduled-scan.yaml` | Nightly cron | Re-scan published images; open GitHub Issues on new CVEs |
| `release.yaml` | Tag push (`v*`) | Build, sign, and push with release tag |

> **Do not edit these files by hand.** They carry an `# Auto-generated by CascadeGuard` header. Re-run `cg build generate` after any `images.yaml` change.

Commit and push everything:

```bash
git add .
git commit -m "chore: initialise CascadeGuard state and CI/CD"
git push
```

---

## Step 6 — Add repository secrets

Add these secrets to your GitHub repository (Settings → Secrets and variables → Actions):

| Secret | Value |
|---|---|
| `CLOUDFLARE_API_TOKEN` | A Cloudflare API token with Workers deploy permission |
| `CLOUDFLARE_ACCOUNT_ID` | Your Cloudflare account ID |

`GITHUB_TOKEN` is injected automatically and requires no manual setup.

The generated workflows request these permissions (already set in the generated files):

```yaml
permissions:
  contents: read
  packages: write      # push container to GHCR
  id-token: write      # Cosign keyless signing via OIDC
  issues: write        # open issues when new CVEs are found
```

---

## Step 7 — Verify the pipeline

After pushing, GitHub Actions will trigger. Once CI completes, confirm the image and deployment are healthy:

```bash
# Check current image status: digest, last build time, base image deps
cg images status

# Run a check against enrolled images
cg images check
```

`cg images status` prints a table of every managed image showing version, registry digest, build time, and dependency relationships. `cg images check` checks for digest drift and new upstream tags.

---

## Optional — Switch to CascadeGuard managed secure base images

CascadeGuard publishes regularly-updated, pre-scanned base images via [cascadeguard-open-secure-images](https://github.com/cascadeguard/cascadeguard-open-secure-images). Switching to a managed base image means CascadeGuard will automatically queue a rebuild of your image whenever the base is updated, with supply chain protections you control through the state repository.

Update your `Dockerfile`:

```dockerfile
# Before
FROM nginx:1.27-alpine

# After — CascadeGuard managed image
FROM ghcr.io/cascadeguard/nginx:1.27-alpine
```

Then re-run `cg images generate` to update state and `cg build generate` to regenerate pipelines. The `autoRebuild: true` flag in `images.yaml` handles triggering rebuilds automatically on any future base image update.

---

## Optional — Protect your GitHub Actions supply chain

Install the CascadeGuard action in other repositories to audit and enforce a policy on the GitHub Actions used in their workflows — protecting against supply chain issues in actions.

```bash
cg actions pin --repo your-org/your-repo
```

This adds a workflow that audits all referenced actions on every PR and blocks merges that introduce unpinned or policy-violating actions.

---

## Next steps

- [Getting Started](../getting-started.md) — installation and first setup
- [CLI Reference](../reference/cli.md) — full reference for `cg images enrol`, `images check`, `build generate`, and all other commands
- [GitHub Actions Integration Guide](../integrations/github-actions.md) — how the generated workflows are structured and how to customise them
- [Security Model](../security-model.md) — SBOM generation, Cosign signing, and scan gating
- [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) — a working state repository with real generated workflows
- [GitLab CI + Argo CD](gitlab-argocd.md) — the same walkthrough for GitLab + ArgoCD + Kargo

# CLI Reference

CascadeGuard is invoked as a Docker container or directly via the `cascadeguard` binary. All commands accept global flags for specifying file paths.

## Global flags

| Flag | Default | Description |
|---|---|---|
| `--images-yaml PATH` | `images.yaml` | Path to the image enrollment file |
| `--state-dir PATH` | `state` | Path to the state directory |

## Commands

### `validate`

Validates your `images.yaml` configuration without making any changes.

```bash
cascadeguard validate
# or via Docker:
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 validate
```

Exits with a non-zero status if the file is malformed or contains invalid values.

---

### `enrol`

Adds a new image entry to `images.yaml`.

```bash
cascadeguard --images-yaml images.yaml enrol \
  --name <name> \
  --registry <registry> \
  --repository <repository> \
  [--provider github|gitlab] \
  [--repo <source-repo>] \
  [--dockerfile <path>] \
  [--branch <branch>] \
  [--rebuild-delay <duration>]
```

| Flag | Required | Description |
|---|---|---|
| `--name` | Yes | Logical name for the image (used in state filenames) |
| `--registry` | Yes | Registry hostname, e.g. `ghcr.io` |
| `--repository` | Yes | Repository path, e.g. `your-org/my-image` |
| `--provider` | No | Source code provider: `github` or `gitlab` |
| `--repo` | No | Source repository, e.g. `your-org/my-app` |
| `--dockerfile` | No | Path to Dockerfile within the source repo |
| `--branch` | No | Branch to track (default: `main`) |
| `--rebuild-delay` | No | Minimum time between rebuilds, e.g. `7d` |

After enrolling, run `generate` and `generate-ci` to update state files and pipelines.

---

### `check`

Checks image and base image states against the registry. Useful for verifying the current CVE posture before a release.

```bash
cascadeguard check
```

---

### `status`

Prints a summary table of every managed image and base image, including version, digest, build time, and dependency information.

```bash
cascadeguard status
# or via Docker:
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 status
```

---

### `build`

Triggers a build for a specific image via the GitHub Actions API.

```bash
cascadeguard build \
  --image <name> \
  [--tag <tag>] \
  [--repo <github-repo>] \
  [--github-token <token>]
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — (required) | Image name as defined in `images.yaml` |
| `--tag` | `latest` | Image tag to build |
| `--repo` | — | GitHub repository, e.g. `org/repo` |
| `--github-token` | `$GITHUB_TOKEN` | GitHub API token with `workflow` permission |

---

### `deploy`

Triggers an ArgoCD sync for a specific application.

```bash
cascadeguard deploy \
  --image <name> \
  --app <argocd-app> \
  [--argocd-server <url>] \
  [--argocd-token <token>]
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — (required) | Image name as defined in `images.yaml` |
| `--app` | — (required) | ArgoCD application name |
| `--argocd-server` | — | ArgoCD server URL |
| `--argocd-token` | `$ARGOCD_TOKEN` | ArgoCD API token |

---

### `test`

Queries GitHub Actions for the latest test run results for a specific image.

```bash
cascadeguard test \
  --image <name> \
  [--repo <github-repo>] \
  [--github-token <token>]
```

---

### `pipeline`

Runs the full pipeline in sequence: `validate → check → build → deploy → test`.

```bash
cascadeguard pipeline \
  [--image <name>] \
  [--tag <tag>] \
  [--repo <github-repo>] \
  [--github-token <token>] \
  [--app <argocd-app>] \
  [--argocd-server <url>] \
  [--argocd-token <token>]
```

All flags are optional. If `--image` is omitted, the pipeline runs for all enrolled images.

---

### `actions pin`

Rewrites GitHub Actions workflow files to pin `uses:` references from floating tags to full commit SHAs. This hardens your CI supply chain against tag-hijacking attacks.

```bash
cascadeguard actions pin \
  [--workflows-dir <path>] \
  [--dry-run] \
  [--refresh] \
  [--github-token <token>]
```

| Flag | Default | Description |
|---|---|---|
| `--workflows-dir` | `.github/workflows` | Path to the workflows directory |
| `--dry-run` | false | Preview changes without writing files |
| `--refresh` | false | Re-pin already-pinned SHAs to the latest SHA for the same tag |
| `--github-token` | `$GITHUB_TOKEN` | GitHub token for resolving SHA lookups |

---

### `actions audit`

Audits workflow files against a declarative actions policy.

```bash
cascadeguard actions audit \
  [--policy <path>] \
  [--workflows-dir <path>]
```

| Flag | Default | Description |
|---|---|---|
| `--policy` | `.cascadeguard/actions-policy.yaml` | Path to the policy file |
| `--workflows-dir` | `.github/workflows` | Path to the workflows directory |

Exits with a non-zero status if any violations are found.

---

### `actions policy init`

Scaffolds a starter `.cascadeguard/actions-policy.yaml` file.

```bash
cascadeguard actions policy init \
  [--output <path>] \
  [--force]
```

| Flag | Default | Description |
|---|---|---|
| `--output` | `.cascadeguard/actions-policy.yaml` | Output path |
| `--force` | false | Overwrite an existing policy file |

---

## Taskfile tasks

State repositories typically use CascadeGuard via the shared Taskfile rather than calling `cascadeguard` directly:

| Task | Equivalent command | Description |
|---|---|---|
| `task generate` | `cascadeguard generate` | Generate state files from `images.yaml` |
| `task synth` | `cascadeguard synth` | Generate Kargo manifests |
| `task generate-and-synth` | Both above in sequence | Full regeneration |
| `task generate-ci` | `cascadeguard generate-ci` | Generate GitHub Actions workflow files |
| `task status` | `cascadeguard status` | Show image status table |

See [Getting Started](../getting-started.md) for the full Taskfile setup.

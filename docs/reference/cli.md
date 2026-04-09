# CLI Reference

CascadeGuard uses a noun-verb command structure. Commands are grouped by the resource they operate on.

## Command groups

| Group | Description |
|---|---|
| `images` | Image lifecycle management |
| `pipeline` | CI/CD orchestration |
| `vuln` | Vulnerability management |
| `actions` | GitHub Actions utilities |

---

## `images` — Image lifecycle management

Flags scoped to `images` subcommands:

| Flag | Default | Description |
|---|---|---|
| `--images-yaml PATH` | `images.yaml` | Path to the image enrollment file |
| `--state-dir PATH` | `state` | Path to the state directory |

### Command design

CascadeGuard commands follow a `<noun> <verb>` naming convention. The noun is the resource (`images`, `pipeline`, `vuln`, `actions`) and the verb is the operation. This keeps related commands grouped and tab-completion predictable.

The `images` subcommands cover the full image lifecycle:

| Command | Network? | Reads | Writes | Purpose |
|---|---|---|---|---|
| `images validate` | No | `images.yaml` | — | Syntax and schema check before making changes |
| `images enrol` | No | — | `images.yaml` | Register a new image for tracking |
| `images check` | **Yes** | state files, live registry | — | Detect digest drift on already-enrolled tags |
| `images check-upstream` | **Yes** | `images.yaml`, Docker Hub | — | Discover new stable upstream tags not yet enrolled |
| `images status` | No | state files | — | Display current enrollment state and metadata |

**Why `check` and `check-upstream` are separate commands:**

- `images check` is an *offline verification* step: it reads the digests already recorded in local state files and confirms they still match what the registry serves. If a tag's digest changed without a new version being enrolled, `check` surfaces that as drift.
- `images check-upstream` is a *network discovery* step: it reaches out to Docker Hub to find entirely new tags (e.g. `alpine:3.22`) that predate any local state. It tells you "there is a newer version you haven't enrolled yet."

Keeping them separate means `check` can run in air-gapped or rate-limited environments as a fast integrity assertion, while `check-upstream` is reserved for the scheduled discovery job that may paginate many tag pages from Docker Hub.

---

### `images validate`

Validates your `images.yaml` configuration without making any changes.

```bash
cascadeguard images validate
cascadeguard images --images-yaml custom.yaml validate
```

Exits with a non-zero status if the file is malformed or contains invalid values.

---

### `images enrol`

Adds a new image entry to `images.yaml`.

```bash
cascadeguard images enrol \
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

### `images check`

Compares locally recorded image digests against the live registry to detect **digest drift** — when a tag's digest changes upstream without a corresponding version bump.

```bash
cascadeguard images check
cascadeguard images --state-dir /path/to/state check
cascadeguard images check --image <name>
cascadeguard images check --format json
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — | Scope check to a single image (matches state file stem); omit to check all |
| `--format` | `table` | Output format: `table` or `json` |

**Exit codes:** `0` all images OK · `1` drift detected, registry unreachable, or image not found

**Status values in output:**

| Status | Meaning |
|---|---|
| `ok` | Recorded digest matches live digest |
| `drift` | Recorded digest differs from live digest — rebuild recommended |
| `error` | Registry could not be reached |
| `skipped` | Image missing version or tag information |
| `unknown` | No local digest recorded yet (run `generate` first) |

> **When to use:** Run after `generate` to confirm enrolled digests still match what is live. Use `check-upstream` to find *new* tags that have not yet been enrolled.

---

### `images check-upstream`

Queries Docker Hub for new stable semver tags across your enrolled base images that are not yet enrolled in `images.yaml`. Useful for staying ahead of upstream releases without manual monitoring.

```bash
cascadeguard images check-upstream
cascadeguard images check-upstream --image <name>
cascadeguard images check-upstream --format json
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — | Scope check to a single image name (as listed in `images.yaml`); omit to check all |
| `--format` | `table` | Output format: `table` or `json` |

**Exit codes:** `0` no new tags found · `1` new tags available (action recommended)

Only stable tags are surfaced — `latest`, `edge`, `nightly`, and pre-release suffixes (`-rc`, `-alpha`, `-beta`, etc.) are filtered out. Results are scoped to the same major version as currently enrolled tags.

> **When to use:** Invoked automatically by the `cascadeguard-actions/check-upstream` action on a daily schedule. Run manually to check for upstream drift before a planned enrolment cycle.

---

### `images status`

Prints a summary table of every managed image and base image, including version, digest, build time, and dependency information.

```bash
cascadeguard images status
# or via Docker:
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v1.0.0 status
```

---

## `pipeline` — CI/CD orchestration

### `pipeline build`

Triggers a build for a specific image via the GitHub Actions API.

```bash
cascadeguard pipeline build \
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

### `pipeline deploy`

Triggers an ArgoCD sync for a specific application.

```bash
cascadeguard pipeline deploy \
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

### `pipeline test`

Queries GitHub Actions for the latest test run results for a specific image.

```bash
cascadeguard pipeline test \
  --image <name> \
  [--repo <github-repo>] \
  [--github-token <token>]
```

---

### `pipeline run`

Runs the full pipeline in sequence: `validate → check → build → deploy → test`.

```bash
cascadeguard pipeline run \
  [--images-yaml <path>] \
  [--state-dir <path>] \
  [--image <name>] \
  [--tag <tag>] \
  [--repo <github-repo>] \
  [--github-token <token>] \
  [--app <argocd-app>] \
  [--argocd-server <url>] \
  [--argocd-token <token>]
```

All flags are optional. If `--image` is omitted, the pipeline runs validate and check for all enrolled images.

---

## `vuln` — Vulnerability management

### `vuln report`

Parses scanner output (Grype and/or Trivy) and writes diffable vulnerability reports.

```bash
cascadeguard vuln report \
  --image <name> \
  --dir <output-dir> \
  [--grype <grype-json>] \
  [--trivy <trivy-json>]
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — (required) | Image name (used in report metadata) |
| `--dir` | — (required) | Output directory, e.g. `images/alpine/reports` |
| `--grype` | — | Path to Grype JSON results file |
| `--trivy` | — | Path to Trivy JSON results file |

---

### `vuln issues`

Creates, updates, or reopens per-CVE GitHub issues from scan results.

```bash
cascadeguard vuln issues \
  --image <name> \
  --repo <github-repo> \
  [--tag <tag>] \
  [--grype <grype-json>] \
  [--trivy <trivy-json>] \
  [--github-token <token>]
```

| Flag | Default | Description |
|---|---|---|
| `--image` | — (required) | Image name |
| `--repo` | — (required) | GitHub repository, e.g. `org/repo` |
| `--tag` | — | Image tag |
| `--grype` | — | Path to Grype JSON results file |
| `--trivy` | — | Path to Trivy JSON results file |
| `--github-token` | `$GITHUB_TOKEN` | GitHub API token |

---

## `actions` — GitHub Actions utilities

### `actions pin`

Rewrites GitHub Actions workflow files to pin `uses:` references from floating tags to full commit SHAs. This hardens your CI supply chain against tag-hijacking attacks.

```bash
cascadeguard actions pin \
  [--workflows-dir <path>] \
  [--dry-run] \
  [--update] \
  [--github-token <token>]
```

| Flag | Default | Description |
|---|---|---|
| `--workflows-dir` | `.github/workflows` | Path to the workflows directory |
| `--dry-run` | false | Preview changes without writing files |
| `--update` | false | Re-pin already-pinned SHAs to the latest SHA for the same tag |
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

| Task | Description |
|---|---|
| `task generate` | Generate state files from `images.yaml` |
| `task synth` | Generate Kargo manifests |
| `task generate-and-synth` | Full regeneration (generate then synth) |
| `task generate-ci` | Generate GitHub Actions workflow files |
| `task status` | Show image status table |

See [Getting Started](../getting-started.md) for the full Taskfile setup.

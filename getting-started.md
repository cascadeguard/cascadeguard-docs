# Getting Started

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)

That's it. No other tools to install.

## Quick Start

One command to scaffold a new state repo:

```bash
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 init
```

This creates:
- `cascadeguard.sh` — a shell wrapper (all commands delegate to Docker)
- `images.yaml` — a template with commented examples

Edit `images.yaml` to enrol your images, then:

```bash
./cascadeguard.sh enrol
./cascadeguard.sh status
```

### Alternative wrappers

If you prefer Task, Make, or want all wrappers:

```bash
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 init --wrapper=taskfile
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 init --wrapper=makefile
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 init --wrapper=all
```

Or call Docker directly without any wrapper:

```bash
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 enrol
```

### Download exemplar files

To start with a working example rather than a blank template:

```bash
docker run --rm -v $(pwd):/workspace ghcr.io/cascadeguard/cascadeguard:v0.1.0 init:exemplar
```

This pulls `images.yaml` and `.cascadeguard.yaml` from the
[cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) repo.

---

## Two usage modes

### Task mode (CI-orchestrated)

You orchestrate the pipeline from your CI system. CascadeGuard handles provider
calls, polling, and supply chain policy.

**Scheduled** (e.g. daily cron):
```bash
./cascadeguard.sh check            # reports upstream changes (table)
./cascadeguard.sh check:json       # same, JSON for CI consumption
```

**Triggered** (from CI, after check or manually):
```bash
./cascadeguard.sh pipeline              # all images with pending changes
./cascadeguard.sh pipeline my-app       # specific image
```

Or use individual steps:

```bash
./cascadeguard.sh build my-app
./cascadeguard.sh deploy my-app staging
./cascadeguard.sh test my-app staging
./cascadeguard.sh deploy my-app production
```

All commands have a `:json` variant.

### Kargo mode (automatic promotion)

CascadeGuard generates Kargo manifests. Kargo handles monitoring and promotion.

```bash
./cascadeguard.sh enrol
./cascadeguard.sh setup:kargo
```

---

## Configuration (optional)

Everything has sensible defaults. Only create `.cascadeguard.yaml` to override.

See [cascadeguard/.cascadeguard.yaml](https://github.com/cascadeguard/cascadeguard/blob/main/.cascadeguard.yaml)
for the full reference.

---

## Commands

| Command | Description |
|---|---|
| `init` | Scaffold state repo (shell wrapper + images.yaml) |
| `init --wrapper=taskfile` | Also create Taskfile.yaml |
| `init --wrapper=makefile` | Also create Makefile |
| `init --wrapper=all` | Create all wrappers |
| `init:exemplar` | Download exemplar files from GitHub |
| `validate` | Lint images.yaml, dry-run |
| `enrol` | Generate/update state files |
| `check` | Poll upstreams, evaluate policy |
| `build <image>` | Trigger CI build+test |
| `deploy <image> <env>` | Deploy to environment |
| `test <image> <env>` | Post-deploy tests |
| `pipeline [image]` | Full automated pipeline |
| `setup:kargo` | Generate Kargo manifests |
| `setup:github` | Generate GitHub Actions workflows |
| `setup:gitlab` | Generate GitLab CI config |
| `setup:jenkins` | Generate Jenkinsfile |
| `status` | Show generated files |
| `clean` | Remove generated files |

---

## Override the image at runtime

```bash
CASCADEGUARD_IMAGE=ghcr.io/cascadeguard/cascadeguard:dev ./cascadeguard.sh check
```

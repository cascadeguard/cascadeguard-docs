# Architecture

## Overview

CascadeGuard is a Docker image that manages the container image lifecycle. It
reads `images.yaml` (your enrollment config) and provides two usage modes:

**Task mode** — you orchestrate the pipeline explicitly from your CI system.
CascadeGuard handles provider calls, polling, and supply chain policy.

**Kargo mode** — CascadeGuard generates Kargo manifests. Kargo monitors
registries and handles promotion automatically.

Both modes share the same `images.yaml` format and state files. The difference
is in how the pipeline is driven.

---

## The image lifecycle

```
Discover → Build+Test → Deploy(env1) → Test(env1) → Deploy(env2) → Test(env2) → ...
```

### Discover
`images.yaml` is the source of truth. On enrolment, CascadeGuard:
- Parses each image's Dockerfile to discover base image dependencies
- Generates state files in `base-images/` and `images/`

A scheduled `task check` (or Kargo warehouse polling) monitors upstream
registries for digest changes and evaluates supply chain policy before acting.

### Build+Test
The CI pipeline owns this step entirely — CascadeGuard just triggers it.
Supported: GitHub Actions (`workflow_dispatch`). GitLab CI coming.

### Deploy → Test → Deploy → Test
CascadeGuard commits the new image tag to your gitops repo (ArgoCD detects and
syncs), then polls for sync completion before running tests and proceeding to
the next environment.

---

## Task mode

```
images.yaml
    │
    ▼
task enrol              generates/updates base-images/ and images/
    │
    ▼
task check              polls upstreams, evaluates policy, reports status
    │                   (scheduled daily/weekly)
    ▼
task pipeline           for each image with pending changes:
    ├── task build      trigger CI (build+test), poll for completion
    ├── task deploy staging   commit to gitops repo, poll ArgoCD sync
    ├── task test staging     trigger test workflow, poll for completion
    ├── task deploy production
    └── task test production
```

`task pipeline` runs all images concurrently, advancing each through its own
pipeline independently as steps complete.

Supply chain policy is checked before `task build` fires:
- `min_upstream_age`: don't act on upstream images newer than N days
- `rebuild_delay`: minimum time between rebuilds of the same image

All tasks have a `:json` variant for CI scripting.

---

## Kargo mode

```
images.yaml
    │
    ▼
task enrol              generates/updates base-images/ and images/
    │
    ▼
task kargo              CDK8s synthesis → dist/cdk8s/cascadeguard.k8s.yaml
    │
    ▼
ArgoCD syncs            deploys Kargo Warehouses, Stages, AnalysisTemplates
    │
    ▼
Kargo monitors          polls registries via Warehouses
    │
    ▼  (upstream digest changes)
Kargo Stage             runs AnalysisTemplate job → parses Dockerfile
    │                   → discovers base images → updates state files
    ▼
Kargo promotes          triggers rebuild, manages env-to-env promotion
```

In Kargo mode, `task check` and `task pipeline` are not used — Kargo handles
monitoring and promotion automatically.

---

## State repository structure

```
your-state-repo/
├── images.yaml          ← the only file you edit
├── .cascadeguard.yaml   ← config: image version, policy, environments
├── Taskfile.yaml        ← includes shared tasks from cascadeguard
├── base-images/         ← auto-generated, commit these
│   └── nginx-alpine.yaml
├── images/              ← auto-generated, commit these
│   └── my-app.yaml
└── dist/                ← auto-generated, commit these
    └── cdk8s/
        └── cascadeguard.k8s.yaml  (Kargo mode only)
```

---

## Provider interface

CascadeGuard uses a generic provider interface for each external system.
Each provider implements two operations: **trigger** and **status**.

| Operation | GitHub Actions | ArgoCD |
|---|---|---|
| trigger | `workflow_dispatch` API | commit to gitops repo at `path_pattern` |
| status | poll `/actions/runs` | poll ArgoCD API for app sync status via `app_pattern` |

Adding a new provider (GitLab, Flux, etc.) requires implementing these two
operations for that provider. The task layer is provider-agnostic.

---

## Supply chain policy

Evaluated by `task check` and `task pipeline` before triggering any action:

```yaml
policy:
  min_upstream_age: 2d   # upstream image must be at least 2 days old
  rebuild_delay: 7d      # at least 7 days between rebuilds of the same image
```

Per-image overrides are supported in `images.yaml` under `policy:`.

---

## Components

### `app/` — Analysis tool
Reads `images.yaml`, parses Dockerfiles, generates/updates state files.
Entry point for `validate`, `enrol`, `check`, `build`, `deploy`, `test`,
`pipeline`, and `status` commands.

### `cdk8s/` — Kargo manifest generator
Reads state files and generates Kargo `Warehouse`, `Stage`, and
`AnalysisTemplate` resources. Entry point for `task kargo`.

### `Taskfile.shared.yaml`
Included by state repos via a pinned URL. Thin `docker run` wrappers — each
task delegates to the Docker image. No local tooling needed beyond Docker and Task.

### `Taskfile.docker.yaml`
Baked into the Docker image. Delegates to the Python CLI for all task-mode
commands, and to CDK8s for `task kargo`.

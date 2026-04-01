# Getting Started

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)
- A Kubernetes cluster with [Kargo](https://kargo.io) and [ArgoCD](https://argo-cd.readthedocs.io) installed (for the full workflow)

## Option 1: Try the Exemplar

The fastest way to see CascadeGuard in action:

```bash
git clone https://github.com/cascadeguard/cascadeguard-exemplar.git
cd cascadeguard-exemplar
task generate-and-synth
task status
```

This generates state files and Kargo manifests for a simple hello-world nginx image. No cluster needed — inspect the output in `base-images/`, `images/`, and `dist/cdk8s/`.

## Option 2: Start Your Own State Repository

### 1. Create a new repo and add a Taskfile

```yaml
# Taskfile.yaml
version: '3'

includes:
  shared:
    taskfile: https://raw.githubusercontent.com/cascadeguard/cascadeguard/v1.0.0/Taskfile.shared.yaml
    flatten: true
```

### 2. Create `images.yaml`

Enroll your first image:

```yaml
- name: my-app
  registry: ghcr.io
  repository: myorg/my-app
  source:
    provider: github
    repo: myorg/my-app
    dockerfile: Dockerfile
    workflow: build.yml
  rebuildDelay: 7d
  autoRebuild: true
```

### 3. Generate and synth

```bash
task generate-and-synth
```

This produces:
- `images/my-app.yaml` — state file for your image
- `dist/cdk8s/image-factory.k8s.yaml` — Kargo manifests ready for ArgoCD

### 4. Commit and push

Commit all generated files — ArgoCD deploys from `dist/cdk8s/`.

## Enrolling Image Types

### Managed image (built by your CI)

```yaml
- name: my-app
  registry: ghcr.io
  repository: myorg/my-app
  source:
    provider: github
    repo: myorg/my-app
    dockerfile: Dockerfile
    workflow: build.yml
  rebuildDelay: 7d
  autoRebuild: true
```

CascadeGuard discovers base images from the Dockerfile and monitors them. When a base image updates, it triggers your `build.yml` workflow.

### External image (third-party, monitored directly)

```yaml
- name: postgres
  registry: docker.io
  repository: library/postgres
  allowTags: ^16-alpine$
  imageSelectionStrategy: Lexical
  rebuildDelay: 30d
  autoRebuild: false
```

No `source` field — CascadeGuard generates a Kargo Warehouse to monitor this image directly.

## Available Tasks

| Task | Description |
|---|---|
| `task generate` | Generate state files from `images.yaml` |
| `task synth` | Generate Kargo manifests from state files |
| `task generate-and-synth` | Run both in sequence |
| `task status` | Show generated files |
| `task clean` | Remove all generated files |
| `task deploy` | Apply manifests to cluster (`kubectl apply`) |

## Configuration

Create a `.cascadeguard.yaml` in your state repo root to control which Docker image is used:

```yaml
# .cascadeguard.yaml
image: ghcr.io/cascadeguard/cascadeguard
version: v1.0.0
```

`Taskfile.shared.yaml` reads this file automatically. To upgrade CascadeGuard, update `version` here and update the `taskfile:` URL in `Taskfile.yaml` to match.

Override at runtime without editing the file (useful in CI):

```bash
CASCADEGUARD_IMAGE=ghcr.io/cascadeguard/cascadeguard:v1.1.0 task generate
```

### Available config keys

| Key | Description | Default |
|---|---|---|
| `image` | Docker image registry and name | `ghcr.io/cascadeguard/cascadeguard` |
| `version` | Docker image tag | `v1.0.0` |

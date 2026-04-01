# Architecture

## Overview

CascadeGuard is a Docker image with two responsibilities:

1. **`generate`** — reads `images.yaml`, parses Dockerfiles, writes state files to `base-images/` and `images/`
2. **`synth`** — reads state files, generates Kargo manifests into `dist/cdk8s/`

That's it. CascadeGuard has no opinions about how or where you run it, or what you do with the output.

```
images.yaml  →  [cascadeguard generate]  →  base-images/*.yaml
                                         →  images/*.yaml

state files  →  [cascadeguard synth]     →  dist/cdk8s/image-factory.k8s.yaml
```

You can run these steps from a local terminal, a GitHub Action, a GitLab CI stage, a Kargo analysis job, a cron job, or anything else that can run a Docker container.

## Repository Structure

```
Your state repo (e.g. cascadeguard-exemplar)
├── images.yaml          ← the only file you edit
├── base-images/         ← auto-generated, commit these
├── images/              ← auto-generated, commit these
├── dist/cdk8s/          ← auto-generated Kargo manifests, commit these
└── Taskfile.yaml        ← thin wrapper, includes shared tasks from cascadeguard
```

## Image Types

| Type | Description | Monitoring manifest generated? |
|---|---|---|
| **Managed** | Built by your CI. CascadeGuard tracks their base images. | No |
| **External** | Third-party images monitored directly. | Yes |
| **Base** | Discovered from Dockerfile `FROM` statements. | Yes |

Managed images (those with a `source:` block in `images.yaml`) are built by your existing CI pipeline — CascadeGuard does not build them. It only generates monitoring configuration for the base images they depend on.

## Example Flow

```
1. You edit images.yaml to enroll a new image

2. Run: docker run --rm -v $(pwd):/workspace cascadeguard:v1.0.0 generate-and-synth
   → state files written to base-images/ and images/
   → Kargo manifests written to dist/cdk8s/

3. Commit and push
   → your CI/CD picks up the change (GitHub Actions, GitLab CI, etc.)

4. If using Kargo + ArgoCD:
   → ArgoCD syncs dist/cdk8s/ to the cluster
   → Kargo Warehouses monitor registries
   → When a base image updates, Kargo triggers a rebuild of dependent images

5. If not using Kargo:
   → Apply the manifests yourself, or use a different GitOps tool
   → Or just use the state files directly in your own pipeline
```

## Components

### Analysis Tool (`app/`)

Python tool that:
- Reads `images.yaml`
- Parses Dockerfiles to discover `FROM` dependencies
- Generates/updates state files in `base-images/` and `images/`

### CDK8s Generator (`cdk8s/`)

Python CDK8s application that:
- Reads `images.yaml` and state files
- Generates Kargo `Warehouse`, `Stage`, and `AnalysisTemplate` resources
- Outputs to `dist/cdk8s/image-factory.k8s.yaml`

The Kargo manifests are optional output — you only need them if you're running Kargo. The state files in `base-images/` and `images/` are useful independently of any orchestration platform.

### Taskfile.shared.yaml

Shared task definitions included by state repositories via a pinned URL:

```yaml
includes:
  shared:
    taskfile: https://raw.githubusercontent.com/cascadeguard/cascadeguard/v1.0.0/Taskfile.shared.yaml
    flatten: true
```

Each task is a `docker run` wrapper — no local tooling needed beyond Docker and Task.

## Data Contract

State files contain these fields for the CDK8s generator to create a Warehouse:

- `repoURL` — full registry/repository path
- `allowTags` — regex for tag matching
- `imageSelectionStrategy` — `Lexical`, `SemVer`, or `NewestBuild`

Managed images intentionally have no `repoURL` and are skipped by the generator — they are built, not monitored.

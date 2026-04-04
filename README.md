# CascadeGuard Documentation

Guardian of the container cascade. Event-driven image lifecycle management with Kargo and ArgoCD.

## Contents

- [Architecture](architecture.md) — How CascadeGuard works
- [Getting Started](getting-started.md) — Set up your own state repository
- [Contributing](contributing.md) — How to contribute to CascadeGuard

## What is CascadeGuard?

CascadeGuard automates the container image rebuild lifecycle. When an upstream base image updates, CascadeGuard detects the change and orchestrates rebuilds of all dependent images — the update cascades down the dependency chain automatically.

**Core capabilities:**

- Monitors upstream base images for digest changes
- Discovers Dockerfile dependencies automatically via Kargo analysis jobs
- Generates Kargo Warehouses and Stages from a simple `images.yaml` enrollment file
- Supports managed images (built by your CI), external images (third-party), and base images
- Configurable rebuild delays and auto-rebuild policies per image

## Repositories

| Repository | Purpose |
|---|---|
| [cascadeguard/cascadeguard](https://github.com/cascadeguard/cascadeguard) | Factory code, Docker image, shared Taskfile |
| [cascadeguard/cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) | Example state repository with hello-world nginx image |
| [cascadeguard/docs](https://github.com/cascadeguard/docs) | This documentation |

## Quick Start

See [Getting Started](getting-started.md) or clone the exemplar:

```bash
git clone https://github.com/cascadeguard/cascadeguard-exemplar.git
cd cascadeguard-exemplar
task generate-and-synth
```

## Licensing

CascadeGuard is licensed under the [Business Source License 1.1](LICENSE).

**Key terms:**
- You may use, modify, and redistribute this software freely for non-production and internal production use.
- You may **not** offer this software to third parties as a commercial container image lifecycle management service or a managed image rebuild service.
- On **2030-04-04**, this version automatically converts to the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

For commercial licensing inquiries, contact licensing@cascadeguard.io.

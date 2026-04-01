# Getting Started

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/installation/)

No local Python setup needed — CascadeGuard runs entirely via Docker.

## Option 1: Try the Exemplar

```bash
git clone https://github.com/cascadeguard/cascadeguard-exemplar.git
cd cascadeguard-exemplar
task enrol
task status
```

## Option 2: Start Your Own State Repository

All you need is a `Taskfile.yaml`. Run `task init` to scaffold it:

```bash
mkdir my-state && cd my-state
task init
```

This creates a `Taskfile.yaml` (no Docker image needed). Then download a
working example:

```bash
task init:exemplar
```

This pulls `images.yaml` and `.cascadeguard.yaml` from the exemplar repo.
You can also just create `images.yaml` manually — no `.cascadeguard.yaml`
required, defaults are sensible.

Edit `images.yaml` to enrol your images, then:

```bash
task enrol
task status
```

## Two usage modes

### Task mode (CI-orchestrated)

You orchestrate the pipeline from your CI system. CascadeGuard handles provider
calls, polling, and supply chain policy.

```
MR/PR:   task validate
Schedule: task check
Trigger: task pipeline -- my-app
```

Or use individual steps:

```bash
task build -- my-app
task deploy -- my-app staging
task test -- my-app staging
```

All tasks have a `:json` variant for CI scripting.

### Kargo mode (automatic promotion)

Generate Kargo manifests and let Kargo handle everything:

```bash
task enrol
task kargo
```

---

## Configuration (optional)

Only create `.cascadeguard.yaml` if you want to override defaults.

See [cascadeguard/.cascadeguard.yaml](https://github.com/cascadeguard/cascadeguard/blob/main/.cascadeguard.yaml)
for the full reference with every option documented.

---

## Tasks reference

| Task | Description |
|---|---|
| `task init` | Scaffold `Taskfile.yaml` |
| `task init:exemplar` | Download exemplar `images.yaml` + `.cascadeguard.yaml` |
| `task validate` | Lint `images.yaml`, dry-run |
| `task enrol` | Generate/update state files |
| `task check` | Poll upstreams, evaluate policy |
| `task pipeline [-- <image>]` | Full automated pipeline |
| `task build -- <image>` | Trigger CI build+test |
| `task deploy -- <image> <env>` | Deploy to environment |
| `task test -- <image> <env>` | Post-deploy tests |
| `task kargo` | Generate Kargo manifests (Kargo mode) |
| `task status` | Show generated files |
| `task clean` | Remove generated files |

All tasks have a `:json` variant.

## Override the image at runtime

```bash
CASCADEGUARD_IMAGE=ghcr.io/cascadeguard/cascadeguard:dev task check
```

# images.yaml Reference

`images.yaml` is the single source of truth for all images managed by CascadeGuard. It lives in the root of your state repository.

## Structure

The file is a YAML list. Each entry describes one image:

```yaml
images:
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
```

> **Note:** CascadeGuard also accepts the file as a bare list (without a top-level `images:` key). Both formats are supported.

## Fields

### `name` _(required)_

A short, unique identifier for the image. Used in:
- State filenames (`cascadeguard/state/images/<name>.yaml`)
- CLI commands (`--image <name>`)
- GitHub Actions job names

Use lowercase letters, numbers, and hyphens. No spaces.

---

### `registry` _(required)_

The registry hostname where the image is published.

```yaml
registry: ghcr.io
```

Common values: `ghcr.io`, `docker.io`, `registry.hub.docker.com`, your private registry hostname.

---

### `repository` _(required)_

The full repository path within the registry (everything after the hostname).

```yaml
repository: your-org/my-app
```

---

### `source` _(optional)_

Describes where CascadeGuard can find the Dockerfile to analyse dependencies.

| Field | Default | Description |
|---|---|---|
| `provider` | `github` | Source code provider. Accepted values: `github`, `gitlab`. |
| `repo` | — | The source repository in `org/repo` format. |
| `dockerfile` | `Dockerfile` | Path to the Dockerfile within the source repository. |
| `branch` | `main` | Branch to read the Dockerfile from. |

If `source` is omitted, CascadeGuard will not perform Dockerfile dependency discovery for this image. It will still track the published image for new versions.

**GitHub example:**
```yaml
source:
  provider: github
  repo: your-org/my-app
  dockerfile: services/api/Dockerfile
  branch: main
```

**GitLab example:**
```yaml
source:
  provider: gitlab
  repo: your-group/my-app
  dockerfile: Dockerfile
  branch: main
```

Set the `GITHUB_TOKEN` (or `GITLAB_TOKEN`) environment variable to allow CascadeGuard to access private repositories.

---

### `rebuildDelay` _(optional)_

Minimum time that must elapse between successive rebuilds of this image. Prevents excessive rebuilds when a base image is updated frequently.

```yaml
rebuildDelay: 7d
```

Accepted units: `d` (days), `h` (hours). Default: no delay (rebuild immediately when a base image changes).

---

### `autoRebuild` _(optional)_

Whether CascadeGuard should automatically trigger a rebuild when a base image dependency is updated.

```yaml
autoRebuild: true
```

Default: `true`. Set to `false` to receive notifications without automatic rebuilds.

---

## Image types

CascadeGuard distinguishes between three types of image:

| Type | How to configure | Description |
|---|---|---|
| **Managed** | `enabled: true` (default) | CascadeGuard builds, scans, and publishes a hardened copy. CI + state resources are generated. |
| **Upstream tracked** | `enabled: false` | CVE posture is monitored but no hardened build is published. No CI or state generated. |
| **External** | Entry without a `source` block | Third-party images tracked directly for new versions. |

---

## Complete example

```yaml
images:
  # Managed image — CascadeGuard builds and publishes this
  - name: api
    registry: ghcr.io
    repository: your-org/api
    source:
      provider: github
      repo: your-org/api
      dockerfile: Dockerfile
      branch: main
    rebuildDelay: 3d
    autoRebuild: true

  # Upstream-tracked only — monitor for CVEs, don't rebuild
  - name: upstream-redis
    registry: docker.io
    repository: library/redis
    enabled: false

  # Multi-stage Dockerfile in a monorepo
  - name: worker
    registry: ghcr.io
    repository: your-org/worker
    source:
      provider: github
      repo: your-org/monorepo
      dockerfile: services/worker/Dockerfile
      branch: main
    rebuildDelay: 7d
```

## See also

- [Getting Started](../getting-started.md) — How to create your first `images.yaml`
- [CLI Reference](cli.md) — The `enrol` command for adding images
- [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) — A complete working example state repository

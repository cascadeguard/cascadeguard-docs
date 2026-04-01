# Contributing

Contributions are welcome. CascadeGuard is open source under the MIT licence.

## Repository Layout

```
cascadeguard/cascadeguard
├── app/            # Python analysis tool (Dockerfile parsing, state generation)
├── cdk8s/          # CDK8s application (Kargo manifest generation)
├── tests/          # Integration and acceptance tests
├── Dockerfile      # Builds the CascadeGuard Docker image
├── Taskfile.docker.yaml   # Internal tasks (runs inside Docker)
├── Taskfile.shared.yaml   # Shared tasks for state repos (docker run wrappers)
└── Taskfile.yaml          # Developer tasks for this repo
```

## Development Setup

```bash
git clone https://github.com/cascadeguard/cascadeguard.git
cd cascadeguard
task app:setup      # Set up app Python environment
task cdk8s:setup    # Set up CDK8s Python environment
```

## Running Tests

```bash
task test:unit        # Unit tests for app/ and cdk8s/
task test:integration # Integration tests
```

## Building the Docker Image Locally

```bash
docker build -t cascadeguard:dev .
```

Test it against the exemplar:

```bash
git clone https://github.com/cascadeguard/cascadeguard-exemplar.git
cd cascadeguard-exemplar
CASCADEGUARD_IMAGE=cascadeguard:dev task generate-and-synth
```

## Making Changes

1. Fork `cascadeguard/cascadeguard`
2. Create a feature branch
3. Make your changes with tests
4. Run `task test:unit` and `task test:integration`
5. Open a pull request

The CDK8s synth workflow will run automatically on your PR and commit updated manifests to the state repository if needed.

## Release Process

Releases are triggered by pushing a version tag:

```bash
git tag -a v1.1.0 -m "v1.1.0"
git push origin v1.1.0
```

This triggers the Docker build workflow, which publishes `ghcr.io/cascadeguard/cascadeguard:v1.1.0`.

State repositories pin to a specific version in their `Taskfile.yaml` include URL and `.cascadeguard.yaml` — upgrading is a deliberate, explicit action.

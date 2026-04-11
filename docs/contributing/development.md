# Development

Local development setup for contributing to the CascadeGuard CLI and CDK8s modules.

## Setup

```bash
task app:setup      # Set up the app Python environment
task cdk8s:setup    # Set up the CDK8s Python environment
```

## Testing

```bash
task test:unit           # Unit tests for app and cdk8s
task test:integration    # Integration tests
task test:acceptance     # Kargo acceptance tests (requires cluster)
```

See [CONTRIBUTING.md](https://github.com/cascadeguard/cascadeguard/blob/main/CONTRIBUTING.md) for branch naming, PR process, and coding standards.

## Building the Docker Image Locally

```bash
docker build -t cascadeguard:dev .
docker run --rm -v $(pwd)/path/to/state:/workspace cascadeguard:dev generate
```

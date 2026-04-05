# Security Model

CascadeGuard is built with a security-first design. Every image it manages is automatically scanned, attested, and signed as part of the standard pipeline.

## What CascadeGuard secures

### Vulnerability scanning

Every image build triggers dual-scanner analysis:

- **[Grype](https://github.com/anchore/grype)** — Anchore's vulnerability scanner, optimised for container images. Detects CVEs across OS packages and language ecosystems.
- **[Trivy](https://github.com/aquasecurity/trivy)** — Aqua Security's scanner. Cross-validates Grype findings and detects misconfigurations.

Running both scanners in parallel reduces false negatives and gives a higher-confidence CVE posture for each image.

The nightly `scheduled-scan.yaml` workflow re-scans every published image against the latest vulnerability databases — without rebuilding — and opens a GitHub Issue for any newly disclosed CVEs.

### Software Bill of Materials (SBOM)

CascadeGuard generates an SBOM for every image using [Syft](https://github.com/anchore/syft). The SBOM is:

- Attached as a build artifact
- Attested and signed alongside the image using Cosign (see below)

SBOMs let you audit exactly what packages and versions are inside each image, which is essential for responding quickly to newly disclosed vulnerabilities.

### Image signing

All images published by CascadeGuard are signed using **[Cosign](https://github.com/sigstore/cosign)** with keyless signing via [Sigstore](https://www.sigstore.dev/). The signature is stored in the OCI registry alongside the image.

To verify a CascadeGuard-published image:

```bash
cosign verify \
  --certificate-identity-regexp="https://github.com/cascadeguard/" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/cascadeguard/nginx:stable-alpine-slim
```

A valid signature proves the image was built by CascadeGuard CI and has not been tampered with since publication.

## Supply chain security

### Pinned GitHub Actions

CascadeGuard provides an `actions pin` command that rewrites all GitHub Actions `uses:` references from floating tags (e.g. `actions/checkout@v4`) to full commit SHAs (e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`). This prevents supply chain attacks where a compromised tag could inject malicious code.

```bash
cascadeguard actions pin
```

Use `--dry-run` to preview changes before writing them, and `--refresh` to re-pin SHAs when new versions are released.

### Actions policy enforcement

The `actions audit` command validates your workflow files against a declarative policy:

```bash
cascadeguard actions policy init      # Scaffold a starter policy
cascadeguard actions audit            # Audit all workflow files
```

The policy file (`.cascadeguard/actions-policy.yaml`) lets you define:

- Allowed action registries
- Required pinning (SHA vs. tag)
- Blocked actions

This is particularly useful for enforcing supply-chain standards across multiple repositories.

## Runtime security practices

| Practice | Detail |
|---|---|
| **Minimal base images** | CascadeGuard's exemplar images use Alpine and distroless bases to reduce attack surface. |
| **Non-root users** | Published images run as non-root by default. |
| **Read-only filesystems** | Recommended in deployment manifests where possible. |
| **Dependency updates** | Base images are monitored for updates; rebuilds are triggered automatically when upstream changes. |

## Responsible disclosure

Security vulnerabilities in CascadeGuard itself should be reported via [private vulnerability reporting](https://github.com/cascadeguard/cascadeguard/security/advisories/new). Do not open public issues for security vulnerabilities.

See [SECURITY.md](https://github.com/cascadeguard/cascadeguard/blob/main/SECURITY.md) for the full disclosure policy and response timeline.

# CascadeGuard — Project Instructions

Shared instructions for AI agents working across CascadeGuard repositories.

## Repositories

| Repo | Purpose |
|---|---|
| [cascadeguard](https://github.com/cascadeguard/cascadeguard) | Core CLI tool |
| [cascadeguard-open-secure-images](https://github.com/cascadeguard/cascadeguard-open-secure-images) | Hardened container images |
| [cascadeguard-app](https://github.com/cascadeguard/cascadeguard-app) | SaaS application |
| [cascadeguard-docs](https://github.com/cascadeguard/cascadeguard-docs) | Documentation |
| [cascadeguard-exemplar](https://github.com/cascadeguard/cascadeguard-exemplar) | Example/reference implementation |

## SDLC Process

Each repository follows the SDLC defined in its own `SDLC.md`. The canonical process covers how features move from idea to production including PR requirements, testing pyramid, and definition of done.

## Pull Request Standards

- Feature branches off `main`, named `<identifier>/<short-description>` (e.g. `CAS-45/add-billing-endpoint`).
- Every commit includes `Co-Authored-By: Paperclip <noreply@paperclip.ing>`.
- PRs must include: summary, test plan, and link to the tracking issue.
- All CI checks must pass before review. No direct commits to `main`.
- Agents do not merge their own PRs.

## Testing Requirements

- **Static analysis**: linting, type checking, secret scanning (100% of code).
- **Unit tests**: pure functions, business logic (80%+ line coverage target).
- **Integration tests**: service boundaries, deployment validation.
- **E2E tests**: critical user flows and deployment smoke tests.
- Every PR must include tests for new/changed behavior. Test failures block merge.

## Security

Vulnerability reports must follow each repo's `SECURITY.md`. Public issues that disclose vulnerabilities are immediately closed and moved to private advisories per the documented SLA.

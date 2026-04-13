# Image Lifecycle Policy

CascadeGuard publishes hardened container images that teams depend on in production. This page defines the official lifecycle policy — how long images are supported, how and when deprecation happens, and what action you need to take at each stage.

## Image Lifecycle States

Every CascadeGuard image is in one of three states:

| State | Meaning | Where you see it |
|-------|---------|-----------------|
| **Active** | Receiving security patches; rebuilt on schedule | Catalog, Dashboard, scan results |
| **Deprecated** | Available, but no longer actively rebuilt; migration recommended | Catalog (amber badge), scan warnings, email notification |
| **EOL (End of Life)** | Removed from active registry after grace period | Redirect to replacement image, this page |

## Support Windows by Tier

> **What counts as a "version"?**
> CascadeGuard tracks lifecycle at the upstream project's **release-track** level — not individual patch releases. For Node.js this means LTS major lines (`node:20`, `node:22`); for Go it means minor lines (`go:1.22`, `go:1.23`). Patch releases within a supported track (e.g. `node:20.15.0 → 20.16.0`) are automatically included and do not individually trigger lifecycle changes. The deprecation clock starts only when a new upstream release track ships and becomes the designated "latest".

### Free Tier

| Image type | Support window |
|------------|---------------|
| Latest supported version | Always supported (Active) |
| Previous supported versions | **90 days** after a newer supported version is released |

**Example:** When `node:22` ships, `node:20` enters a 90-day deprecation window. At T+90 days, `node:20` reaches EOL.

### Paid Tier

| Image type | Support window |
|------------|---------------|
| Latest supported version | Always supported (Active) |
| Previous supported versions | **180 days** after a newer supported version is released |

Extended support beyond 180 days is available on request — contact your account team.

Paid tier also includes custom notification channels (webhook, Slack) and priority rebuild requests.

## Deprecation Timeline

The following timeline applies when a new supported version is released. When discovery runs detect a new version, the published date is written to the state file and the previous version is automatically marked `deprecated`.

| Time | Event |
|------|-------|
| T+0 | New version discovered; published date written to state file; previous version status set to `deprecated`; amber badge appears in catalog |
| T+0 | Email notification sent to users watching that image |
| T+45 days (Free) / T+90 days (Paid) | Reminder notification — "halfway to EOL on [image]" |
| T+90 days (Free) / T+180 days (Paid) | Image reaches EOL |
| T+90/180 + 30 days | Image removed from active registry |
| T+90/180 + 120 days | Digest purged entirely (no further pulls) |

> **Note:** Digests for EOL images remain pullable for 90 days after EOL to allow time to migrate. After that, the digest is permanently removed.

## What You Should Do

### When an image is Deprecated (amber badge)

1. Check the catalog or scan results for the `recommended_replacement` field — this tells you which image to migrate to.
2. Update your `images.yaml` or Dockerfile to reference the replacement.
3. Run `cascadeguard scan` to verify no deprecated images remain in your dependency tree.
4. Target migration before the EOL date shown in the catalog.

### When an image reaches EOL

1. Pulls of the EOL digest will continue to work for 90 days post-EOL.
2. After 90 days, the digest is removed — any pipeline still referencing it will fail.
3. If you need extended access beyond the grace period, contact support.

## Upstream Deprecation Handling

CascadeGuard also tracks upstream deprecations from Docker Hub and other registries. When an upstream image is officially deprecated (e.g., `openjdk`):

- CascadeGuard marks the image `deprecated` immediately, regardless of version age.
- A `recommended_replacement` is provided (e.g., `eclipse-temurin`).
- Scan worker flags `DEPRECATED_BASE_IMAGE` findings in Dockerfile scans.
- A **90-day grace period** applies before EOL, regardless of tier.

Upstream deprecations are announced on [docker.com/blog](https://www.docker.com/blog/) and in the CascadeGuard changelog.

## Staying Notified

Registered users can subscribe to lifecycle state changes for any image:

1. Open the **Dashboard** and navigate to the **Catalog** tab.
2. Click the image you want to track and enable **Lifecycle notifications**.
3. You will receive email notifications at T+0 (deprecation) and T+3 months (reminder).

Paid tier users can configure additional notification channels (webhook, Slack) from their account settings.

## Questions or Exceptions

For questions about this policy, or to request a timeline exception, open an issue via the [CascadeGuard support portal](https://cascadeguard.app/support) or reach out to your account team.

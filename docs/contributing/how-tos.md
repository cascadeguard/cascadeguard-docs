# Contributing How-To Guides

CascadeGuard documentation includes a collection of task-focused how-to guides. This document explains how to write, structure, and contribute new guides.

## What is a how-to guide?

A **how-to guide** answers the question "how do I achieve X?" It assumes the reader has some familiarity with CascadeGuard and guides them through a specific task. How-tos are distinct from:

| Type | Answers | Example |
|---|---|---|
| **Tutorial** | "How do I get started?" | Getting Started guide |
| **How-to** | "How do I achieve X?" | How to integrate with GitLab CI |
| **Reference** | "What does X do?" | CLI reference |
| **Explanation** | "Why does X work this way?" | Security model |

How-to guides should be **goal-oriented** and assume a working CascadeGuard installation. They skip setup details already covered in Getting Started.

## How-to template

```markdown
# How-to: <Short goal statement>

Brief one-paragraph summary of what this guide accomplishes and when you would
need it.

## Prerequisites

- Item one (what the reader needs before starting)
- Item two

---

## Step 1 — <Action>

Explain what this step does and why.

```bash
example command
```

Expected output or confirmation the step succeeded.

## Step 2 — <Action>

...

---

## Troubleshooting

### <Error message or symptom>

Cause and resolution.
```

## Style conventions

### Headings

- Top-level heading (`#`) is the guide title — prefix with "How-to:".
- Use `##` for top-level steps and major sections.
- Use `###` for sub-steps and troubleshooting items.

### Step structure

- Name steps with a verb: "Configure", "Install", "Enable", not "Configuration".
- Each step should do one thing. Split complex steps rather than combining them.
- Separate steps with a horizontal rule (`---`) when there is a natural pause (e.g., a long-running operation).

### Progressive complexity

Order content from simple to complex:

1. Start with the **common case** — the path most readers will follow.
2. Add **variations** after the common case is complete (e.g., different registries, additional flags).
3. Put **advanced options** in a collapsible section or at the end, clearly marked.

This keeps the guide approachable for readers who only need the basics while remaining useful to readers who need more.

### Code blocks

- Always specify the language after the opening fence: ```` ```bash ````, ```` ```yaml ````, ```` ```typescript ````.
- Show commands as they would be typed — include the prompt character (`$`) only if needed to distinguish input from output.
- If a command produces important output, show it immediately after in a separate block or inline comment.

### Prerequisites section

- List concrete, specific prerequisites, not vague ones. "A GitHub repository with at least one Dockerfile" not "a GitHub repository".
- Link to the relevant guide when a prerequisite itself requires setup.

### Notes and callouts

Use blockquotes for callouts:

```markdown
> **Note:** brief informational aside.

> **Warning:** something that could cause data loss or significant disruption.

> **Tip:** useful shortcut or alternative.
```

Keep callouts short — one or two sentences. Long callouts suggest the content belongs in the main body.

### Links

- Link to related guides and reference docs inline where they are first mentioned.
- Use descriptive link text: `[CLI reference](../reference/cli.md)` not `[here](../reference/cli.md)`.

## File naming and location

| Guide type | Location | Naming convention |
|---|---|---|
| Integration guides | `docs/guides/` | `<platform>-<tool>.md` (e.g. `gitlab-argocd.md`) |
| Contributing guides | `docs/contributing/` | descriptive noun (e.g. `how-tos.md`) |

Use lowercase with hyphens. No spaces or underscores.

## Example: short how-to

The following is a minimal example illustrating the template in practice.

---

### How-to: Enrol a new image

This guide shows how to add a new container image to CascadeGuard's management in an existing state repository.

**Prerequisites**

- A working state repository with CascadeGuard initialised (`cascadeguard/state/` exists).
- Write access to `images.yaml` in that repository.

---

**Step 1 — Add the image to `images.yaml`**

Open `images.yaml` and append a new entry:

```yaml
images:
  - name: my-worker
    registry: ghcr.io
    repository: your-org/my-worker
    source:
      provider: github
      repo: your-org/my-worker
      dockerfile: Dockerfile
      branch: main
    rebuild_delay: 7d
```

`rebuild_delay` controls how long CascadeGuard waits after a base-image update before triggering a rebuild. `7d` is a reasonable default; set it lower for security-sensitive images.

**Step 2 — Regenerate state files**

```bash
task generate
```

CascadeGuard reads the updated `images.yaml`, inspects the registry, and writes a new state file to `cascadeguard/state/images/my-worker.yaml`.

**Step 3 — Commit and push**

```bash
git add images.yaml cascadeguard/state/images/my-worker.yaml
git commit -m "enrol my-worker image"
git push
```

The CI pipeline will pick up the new enrollment on the next run.

---

## Submitting a new how-to

1. Copy the [template](#how-to-template) into the appropriate directory.
2. Write the guide following the conventions above.
3. Open a PR against `main` with the title `docs: add how-to for <topic>`.
4. Ensure the guide is linked from the relevant index or navigation file.

For questions about where a guide belongs or whether a topic warrants a how-to, open a GitHub issue with the `docs` label.

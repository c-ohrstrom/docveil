# ADR 0002: Three packages in one uv workspace

- Status: Accepted
- Date: 2026-10-06

## Context

We want to support local LLMs and remote provider APIs, use the logic from both a CLI and a future hosted service, and keep model setup separate from anonymization logic.

## Decision

One GitHub repo, one uv workspace, three packages:

- `docveil-llm`: provider interface and adapters; local runtime management behind the `[local]` extra.
- `docveil-core`: anonymization logic; depends only on the provider interface.
- `docveil`: the CLI; the composition root that wires the other two.

A future `docveil-server` will be a second composition root.

## Consequences

- Providers can be added without touching the core.
- A hosted deployment can use core and providers without local-runtime dependencies.
- Changes across packages land in one commit; packages can still be released separately.
- Slightly more setup (three `pyproject.toml` files, versioning between packages).

# ADR 0005: mypy as the type checker

- Status: Accepted
- Date: 2026-10-07

## Context

The workspace needs one static type checker, run locally and in CI. The candidates were mypy and pyright (or its PyPI-packaged fork basedpyright).

## Decision

Use mypy in strict mode, with the pydantic mypy plugin, configured at the workspace root and run over all packages.

## Consequences

- Pure-Python dev dependency; no Node runtime needed.
- The pydantic plugin gives accurate types for models and settings.
- Slower than pyright on large codebases, which is not a concern at this size.

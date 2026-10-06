# ADR 0004: Async, I/O-free core that is ready to be hosted

- Status: Accepted
- Date: 2026-10-06

## Context

docveil starts as a CLI but should later be hostable as a service. Retrofitting async, removing terminal prompts and changing file-path APIs later is costly.

## Decision

`docveil-core`:

- is async-first, with a `run_sync()` helper for scripts;
- works on bytes and streams; path helpers are thin conveniences;
- does no printing or prompting; it reports progress through typed events;
- splits processing into `analyze()` and `apply(decisions)` so review can happen in any UI;
- stores mappings through a `MappingStore` protocol.

## Consequences

- The CLI and a future server share the same core without changes.
- Slightly more ceremony in the CLI (an event-to-Rich bridge, `asyncio.run`).
- Tests need pytest-asyncio.

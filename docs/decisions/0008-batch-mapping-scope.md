# ADR 0008: One mapping per run, shared across files

- Status: Accepted
- Date: 2026-10-07

## Context

When several files are processed, the same person may appear in all of them. Readers of the anonymised set need `[PERSON_1]` to mean the same person in every file.

## Decision

- **Default: one mapping per run.** Every file in a run (a folder, or several files on the command line) shares one mapping. The same entity gets the same placeholder in every file.
- Files are processed in sorted path order, so numbering is deterministic: placeholders are numbered by first appearance in that order.
- **`--per-file`** gives each file its own mapping; numbering restarts in each file.
- **`--mapping PATH`** continues across runs: the mapping is loaded from `PATH` if it exists and written back after the run. Without it, the mapping is kept in memory and discarded.
- Config: `[replace] mapping_scope = "run" | "file"`.

## Consequences

- Batch output is consistent by default.
- Processing order affects numbering, so it is fixed (sorted paths).
- `--mapping` is the only way to write a mapping to disk; see [privacy.md](../privacy.md#mappings).

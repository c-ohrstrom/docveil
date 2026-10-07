# docveil (CLI)

The command-line tool users install. A thin layer that wires `docveil-llm` and `docveil-core` together and handles everything interactive.

Built with Typer and Rich. The command is `docveil`; the import name is `docveil_cli`.

## Responsibilities

- Load config (`docveil.toml` in the current directory or user config dir) and merge CLI options.
- Build the provider: local by default (`ensure_local`), or from `--provider`.
- Enforce the remote opt-in (see [privacy.md](../privacy.md)).
- Ask before large model downloads.
- Show progress, the review table and the final summary.
- Read and write files; the core works on bytes.

## Commands

```
docveil doctor                        # hardware, chosen tier, runtime and model status
docveil setup                         # prepare runtime and model ahead of time
docveil scrub PATH... [-o OUT]       # anonymize files and/or folders
      [--recursive]                   #   include subfolders
      [--review]                      #   accept/reject entities before applying
      [--style tag|generic]           #   fake later
      [--types PERSON,ORG,...]
      [--provider SPEC]               #   e.g. lmstudio:<model>, anthropic:<model>
      [--allow-remote]
      [--per-file]                    #   separate mapping per file
      [--mapping PATH]                #   load/save the mapping to continue across runs
docveil restore FILE --mapping PATH   # reverse using a saved mapping
docveil models list|pull|use          # manage local models
```

Default output names: `report.docx` → `report.clean.docx`; a folder `reports/` → `reports.clean/` with the same structure.

## Mappings across files

See [requirements](../requirements.md#replacement-and-mappings).

| You run | Result |
|---|---|
| `docveil scrub a.docx b.docx` or `docveil scrub reports/` | **One mapping for the whole run.** "Anna Berg" is `[PERSON_1]` in every file. |
| `… --per-file` | Each file has its own mapping. "Anna Berg" may be `[PERSON_1]` in one file and `[PERSON_3]` in another. |
| `… --mapping project.json` | The mapping is loaded from `project.json` if it exists and saved back after the run, so later runs continue the numbering. |

- Files are processed in sorted path order, so numbering is the same each time.
- Without `--mapping`, the mapping is kept in memory only and discarded after the run.
- A saved mapping contains the original values. It is plain JSON for now; keep it somewhere safe (see [privacy.md](../privacy.md#mappings)).

## Wiring (sketch)

```python
async def scrub(file, provider_spec, allow_remote, ...):
    config = load_config(overrides=...)

    if provider_spec is None:
        from docveil_llm.local import ensure_local
        provider = await ensure_local(on_progress=rich_progress, confirm_download=ask_user)
    else:
        provider = get_provider(provider_spec)

    if not provider.capabilities.is_local and not allow_remote:
        abort("This sends document text to an external API. Use --allow-remote to continue.")

    anon = Anonymizer(provider=provider, config=config)
    data = Path(file).read_bytes()

    if review:
        analysis = await anon.analyze(data, fmt=fmt, on_event=rich_reporter)
        decisions = review_table(analysis.entities)
        result = await anon.apply(analysis, decisions)
    else:
        result = await anon.process(data, fmt=fmt, on_event=rich_reporter)

    out.write_bytes(result.output)
    print_summary(result, provider.capabilities)
```

## Review mode

A Rich table of found entities with type, placeholder, number of occurrences and a short context snippet. The user can accept all, reject individual entities, or add missed strings. Later this could become a Textual TUI.

## Summary output

- Number of entities per type
- Provider and model used, and whether it was local
- Output path, and mapping path if saved
- Verification warnings, if any

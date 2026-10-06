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
docveil scrub FILE [-o OUT]           # anonymize one file
      [--review]                      #   accept/reject entities before applying
      [--style tag|generic|fake]
      [--types PERSON,ORG,...]
      [--provider SPEC]               #   e.g. anthropic:<model>
      [--allow-remote]
      [--save-mapping PATH]
docveil scrub DIR --recursive         # batch; one shared mapping
docveil restore FILE --mapping PATH   # reverse using a saved mapping
docveil models list|pull|use          # manage local models
```

Default output name: `report.docx` → `report.clean.docx`.

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

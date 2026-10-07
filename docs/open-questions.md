# Open questions

Decisions that are not made yet. Move each one into an ADR when it is decided.

## Open

1. **Default runtime order on Apple Silicon:** llama.cpp or MLX first? Compare speed and JSON reliability on the eval set. See [ADR 0006](decisions/0006-local-runtimes.md), which also lists a smaller runtime question.

## Deferred

These are noted for later and do not block the first iteration.

2. **Which models per tier?** Decide after comparing candidates on the eval set. Criteria: recall per entity type and language (English and Swedish), JSON validity rate, speed on the tier's hardware, and a licence compatible with fully open-source distribution and hosting (prefer Apache-2.0 or MIT).
3. **Mapping encryption.** For now, saving a mapping is opt-in and the user is responsible for storing it ([privacy.md](privacy.md#mappings)). If built-in encryption is added: support both the OS keychain (`keyring`, convenient) and a passphrase (portable, needed to restore on another machine). Likely `cryptography` with AES-GCM and a scrypt-derived key; the `age` format is an alternative.
4. **Licensing.** The repo is private for now. The goal is a fully open-source project under a permissive licence (Apache-2.0 or MIT). Known conflicts:
   - **PyMuPDF is AGPL-3.0** (or commercial). Prefer a permissively licensed way to redact PDFs; failing that, keep PDF support in an optional extra and document the AGPL terms that come with it.
   - **Model licences** vary (some GLiNER variants are non-commercial; some LLM families have use restrictions). Check each against the criteria in question 2.
5. **Hosted version.** The first iteration is local on a MacBook ([overview](overview.md#first-iteration)). The expected direction is a self-hostable server first, a managed service maybe later. Decide when the hosted work starts; it affects auth, storage and which providers are allowed.

## Decided

| Question | Decision |
|---|---|
| Type checker | mypy, [ADR 0005](decisions/0005-mypy-type-checker.md) |
| Local runtime | Use a running server, otherwise start the first available of llama.cpp, MLX, LM Studio, Ollama, [ADR 0006](decisions/0006-local-runtimes.md) |
| Languages | English and Swedish, [ADR 0007](decisions/0007-languages-en-sv.md) |
| Batch consistency | One mapping per run, `--per-file` and `--mapping` options, [ADR 0008](decisions/0008-batch-mapping-scope.md) |
| Default replacement style | `tag`; `fake` is a later opt-in ([docveil-core](packages/docveil-core.md#5-replacement-replace)) |
| Dates | Off by default, enabled in config ([docveil-core](packages/docveil-core.md#entity-types)) |

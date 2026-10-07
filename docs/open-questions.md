# Open questions

Things not settled yet. When one is settled, move it into [requirements.md](requirements.md).

## Open

1. **Default runtime order on Apple Silicon:** llama.cpp or MLX first? Compare speed and JSON reliability on the eval set. See [requirements](requirements.md#local-runtimes).
2. **Getting `llama-server` without Homebrew:** official release binaries, or `llama-cpp-python`?

## Deferred

These are noted for later and do not block the first iteration.

3. **Which models per tier?** Decide after comparing candidates on the eval set. Criteria: recall per entity type and language (English and Swedish), JSON validity rate, speed on the tier's hardware, and a licence compatible with fully open-source distribution and hosting (prefer Apache-2.0 or MIT).
4. **Mapping encryption.** For now, saving a mapping is opt-in and the user is responsible for storing it ([privacy.md](privacy.md#mappings)). If built-in encryption is added: support both the OS keychain (`keyring`, convenient) and a passphrase (portable, needed to restore on another machine). Likely `cryptography` with AES-GCM and a scrypt-derived key; the `age` format is an alternative.
5. **Licensing.** The repo is private for now. The goal is a fully open-source project under a permissive licence (Apache-2.0 or MIT). Known conflicts:
   - **PyMuPDF is AGPL-3.0** (or commercial). Prefer a permissively licensed way to redact PDFs; failing that, keep PDF support in an optional extra and document the AGPL terms that come with it.
   - **Model licences** vary (some GLiNER variants are non-commercial; some LLM families have use restrictions). Check each against the criteria in question 3.
6. **Hosted version.** The first iteration is local on a MacBook ([overview](overview.md#first-iteration)). The expected direction is a self-hostable server first, a managed service maybe later. Decide when the hosted work starts; it affects auth, storage and which providers are allowed.

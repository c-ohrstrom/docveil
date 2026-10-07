# Open questions

Decisions that are not made yet. Move each one into an ADR when it is decided.

1. **Which models per tier?** The registry has placeholders. Decide after comparing candidates on the eval set, including non-English documents.
2. **Which languages must be supported from the start?** Affects model choice, NER model and rules (e.g. Swedish personal identity numbers).
3. **Ollama not installed:** only print instructions, offer to install it, or fall back to a bundled llama.cpp runtime?
4. **Default replacement style:** `tag` is safest and clearest; `fake` reads more naturally. Keep `tag` as default?
5. **Mapping encryption:** passphrase, OS keychain, or both? Which library?
6. **Dates:** should dates be treated as identifying by default? They often are in combination, but replacing them hurts readability.
7. **Batch consistency:** should one mapping always be shared across a folder, or should it be configurable per run?
8. **Licensing:** which license for the repo, and are all chosen models' licenses compatible with how the tool will be distributed and hosted?
9. **Hosted version:** what shape — a self-hostable server, a SaaS, or both? Affects auth, storage and which providers are allowed.

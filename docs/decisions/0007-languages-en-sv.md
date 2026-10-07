# ADR 0007: English and Swedish from the start, rules as locale packs

- Status: Accepted
- Date: 2026-10-07

## Context

Language support affects the LLM choice, the NER model, the rules and the eval set. Supporting few languages well is better than many poorly.

## Decision

- Support **English (`en`) and Swedish (`sv`)** from the first version.
- Country-specific rules live in **locale packs** (`detect/rules/locales/en.py`, `sv.py`). Language-neutral rules (email, URL, IP, IBAN, card numbers) are shared.
- The Swedish pack covers personnummer and samordningsnummer (with Luhn check), organisationsnummer, Swedish phone numbers and postal codes.
- Config: `languages = ["en", "sv"]`. All configured packs run on every document; detecting the language per document is not needed for rules.
- The eval set has English and Swedish documents from M0.
- Model and NER choices must be evaluated on both languages.

## Consequences

- A multilingual NER model is needed (a multilingual GLiNER variant).
- Adding a language later means adding a locale pack and eval documents.

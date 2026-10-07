# docveil-core

All anonymization logic. Receives a ready `LLMProvider` (or none, for rules + NER only) and a config, and turns a document into an anonymized document.

It does no terminal I/O and does not know how the provider was created. See [requirements](../requirements.md#architecture).

## Public API

```python
from docveil_core import Anonymizer, Config

anon = Anonymizer(provider=provider, config=Config.load("docveil.toml"))

# one step
result = await anon.process(data, fmt="docx")          # bytes in, Result out
result.output        # bytes
result.entities      # what was found and replaced

# two steps, for review
analysis = await anon.analyze(data, fmt="docx")
decisions = review(analysis.entities)                  # done by CLI / web UI
result = await anon.apply(analysis, decisions)

# convenience for scripts
from docveil_core import run_sync
run_sync(anon.process_path("report.docx", out="report.clean.docx"))
```

Progress is reported with an optional `on_event` callback receiving typed events (`DocumentLoaded`, `ChunkStarted`, `ChunkDone`, `EntitiesFound`, `Finished`).

## Pipeline

```
load → segments → chunk → detect (rules, NER, LLM) → merge → group aliases
     → assign placeholders → [review] → apply → write → (verify)
```

### 1. Documents (`documents/`)

Each format turns into a list of **text segments** (roughly paragraphs, table cells, headers) with a way to write replacements back.

| Format | Library | Notes |
|---|---|---|
| `.txt`, `.md` | – | Segment by paragraph |
| `.docx` | python-docx | Text is split across formatting runs; match on paragraph text, map offsets back to runs. Also cover tables, headers, footers, comments, core properties. |
| `.pdf` | PyMuPDF | Real redaction with `add_redact_annot` + `apply_redactions`, then insert placeholder text. Clear metadata. Text-based PDFs only. |

### 2. Chunking

- Group segments into chunks that fit `provider.capabilities.context_tokens`, leaving room for the prompt and output.
- Overlap chunks slightly so entities on boundaries are not missed.

### 3. Detection (`detect/`)

All detectors implement:

```python
class Detector(Protocol):
    async def detect(self, chunk: Chunk) -> list[Candidate]: ...
```

| Detector | Finds | Notes |
|---|---|---|
| `rules` | Email, phone, URL, IP, IBAN, card numbers (Luhn), national ID numbers, postal codes, dates (only when `DATE` is enabled) | Regex plus checksums; always on |
| `ner` | Person, organisation, location | Multilingual GLiNER (optional extra), runs on CPU |
| `llm` | Context-dependent and indirect identifiers, aliases | Uses `complete_json` with a schema; skipped if no provider |

#### Languages and locale packs

English and Swedish are supported from the start ([requirements](../requirements.md#languages)). Language-neutral rules (email, URL, IP, IBAN, cards) are shared. Country-specific rules live in locale packs under `detect/rules/locales/`:

| Pack | Covers |
|---|---|
| `en` | UK and US phone formats, postcodes and ZIP codes, common ID formats (e.g. US SSN, UK NI number) |
| `sv` | Personnummer and samordningsnummer (Luhn), organisationsnummer, Swedish phone numbers, postnummer |

All configured packs run on every document.

#### LLM output

The LLM returns entity **strings and types**, never offsets or rewritten text ([requirements](../requirements.md#detection)):

```json
{"entities": [{"text": "Anna Berg", "type": "PERSON", "aliases": ["Anna", "Ms. Berg"]}]}
```

### 4. Merge (`detect/merge.py`)

- Locate every occurrence of every candidate string in the original text: normalised whitespace, case-insensitive where safe, word boundaries (so "Ann" does not match inside "Annual").
- Resolve overlapping spans (prefer the longest; record which detectors agreed).
- Group aliases into one entity ("Anna Berg", "Anna", "Ms. Berg" → one person).

### 5. Replacement (`replace/`)

| Strategy | Example |
|---|---|
| `tag` (default) | `[PERSON_1]`, `[ORG_2]` |
| `generic` | "Person A", "Company B" |
| `fake` (later, opt-in) | Realistic fake names from Faker, consistent per entity |

- The **mapping** (entity → placeholder) is shared across chunks and, by default, across all files in a run ([requirements](../requirements.md#replacement-and-mappings)). `mapping_scope = "file"` gives each file its own mapping.
- Mappings are stored through a `MappingStore` protocol: an in-memory store by default, `FileMappingStore` when the user asks to keep the mapping, and a database store for the server later.
- The mapping is not written to disk unless the user asks. Usually the original document is kept, so the mapping is not needed. A saved mapping is plain JSON for now; storing it safely is up to the user (see [privacy.md](../privacy.md#mappings)).

### 6. Verification (optional)

- Run the rules again on the output.
- Optionally ask the LLM whether anything identifying is left; report findings as warnings rather than silently changing the output.

## Entity types

Initial set, configurable:

`PERSON`, `ORG`, `LOCATION`, `ADDRESS`, `EMAIL`, `PHONE`, `URL`, `IP`, `ID_NUMBER`, `FINANCIAL`, `DATE` (off by default), `OTHER`.

Dates are rare in the expected documents and replacing them hurts readability, so `DATE` is off by default and can be enabled in config. Dates inside other identifiers (e.g. the birth date in a personnummer) are covered by that identifier's rule.

## Config

TOML, loaded with pydantic-settings; CLI options override it.

```toml
[detect]
types = ["PERSON", "ORG", "LOCATION", "EMAIL", "PHONE", "ID_NUMBER"]
detectors = ["rules", "ner", "llm"]
languages = ["en", "sv"]

[replace]
style = "tag"            # tag | generic (fake later)
mapping_scope = "run"    # run | file

[verify]
enabled = true
```

## Evaluation

`tests/eval/` at the repo root holds 20–50 synthetic documents in English and Swedish with labelled entities. A scorer reports recall and precision per entity type, per detector and per language. It is used to compare models, prompts and detector combinations.

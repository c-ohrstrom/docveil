# Requirements

What docveil must do and the constraints it works under. These are working requirements: change them when we learn something that points elsewhere. Each requirement has a short reason so it is clear what changing it would affect. The package docs describe how the requirements are met.

## Scope of the first iteration

- The CLI runs fully locally on a MacBook with Apple Silicon.
- Supported formats: `.txt`, `.md`, `.docx` and text-based `.pdf`.
- Linux, NVIDIA GPUs and a hosted service come later. The design must not rule them out (see [Architecture](#architecture)).

## Languages

- English and Swedish documents are supported from the start.
- Country-specific rules are grouped in **locale packs** (`en`, `sv`); language-neutral rules (email, URL, IP, IBAN, card numbers) are shared. All configured packs run on every document.
- The Swedish pack covers personnummer and samordningsnummer (with Luhn check), organisationsnummer, Swedish phone numbers and postnummer.
- The eval set contains English and Swedish documents, and models and NER are compared on both. A multilingual NER model is therefore needed.

*Why:* these are the languages of the documents the tool is first meant for. Adding a language later means adding a locale pack and eval documents.

## Detection

- **The LLM only detects.** It returns a list of entity strings, types and aliases, validated against a JSON schema. Code finds every occurrence in the original text and replaces it. A model never regenerates the document's text or formatting.
  - *Why:* LLMs asked to rewrite text change wording, drop sentences, hallucinate and break formatting, and they are unreliable at returning character offsets. Detection-only output is deterministic to apply and can be reviewed before applying.
  - *Cost:* matching needs care (normalisation, word boundaries, text split across DOCX runs), and entities the LLM paraphrases may be missed; the other layers and the verification pass reduce this.
- **Several layers**: rules (regex plus checksums), a NER model and an LLM, merged into one result, with an optional verification pass.
- **Recall over precision.** A missed name is a leak; an unnecessary replacement is an inconvenience.
- **Dates are off by default** and can be enabled in config. They are rare in the expected documents and replacing them hurts readability. Dates inside other identifiers (such as a personnummer) are covered by that identifier's rule.

## Replacement and mappings

- The default style is `tag` (`[PERSON_1]`). `generic` ("Person A") is also supported. `fake` (realistic fake names) comes later as an opt-in.
- The same entity gets the same placeholder throughout a document.
- **One mapping per run by default.** All files in a run (a folder, or several files) share one mapping, so "Anna Berg" is `[PERSON_1]` in every file.
  - Files are processed in sorted path order, so numbering is deterministic.
  - `--per-file` gives each file its own mapping.
  - `--mapping PATH` loads the mapping from `PATH` if it exists and saves it back after the run, so numbering continues across runs.
- The mapping is not written to disk unless the user asks with `--mapping`. Usually the original document is kept, so the mapping is rarely needed. A saved mapping is plain JSON for now; storing or encrypting it is up to the user (see [privacy.md](privacy.md#mappings)).

## LLM access

- One provider interface (`LLMProvider`) for every model, local or remote, with `complete` and `complete_json`.
- One thin `openai_compat` adapter (httpx) covers llama.cpp server, MLX server, LM Studio, Ollama, vLLM, OpenAI and other compatible APIs. A native `anthropic` adapter covers Anthropic.
  - *Why:* two adapters reach most of the field without a heavy multi-provider library. We own the retry and error-mapping logic in return.
- Differences between providers (for example native JSON-schema output) are expressed through `Capabilities` flags, with a prompt-based fallback.
- If a provider's compatible mode lacks something important, a native adapter can be added.
- Remote providers are an explicit opt-in (see [privacy.md](privacy.md#remote-providers)).

## Local runtimes

docveil uses whatever local runtime it can, and starts it itself when possible:

1. **Use a running server** if one answers on its default localhost port: LM Studio (`:1234`), `llama-server` (`:8080`) or Ollama (`:11434`).
2. **Otherwise, start the first available runtime**, in this order (configurable):

   | Runtime | Available when | How docveil starts it | Models |
   |---|---|---|---|
   | llama.cpp | `llama-server` on `PATH` | Child process on a free localhost port | GGUF from Hugging Face |
   | MLX (Apple Silicon) | `docveil-llm[mlx]` installed | `mlx_lm.server` as a child process | MLX models from Hugging Face |
   | LM Studio | `lms` CLI found | `lms server start`, then `lms load <model>` | `lms get <model>` |
   | Ollama | `ollama` CLI found | `ollama serve` in the background | `ollama pull <model>` |

3. If nothing is available, show an error that lists the options (`brew install llama.cpp`, the `[mlx]` extra, LM Studio or Ollama).

Constraints:

- docveil never installs system software itself.
- Models are downloaded only after the user has been asked.
- Processes docveil started as children are stopped when it is done. Servers and apps that were already running are left as they were.

*Why:* no dependency on a single runtime; the tool works with what the user already has. See [open questions](open-questions.md) for the default order on Apple Silicon.

## Architecture

- **One repo, one uv workspace, three packages**: `docveil-llm` (provider interface, adapters, local runtimes behind the `[local]` extra), `docveil-core` (anonymization logic, depends only on the provider interface) and `docveil` (the CLI, which wires the other two together). A future `docveil-server` is a second such wiring point.
  - *Why:* providers can be added without touching the core, and a hosted deployment can use core and providers without the local-runtime dependencies. Changes across packages still land in one commit.
- **The core is async and does no I/O of its own**: it works on bytes, reports progress through typed events, does no printing or prompting, splits processing into `analyze()` and `apply(decisions)` so review can happen in any UI, and stores mappings through a `MappingStore` protocol. A `run_sync()` helper covers scripts.
  - *Why:* the CLI and a future server can share the core unchanged; retrofitting this later is costly.

See [architecture.md](architecture.md) for the dependency rules and repo layout.

## Privacy

See [privacy.md](privacy.md). In short: local by default, remote only by explicit opt-in with a warning and local masking first, no telemetry, and no document text in logs.

## Development

- Python 3.11+, managed with uv.
- ruff for linting and formatting.
- mypy in strict mode, with the pydantic plugin, configured at the workspace root. *Why:* pure-Python dev dependency (no Node), accurate types for pydantic models.
- pytest with pytest-asyncio; respx for mocking HTTP.
- CI (GitHub Actions) runs lint, type check and tests on macOS and Linux.
- An eval set of labelled synthetic documents exists from M0, so model and prompt changes can be measured.

# Roadmap

Providers come before local runtime management: the whole pipeline can be built against an already-running LM Studio or `llama-server`, and automated setup is added afterwards.

## M0 – Repo setup

- [ ] uv workspace with `packages/docveil-llm`, `packages/docveil-core`, `packages/docveil`
- [ ] ruff, mypy, pytest configured at the root
- [ ] CI (GitHub Actions): lint, type check, tests on macOS and Linux
- [ ] `tests/eval/` skeleton with a first handful of labelled synthetic documents in English and Swedish

## M1 – `docveil-llm` providers

- [ ] `Message`, `Capabilities`, error hierarchy
- [ ] `LLMProvider` protocol
- [ ] Provider string parsing (`spec.py`) and `get_provider()`
- [ ] `openai_compat` adapter with retries and error mapping
- [ ] `complete_json` with native schema support and prompt fallback
- [ ] Tests with mocked HTTP; optional integration test against LM Studio or `llama-server`

## M2 – `docveil-core` MVP

- [ ] Entity and candidate models
- [ ] `.txt` / `.md` documents
- [ ] Chunking based on `context_tokens`
- [ ] Rules detector with `en` and `sv` locale packs
- [ ] LLM detector (prompt + schema)
- [ ] Merge: locate spans, resolve overlaps
- [ ] `tag` replacement strategy and in-memory mapping
- [ ] `analyze` / `apply` / `process`, events
- [ ] Eval scorer: recall and precision per type and language

## M3 – CLI

- [ ] `docveil doctor` (provider status only at this point)
- [ ] `docveil scrub` for text files
- [ ] `--provider`, `--allow-remote`, remote warning
- [ ] Config file loading
- [ ] Rich progress and summary

## M4 – `docveil-llm[local]`

- [ ] Hardware probe (macOS / Apple Silicon)
- [ ] `registry.toml` and tier selection
- [ ] Detect a running LM Studio, `llama-server` or Ollama
- [ ] llama.cpp runtime: find `llama-server`, download GGUF with progress, start and stop
- [ ] MLX runtime (`[mlx]` extra)
- [ ] LM Studio runtime (`lms server start`, `lms get`, `lms load`)
- [ ] Ollama runtime (`ollama serve`, `ollama pull`)
- [ ] `ensure_local()`
- [ ] `docveil setup`, `docveil models`, full `docveil doctor`
- [ ] Ask before downloads; check disk space

## M5 – Detection quality

- [ ] GLiNER NER detector
- [ ] Alias grouping
- [ ] Verification pass
- [ ] Expand eval set (more English and Swedish documents)
- [ ] Compare models per tier and update `registry.toml`
- [ ] Choose the default runtime order on Apple Silicon (llama.cpp or MLX first)

## M6 – Formats

- [ ] `.docx` with formatting preserved (runs, tables, headers, footers)
- [ ] `.pdf` redaction with PyMuPDF
- [ ] Metadata cleaning for both

## M7 – UX

- [ ] `--review` mode
- [ ] Multiple files and folders with one shared mapping, `--per-file`, `--mapping PATH`
- [ ] `generic` replacement style
- [ ] `docveil restore`
- [ ] Local masking before remote calls
- [ ] `anthropic` adapter

## M8 – Later

- [ ] `fake` replacement style (opt-in)
- [ ] Encrypted mappings
- [ ] Linux + NVIDIA support in the hardware probe and runtimes
- [ ] `MappingStore` database backend
- [ ] `docveil-server` (HTTP API on top of core)
- [ ] OCR for scanned PDFs
- [ ] TUI or web UI

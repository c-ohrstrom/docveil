# Roadmap

Providers come before local runtime management: the whole pipeline can be built against an already-running Ollama, and automated setup is added afterwards.

## M0 – Repo setup

- [ ] uv workspace with `packages/docveil-llm`, `packages/docveil-core`, `packages/docveil`
- [ ] ruff, mypy, pytest configured at the root
- [ ] CI (GitHub Actions): lint, type check, tests on macOS and Linux
- [ ] `tests/eval/` skeleton with a first handful of labelled synthetic documents

## M1 – `docveil-llm` providers

- [ ] `Message`, `Capabilities`, error hierarchy
- [ ] `LLMProvider` protocol
- [ ] Provider string parsing (`spec.py`) and `get_provider()`
- [ ] `openai_compat` adapter with retries and error mapping
- [ ] `complete_json` with native schema support and prompt fallback
- [ ] Tests with mocked HTTP; optional integration test against Ollama

## M2 – `docveil-core` MVP

- [ ] Entity and candidate models
- [ ] `.txt` / `.md` documents
- [ ] Chunking based on `context_tokens`
- [ ] Rules detector
- [ ] LLM detector (prompt + schema)
- [ ] Merge: locate spans, resolve overlaps
- [ ] `tag` replacement strategy and in-memory mapping
- [ ] `analyze` / `apply` / `process`, events
- [ ] Eval scorer: recall and precision per type

## M3 – CLI

- [ ] `docveil doctor` (provider status only at this point)
- [ ] `docveil scrub` for text files
- [ ] `--provider`, `--allow-remote`, remote warning
- [ ] Config file loading
- [ ] Rich progress and summary

## M4 – `docveil-llm[local]`

- [ ] Hardware probe (macOS/Apple Silicon, Linux + NVIDIA)
- [ ] `registry.toml` and tier selection
- [ ] Ollama runtime: detect, start, pull with progress
- [ ] `ensure_local()`
- [ ] `docveil setup`, `docveil models`, full `docveil doctor`
- [ ] Ask before downloads; check disk space

## M5 – Detection quality

- [ ] GLiNER NER detector
- [ ] Alias grouping
- [ ] Verification pass
- [ ] Expand eval set (more documents, non-English)
- [ ] Compare models per tier and update `registry.toml`

## M6 – Formats

- [ ] `.docx` with formatting preserved (runs, tables, headers, footers)
- [ ] `.pdf` redaction with PyMuPDF
- [ ] Metadata cleaning for both

## M7 – UX

- [ ] `--review` mode
- [ ] Batch folders with shared mapping
- [ ] `generic` and `fake` replacement styles
- [ ] Encrypted mappings, `docveil restore`
- [ ] Local masking before remote calls
- [ ] `anthropic` adapter

## M8 – Later

- [ ] llama.cpp runtime (no Ollama needed)
- [ ] `MappingStore` database backend
- [ ] `docveil-server` (HTTP API on top of core)
- [ ] OCR for scanned PDFs
- [ ] TUI or web UI

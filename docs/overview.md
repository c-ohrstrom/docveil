# Overview

## Problem

Reports, articles and other documents often contain names, organisations, places, contact details and other information that can identify people. Before sharing such documents, that information needs to be replaced with generic placeholders, without otherwise changing the document.

## Goals

- Detect identifying information in documents and replace it with consistent placeholders (`[PERSON_1]`, "Person A", and later realistic fake names as an opt-in).
- Run **fully locally** by default: the tool checks the machine's capabilities, picks a suitable local model, downloads it and starts it.
- Allow **remote LLM providers** (OpenAI, Anthropic, any OpenAI-compatible API) as an explicit opt-in.
- Preserve the document: formatting, structure and all non-identifying text stay unchanged.
- Be usable as a **Python library**, through a **CLI**, and later as a **hosted service**.
- Support `.txt`, `.md`, `.docx` and text-based `.pdf`.
- Support English and Swedish documents from the start ([ADR 0007](decisions/0007-languages-en-sv.md)).

## First iteration

The first target is the CLI running fully locally on a MacBook with Apple Silicon. Linux, NVIDIA GPUs and the hosted service come later, but the design keeps them possible (see [ADR 0004](decisions/0004-async-core-io-free.md)).

For hosting, the expected order is a self-hostable server first and a managed service, if any, later. This is not decided yet; see [open questions](open-questions.md).

## Non-goals (for now)

- OCR of scanned documents.
- Guaranteed, certified anonymization. The tool is assistive; output should be reviewed for sensitive use.
- Images, audio or video.
- A graphical desktop app (the library design keeps this possible later).

## Guiding principles

1. **The LLM detects, code replaces.** The model never rewrites the document. See [ADR 0001](decisions/0001-llm-detects-code-replaces.md).
2. **Several detection layers.** Rules, a small NER model and an LLM complement each other; the LLM is not the only safety net.
3. **Recall over precision.** A missed name is a leak; an unnecessary replacement is an inconvenience. Tune and measure for recall.
4. **Consistency.** The same entity gets the same placeholder throughout a document (and across a batch).
5. **Local and private by default.** Nothing leaves the machine unless the user explicitly allows it. See [privacy.md](privacy.md).
6. **Clear boundaries.** The LLM layer knows nothing about anonymization; the core knows nothing about how models are installed or which provider is used. See [architecture.md](architecture.md).
7. **Measure.** An evaluation set with labelled synthetic documents exists from the first milestone, so model and prompt changes can be compared.

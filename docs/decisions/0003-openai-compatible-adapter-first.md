# ADR 0003: One OpenAI-compatible adapter covers most providers

- Status: Accepted
- Date: 2026-10-06

## Context

We need to talk to local runtimes and to remote providers. Writing one adapter per provider is a lot of work. Large multi-provider libraries add heavy dependencies and a large surface we do not need.

## Decision

- Write a thin `openai_compat` adapter using httpx. It covers llama.cpp server, MLX server, LM Studio, Ollama's `/v1` endpoint, vLLM, OpenAI and other compatible APIs.
- Write a native `anthropic` adapter.
- Local model management (downloads, starting runtimes) lives in `docveil_llm.local`, not in the adapters. See [ADR 0006](0006-local-runtimes.md).
- Support differences (e.g. native JSON-schema output) through `Capabilities` flags, with a prompt-based fallback.

## Consequences

- Two adapters reach most of the field.
- We own retry and error-mapping logic.
- If a provider's compatible mode lacks something important, add a native adapter for it.

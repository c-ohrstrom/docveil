# ADR 0006: docveil starts whichever local runtime is available

- Status: Accepted
- Date: 2026-10-07

## Context

The first plan used Ollama as the only managed local runtime, with llama.cpp later. We prefer not to depend on any single runtime. The first target is a MacBook with Apple Silicon (see [overview](../overview.md#first-iteration)), where llama.cpp (Metal), MLX, LM Studio and Ollama all run well.

All of these expose an OpenAI-compatible server, so the provider layer ([ADR 0003](0003-openai-compatible-adapter-first.md)) does not change. Only local runtime management (`docveil_llm.local`) is affected.

## Decision

docveil uses whatever local runtime it can, and starts it itself when possible. `ensure_local()` works in this order:

1. **Use a running server** if one answers on its default localhost port: LM Studio (`:1234`), `llama-server` (`:8080`) or Ollama (`:11434`).
2. **Otherwise, start the first available runtime**, in this order (configurable with `runtime = "..."`):

   | Runtime | Available when | How docveil starts it | Models |
   |---|---|---|---|
   | llama.cpp | `llama-server` on `PATH` | Child process on a free localhost port | GGUF from Hugging Face |
   | MLX (Apple Silicon) | `docveil-llm[mlx]` installed | `mlx_lm.server` as a child process | MLX models from Hugging Face |
   | LM Studio | `lms` CLI found | `lms server start`, then `lms load <model>` | `lms get <model>` |
   | Ollama | `ollama` CLI found | `ollama serve` in the background | `ollama pull <model>` |

3. If nothing is available, raise an error that lists the options (`brew install llama.cpp`, the `[mlx]` extra, LM Studio or Ollama).

Rules:

- docveil never installs system software itself.
- Models are downloaded only after the CLI asks the user.
- Processes that docveil started as children are stopped when the provider is closed. Servers that were already running, and apps such as LM Studio, are left as they were; a model docveil loaded into them may be unloaded.

## Still to decide

- Default order on Apple Silicon: llama.cpp or MLX first. Compare speed and JSON reliability on the eval set (M5). llama.cpp supports grammar-constrained JSON-schema output, which matters for `complete_json`; check what `mlx_lm.server` supports.
- How to get `llama-server` without Homebrew (official release binaries, or `llama-cpp-python`).

## Consequences

- No dependency on one runtime; the tool works with what the user already has.
- Four runtime modules to maintain, each with start, health check, model download and stop.
- Models end up in different stores (Hugging Face cache, LM Studio, Ollama); `docveil models` shows where.
- The registry maps each tier to a model per runtime.

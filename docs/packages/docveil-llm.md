# docveil-llm

The LLM access layer. Gives the rest of the project one interface to any model, local or remote, and optionally prepares a local model on the user's machine.

It must not contain anything specific to anonymization.

## Public API

```python
from docveil_llm import get_provider, LLMProvider, Capabilities, Message

provider = get_provider("ollama:qwen3:8b")
answer = await provider.complete_json(messages, schema=MySchema)

# local mode (requires docveil-llm[local])
from docveil_llm.local import ensure_local
provider = await ensure_local(on_progress=callback)
```

## Provider interface

```python
class Capabilities(BaseModel):
    provider: str                 # "ollama", "openai", "anthropic", …
    model: str
    context_tokens: int
    structured_output: bool       # native JSON-schema support
    is_local: bool                # does data stay on this machine?

class LLMProvider(Protocol):
    capabilities: Capabilities

    async def complete(self, messages: list[Message], **opts) -> str: ...
    async def complete_json(self, messages: list[Message], schema: type[T], **opts) -> T: ...
    async def close(self) -> None: ...
```

### `complete_json` behaviour

- If `capabilities.structured_output` is true, send the pydantic model's JSON schema natively.
- Otherwise, add the schema to the prompt, ask for JSON only, and parse it.
- In both cases, validate with pydantic. On failure, retry once with the validation error included. Then raise `StructuredOutputError`.

### Errors

A small hierarchy so callers do not depend on provider-specific exceptions:

- `LLMError` (base)
  - `ProviderUnavailableError`: cannot connect, runtime not running
  - `AuthenticationError`: missing or invalid API key
  - `RateLimitError`: includes retry-after if known
  - `ContextLengthError`: input too large for the model
  - `StructuredOutputError`: no valid JSON after retries

Transient errors (connection, rate limit) are retried with backoff inside the provider.

## Provider adapters

See [ADR 0003](../decisions/0003-openai-compatible-adapter-first.md).

| Adapter | Covers |
|---|---|
| `openai_compat` | Ollama (`/v1`), llama.cpp server, LM Studio, vLLM, OpenAI, and other OpenAI-compatible APIs |
| `anthropic` | Anthropic's API |

More native adapters can be added later if a provider's OpenAI-compatible mode is missing features.

## Provider strings

```
ollama:qwen3:8b                               # local Ollama at default host
openai:<model>                                # OPENAI_API_KEY
anthropic:<model>                             # ANTHROPIC_API_KEY
openai-compat:http://gpu-box:8000/v1#<model>  # any compatible endpoint
```

- `spec.py` parses these into a provider config.
- API keys come from environment variables or the OS keychain. They are never stored in config files.
- `is_local` is true for `ollama:` and for `openai-compat:` endpoints on localhost; false otherwise. A config flag can mark a self-hosted endpoint as trusted.

## Local runtime management (`docveil_llm.local`)

Installed with the `[local]` extra. Only imported when local mode is used.

### `ensure_local()`

1. **Probe hardware** (`hardware.py`): total and available RAM, CPU, GPU vendor and VRAM, Apple Silicon unified memory, free disk space.
2. **Choose a tier** (`selector.py`) from `registry.toml`, unless the user overrides the model.
3. **Ensure the runtime** (`runtimes/ollama.py`): check that Ollama is installed; start `ollama serve` if it is not running; if it is not installed, raise an error with install instructions.
4. **Ensure the model**: pull it if missing, reporting progress through a callback. The caller (CLI) is responsible for asking the user before a large download.
5. Return a ready `LLMProvider`.

### Hardware probe

| Platform | Source |
|---|---|
| RAM, CPU, disk | psutil |
| NVIDIA GPU | pynvml |
| Apple Silicon | `sysctl` (chip, unified memory); treat ~70% of unified memory as usable |
| AMD GPU | `rocm-smi` if present (later) |

Output is a `HardwareProfile` pydantic model, also shown by `docveil doctor`.

### Model registry

Tiers are data, not code, so models can be swapped as better ones are released.

```toml
[[tier]]
name = "minimal"      # < 8 GB, no GPU
min_mem_gb = 0
llm = "none"          # core runs rules + NER only

[[tier]]
name = "small"        # 8–16 GB
min_mem_gb = 8
llm = "qwen3:4b"

[[tier]]
name = "medium"       # 16–32 GB or 8 GB+ VRAM
min_mem_gb = 16
llm = "qwen3:8b"

[[tier]]
name = "large"        # 32 GB+ or 16 GB+ VRAM
min_mem_gb = 32
llm = "qwen3:14b"
```

Model choices are placeholders until the evaluation set (see [roadmap](../roadmap.md)) has compared candidates, including on non-English text.

### Runtimes

| Runtime | Status |
|---|---|
| Ollama | First. Handles GPU detection, model storage and serving. |
| llama.cpp (`llama-server` + GGUF from Hugging Face) | Later, so the tool works without Ollama installed. |

## Testing

- Unit tests with mocked HTTP (respx) for each adapter, including error mapping and `complete_json` fallback.
- Hardware probe tests with faked system info.
- Optional integration tests against a real Ollama, skipped when it is not running.

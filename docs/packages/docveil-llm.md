# docveil-llm

The LLM access layer. Gives the rest of the project one interface to any model, local or remote, and optionally prepares a local model on the user's machine.

It must not contain anything specific to anonymization.

## Public API

```python
from docveil_llm import get_provider, LLMProvider, Capabilities, Message

provider = get_provider("lmstudio:<model>")
answer = await provider.complete_json(messages, schema=MySchema)

# local mode (requires docveil-llm[local])
from docveil_llm.local import ensure_local
provider = await ensure_local(on_progress=callback)
```

## Provider interface

```python
class Capabilities(BaseModel):
    provider: str                 # "llamacpp", "lmstudio", "openai", "anthropic", …
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

See [requirements](../requirements.md#llm-access).

| Adapter | Covers |
|---|---|
| `openai_compat` | llama.cpp server, MLX server, LM Studio, Ollama (`/v1`), vLLM, OpenAI, and other OpenAI-compatible APIs |
| `anthropic` | Anthropic's API |

More native adapters can be added later if a provider's OpenAI-compatible mode is missing features.

## Provider strings

```
lmstudio:<model>                              # LM Studio at localhost:1234
llamacpp[:<model>]                            # llama-server at localhost:8080
ollama:<model>                                # Ollama at localhost:11434 (/v1)
openai:<model>                                # OPENAI_API_KEY
anthropic:<model>                             # ANTHROPIC_API_KEY
openai-compat:http://gpu-box:8000/v1#<model>  # any compatible endpoint
```

The default (no `--provider`) is the managed local runtime from `ensure_local()`; see below.

- `spec.py` parses these into a provider config. The local shortcuts are `openai_compat` with a default URL.
- API keys come from environment variables or the OS keychain. They are never stored in config files.
- `is_local` is true for the local shortcuts and for `openai-compat:` endpoints on localhost; false otherwise. A config flag can mark a self-hosted endpoint as trusted.

## Local runtime management (`docveil_llm.local`)

Installed with the `[local]` extra. Only imported when local mode is used.

### `ensure_local()`

See [requirements](../requirements.md#local-runtimes). docveil uses whatever local runtime it can, and starts it itself when possible.

1. **Use a running server** (`runtimes/detect.py`): if LM Studio, `llama-server` or Ollama answers on its default localhost port, return a provider for it.
2. **Probe hardware** (`hardware.py`): total and available RAM, CPU, Apple Silicon unified memory, GPU, free disk space.
3. **Choose a tier** (`selector.py`) from `registry.toml`, unless the user overrides the model.
4. **Pick a runtime**: the first available of llama.cpp, MLX, LM Studio and Ollama (order configurable). If none is available, raise an error with the options. docveil never installs system software itself.
5. **Ensure the model** in that runtime's store, reporting progress through a callback. The caller (CLI) is responsible for asking the user before a large download.
6. **Start the runtime** (`runtimes/<name>.py`) and wait until it is healthy.
7. Return a ready `LLMProvider`. Closing it stops anything docveil started as a child process.

### Hardware probe

macOS on Apple Silicon comes first; Linux and NVIDIA come later.

| Platform | Source |
|---|---|
| RAM, CPU, disk | psutil |
| Apple Silicon | `sysctl` (chip, unified memory); treat ~70% of unified memory as usable |
| NVIDIA GPU | pynvml (later) |
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
gguf = "<hf-repo>/<file>.gguf"
mlx = "<hf-repo>"
lmstudio = "<model-key>"
ollama = "<model:tag>"

[[tier]]
name = "medium"       # 16–32 GB
min_mem_gb = 16
gguf = "<hf-repo>/<file>.gguf"
mlx = "<hf-repo>"
lmstudio = "<model-key>"
ollama = "<model:tag>"

[[tier]]
name = "large"        # 32 GB+
min_mem_gb = 32
gguf = "<hf-repo>/<file>.gguf"
mlx = "<hf-repo>"
lmstudio = "<model-key>"
ollama = "<model:tag>"
```

Model choices are left open until the evaluation set (see [roadmap](../roadmap.md)) has compared candidates on English and Swedish text.

### Runtimes

Tried in this order when no server is already running:

| Runtime | Available when | Started with | Models from |
|---|---|---|---|
| llama.cpp | `llama-server` on `PATH` | child process | Hugging Face (GGUF) |
| MLX (Apple Silicon) | `[mlx]` extra installed | `mlx_lm.server` child process | Hugging Face (MLX) |
| LM Studio | `lms` CLI found | `lms server start` + `lms load` | `lms get` |
| Ollama | `ollama` CLI found | `ollama serve` | `ollama pull` |

## Testing

- Unit tests with mocked HTTP (respx) for each adapter, including error mapping and `complete_json` fallback.
- Hardware probe tests with faked system info.
- Runtime tests with a fake server process.
- Optional integration tests against a real LM Studio or `llama-server`, skipped when none is running.

# Architecture

## Packages

The project is one GitHub repo containing three Python packages in a [uv workspace](https://docs.astral.sh/uv/concepts/projects/workspaces/). See [ADR 0002](decisions/0002-three-package-workspace.md).

| PyPI package | Import name | Responsibility |
|---|---|---|
| `docveil` | `docveil_cli` | The CLI. This is what end users install (`uv tool install docveil`). |
| `docveil-core` | `docveil_core` | All anonymization logic: documents, detection, merging, replacement, mappings. |
| `docveil-llm` | `docveil_llm` | Talking to LLMs (any provider), plus optional local runtime management (`docveil-llm[local]`). |

A fourth package, `docveil-server`, is planned later for hosting.

## Dependency rules

```
        ┌──────────────┐        ┌──────────────────┐
        │  docveil     │        │  docveil-server  │   (later)
        │  (CLI)       │        │  (HTTP API)      │
        └──────┬───────┘        └────────┬─────────┘
               │  composition roots       │
       ┌───────┴──────────┬───────────────┘
       ▼                  ▼
┌──────────────┐   ┌───────────────────────────────┐
│ docveil-core │──▶│ docveil-llm                   │
│              │   │  ├─ providers  (always)       │
│              │   │  └─ local      (extra [local])│
└──────────────┘   └───────────────────────────────┘
```

1. **`docveil-llm` knows nothing about anonymization.** No entity types, no anonymization prompts. It could be reused in an unrelated project.
2. **`docveil-core` depends only on the `LLMProvider` protocol** from `docveil-llm`. It never probes hardware, installs runtimes or reads API keys. It is handed a ready provider.
3. **The CLI and the future server are the only composition roots.** They choose and prepare a provider, load config, and pass both to the core.
4. **`docveil-core` does no terminal I/O.** No printing, no prompts. It reports progress through events and supports review through data (`analyze` → decisions → `apply`).
5. **`docveil_llm.local` is only imported when local mode is used**, so hosted deployments do not need its dependencies (`psutil`, `pynvml`, …).

## Two jobs in the LLM layer

| Job | Where | Needed when |
|---|---|---|
| Talk to a model (prompt in, text/JSON out) | `docveil_llm.providers` | Always |
| Run a model locally (probe hardware, pick model, install/start runtime, download model) | `docveil_llm.local` | Only for local mode on the user's machine |

A hosted deployment connects to an existing endpoint (an API or a shared GPU server) and never starts processes, which is why these jobs are separated.

## Repo layout

```
docveil/                              # repo root
├── pyproject.toml                    # [tool.uv.workspace] members = ["packages/*"]
├── uv.lock
├── docs/                             # these planning docs
├── packages/
│   ├── docveil-llm/
│   │   ├── pyproject.toml            # extras: [local]
│   │   └── src/docveil_llm/
│   │       ├── __init__.py           # get_provider("ollama:qwen3:8b")
│   │       ├── types.py              # Message, Capabilities, errors
│   │       ├── provider.py           # LLMProvider protocol
│   │       ├── spec.py               # parse provider strings
│   │       ├── providers/
│   │       │   ├── openai_compat.py  # Ollama, llama.cpp, LM Studio, vLLM, OpenAI, …
│   │       │   └── anthropic.py
│   │       └── local/
│   │           ├── __init__.py       # ensure_local()
│   │           ├── hardware.py
│   │           ├── registry.toml
│   │           ├── selector.py
│   │           └── runtimes/
│   │               ├── ollama.py
│   │               └── llamacpp.py   # later
│   ├── docveil-core/
│   │   ├── pyproject.toml            # extras: [docx], [pdf], [ner]
│   │   └── src/docveil_core/
│   │       ├── __init__.py           # Anonymizer, Config
│   │       ├── pipeline.py
│   │       ├── config.py
│   │       ├── events.py
│   │       ├── entities.py
│   │       ├── detect/               # rules, ner, llm, merge
│   │       ├── documents/            # text, docx, pdf
│   │       └── replace/              # strategies, mapping, MappingStore
│   └── docveil/
│       ├── pyproject.toml            # [project.scripts] docveil = "docveil_cli.main:app"
│       └── src/docveil_cli/
│           ├── main.py
│           ├── wiring.py             # build provider + Anonymizer from options/config
│           └── ui/                   # Rich progress, review table
└── tests/
    ├── integration/                  # cross-package tests
    └── eval/                         # labelled synthetic documents + scorer
```

Each package also has its own `tests/` directory for unit tests.

## Typical flow

```
docveil scrub report.docx
  │
  ├─ CLI: load config, parse options
  ├─ CLI: build provider
  │     ├─ default → docveil_llm.local.ensure_local()
  │     │            probe hardware → choose tier → ensure runtime → ensure model
  │     └─ --provider X → docveil_llm.get_provider(X)  (+ remote check)
  ├─ CLI: Anonymizer(provider, config)
  ├─ core: load document → segments
  ├─ core: chunk → detect (rules, NER, LLM) → merge → assign placeholders
  ├─ CLI: optional review of found entities
  ├─ core: apply decisions → write output document
  └─ CLI: print summary (entities, provider used, output path)
```

## Tech stack

| Area | Choice |
|---|---|
| Python | 3.11+ |
| Packaging / workspace | uv |
| Data models / config | pydantic, pydantic-settings, TOML |
| HTTP | httpx (async) |
| CLI | Typer + Rich |
| Hardware | psutil, pynvml, `sysctl` on macOS |
| Documents | python-docx, PyMuPDF |
| NER | GLiNER (optional extra) |
| Fake names | Faker |
| Tests | pytest, pytest-asyncio, respx (HTTP mocking) |
| Lint / types | ruff, mypy or pyright |

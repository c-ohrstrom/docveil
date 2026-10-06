# docveil – planning docs

docveil finds identifying information (names, organisations, contact details, ID numbers, …) in documents such as reports and articles and replaces it with generic placeholders. It is designed to run fully locally with a local LLM, with remote LLM providers as an explicit opt-in.

These docs describe the planned implementation. They are working documents: update them when a decision changes.

## Contents

| Document | What it covers |
|---|---|
| [overview.md](overview.md) | Goals, non-goals and guiding principles |
| [architecture.md](architecture.md) | Package split, dependency rules, repo layout, how the pieces are wired |
| [packages/docveil-llm.md](packages/docveil-llm.md) | LLM access layer: provider interface and local runtime management |
| [packages/docveil-core.md](packages/docveil-core.md) | Anonymization logic: pipeline, detection, replacement, documents |
| [packages/docveil-cli.md](packages/docveil-cli.md) | The `docveil` command-line tool |
| [privacy.md](privacy.md) | Privacy and security requirements |
| [roadmap.md](roadmap.md) | Milestones and task checklists |
| [open-questions.md](open-questions.md) | Things not yet decided |
| [decisions/](decisions/) | Architecture decision records (ADRs) |

## Decision records

| ADR | Decision |
|---|---|
| [0001](decisions/0001-llm-detects-code-replaces.md) | The LLM only detects; code locates and replaces |
| [0002](decisions/0002-three-package-workspace.md) | Three packages in one uv workspace |
| [0003](decisions/0003-openai-compatible-adapter-first.md) | One OpenAI-compatible adapter covers most providers |
| [0004](decisions/0004-async-core-io-free.md) | Async, I/O-free core that is ready to be hosted |

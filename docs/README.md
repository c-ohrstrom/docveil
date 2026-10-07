# docveil – planning docs

docveil finds identifying information (names, organisations, contact details, ID numbers, …) in documents such as reports and articles and replaces it with generic placeholders. It is designed to run fully locally with a local LLM, with remote LLM providers as an explicit opt-in.

These docs describe the planned implementation. They are working documents: update them when something changes. [requirements.md](requirements.md) is the place to start.

## Contents

| Document | What it covers |
|---|---|
| [overview.md](overview.md) | Goals, non-goals and guiding principles |
| [requirements.md](requirements.md) | What docveil must do, with the reason for each requirement |
| [architecture.md](architecture.md) | Package split, dependency rules, repo layout, how the pieces are wired |
| [packages/docveil-llm.md](packages/docveil-llm.md) | LLM access layer: provider interface and local runtime management |
| [packages/docveil-core.md](packages/docveil-core.md) | Anonymization logic: pipeline, detection, replacement, documents |
| [packages/docveil-cli.md](packages/docveil-cli.md) | The `docveil` command-line tool |
| [privacy.md](privacy.md) | Privacy and security requirements |
| [roadmap.md](roadmap.md) | Milestones and task checklists |
| [open-questions.md](open-questions.md) | Things not yet decided |

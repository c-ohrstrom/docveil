# Privacy and security

docveil handles exactly the data it is meant to protect, so the tool itself must not leak it.

## Local by default

- The default provider is local. No document text leaves the machine unless the user opts in.
- After setup (runtime and model downloaded), local runs make no network requests. A test enforces this by blocking sockets except to localhost.
- No telemetry.

## Remote providers

Remote providers are allowed, but only as an explicit choice:

1. **Explicit opt-in**: `--allow-remote` or `allow_remote = true` in config. Without it, a remote provider is refused.
2. **Clear warning**: state which provider and model will receive the text.
3. **Mask locally first**: run the rules and NER detectors locally, replace what they find, and only send the partly masked text to the remote model. The remote model never sees the easy-to-find identifiers.
4. **Record it**: the summary and result metadata state which provider and model were used.

## Secrets

- API keys come from environment variables or the OS keychain, never from config files or command-line arguments.
- Keys are never logged.

## Mappings

- A mapping file reveals the original data. Saving it is opt-in.
- Mappings should be encrypted at rest (passphrase or keychain-stored key).
- The default output never contains original values.

## Logging

- Logs must not contain document text or entity values by default.
- A debug mode may log them, with a clear warning.

## Document metadata

Author, last-modified-by, comments, tracked changes, file properties and PDF metadata are cleaned as part of processing.

## Limitations to communicate

- Detection is not perfect; output should be reviewed for sensitive use.
- Indirect identification (a unique job title plus a small town) is only partly covered.
- Scanned PDFs are not supported.

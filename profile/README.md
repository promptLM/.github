
# promptLM

**Prompt Lifecycle Management for agentic applications**

promptLM is an open-source framework for teams that want to treat prompts like production assets: design, evaluate, version, release, and consume prompts with confidence.

[Website](https://promptlm.dev) • [Core Framework](https://github.com/promptLM/promptlm-app) • [Client SDKs](https://github.com/promptLM/promptlm-client-sdk) • [All Repositories](https://github.com/orgs/promptLM/repositories)

## Status

`Early alpha` — APIs and workflows are evolving quickly. Feedback and contributions are welcome.

## Why promptLM

When prompts power real features, ad-hoc copy/paste is not enough. You need:
- Versioned prompt specs
- Repeatable evaluation gates
- Safe release workflows
- Retrieval in app runtime through SDKs
- Cost-aware integration testing support

## Core capabilities

- Prompt specs and lifecycle tracking in Git
- Multi-model execution (OpenAI, Anthropic, optional LiteLLM gateway)
- Evaluation and release gating
- CLI, web app, and API workflows
- SDK support for consuming released prompts in applications

## Featured repositories

| Repository | Purpose |
| --- | --- |
| [`promptlm-app`](https://github.com/promptLM/promptlm-app) | Core framework, CLI, web app, and lifecycle workflows |
| [`promptlm-client-sdk`](https://github.com/promptLM/promptlm-client-sdk) | Java, TypeScript, and Python SDK workspace |
| [`promptlm-junit-gitea-artifactory`](https://github.com/promptLM/promptlm-junit-gitea-artifactory) | JUnit 5 + Testcontainers support for Gitea and Artifactory integration tests |


## Quick start

### Install `promptlm-cli` (macOS/Linux)

```bash
curl -fsSL https://raw.githubusercontent.com/promptLM/promptlm-app/main/scripts/install.sh | bash
```

### Install `promptlm-cli` (Windows PowerShell)

```powershell
$script = irm https://raw.githubusercontent.com/promptLM/promptlm-app/main/scripts/install.ps1
& ([scriptblock]::Create($script))
```

### Verify

```bash
promptlm-cli --version
```

## Contributing

- [Contributing Guide](https://github.com/promptLM/promptlm-app/blob/main/CONTRIBUTING.md)
- [Code of Conduct](https://github.com/promptLM/promptlm-app/blob/main/CODE_OF_CONDUCT.md)
- [Issue Tracker](https://github.com/promptLM/promptlm-app/issues)

## License

promptLM is open source under the [Apache-2.0 License](https://github.com/promptLM/promptlm-app/blob/main/LICENSE).
```

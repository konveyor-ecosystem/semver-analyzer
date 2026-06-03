# Build & Run

This directory contains scripts for running migrations, building the container image, and packaging ZIP archives.

## Quick Start (Container)

```bash
# Pull the image
podman pull quay.io/konveyor/patternfly-tools:latest

# Set LLM credentials (GCP Vertex AI is the default)
export GCP_PROJECT_ID=my-project
export GCP_LOCATION=us-east5

# Run a migration
./run_container.sh --migrate /path/to/your-app
```

## Guides

| Topic | Link |
|-------|------|
| Migrate an app (container or ZIP) | [Running the Pipeline](../docs/running-the-pipeline.md) |
| Regenerate rules for a different version | [Recreating Rules](../docs/recreating-rules.md) |
| Browse pre-generated rulesets | [Generated Rules](../docs/generated-rules.md) |

## Scripts

| Script | Purpose |
|--------|---------|
| `run_container.sh` | Run migration in a container (mount or bake mode) |
| `run.sh` | Run migration directly on the host |
| `eval.sh` | Compare a migration branch against pf-codemods baseline |
| `build.sh` | Build all tools from source and package a ZIP archive |
| `Containerfile` | Multi-stage container build (Go + Rust + 7 rule generation stages) |

Run `run_container.sh`, `run.sh`, or `eval.sh` with `--help` for CLI options. Full CLI reference tables and container build args are in [Running the Pipeline](../docs/running-the-pipeline.md) and [Recreating Rules](../docs/recreating-rules.md).

## LLM Providers

The default provider is GCP Vertex AI. Override with `GOOSE_PROVIDER` and the corresponding key.

| Provider | Variables |
|----------|-----------|
| GCP Vertex AI (default) | `GCP_PROJECT_ID`, `GCP_LOCATION`, `~/.config/gcloud/` credentials |
| OpenAI | `GOOSE_PROVIDER=openai`, `OPENAI_API_KEY` |
| Google Gemini | `GOOSE_PROVIDER=google`, `GOOGLE_API_KEY` |
| Anthropic | `GOOSE_PROVIDER=anthropic`, `ANTHROPIC_API_KEY` |

Set `GOOSE_MODEL` to override the model (default: `claude-opus-4-6`).

## Source Repositories

| Repo | Purpose |
|------|---------|
| [konveyor/kantra](https://github.com/konveyor/kantra) | Static analysis CLI |
| [konveyor-ecosystem/semver-analyzer](https://github.com/konveyor-ecosystem/semver-analyzer) | Breaking change detection |
| [konveyor-ecosystem/frontend-analyzer-provider](https://github.com/konveyor-ecosystem/frontend-analyzer-provider) | Frontend analysis provider |
| [konveyor-ecosystem/fix-engine](https://github.com/konveyor-ecosystem/fix-engine) | Fix engine CLI |

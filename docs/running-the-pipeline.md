# Running the Migration Pipeline

The migration pipeline takes a PatternFly 5 application and migrates it to PatternFly 6 in two phases:

1. **Phase 1 (automated):** Pre-generated Konveyor rules are applied via [kantra](https://github.com/konveyor/kantra) static analysis, then [fix-engine](https://github.com/konveyor-ecosystem/fix-engine) applies pattern-based and LLM-assisted fixes.
2. **Phase 2 (AI agent):** An AI agent (Goose, Claude, or OpenCode) fixes remaining build errors, type issues, and test failures.

Rules for 7 libraries are pre-generated and bundled. See [Generated Rules](generated-rules.md) for what's included.

## Scripts Overview

There are three scripts in `build/`:

| Script | What it does |
|--------|-------------|
| `run.sh` | The migration engine. Runs Phase 1 + Phase 2 directly on the host. Also supports `--generate-rules` mode. |
| `eval.sh` | Compares a migration branch against a pf-codemods baseline. Produces an HTML report. |
| `run_container.sh` | Wrapper that runs `run.sh` inside a container. Handles image pulling, credential mounting, and log syncing. Can also run `eval.sh` inside the container via `--enable-eval` or `--eval-only`. |

If you're using the container approach, you only interact with `run_container.sh`. If you're using the ZIP archive, you use `run.sh` and `eval.sh` directly.

## Container (Recommended)

```bash
# Pull the image
podman pull quay.io/konveyor/patternfly-tools:latest

# Set LLM credentials (GCP Vertex AI is the default)
export GCP_PROJECT_ID=my-project
export GCP_LOCATION=us-east5

# Run the migration
cd build/
./run_container.sh --migrate /path/to/your-app
```

Results are applied directly to your app directory. Logs go to `.pf-migration-logs/`.

### Mount vs Bake Mode

**Mount mode** (default) mounts the app directory into the container. Changes appear in real-time on the host.

**Bake mode** (`--bake`) copies the app into a temporary image, runs migration inside it, then syncs results back. Use this when mount performance is slow (e.g., Docker on Mac).

### LLM Providers

The default provider is GCP Vertex AI. Override with `GOOSE_PROVIDER` and the corresponding key.

| Provider | Variables |
|----------|-----------|
| GCP Vertex AI (default) | `GCP_PROJECT_ID`, `GCP_LOCATION`, `~/.config/gcloud/` credentials |
| OpenAI | `GOOSE_PROVIDER=openai`, `OPENAI_API_KEY` |
| Google Gemini | `GOOSE_PROVIDER=google`, `GOOGLE_API_KEY` |
| Anthropic | `GOOSE_PROVIDER=anthropic`, `ANTHROPIC_API_KEY` |

GCP Vertex AI requires `~/.config/gcloud/application_default_credentials.json`. When using the container, `run_container.sh` automatically mounts your host's `~/.config/gcloud/` directory into the container if it exists. Generate credentials with `gcloud auth application-default login`.

## ZIP Archive (Fallback)

If you can't use containers, download the ZIP archive and run directly on the host.

**Prerequisites:** Goose CLI (or claude/opencode), yq or python3, git, unbuffer.

```bash
# Extract and run
unzip patternfly-tools-*.zip
cd patternfly-tools/
./run.sh --migrate /path/to/your-app
```

## Example: quipucords-ui

```bash
git clone https://github.com/quipucords/quipucords-ui.git
cd quipucords-ui
git checkout 2.1.0

cd /path/to/semver-analyzer/build
./run_container.sh --migrate /path/to/quipucords-ui --base-branch 2.1.0
```

For prior run results on quipucords-ui, see [patternfly6-migration-bench](https://github.com/jwmatthews/patternfly6-migration-bench/blob/main/results/). To evaluate your own run, see [Evaluation](#evaluation) below.

## Example: Console Plugins

Two OpenShift console plugins that demonstrate the pipeline on real-world apps:

- **[forklift-console-plugin](https://github.com/kubev2v/forklift-console-plugin)** — Forklift migration toolkit UI
- **[kubevirt-plugin](https://github.com/kubevirt-ui/kubevirt-plugin)** — KubeVirt virtualization UI

Both achieved Deps/Build/Tests pass in [jmontleon/semver-comparison](https://github.com/jmontleon/semver-comparison#results-by-repository).

```bash
git clone https://github.com/kubev2v/forklift-console-plugin.git
./run_container.sh --migrate /path/to/forklift-console-plugin
```

To verify the migration, build and test the app, then deploy on a kind cluster with OpenShift console. See [semver-comparison](https://github.com/jmontleon/semver-comparison) for deployment details.

## Evaluation

Evaluation compares a migration branch against a [pf-codemods](https://github.com/patternfly/pf-codemods) baseline and generates an HTML report.

### Container

```bash
# Run evaluation after migration
./run_container.sh --enable-eval --migrate /path/to/app

# Or evaluate an existing migration branch (skip migration)
./run_container.sh --eval-only semver/goose/042926-1043 --migrate /path/to/app
```

### Local (ZIP)

```bash
./eval.sh --migrate /path/to/app --branch semver/goose/042926-1043
```

The evaluation:
1. Creates a `pf-codemods-MMDDYY-HHMM` branch from the base, runs `npx @patternfly/pf-codemods@latest --v6 --fix`
2. Runs the evaluation agent comparing base → pf-codemods → migration branch
3. Generates `pf-migration-comparison-report.html` in the logs directory

## Logs

Logs are saved to `.pf-migration-logs/<timestamp>/` (or `--log-dir` if specified).

| File | Contents |
|------|----------|
| `kantra.log` | Static analysis output |
| `provider.log` | Frontend analyzer provider |
| `fix-pattern.log` | Pattern-based fix output |
| `fix-llm.log` | LLM-assisted fix output |
| `fix-debug/` | Per-file fix-engine debug logs |
| `agent-goose.log` | AI agent transcript |
| `eval-agent.log` | Evaluation agent transcript (if `--enable-eval`) |
| `pf-migration-comparison-report.html` | Evaluation report (if `--enable-eval`) |

## CLI Reference

### `run_container.sh`

| Option | Default | Description |
|--------|---------|-------------|
| `--migrate <PATH>` | *(required)* | Path to the application to migrate |
| `--bake` | off | Bake app into image instead of mounting |
| `--goose-config <PATH>` | baked default | Override goose config directory |
| `--image <NAME>` | `quay.io/konveyor/patternfly-tools:latest` | Container image |
| `--keep` | off | Keep container after completion (for debugging) |
| `--no-memory` | off | Disable memory extension and skip memory volume mount |
| `--log-dir <PATH>` | `.pf-migration-logs/` | Directory to sync logs to |
| `--enable-eval` | off | Run evaluation after migration |
| `--eval-only <BRANCH>` | — | Evaluate existing branch (skips migration) |
| `--agent <NAME>` | `goose` | Agent: goose, claude, opencode |
| `--base-branch <NAME>` | `main` | Branch of the application to migrate |
| `--llm-timeout <SECS>` | `300` | LLM timeout per fix |
| `--non-interactive` | off | Skip all prompts |

### `run.sh`

| Option | Default | Description |
|--------|---------|-------------|
| `--migrate <PATH>` | *(required)* | Project to migrate |
| `--generate-rules` | — | Generate new rules instead of migrating |
| `--agent <NAME>` | `goose` | Agent: goose, claude, opencode |
| `--rules-dir <PATH>` | pre-packaged | Custom rules directory |
| `--base-branch <NAME>` | `main` | Base branch |
| `--skip-agent` | off | Skip AI agent step (Phase 2) |
| `--llm-timeout <SECS>` | `300` | LLM timeout |
| `--non-interactive` | off | Skip prompts |
| `--from / --to <REF>` | — | Git refs for rule generation |
| `--dep-from / --dep-to <REF>` | — | Dependency repo refs for rule generation |
| `--from-node-version <V>` | — | Node version for --from ref |
| `--to-node-version <V>` | — | Node version for --to ref |
| `--from-install-command <C>` | — | Install command for --from ref |
| `--to-install-command <C>` | — | Install command for --to ref |
| `--from-build-command <C>` | — | Build command for --from ref |
| `--to-build-command <C>` | — | Build command for --to ref |

### `eval.sh`

| Option | Default | Description |
|--------|---------|-------------|
| `--migrate <PATH>` | *(required)* | Project path |
| `--branch <BRANCH>` | *(required)* | Migration branch to evaluate |
| `--base-branch <NAME>` | `main` | Base branch |
| `--agent <NAME>` | `goose` | Agent: goose, claude, opencode |
| `--non-interactive` | off | Skip prompts |

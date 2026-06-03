# LLM Integration

semver-analyzer can optionally use an external LLM for analysis tasks. The LLM is invoked as a CLI subprocess — there are no direct API integrations. No API keys are configured in semver-analyzer itself; the external tool handles authentication.

## When LLM Is Used

The default SD (Source-level Diff) pipeline is mostly deterministic but makes **1-2 LLM calls** for rename inference (constant rename patterns and interface renames) and hierarchy inference. These are skipped with `--no-llm`.

The BU (Behavioral) pipeline (`--behavioral`) makes **many more** LLM calls — per changed file, per private function with breaks, etc. This is the heavier LLM use case.

| Pipeline | LLM usage | What it covers |
|----------|-----------|----------------|
| SD (default) | 1-2 calls for rename/hierarchy inference | API diff, composition trees, CSS tokens, React patterns, DOM/ARIA, prop defaults |
| SD + `--no-llm` | Zero calls | Same as above, but skips rename/hierarchy inference |
| BU (`--behavioral`) | Many calls (per-file + per-function) | Test-delta correlation, behavioral inference, call graph propagation |
| BU + `--no-llm` | Zero calls | Test-delta heuristics only, no LLM semantic analysis |

For most library migration analysis, the default SD pipeline (with or without `--no-llm`) is sufficient.

## Installing Goose

[Goose](https://github.com/block/goose) is the recommended LLM CLI tool. It supports multiple LLM providers (Anthropic, OpenAI, Google, Ollama, etc.).

```bash
curl -fsSL https://github.com/block/goose/releases/download/stable/download_cli.sh | bash
goose --version
```

On first run, Goose prompts you to configure a provider and API key. See the [Goose documentation](https://block.github.io/goose/docs/getting-started/installation) for details.

## Usage

```bash
# Default pipeline with LLM rename inference
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  -o report.json

# Default pipeline, no LLM at all
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  --no-llm \
  -o report.json

# Behavioral pipeline with full LLM analysis
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  --behavioral \
  --llm-command "goose run --no-session -q -t" \
  -o report.json

# Behavioral pipeline, no LLM (test-delta heuristics only)
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  --behavioral --no-llm \
  -o report.json
```

### CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--behavioral` | off | Use the BU pipeline instead of the default SD pipeline |
| `--llm-command <cmd>` | `goose run --no-session -q -t` | CLI command to invoke for LLM analysis |
| `--llm-timeout <secs>` | `120` | Timeout per LLM invocation |
| `--llm-all-files` | off | Send all changed files to LLM, not just those with test changes. Requires `--behavioral` |
| `--no-llm` | off | Skip all LLM analysis |

### Goose Flags Explained

| Flag | Purpose |
|------|---------|
| `run` | Run Goose in single-prompt mode |
| `--no-session` | Don't create or use a persistent session |
| `-q` | Quiet mode — suppress interactive UI output |
| `-t` | Treat the final argument as the prompt text (not a file path) |

## CLI Contract for Custom Providers

Any CLI tool can be used as long as it follows this contract:

1. **Prompt as last argument:** The `--llm-command` string is split on whitespace, and the entire prompt is appended as a single final argument
2. **Response on stdout:** The tool must write its response to stdout
3. **Exit code 0:** Non-zero exit codes are treated as errors
4. **JSON in response:** The response must contain valid JSON, either in a fenced code block (`` ```json ... ``` ``) or inline. The parser tries multiple extraction strategies

Given `--llm-command "my-tool --format json"`, the analyzer runs:

```
my-tool --format json "<entire prompt text>"
```

The prompt text can be very long (thousands of characters). The tool must handle large arguments.

### Wrapper Script Example

```bash
#!/bin/bash
# llm-wrapper.sh — adapts stdin-based tools for semver-analyzer
echo "$1" | my-llm-tool --stdin --json
```

```bash
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  --behavioral \
  --llm-command "./llm-wrapper.sh"
```

## What the LLM Analyzes

### SD Pipeline (default, 1-2 calls)

| Task | When it runs |
|------|-------------|
| Constant rename inference | Once, if many constants were removed — detects regex rename patterns |
| Interface rename detection | Once, if many interfaces were removed — finds renames with low lexical similarity |
| Hierarchy inference | Once per package — infers component parent-child relationships |

### BU Pipeline (`--behavioral`, many calls)

| Task | When it runs |
|------|-------------|
| File behavioral analysis | Per changed source file — analyzes diffs for behavioral changes |
| Propagation check | Per private function with breaks — checks if break propagates to public API |
| Constant rename inference | Same as SD |
| Interface rename detection | Same as SD |
| Hierarchy inference | Same as SD |

## Cost Considerations

- **SD pipeline (default):** 1-2 LLM calls total. Negligible cost.
- **BU pipeline (`--behavioral`):** Dozens to hundreds of calls depending on the number of changed files. For large libraries (e.g., PatternFly v5 → v6), expect significant cost.
- Cost depends on the model configured in your LLM tool. More capable models produce better results.
- The `--llm-timeout` flag prevents individual calls from hanging (default: 120s).
- Use `--no-llm` for zero LLM cost on either pipeline.
- The analyzer runs up to 5 concurrent LLM calls.

# semver-analyzer

Deterministic, structured analysis of semantic versioning breaking changes between two git refs. Extracts API surfaces, diffs them, performs source-level analysis, and generates [Konveyor](https://www.konveyor.io/) migration rules with fix strategies.

Supports TypeScript/JavaScript/React and Java projects.

## Quick Start

```bash
cargo build --release

# Analyze breaking changes between two versions
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 \
  --to v2.0.0 \
  -o report.json

# Generate Konveyor migration rules from the report
semver-analyzer konveyor typescript \
  --from-report report.json \
  --output-dir ./rules
```

## Prerequisites

- **Rust** (stable toolchain)
- **Node.js >= 18** and **npm/yarn/pnpm**
- **Git**

## Commands

| Command | Description |
|---------|-------------|
| `analyze typescript` | Full analysis pipeline: extract, diff, source-level analysis |
| `analyze java` | Same pipeline for Java projects (requires `--features java`) |
| `konveyor typescript` | Generate Konveyor YAML rules from a report or inline |
| `konveyor java` | Generate Java migration rules |
| `extract typescript` | Extract API surface at a single ref |
| `diff` | Compare two extracted API surface files |

Run `semver-analyzer <command> --help` for full option details, or see [Getting Started](docs/getting-started.md) for complete CLI reference.

## Migration Pipeline

Pre-generated rules can be consumed by a full migration pipeline that applies fixes automatically. See [Running the Pipeline](docs/running-the-pipeline.md) for walkthroughs using the container image or ZIP archive.

```bash
# Quick example: migrate an app using the container
export GCP_PROJECT_ID=my-project GCP_LOCATION=us-east5
./build/run_container.sh --migrate /path/to/your-app
```

## Development

```bash
cargo test                          # all tests
cargo test -p semver-analyzer-ts    # single crate
cargo build --release               # release build
cargo build --release --features java  # with Java support
```

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](docs/getting-started.md) | How the analyzer works, CLI reference, architecture |
| [Running the Pipeline](docs/running-the-pipeline.md) | Migrate apps with the container or ZIP |
| [Recreating Rules](docs/recreating-rules.md) | Regenerate rules for different versions or libraries |
| [Generated Rules](docs/generated-rules.md) | Browse the 7 pre-generated rulesets |
| [Rule Format](docs/konveyor-rules.md) | Rule anatomy, conditions, fix strategies |
| [Report Format](docs/report-format.md) | JSON report schema reference |
| [LLM Integration](docs/llm-integration.md) | Behavioral pipeline, LLM setup, cost |

## Known Limitations

- **ESM/CJS declaration deduplication**: Projects emitting both ESM and CJS builds will have roughly doubled symbol counts.
- **Language support**: Java requires `--features java` at build time.

## License

See [LICENSE](LICENSE) for details.

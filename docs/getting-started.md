# Getting Started

semver-analyzer detects breaking changes between two git refs of a TypeScript (or Java) library and generates [Konveyor](https://www.konveyor.io/) migration rules with fix strategies. It works by extracting the public API surface at each ref, diffing them, and running source-level analysis to catch changes that don't appear in type signatures alone (CSS tokens, component composition, React patterns).

## Prerequisites

- **Rust** (stable toolchain)
- **Node.js >= 18** with npm/yarn/pnpm
- **Git**

## Build

```bash
git clone https://github.com/konveyor-ecosystem/semver-analyzer.git
cd semver-analyzer
cargo build --release
# Binary: target/release/semver-analyzer
```

For Java support, add `--features java`.

## Basic Usage

Analyze a library and generate migration rules in two steps:

```bash
# 1. Analyze breaking changes
semver-analyzer analyze typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  -o report.json

# 2. Generate Konveyor rules
semver-analyzer konveyor typescript \
  --from-report report.json \
  --output-dir ./rules
```

The `konveyor` command can also run inline (without a pre-existing report):

```bash
semver-analyzer konveyor typescript \
  --repo /path/to/library \
  --from v1.0.0 --to v2.0.0 \
  --output-dir ./rules
```

### Case Study: PatternFly

To analyze [PatternFly React](https://github.com/patternfly/patternfly-react) v5 → v6 and generate migration rules, see [Recreating Rules](recreating-rules.md) for a full walkthrough including CSS dependency analysis.

## CLI Reference

### `analyze typescript`

Full pipeline: extract API surfaces at both refs, diff, source-level analysis.

| Option | Description |
|--------|-------------|
| `--repo <path>` | Path to local git repository |
| `--from <ref>` | Old git ref (tag, branch, SHA) |
| `--to <ref>` | New git ref |
| `-o, --output <path>` | Output file (JSON, defaults to stdout) |
| `--behavioral` | Use behavioral (BU) pipeline instead of default source-level (SD) |
| `--no-llm` | Skip LLM-based analysis (static only) |
| `--llm-command <cmd>` | Command for LLM analysis (see [LLM Integration](llm-integration.md)) |
| `--llm-timeout <secs>` | Timeout per LLM invocation (default: 120) |
| `--llm-all-files` | Send all changed files to LLM, not just those with test changes |
| `--build-command <cmd>` | Custom build command (auto-detected by default) |
| `--dep-repo <path>` | Dependency repo path (e.g., CSS framework) |
| `--dep-from <ref>` | Old ref for dependency repo |
| `--dep-to <ref>` | New ref for dependency repo |
| `--dep-build-command <cmd>` | Build command for dependency repo |

### `konveyor typescript`

Generate Konveyor-compatible YAML rules. Two modes: from a saved report (`--from-report`) or inline (`--repo`).

| Option | Description |
|--------|-------------|
| `--from-report <path>` | Load a pre-existing analysis report |
| `--repo <path>` | Run full analysis then generate rules |
| `--from / --to <ref>` | Git refs (when using `--repo`) |
| `--output-dir <path>` | Output directory for the ruleset |
| `--rename-patterns <path>` | YAML file with regex-based rename patterns |
| `--no-consolidate` | Keep one rule per change (disable merging) |
| `--file-pattern <glob>` | File glob for rules (default: `*.{ts,tsx,js,jsx,mjs,cjs}`) |
| `--ruleset-name <name>` | Ruleset name (default: `semver-breaking-changes`) |

When using `--repo`, all `analyze` options (LLM, build, dep-repo) are also accepted. Run `semver-analyzer konveyor typescript --help` for the full list.

### `extract typescript`

Extract the public API surface at a single ref:

```bash
semver-analyzer extract typescript --repo /path/to/repo --ref v5.0.0 -o surface.json
```

### `diff`

Compare two previously extracted API surface files (language-agnostic):

```bash
semver-analyzer diff --from old-surface.json --to new-surface.json -o changes.json
```

## How It Works

The analyzer combines two pipelines. The **TD** (structural) pipeline always runs. By default, the **SD** (source-level) pipeline runs alongside it. Optionally, the **BU** (behavioral) pipeline can replace SD via `--behavioral`.

### TD — Structural Analysis (always runs)

Extracts and diffs the public API surface:

1. Creates git worktrees for each ref
2. Detects the package manager and installs dependencies
3. Runs `tsc --declaration --emitDeclarationOnly` with monorepo-aware fallbacks
4. Parses `.d.ts` files with [OXC](https://oxc.rs/) and builds the `ApiSurface`
5. Diffs old vs new with 4-phase matching (exact → relocation → fingerprint+LCS rename → unmatched)
6. Detects 30+ change categories: removed exports, signature changes, type changes, generics, enum members, etc.
7. Diffs `package.json` for manifest-level breaks (entry points, exports map, peer deps)

### SD — Source-Level Analysis (default)

Deterministic, AST-based analysis of source code changes. No LLM involved.

- **Component composition trees** — parent-child relationships via 10 evidence signals
- **CSS token analysis** — BEM-structured class/variable tracking, removed classes, renamed variables
- **React API changes** — portal usage, forwardRef/memo wrapping, context dependencies
- **Prop defaults and bindings** — default values, prop-to-CSS-class binding changes
- **DOM structure** — rendered element trees, ARIA attributes, roles
- **Deprecated replacement detection** — relocated components with rendering swap signals

### BU — Behavioral Analysis (opt-in)

Opt-in via `--behavioral`. Uses test-delta heuristics and optional LLM inference:

1. Finds changed source files and extracts function bodies at both refs
2. Cross-references with TD findings to avoid duplicates
3. If test assertions changed → HIGH confidence behavioral break
4. If LLM enabled → sends diffs for semantic analysis
5. Walks up the call graph for private functions with behavioral breaks

## Architecture

```
semver-analyzer (binary)
├── src/main.rs              # CLI entry, report building
├── src/orchestrator.rs      # Pipeline orchestrator (TD+SD or TD+BU)
└── src/cli/mod.rs           # Clap CLI definitions

crates/
├── core/                    # Language-agnostic types and diff engine
│   └── src/
│       ├── traits.rs        # Pluggable language support trait
│       ├── shared.rs        # SharedFindings (DashMap + broadcast)
│       ├── diff/            # 6-phase API surface differ
│       └── types/           # ApiSurface, Symbol, AnalysisReport
├── ts/                      # TypeScript/JavaScript support
│   └── src/
│       ├── extract/         # OXC-based .d.ts API extraction
│       ├── canon/           # 6-rule type canonicalization
│       ├── source_profile/  # Component source profile extraction
│       ├── composition/     # Composition tree builder (v2)
│       ├── sd_pipeline.rs   # Source-level diff pipeline
│       ├── konveyor.rs      # Konveyor rule generation (TD)
│       ├── konveyor_v2.rs   # Konveyor rule generation (SD)
│       └── worktree/        # Git worktree lifecycle, tsc, pkg mgr
├── konveyor-core/           # Shared Konveyor rule types and utilities
└── llm/                     # LLM behavioral analysis
```

The core crate defines a `Language` trait, making the architecture language-pluggable. TypeScript and Java are the current implementations.

## Next Steps

- [Running the Pipeline](running-the-pipeline.md) — migrate an app using pre-generated rules
- [Recreating Rules](recreating-rules.md) — regenerate rules for different versions or libraries
- [Generated Rules](generated-rules.md) — browse the 7 pre-generated rulesets
- [Rule Format](konveyor-rules.md) — rule anatomy, conditions, fix strategies
- [Report Format](report-format.md) — JSON report schema reference
- [LLM Integration](llm-integration.md) — behavioral pipeline setup and cost

# Onboarding: semver-analyzer Ecosystem

Quick-reference for new contributors. Covers what each piece of the ecosystem does, where to find it, and how to run the tool.

## External Resources

### Repositories

| Repo | What it is | Link |
|------|-----------|------|
| **semver-analyzer** (this repo) | Rust CLI that detects breaking changes between two versions of a TypeScript/React library and generates Konveyor migration rules | You're here |
| **pf-tools-builder** | Container image + scripts that package semver-analyzer, kantra, fix-engine, and Goose into a single migration pipeline | [pranavgaikwad/semver-ansible/pf-tools-builder](https://github.com/pranavgaikwad/semver-ansible/tree/main/pf-tools-builder) |
| **patternfly6-migration-bench** | 85-case benchmark suite for evaluating migration tool quality against known PF5-to-PF6 breaking changes. Includes a Claude Code `/evaluate-migration` slash command for scoring | [jwmatthews/patternfly6-migration-bench](https://github.com/jwmatthews/patternfly6-migration-bench) |
| **patternfly-react** | The library being analyzed. PF5 -> PF6 is the primary test case | [patternfly/patternfly-react](https://github.com/patternfly/patternfly-react) |
| **patternfly (CSS)** | CSS dependency repo used for CSS token analysis (`--dep-repo`) | [patternfly/patternfly](https://github.com/patternfly/patternfly) |

### Gists and Examples

| Resource | What it is | Link |
|----------|-----------|------|
| Container build & run | How to build the container image for PF 5.3 -> 6.4 and run it against a target app | [gist: 4b2318aec891bda972c532f330a5d323](https://gist.github.com/jwmatthews/4b2318aec891bda972c532f330a5d323) |
| quipucords-ui walkthrough | End-to-end example running the full pipeline against quipucords-ui (PF 5.3), including rule generation, migration, and test results | [gist: 615f920b8e85f4a15f6b9bbcffd76dd4](https://gist.github.com/jwmatthews/615f920b8e85f4a15f6b9bbcffd76dd4) |
| Last eval results | Evaluation output (scorecard, report, recommendations) from a benchmark run | [results/2026-05-05-semver-goose-050526-1644](https://github.com/jwmatthews/patternfly6-migration-bench/tree/main/results/2026-05-05-semver-goose-050526-1644) |
| Eval recommendations | Prioritized list of issues found in the last eval, with root causes and fixes | [recommendations.md](https://github.com/jwmatthews/patternfly6-migration-bench/blob/main/results/2026-05-05-semver-goose-050526-1644/recommendations.md) |

---

## How It All Fits Together

The full migration pipeline has these stages:

```
┌──────────────────────────────────────────────────────────────────┐
│  1. semver-analyzer analyze    → report.json (breaking changes) │
│  2. semver-analyzer konveyor   → YAML rules + fix-strategies    │
│  3. kantra analyze             → violations in target app       │
│  4. fix-engine-cli fix         → pattern-based fixes            │
│  5. fix-engine-cli fix --llm   → LLM-assisted fixes             │
│  6. goose agent                → remaining build/test errors     │
└──────────────────────────────────────────────────────────────────┘
```

**This repo handles steps 1-2.** Steps 3-6 are handled by the pf-tools-builder container. You can run steps 1-2 standalone without any container.

---

## Running semver-analyzer Locally

### Prerequisites

- **Rust** (stable toolchain) — `rustup` to install
- **Node.js 18+** — required by PatternFly for `tsc` and dependency installation
- **Git**
- **~10 GB disk space** for PatternFly worktrees (2 worktrees x ~2 GB `node_modules` each, plus CSS repo)

### Build

```bash
cargo build --release
# Binary at target/release/semver-analyzer
```

### Quick run: PF 5.4 -> 6.4 (simple case)

This is the easy path. One Node version, no container needed.

```bash
# Option A: Use the convenience script
hack/run-patternfly.sh --release

# Option B: Manual
git clone https://github.com/patternfly/patternfly-react.git /tmp/pf-react
cd /tmp/pf-react && git fetch --tags

semver-analyzer analyze typescript \
  --repo /tmp/pf-react \
  --from v5.4.0 \
  --to v6.4.0 \
  --no-llm \
  -o report.json

# Generate migration rules from the report
semver-analyzer konveyor typescript \
  --from-report report.json \
  --output-dir ./rules
```

The convenience script handles cloning, tag fetching, and cleanup. Use `--keep` to preserve the clone between runs. Use `--konveyor` to also generate rules.

### With CSS dependency analysis

Adds CSS token removal/rename detection. Takes longer but produces more rules.

```bash
git clone https://github.com/patternfly/patternfly.git /tmp/pf-css
cd /tmp/pf-css && git fetch --tags

semver-analyzer analyze typescript \
  --repo /tmp/pf-react \
  --from v5.4.0 \
  --to v6.4.1 \
  --build-command "yarn build:generate && yarn build:esm" \
  --dep-repo /tmp/pf-css \
  --dep-from v5.4.2 \
  --dep-to v6.1.0 \
  --dep-build-command "yarn install && npx gulp buildPatternfly" \
  --no-llm \
  -o report.json
```

### Stress test: PF 5.3 -> 6.4 (multi-node-version)

This scenario requires **node 18** for PF 5.3 and **node 20** for PF 6.4. Use nvm:

```bash
# Install both Node versions with corepack
nvm install 18 && nvm exec 18 bash -c 'export NODE_ENV=development && corepack enable'
nvm install 20 && nvm exec 20 bash -c 'export NODE_ENV=development && corepack enable'
```

Then use the pf-tools-builder `run.sh` script which handles node version switching:

```bash
# Clone the tooling
git clone https://github.com/pranavgaikwad/semver-ansible.git
cd semver-ansible/pf-tools-builder

# Generate rules for 5.3 -> 6.4
time ./run.sh --generate-rules \
    --from v5.3.3 --to v6.4.1 \
    --dep-from v5.3.0 --dep-to v6.4.0 \
    --from-node-version 18 --to-node-version 20 \
    --from-install-command "corepack yarn install" \
    --non-interactive
```

This takes ~4 minutes and produces ~4,000 rules (mostly CSS).

Alternatively, build and use the container (better for reproducibility):

```bash
podman build --no-cache \
  --build-arg PF_REACT_FROM=v5.3.3 \
  --build-arg PF_REACT_TO=v6.4.1 \
  --build-arg PF_DEP_FROM=v5.3.0 \
  --build-arg PF_DEP_TO=v6.4.0 \
  --build-arg FROM_NODE_VERSION=18 \
  --build-arg TO_NODE_VERSION=20 \
  --build-arg FROM_INSTALL_CMD="corepack yarn install" \
  -f Containerfile \
  -t pf-tools-from533 .
```

### Why use a container at all?

You don't need one for basic development. The container exists for:

1. **Multi-node-version runs** (PF 5.3 needs node 18, PF 6.4 needs node 20) — the container manages nvm switching
2. **Full pipeline runs** (steps 1-6) — packages semver-analyzer, kantra, fix-engine, and Goose together
3. **Reproducible eval runs** — ensures identical environments for benchmark comparisons
4. **Sharing with non-Rust users** — no Rust toolchain needed to run migrations

For day-to-day development on semver-analyzer itself, just use `cargo build` and run directly.

---

## Running the Eval Benchmark

The benchmark measures how well the tool's output drives correct migrations.

```bash
# Clone the benchmark
git clone https://github.com/jwmatthews/patternfly6-migration-bench.git
cd patternfly6-migration-bench
npm install
npm run build  # Should build cleanly against PF5

# Run your migration tool against it, push as a branch
# Then score with Claude Code:
claude
# /evaluate-migration run/my-tool-branch
```

The baseline to beat: **pf-codemods gets 66/85 fully correct (77.6%)**.

---

## Key Output Artifacts

| Artifact | What it contains |
|----------|-----------------|
| `report.json` | Full breaking change analysis — structural changes, source-level changes, composition trees, CSS profile diffs |
| `rules/ruleset.yaml` | Ruleset metadata |
| `rules/breaking-changes.yaml` | Migration rules consumable by kantra |
| `fix-guidance/fix-strategies.json` | Machine-readable fix strategies per rule (Rename, RemoveProp, PropValueChange, etc.) |
| `fix-guidance/fix-guidance.yaml` | Human-readable fix descriptions per rule (**known alignment bug** — see below) |

---

## Known Issues

### fix-guidance.yaml alignment bug (P0)

fix-guidance.yaml has a catastrophic alignment bug where ~99.7% of entries have mismatched ruleIDs and descriptions. The ruleID sequence and the symbol/description sequence were independently generated correctly, but merged with the wrong partner. fix-strategies.json is correct and unaffected. See [recommendations.md](https://github.com/jwmatthews/patternfly6-migration-bench/blob/main/results/2026-05-05-semver-goose-050526-1644/recommendations.md) for full details.

### Wrong rename inferences (P1)

4 rules in fix-strategies.json have incorrect rename mappings where the analyzer assumed a removed prop and an added prop were related when they weren't. See `open-issues.md` and the recommendations doc for specifics.

### ESM/CJS symbol duplication

The analyzer picks up `.d.ts` files from both `dist/esm/` and `dist/cjs/`, roughly doubling the symbol and change count. Does not affect rule correctness, but inflates numbers.

---

## Development

```bash
cargo test                          # Full test suite
cargo test -p semver-analyzer-ts    # TypeScript crate only (~650 tests)
cargo build                         # Debug build
cargo build --release               # Release build (much faster at runtime)
```

See `AGENTS.md` for detailed architecture notes, critical invariants, and things to watch out for when modifying specific subsystems.

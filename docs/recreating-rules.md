# Recreating Rules

The container image and ZIP archive ship with pre-generated rules for 7 libraries (see [Generated Rules](generated-rules.md)). You'd regenerate rules if you want a different version range, a different library, or to pick up analyzer improvements.

There are three ways to regenerate, from simplest to most flexible.

## Via `run.sh --generate-rules`

The easiest way for PatternFly. Clones both repos, prompts for version tags (or accepts them as flags), runs the analysis, and generates rules.

```bash
cd build/

# Interactive — prompts for version tags
./run.sh --generate-rules

# Non-interactive — specify everything
./run.sh --generate-rules \
  --from v5.3.3 --to v6.4.1 \
  --dep-from v5.4.0 --dep-to v6.4.0 \
  --from-node-version 18 --to-node-version 20 \
  --from-install-command "corepack yarn install" \
  --non-interactive
```

Output goes to a temp directory (path printed and saved to `.semver_runner`). The generated rules and report are ready to use with `run.sh --migrate --rules-dir`.

## Via `cargo` (Any Library)

For non-PatternFly libraries or full control over the process.

```bash
cargo build --release

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

### PatternFly Walkthrough

PatternFly needs both the React repo and the CSS repo (for CSS token analysis):

```bash
# Clone both repos
git clone https://github.com/patternfly/patternfly-react.git /tmp/pf-react
git clone https://github.com/patternfly/patternfly.git /tmp/pf-css

# Analyze with CSS dependency
semver-analyzer analyze typescript \
  --repo /tmp/pf-react \
  --from v5.3.3 --to v6.4.1 \
  --dep-repo /tmp/pf-css \
  --dep-from v5.4.0 --dep-to v6.4.0 \
  --dep-build-command "export NODE_ENV=development && yarn install && npx gulp buildPatternfly" \
  --build-command "corepack yarn build" \
  --from-node-version 18 --to-node-version 20 \
  --from-install-command "corepack yarn install" \
  --no-llm \
  -o pf-report.json

# Generate rules
semver-analyzer konveyor typescript \
  --from-report pf-report.json \
  --output-dir ./pf-rules
```

See [Getting Started](getting-started.md) for full CLI reference.

## Via Container Build

Rebuild the container image with different versions by overriding build args:

```bash
podman build --format docker --layers=false \
  -t quay.io/konveyor/patternfly-tools:custom \
  --build-arg PF_REACT_FROM=v5.3.3 \
  --build-arg PF_REACT_TO=v6.5.0 \
  -f build/Containerfile build/
```

Use `--format docker` for SHELL directive support. Use `--layers=false` to save disk on large builds.

### Key Build Args

| Arg | Default | Description |
|-----|---------|-------------|
| `PF_REACT_FROM` | `v5.3.3` | PatternFly React source version |
| `PF_REACT_TO` | `v6.4.1` | PatternFly React target version |
| `PF_DEP_FROM` | `v5.4.0` | PatternFly CSS source version |
| `PF_DEP_TO` | `v6.4.0` | PatternFly CSS target version |
| `PF_FROM_NODE_VERSION` | `18` | Node.js version for source ref |
| `PF_TO_NODE_VERSION` | `20` | Node.js version for target ref |
| `SEMVER_REPO` | `konveyor-ecosystem/semver-analyzer` | semver-analyzer repo |
| `SEMVER_BRANCH` | `main` | semver-analyzer branch |
| `FIX_ENGINE_REPO` | `konveyor-ecosystem/fix-engine` | fix-engine repo |
| `FIX_ENGINE_BRANCH` | `main` | fix-engine branch |
| `KANTRA_VERSION` | `v0.9.2-rc.1` | Kantra release |
| `KANTRA_ARCH` | `amd64` | Architecture (`amd64` or `arm64`) |

Each of the 7 library stages has its own ARGs for repo URL, from/to refs, and build commands. See the [Containerfile](../build/Containerfile) for the full list.

### Build Stages

| Stage | Purpose |
|-------|---------|
| go-builder | Build kantra (Go) |
| rust-builder | Build semver-analyzer, frontend-analyzer-provider, fix-engine (Rust) |
| 3a–3g | Generate rules for each of the 7 libraries (run in parallel) |
| runtime | Final image with all tools, rules, and runtime dependencies |

## Output Files

| File | Description |
|------|-------------|
| `ruleset.yaml` | Ruleset metadata |
| `breaking-changes.yaml` | Migration rules (Konveyor format) |
| `fix-strategies.json` | Fix strategies keyed by rule ID |
| `semver_report.json` | Full analysis report |

For rule format details, see [Rule Format](konveyor-rules.md). For report schema, see [Report Format](report-format.md).

## Building the ZIP Archive

`build/build.sh` compiles all tools from source and generates rules for all 7 libraries into a distributable ZIP.

```bash
cd build/
./build.sh
```

Prompts for target platform and kantra release. Requires: Go 1.23+, Rust (via rustup), Node.js 18+ and 20+ (via nvm), git, curl, unzip, python3. All repo URLs and branches are overridable via environment variables (e.g., `SEMVER_REPO_BRANCH=updates ./build.sh`).

## EC2 Build (Multi-Arch)

For building multi-arch container images natively on EC2 (instead of QEMU emulation), an Ansible playbook provisions amd64 + arm64 instances, builds on each, and pushes a manifest list to Quay.io.

```bash
pip install ansible boto3 botocore
ansible-galaxy collection install amazon.aws community.crypto

ansible-playbook build/ec2-build.yml \
  -e @build/vault.yml --vault-password-file ~/.vault_pass \
  -e guid=my-build-01
```

See `build/ec2-build.yml` for full variable reference (AWS region, instance types, registry config, Containerfile ARG overrides).

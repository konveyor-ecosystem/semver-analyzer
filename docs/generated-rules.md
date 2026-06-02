# Generated Rules

The container image and ZIP archive ship with pre-generated Konveyor migration rules for 7 libraries. These rules are produced by semver-analyzer and consumed by [kantra](https://github.com/konveyor/kantra) during the migration pipeline.

## Rulesets

| Library | From | To | Rules | Description |
|---------|------|----|-------|-------------|
| [patternfly](sample-rules/patternfly/) | v5.3.3 | v6.4.1 | 4,152 | PatternFly React — API, CSS, composition, deps |
| [console-sdk](sample-rules/console-sdk/) | release-4.17 | release-4.19 | 241 | OpenShift Console Plugin SDK |
| [react-component-groups](sample-rules/react-component-groups/) | v5.5.3 | v6.4.0 | 77 | PatternFly React Component Groups |
| [dynamic-plugin-sdk](sample-rules/dynamic-plugin-sdk/) | 097b4c9 | c362b0a | 58 | OpenShift Dynamic Plugin SDK |
| [topology](sample-rules/topology/) | v5.4.1 | v6.4.0 | 12 | PatternFly React Topology |
| [react-types](sample-rules/react-types/) | v17 | v18 | 11 | React type definitions |
| [react](sample-rules/react/) | v17.0.2 | v18.3.1 | 8 | React core (peer dep changes) |

## Where Rules Live

| Location | Path |
|----------|------|
| Container image | `/opt/patternfly-tools/rules/<library>/` |
| ZIP archive | `patternfly-tools/rules/<library>/` |
| This repo | [`docs/sample-rules/<library>/`](sample-rules/) |

### Inspecting rules in a built container

```bash
podman create --name pf-inspect quay.io/konveyor/patternfly-tools:latest
podman cp pf-inspect:/opt/patternfly-tools/rules/ ./extracted-rules
podman rm pf-inspect
```

## Rule Files

Each library directory contains:

| File | Description |
|------|-------------|
| `semver_rules/ruleset.yaml` | Ruleset metadata |
| `semver_rules/breaking-changes-api.yaml` | API breaking change rules |
| `semver_rules/breaking-changes-css.yaml` | CSS token change rules (PatternFly only) |
| `semver_rules/breaking-changes-composition.yaml` | Component composition rules |
| `semver_rules/breaking-changes-deps.yaml` | Dependency/manifest change rules |
| `fix-guidance/fix-strategies.json` | Fix strategies keyed by rule ID |
| `fix-guidance/fix-guidance.yaml` | Human-readable fix guidance |

Not every library has all file types — only PatternFly has CSS rules, for example.

For rule format details, see [Rule Format](konveyor-rules.md). To regenerate rules for different versions, see [Recreating Rules](recreating-rules.md).

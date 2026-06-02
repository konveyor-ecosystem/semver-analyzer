# Rule Format Reference

The `konveyor` command generates [Konveyor](https://www.konveyor.io/)-compatible YAML rules from a breaking change analysis. These rules detect migration issues in consumer codebases — applications that depend on the library being analyzed.

Rules are consumed by [kantra](https://github.com/konveyor/kantra) with a frontend-analyzer-provider for AST-level conditions (`frontend.referenced`, `frontend.cssclass`, etc.).

For the pre-generated rulesets shipped in the container/ZIP, see [Generated Rules](generated-rules.md). For `semver-analyzer konveyor` CLI usage, see [Getting Started](getting-started.md#cli-reference).

## Output Structure

```
<library>/
├── semver_rules/
│   ├── ruleset.yaml                    # Ruleset metadata
│   ├── breaking-changes-api.yaml       # API breaking change rules
│   ├── breaking-changes-css.yaml       # CSS token change rules (PF only)
│   ├── breaking-changes-composition.yaml  # Component composition rules
│   └── breaking-changes-deps.yaml      # Dependency/manifest rules
├── fix-guidance/
│   ├── fix-guidance.yaml               # Per-rule migration guidance
│   └── fix-strategies.json             # Machine-readable fix strategies (keyed by rule ID)
└── rule_count                          # Total rule count
```

Not every library has all file types — only PatternFly has CSS rules, for example.

## Rule Anatomy

```yaml
- ruleID: semver-button-variant-type-changed
  labels:
    - "source=semver-analyzer"
    - "change-type=type-changed"
    - "kind=property"
    - "has-codemod=false"
    - "package=@patternfly/react-core"
  effort: 3
  category: mandatory
  description: "Type of 'variant' changed on Button"
  message: |
    The type of `variant` on `Button` changed from
    `'primary' | 'secondary' | 'tertiary'` to
    `'primary' | 'secondary' | 'danger'`.

    The value `tertiary` is no longer valid.
    Review usages and update to a supported value.
  links:
    - url: https://github.com/patternfly/patternfly-react/releases/tag/v6.0.0
      title: Release notes
  when:
    frontend.referenced:
      pattern: "^variant$"
      location: JSX_PROP
      component: "^Button$"
```

| Field | Description |
|-------|-------------|
| `ruleID` | Unique identifier. Key in fix-strategies.json |
| `labels` | Metadata for filtering (see [Labels](#labels)) |
| `effort` | Estimated migration effort (1-10) |
| `category` | `mandatory` (must fix) or `potential` (may need review) |
| `description` | One-line summary |
| `message` | Detailed migration guidance |
| `when` | Condition that triggers the rule |

## Rule Categories

Rules come from three pipelines. TD (Top-Down) always runs. SD (Source-level Diff) runs by default; BU (Bottom-Up / Behavioral) is opt-in via `--behavioral`.

### Structural (TD — Top-Down)

| `change-type` | What it detects |
|---|---|
| `removed` | Symbol removed (prop, constant, interface) |
| `renamed` | Symbol renamed |
| `type-changed` | Type annotation changed |
| `signature-changed` | Interface base class or return type changed |
| `visibility-changed` | Export visibility narrowed |
| `prop-value-change` | Union value removed from prop type |
| `component-removal` | Component fully removed or restructured |
| `new-sibling-component` | New required child component added |
| `css-class` | CSS class prefix renamed |
| `css-variable` | CSS variable prefix/suffix renamed |
| `dependency-update` | npm package needs version bump |
| `manifest` | package.json structural changes |

### Source-Level (SD — Source-level Diff)

| `change-type` | What it detects |
|---|---|
| `conformance` | Incorrect component nesting |
| `composition` | Family member removed |
| `deprecated-migration` | Component moved to/from `/deprecated` |
| `prop-to-child` | Prop moved to new child component |
| `child-to-prop` | Child replaced by parent prop |
| `context-dependency` | React context provider/consumer changed |
| `prop-value-removed` | Specific string value removed |
| `required-prop-added` | New required prop |
| `test-impact` | Testing Library queries affected |
| `css-removal` | Entire CSS component block removed |
| `prop-attribute-override` | Component manages HTML attribute from prop |
| `composition-inversion` | Subcomponent replaced by render prop |

### Behavioral (BU — Bottom-Up, opt-in)

| `change-type` | What it detects |
|---|---|
| `behavioral` | Implementation behavior changed |

## Condition Types

| YAML Key | What it matches |
|----------|----------------|
| `frontend.referenced` | AST-level symbol matching (imports, JSX, types) |
| `frontend.cssclass` | CSS class name patterns |
| `frontend.cssvar` | CSS custom property patterns |
| `frontend.dependency` | package.json dependency name + version |
| `builtin.filecontent` | Regex match in file contents |
| `or: [...]` | Any sub-condition matches |
| `and: [...]` | All sub-conditions match |

### `frontend.referenced` Locations

| Location | Matches |
|----------|---------|
| `IMPORT` | `import { Button } from '...'` |
| `JSX_COMPONENT` | `<Button>` |
| `JSX_PROP` | `<Button variant="primary">` |
| `FUNCTION_CALL` | `getByRole('button')` |
| `TYPE_REFERENCE` | `const x: ButtonProps = ...` |

### `frontend.referenced` Filters

| Filter | Purpose |
|--------|---------|
| `component` | Scope JSX_PROP to a specific component (regex) |
| `parent` | Require JSX_COMPONENT inside this parent |
| `notParent` | Require NOT inside this parent |
| `child` | Require parent to contain this child |
| `notChild` | Require parent to NOT contain non-matching children |
| `requiresChild` | Require parent to contain at least one of these children |
| `value` | Match a specific prop value (regex) |
| `from` | Scope by import source package (regex) |
| `filePattern` | File path filter (regex) |

## Fix Strategies

Strategies in `fix-strategies.json` describe how to resolve each rule. Consumed by [fix-engine](https://github.com/konveyor-ecosystem/fix-engine), which applies automated code transformations based on these strategies during the migration pipeline.

| Strategy | Automated? |
|----------|------------|
| `Rename` | Yes |
| `RemoveProp` | Yes |
| `CssVariablePrefix` | Yes |
| `ImportPathChange` | Yes |
| `EnsureDependency` | Yes |
| `PropValueChange` | Partial |
| `PropTypeChange` | Partial |
| `PropToChild` | Review needed |
| `ChildToProp` | Review needed |
| `CompositionChange` | Review needed |
| `DeprecatedMigration` | Review needed |
| `LlmAssisted` | No |
| `Manual` | No |

## Labels

Every rule carries labels for filtering and categorization.

| Label | Values |
|-------|--------|
| `source` | `semver-analyzer` (always present) |
| `change-type` | See [Rule Categories](#rule-categories) |
| `kind` | `property`, `function`, `method`, `class`, `interface`, `type-alias`, `constant`, `module-export` |
| `has-codemod` | `true` / `false` |
| `package` | npm package name |
| `family` | Component family name |
| `target-component` | Target component for prop migrations |
| `target-package` | Target package for deprecated migrations |
| `impact` | `frontend-testing`, `visual-regression` |
| `ai-generated` | Present if rule came from LLM analysis |

Filter with kantra's `--label-selector`:

```bash
kantra analyze --rules ./rules --label-selector "change-type=css-class || change-type=css-variable"
kantra analyze --rules ./rules --label-selector "has-codemod=true"
```

## Customization

The `semver-analyzer konveyor` command's `--rename-patterns` flag accepts a YAML file with explicit rules that supplement algorithmic detection.

### `rename_patterns` — Regex symbol renames

```yaml
rename_patterns:
  - match: "^c_(.+)_PaddingTop$"
    replace: "c_${1}_PaddingBlockStart"
```

### `prop_renames` — Explicit prop renames

```yaml
prop_renames:
  - old_prop: isOpen
    new_prop: isExpanded
    components: "^(Accordion|Dropdown)$"
    package: "@patternfly/react-core"
```

### `composition_rules` — Component nesting

```yaml
composition_rules:
  - child_pattern: "^Icon$"
    parent: "^Button$"
    category: mandatory
    effort: 2
```

### `value_reviews` — Flag specific prop values

```yaml
value_reviews:
  - prop: variant
    component: "^Button$"
    value: "^tertiary$"
    category: potential
```

### `token_mappings` — Constant renames

```yaml
token_mappings:
  global_success_color_100: "t_global_color_status_success_100"
```

### `css_var_renames` — CSS custom property renames

```yaml
css_var_renames:
  - from: "--pf-v5-global--BackgroundColor--100"
    to: "--pf-t--global--background--color--100"
```

### `missing_imports` — Co-requisite import detection

```yaml
missing_imports:
  - has_pattern: "import.*from '@patternfly/react-core'"
    missing_pattern: "import.*createIcon"
    file_pattern: "\\.(ts|tsx|js|jsx)$"
```

### `component_warnings` — DOM/CSS rendering changes

```yaml
component_warnings:
  - pattern: "^TextArea$"
    package: "@patternfly/react-core"
    category: potential
    description: "TextArea internal DOM restructured"
```

## Rule Consolidation

By default, related rules targeting the same file, kind, and change type are merged into a single rule with `or`-combined conditions. This reduces rule count significantly.

Pass `--no-consolidate` to `semver-analyzer konveyor` to keep one rule per declaration change (useful for debugging or per-change fix strategies).

Consolidation also runs post-processing:
- Redundant prop suppression when a broader component rule covers the same component
- Redundant value suppression when a type-changed rule covers the same prop
- Duplicate condition merging for identical `when` clauses

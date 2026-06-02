# API Diffing Tools by Language

Existing tools that extract and/or diff public API surfaces between library
versions. These could replace large portions of semver-analyzer's custom
diffing logic.

## Language-specific

| Language | Tool | What it does | Link |
|----------|------|-------------|------|
| Java | **japicmp** | Compares two JARs, reports binary/source-level breaking changes. Battle-tested, used by major OSS projects. | https://github.com/siom79/japicmp |
| Java | **revapi** | API compatibility checker with pluggable rules and Maven integration. | https://revapi.org |
| Go | **apidiff** | Official Go team tool. Compares exported API surfaces between module versions. | https://pkg.go.dev/golang.org/x/exp/cmd/apidiff |
| .NET | **ApiCompat** | Microsoft's tool for comparing assembly API surfaces across versions. | https://github.com/dotnet/sdk |
| Python | **griffe** | Extracts Python API, diffs between versions, detects breaking changes. | https://github.com/mkdocstrings/griffe |
| TypeScript | **api-extractor** | Extracts API surface into `.api.json` reports. Diffing is a separate step. | https://api-extractor.com |

## React-specific

| Tool | What it does | Link |
|------|-------------|------|
| **react-docgen-typescript** | Extracts component props, types, required/optional, defaults, JSDoc descriptions into structured JSON. Uses the TS compiler API, so it resolves generics, interfaces, and inheritance. Best candidate for diffing two versions of a React component library. | https://github.com/styleguidist/react-docgen-typescript |
| **react-docgen** | Same idea but uses Babel instead of the TS compiler. Weaker on complex TS types since Babel parses syntax but doesn't run the type checker. | https://github.com/reactjs/react-docgen |
| **react-scanner** | Scans a *consuming* codebase to find which components/props are actually used, how often, and with what values. Complementary — useful for impact analysis (which breaking changes matter for a given project). | https://github.com/moroshko/react-scanner |

## General / tree-diffing

| Tool | What it does | Link |
|------|-------------|------|
| **GumTree** | Language-agnostic AST diffing. Produces edit scripts (insert/delete/update/move). Supports many languages via tree-sitter. | https://github.com/GumTreeDiff/gumtree |
| **Chawathe** | The algorithm GumTree builds on. Computes minimum edit scripts between ordered labeled trees. | Chawathe et al., 1996 |
| **tree-sitter** | Parser generator with uniform AST output. Paired with GumTree, gives you AST diffing for any language with a grammar. | https://tree-sitter.github.io |

## Prototype idea

A simpler semver-analyzer could be:

1. **API extraction**: Use `react-docgen-typescript` for React component props,
   `api-extractor` for general TS APIs, `japicmp` for Java.
2. **Impact analysis**: Use `react-scanner` on the consuming project to find
   which components/props are actually used — skip breaking changes that don't
   affect the project.
3. **Diffing**: Diff the JSON outputs directly (component-level for React,
   GumTree edit scripts for general TS/Java).
4. **Semantic classification**: Thin layer (~2-3k lines) that classifies each
   diff as breaking/non-breaking based on language rules.
5. **Rule generation**: Convert classified diffs to kantra YAML (~2-3k lines).

Estimated total: 5-10k lines vs. current 98k+.

For the PatternFly use case specifically, `react-docgen-typescript` covers the
bulk of breaking changes (props added/removed/changed, type changes,
required/optional flips, default value changes). Hooks, context providers, and
re-export relationships would need supplementing with `api-extractor` or custom
TS compiler scripts.

Note: react-docgen-typescript appears dormant (~11 months no commits as of May
2026). react-docgen (under the official reactjs org) is actively maintained and
has TS support via Babel, though weaker on complex types. react-scanner appears
abandoned. Prefer react-docgen for new work.

## Two layers of breaking changes

**Structural (type-level)**: Prop removed, type changed, component renamed. Code
won't compile. Detectable by diffing API extraction output (api-extractor,
react-docgen). This is ~96% of changes by volume in the PF 5→6 migration.

**Behavioral (render-level)**: Types are identical but runtime output changed.
A component renders `<section>` instead of `<div>`, an `aria-labelledby` is
removed, a CSS class is renamed from `pf-v5-*` to `pf-v6-*`. Breaks tests,
CSS selectors, screen reader behavior. Not visible in type signatures.

None of the API extraction tools (react-docgen, api-extractor) detect behavioral
changes — they only look at the component interface, not the body.

**GumTree can fill this gap.** Point it at the PF5 and PF6 versions of a
component's `.tsx` file and it produces edit scripts: "div replaced with
section", "aria-labelledby attribute removed". A thin classification layer
decides which edits are breaking.

## Composition constraints

semver-analyzer spends ~5,200 lines of Rust reverse-engineering parent-child
nesting constraints (e.g., "DropdownItem must be inside DropdownList inside
Dropdown") from CSS `>` selectors, React context, DOM nesting rules, BEM
naming, and cloneElement patterns.

This is fragile and indirect. The same constraints are usually already available
from simpler sources:

- **TypeScript types** — if `Modal` types `children` as
  `ReactElement<ModalHeaderProps> | ReactElement<ModalBodyProps>`, the structural
  layer already catches changes
- **Migration guides** — PatternFly publishes one per major version, explicitly
  listing nesting changes
- **Storybook examples** — canonical usage, directly diffable between versions
- **Runtime validation** — some PF components console.warn on incorrect nesting

Diffing Storybook examples between versions is a more direct and robust approach
than reverse-engineering constraints from CSS methodology.

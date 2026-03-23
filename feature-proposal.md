# Feature Proposal: File Tree Diagram

## Summary

Add a new `filetree` diagram type that renders directory/file hierarchies as a styled tree with proper connecting lines and file-type icons — similar to what you see in the VS Code file explorer.

## Motivation

Directory tree structures are among the most common diagrams in software documentation. Today, authors use ASCII art like this:

```
my-project/
├── src/
│   ├── components/
│   │   └── MyComponent.js
│   ├── index.css
│   └── index.js
├── package.json
└── README.md
```

This works, but it has real problems:

1. **Fragile formatting** — ASCII trees break if a proportional font is used, if the page is reflowed, or if a CMS strips whitespace. They depend entirely on monospace alignment.
2. **No semantic meaning** — a screen reader or search engine sees a wall of box-drawing characters, not a navigable tree.
3. **No visual richness** — no icons to distinguish folders from files, no colour, no way to highlight a specific node.
4. **Painful to maintain** — adding a file in the middle means re-drawing every `├`, `│`, and `└` connector below it.

A native Mermaid `filetree` diagram solves all of these:

- **Renders with real SVG lines** that always connect properly regardless of font or zoom.
- **Adds file-type and folder icons** automatically, based on extension and trailing `/`.
- **Is semantic** — the tree is described by indentation alone; Mermaid handles the visual output.
- **Is easy to edit** — add or remove a line, adjust indentation, done.

## Syntax

The diagram type keyword is `filetree`. The parser accepts **two input formats** — auto-detected, no configuration needed:

### Format 1: Indentation-based (recommended for new content)

Hierarchy is expressed through indentation (spaces or tabs, consistent within a diagram). Directories are identified by a trailing `/`. Everything else is treated as a file.

````markdown
```mermaid
filetree
    my-project/
        node_modules/
        public/
            favicon.ico
            index.html
            robots.txt
        src/
            components/
                MyComponent.js
            index.css
            index.js
        .gitignore
        package.json
        README.md
```
````

### Format 2: ASCII tree (for existing content)

The parser also accepts the widely-used `tree` command output format using box-drawing characters (`├──`, `└──`, `│`). This means **existing ASCII tree blocks in Markdown files can be converted to rendered diagrams by simply changing the code fence language to `mermaid` and adding the `filetree` keyword.**

````markdown
```mermaid
filetree
my-project/
├── src/
│   ├── components/
│   │   └── MyComponent.js
│   ├── index.css
│   └── index.js
├── package.json
└── README.md
```
````

The parser auto-detects which format is in use by looking for `├`, `└`, or `│` characters. Both formats produce the identical AST and rendered output.

**Why support both?** There are millions of existing Markdown files containing ASCII tree blocks. Supporting this format means authors can upgrade to a rendered diagram with a one-line change (```` ```mermaid ```` + `filetree`) instead of re-writing the entire tree structure. For new content, the indentation format is simpler to write and maintain.

### Highlighting nodes

Individual nodes can be annotated with a CSS class using `:::` (consistent with existing Mermaid class syntax) to draw attention to specific files or folders:

````markdown
```mermaid
filetree
    src/
        components/
            Button.jsx :::highlight
        App.jsx :::highlight
        index.js
```
````

### Icons

Icons are assigned automatically:

| Signal | Icon |
|---|---|
| Trailing `/` | 📁 Folder (closed) |
| `.js`, `.mjs`, `.cjs` | JS icon |
| `.ts`, `.tsx` | TS icon |
| `.py` | Python icon |
| `.json` | JSON/braces icon |
| `.md` | Markdown icon |
| `.html` | HTML icon |
| `.css`, `.scss` | Stylesheet icon |
| `.jpg`, `.png`, `.svg`, `.gif` | Image icon |
| `.yml`, `.yaml` | YAML icon |
| `.sh`, `.bash` | Terminal icon |
| `.lock` | Lock icon |
| `.env` | Key/env icon |
| `.gitignore`, `.eslintrc`, etc. | Gear/config icon |
| *fallback* | Generic file icon |

Authors can override the automatic icon with an explicit `icon()` annotation:

```
filetree
    data/
        model.bin icon(database)
        weights.h5 icon(database)
```

The icon set should be based on a small, self-contained SVG sprite sheet bundled with Mermaid (no external dependencies). The initial set targets the most common file types; it is extensible via Mermaid's theming/configuration API.

### Inline descriptions

Nodes can carry a visible description using `##`. Everything after `##` on the line is rendered next to the label in a lighter, italic font — like a code comment in an IDE:

````markdown
```mermaid
filetree
    my-project/
        src/
            index.js       ## app entry point
            config.ts      ## runtime configuration
            utils/         ## shared helpers
        package.json       ## project manifest
        README.md
```
````

This is distinct from `%%` comments (below), which are **not** rendered at all.

Descriptions can be combined with other annotations:

```
filetree
    src/
        App.tsx :::highlight  ## main application component
        data.bin icon(database)  ## trained model weights
```

### Comments

Line comments with `%%` (existing Mermaid convention) are **invisible** — stripped during parsing:

```
filetree
    src/
        %% Generated files — do not edit
        generated/
        index.js
```

## Rendering

The rendered output should resemble a VS Code–style file explorer:

- **Tree lines**: Thin vertical and horizontal connector lines drawn as SVG `<line>` or `<path>` elements. Each node gets an L-shaped connector to its parent; sibling connectors are joined by a vertical bar. The last child in a group uses a └-shaped terminator.
- **Icons**: Small inline SVGs placed to the left of each label, sized to match the text.
- **Labels**: Rendered as SVG `<text>` elements, left-aligned after the icon.
- **Indentation**: Fixed per-level indent (configurable via theme, default ~20 px).
- **Theming**: Respects Mermaid's existing theme system (`default`, `dark`, `forest`, `neutral`). Folder labels are bold; file labels are normal weight.

### Conceptual SVG structure

```
<g class="filetree">
  <g class="ft-node ft-dir" data-depth="0">
    <line class="ft-connector" ... />
    <use href="#icon-folder" ... />
    <text>my-project/</text>
  </g>
  <g class="ft-node ft-file" data-depth="1">
    <line class="ft-connector" ... />
    <use href="#icon-json" ... />
    <text>package.json</text>
  </g>
  ...
</g>
```

## Configuration options

Exposed through `mermaidConfig.filetree`:

| Option | Type | Default | Description |
|---|---|---|---|
| `indentWidth` | number | `20` | Pixels per indent level |
| `iconSize` | number | `16` | Icon width/height in px |
| `lineHeight` | number | `28` | Vertical spacing between nodes |
| `showIcons` | boolean | `true` | Toggle icon display |
| `connectorColor` | string | theme-dependent | Colour of tree lines |
| `connectorWidth` | number | `1` | Stroke width of tree lines |
| `defaultExpandDepth` | number | `Infinity` | Auto-collapse beyond this depth (future) |

## Accessibility

- The tree is wrapped in `<g role="tree">`, each node in `<g role="treeitem">`.
- Depth is communicated via `aria-level`.
- Labels are readable by screen readers (no decorative characters in the text content).
- The connecting lines are `aria-hidden="true"`.

## Interaction (future / optional)

These are out of scope for the initial implementation but noted for design compatibility:

- **Collapse/expand** folders on click.
- **Click-to-link** — nodes could carry URLs, opening on click.
- **Tooltips** — hover to show metadata (file size, description, etc.).

## Design note: indentation character

A natural question is whether spaces are safe for indentation, given that the motivation section highlights whitespace fragility as a problem with ASCII trees. The important distinction is that Mermaid source text lives inside **fenced code blocks** (` ```mermaid `), which preserve whitespace faithfully across all major Markdown renderers (GitHub, GitLab, Docusaurus, Hugo, Notion, etc.). The whitespace-stripping issue affects *rendered* ASCII art in plain paragraphs — not content inside code fences.

Alternatives considered:

- **Underscore or other visible delimiter** (e.g. `____src/`) — preserves structure even if whitespace is stripped, but hurts readability, feels foreign, and breaks the precedent set by Mermaid's existing `mindmap` diagram which already uses space-based indentation without issues.
- **Explicit depth markers** (e.g. `- src/` with nested `-` lists) — adds syntactic noise and diverges from the intuitive "indent = depth" model that mirrors how file paths naturally nest.

**Recommendation:** Use spaces (consistent with `mindmap`). The fenced code block guarantee makes this safe. If a future edge case arises where a platform mangles whitespace inside code fences, that's a platform bug — and an `_`-based fallback parser mode could be added at that point without breaking the primary syntax.

## Prior art / related

- VS Code file explorer (primary visual inspiration)
- GitHub's repository file tree view
- `tree` CLI command output
- Mermaid `mindmap` diagram (uses indentation-based syntax — direct precedent)
- [D3 tree layout](https://d3js.org/) for SVG tree rendering
- [file-icons/atom](https://github.com/file-icons/atom) for file type → icon mapping

## Who benefits

- **Documentation authors** — READMEs, tutorials, onboarding guides, architecture docs.
- **Educators** — teaching project structure, explaining frameworks.
- **Blog / article writers** — embed in Markdown-native platforms (GitHub, Notion, Docusaurus, etc.).
- **Anyone** who currently pastes an ASCII tree into a code block and wishes it looked better.

## Codebase integration

For Mermaid maintainers evaluating this proposal — here's an honest assessment of what it takes:

### How big is this?

The `filetree` diagram is **small-to-medium in scope** — comparable to the `mindmap` or `kanban` diagrams, both of which use indentation-based parsing. Estimated line counts:

| Component | Est. lines | Notes |
|---|---|---|
| Parser (dual-format) | ~200 | Two mode auto-detect + shared line parser — no grammar generator needed |
| Database/state | ~150 | Flat AST, tree-build logic, icon resolution |
| Renderer | ~200 | Linear layout (no graph algorithms), connector lines, icons, labels |
| Icon maps + sprites | ~250 | Extension/filename maps + inline SVG `<symbol>` definitions |
| Types | ~40 | Node, AST, config interfaces |
| Styles | ~50 | Theme variable mapping |
| Tests | ~300 | Parser (both formats), icon resolution, renderer snapshots |
| Detector + registration | ~30 | Standard lazy-loading pattern |
| **Total** | **~1,200** | |

For comparison: `mindmap` is ~800–1,000 lines, `kanban` is similar, `pie` is ~400–500. The icon sprite data is the largest single chunk but compresses extremely well (it's repetitive SVG path strings).

### Bundle size impact

Since Mermaid uses **lazy loading** (diagrams are loaded only when their detector matches), the `filetree` code is **zero cost** for users who don't use it. For users who do:

- **Estimated addition**: ~8–12 KB minified + gzipped (parser + renderer + icon sprites)
- **No new dependencies** — no graph layout library (unlike flowchart/ELK), no date library (unlike gantt), no D3 module (unlike sankey)
- Could be excluded from `@mermaid-js/tiny` if desired (same as mindmap/architecture)

### Parser technology

The `filetree` syntax is simple enough that a **hand-written line parser** is the right choice — the same approach used internally by mindmap's JISON grammar (which just counts spaces). A JISON or Langium grammar would be overkill for what amounts to "split lines, count indent, strip decorators."

That said, if Mermaid's maintainers prefer Langium for consistency with the direction of newer diagrams, the grammar would be straightforward (~60–80 lines).

### Architectural fit

`filetree` slots cleanly into the existing diagram architecture:

1. **Detector**: regex `/^\s*filetree/` — standard pattern
2. **Lazy loader**: standard dynamic `import()` — no eager code in the main bundle
3. **DiagramDefinition**: `{ parser, db, renderer, styles }` — standard 4-component contract
4. **Theming**: adds a `filetree` block to each theme definition — standard pattern used by all diagrams
5. **No new shared utilities** — doesn't need new layout engines, graph libraries, or coordinate systems

The only novel element is the **icon sprite sheet**, which other diagrams don't have (except `architecture-beta` which has its own icon system). This could be a reusable pattern for future icon-bearing diagrams.

### What a PR looks like

```
packages/mermaid/src/diagrams/filetree/
├── detector.ts                 (~20 lines)
├── filetreeDiagram.ts          (~15 lines)
├── filetreeParser.ts           (~200 lines)  ← dual-format parser
├── filetreeDb.ts               (~150 lines)
├── filetreeRenderer.ts         (~200 lines)
├── filetreeStyles.ts           (~50 lines)
├── filetreeIcons.ts            (~250 lines)  ← maps + SVG sprites
├── filetreeTypes.ts            (~40 lines)
└── __tests__/
    ├── filetreeParser.spec.ts  (~200 lines)
    ├── filetreeRenderer.spec.ts(~60 lines)
    └── filetreeIcons.spec.ts   (~40 lines)
```

Plus:

- One line in `diagram-orchestration.ts` to register the detector
- Theme variable additions in each theme file (~10 lines each × 4 themes)
- Docs page (`docs/syntax/filetree.md`)

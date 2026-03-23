# Technical Specification: `filetree` Diagram Type

> Companion spec for the `filetree` feature proposal.  
> Intended audience: Mermaid contributors implementing or reviewing the PR.

---

## 1. Grammar

The parser accepts a `filetree` block in **two input formats**, auto-detected:

- **Indent format** — hierarchy from whitespace indentation (like `mindmap`)
- **ASCII format** — hierarchy from box-drawing characters (`├──`, `└──`, `│`)

Both formats produce the same AST. Directories are identified by a trailing `/`. An optional `:::className` suffix attaches a CSS class. An optional `icon(name)` annotation overrides the auto-detected icon.

### 1.1 Format detection

After stripping the `filetree` keyword line, comments, and blank lines, the parser scans the remaining lines for box-drawing characters (`│`, `├`, `└`, `─`). If any are found, ASCII format is used; otherwise, indent format.

```typescript
function detectFormat(lines: string[]): 'indent' | 'ascii' {
  const boxChars = /[│├└─]/;
  return lines.some(l => boxChars.test(l)) ? 'ascii' : 'indent';
}
```

### 1.2 PEG-style grammar — indent format (simplified)

```peg
FileTree       ← 'filetree' NEWLINE Node+
Node           ← INDENT* Label Annotation* Description? NEWLINE
Label          ← [^\n:#]+                         # file or dir name (trimmed)
Annotation     ← ClassAnnotation / IconAnnotation
ClassAnnotation← ':::' ClassName
ClassName      ← [a-zA-Z_][a-zA-Z0-9_-]*
IconAnnotation ← 'icon(' IconName ')'
IconName       ← [a-zA-Z0-9_-]+
Description    ← '##' [^\n]*                      # inline description (rendered)
INDENT         ← SPACE SPACE SPACE SPACE / TAB    # 4 spaces or 1 tab = 1 level
NEWLINE        ← '\n'
COMMENT        ← '%%' [^\n]* NEWLINE              # stripped during lexing
```

> **`%%` vs `##`**: `%%` comments are *invisible* — stripped during lexing and never rendered (consistent with all other Mermaid diagrams). `##` descriptions are *visible* — rendered next to the node label in a lighter, italic style, similar to code comments in an IDE.

### 1.3 PEG-style grammar — ASCII format

```peg
FileTreeAscii  ← 'filetree' NEWLINE AsciiNode+
AsciiNode      ← Prefix* Label Annotation* Description? NEWLINE
Prefix         ← '├── ' / '└── ' / '│   ' / '    '   # each prefix = 1 depth level
Label          ← [^\n:#]+
Annotation     ← ClassAnnotation / IconAnnotation
Description    ← '##' [^\n]*
COMMENT        ← '%%' [^\n]* NEWLINE
```

The depth of an ASCII node is determined by counting the number of 4-character prefix segments (`├──`, `└──`, `│`, `    `) before the label. Lines with no prefix are at depth 0 (root).

### 1.4 Indentation rules (indent format)

1. The **first non-empty, non-comment line** after `filetree` establishes the **root indent** (the column at which the root node starts). This allows the author to indent the whole block for aesthetic reasons.
2. A child must be indented exactly **one** indent unit deeper than its parent.
3. Mixed tabs and spaces within a single diagram is a parse error.
4. Blank lines are ignored (they do not close any scope).

### 1.5 Node type detection

| Condition | Node type             |
| ------------------- | ----------- |
| Label ends with `/` | `directory` |
| Otherwise           | `file`      |

The trailing `/` is **not** included in the rendered label; it is consumed by the parser as a type signal.

### 1.6 Example AST

Both of these inputs produce the **identical** AST:

**Indent format:**

```
filetree
    my-project/
        src/
            index.js  ## app entry point
        package.json
```

**ASCII format:**

```
filetree
my-project/
├── src/
│   └── index.js  ## app entry point
└── package.json
```

Parsed AST:

```json
{
  "type": "filetree",
  "nodes": [
    {
      "label": "my-project",
      "nodeType": "directory",
      "depth": 0,
      "description": null,
      "children": [
        {
          "label": "src",
          "nodeType": "directory",
          "depth": 1,
          "description": null,
          "children": [
            {
              "label": "index.js",
              "nodeType": "file",
              "depth": 2,
              "iconId": "js",
              "description": "app entry point",
              "children": []
            }
          ]
        },
        {
          "label": "package.json",
          "nodeType": "file",
          "depth": 1,
          "iconId": "json",
          "description": null,
          "children": []
        }
      ]
    }
  ]
}
```

---

## 2. Icon resolution

Icons are resolved **after** parsing, in a post-parse decoration step.

### 2.1 Resolution order

1. **Explicit `icon()` annotation** — if present, use that icon ID verbatim.
2. **Exact filename match** — lookup against a known-filenames map (e.g. `.gitignore` → `config`, `Dockerfile` → `docker`, `Makefile` → `terminal`).
3. **Extension match** — extract the final extension (e.g. `.spec.ts` → `.ts`) and look up in an extension map.
4. **Directory** — if `nodeType === "directory"`, use `folder`.
5. **Fallback** — `file-generic`.

### 2.2 Icon maps

```typescript
// Known filenames → icon ID
const FILENAME_ICONS: Record<string, string> = {
  '.gitignore': 'git',
  '.eslintrc': 'config',
  '.eslintrc.js': 'config',
  '.eslintrc.json': 'config',
  '.prettierrc': 'config',
  'Dockerfile': 'docker',
  'docker-compose.yml': 'docker',
  'docker-compose.yaml': 'docker',
  'Makefile': 'terminal',
  'LICENSE': 'license',
  'README.md': 'markdown',
  '.env': 'env',
  '.env.local': 'env',
  '.env.production': 'env',
  'tsconfig.json': 'typescript',
  'vite.config.ts': 'config',
  'webpack.config.js': 'config',
  'package-lock.json': 'lock',
  'yarn.lock': 'lock',
  'pnpm-lock.yaml': 'lock',
};

// Extension → icon ID
const EXTENSION_ICONS: Record<string, string> = {
  '.js': 'javascript',
  '.mjs': 'javascript',
  '.cjs': 'javascript',
  '.jsx': 'react',
  '.ts': 'typescript',
  '.tsx': 'react',
  '.py': 'python',
  '.rb': 'ruby',
  '.rs': 'rust',
  '.go': 'go',
  '.java': 'java',
  '.kt': 'kotlin',
  '.cs': 'csharp',
  '.cpp': 'cpp',
  '.c': 'c',
  '.h': 'c',
  '.swift': 'swift',
  '.html': 'html',
  '.htm': 'html',
  '.css': 'css',
  '.scss': 'css',
  '.sass': 'css',
  '.less': 'css',
  '.json': 'json',
  '.yaml': 'yaml',
  '.yml': 'yaml',
  '.toml': 'config',
  '.xml': 'xml',
  '.svg': 'image',
  '.png': 'image',
  '.jpg': 'image',
  '.jpeg': 'image',
  '.gif': 'image',
  '.webp': 'image',
  '.ico': 'image',
  '.md': 'markdown',
  '.mdx': 'markdown',
  '.txt': 'text',
  '.sh': 'terminal',
  '.bash': 'terminal',
  '.zsh': 'terminal',
  '.ps1': 'terminal',
  '.bat': 'terminal',
  '.sql': 'database',
  '.graphql': 'graphql',
  '.gql': 'graphql',
  '.lock': 'lock',
  '.log': 'text',
  '.csv': 'table',
  '.vue': 'vue',
  '.svelte': 'svelte',
};
```

### 2.3 Icon assets

All icons are shipped as a single SVG sprite sheet (`filetreeIcons.svg`) embedded in the Mermaid bundle at build time. Each icon is a `<symbol>` with a `viewBox="0 0 16 16"` and referenced via `<use href="#ft-icon-{id}" />`.

Icons should be simple, single-colour, and theme-aware (they inherit `currentColor` or respond to CSS custom properties). The initial set should cover ~25–30 icon IDs. Additional icons can be added in follow-up PRs.

### 2.4 Icon sourcing

Recommended open-source icon sources (all permissively licensed):

| Source | License | Notes |
|---|---|---|
| [Material Icon Theme](https://github.com/PKief/vscode-material-icon-theme) | MIT | 19M+ installs, extensive file-type coverage, the most popular VS Code icon theme |
| [vscode-icons](https://github.com/vscode-icons/vscode-icons) | MIT | 14M+ installs, wide file-type coverage |
| [Codicons](https://github.com/microsoft/vscode-codicons) | MIT | VS Code's built-in icon set — good for generic file/folder/terminal icons |
| [Lucide](https://lucide.dev/) | ISC | Clean 24x24 stroke icons, scalable; good fallback/general purpose set |
| [Seti UI](https://github.com/jesseweed/seti-ui) | MIT | Origin of VS Code's default file icons |

For the production implementation, the recommended approach is:

1. **Draw simplified 16x16 SVG icons** inspired by these sources (not copied verbatim) to avoid any attribution ambiguity.
2. Keep them single-colour with `currentColor` fills so they adapt to themes.
3. Ship as an embedded SVG `<defs>` block — no external font files or network requests.

A working interactive demo (`demo.html`) accompanies this spec, using hand-drawn inline SVG icons with attribution to the above sources.

---

## 3. Renderer

### 3.1 Layout algorithm

The file tree is rendered top-to-bottom in document order (DFS pre-order traversal of the AST). This is a simple linear layout — no graph algorithms required.

For each node at index `i`:

```
x = marginLeft + (node.depth * indentWidth)
y = marginTop  + (i * lineHeight)
```

Where `indentWidth` and `lineHeight` come from config (defaults: 20 px, 28 px).

### 3.2 Connector lines

Each node draws connectors back to its parent:

1. **Horizontal stub**: A short horizontal line from `(x - halfIndent, y + halfLine)` to `(x - iconPadding, y + halfLine)`.
2. **Vertical bar**: For each ancestor depth where the current node is **not** the last child at that depth, draw a vertical line segment from `(ancestorX + halfIndent, prevSiblingY)` to `(ancestorX + halfIndent, y + halfLine)`.

More precisely, the connector drawing works as follows:

```
For each node (skipping the root):
    nodeX = marginLeft + node.depth * indentWidth
    nodeY = marginTop + nodeIndex * lineHeight

    // Horizontal connector (elbow → node)
    draw line from (nodeX - indentWidth/2, nodeY + lineHeight/2)
                 to (nodeX - 4, nodeY + lineHeight/2)

    // Vertical connector to parent
    parentY = marginTop + parentIndex * lineHeight
    draw line from (nodeX - indentWidth/2, parentY + lineHeight)
                 to (nodeX - indentWidth/2, nodeY + lineHeight/2)

    // If NOT last child: extend the vertical bar down to the next sibling
    // (handled by the next sibling drawing its own vertical segment upward)
```

The last child at each level uses an L-shape (└); other children use a T-shape (├), achieved naturally by the vertical line **not** extending below the last child's Y.

### 3.3 Rendering pipeline

```
1.  Parse        → AST (tree of nodes)
2.  Decorate     → attach iconId to each node (§2 icon resolution)
3.  Flatten      → DFS pre-order traversal → ordered list of { node, index, parentIndex }
4.  Measure      → compute SVG text widths (via temporary off-screen <text>) for bounding box
5.  Draw icons   → place <use> elements referencing the sprite sheet
6.  Draw labels  → place <text> elements
6b. Draw descs   → if node.description exists, place <text class="ft-description"> after label
7.  Draw lines   → place <line>/<path> connector elements
8.  Apply classes → add user-specified :::class CSS classes to node <g> groups
9.  Size SVG     → set viewBox based on max width/height of rendered nodes
```

### 3.4 SVG output structure

```xml
<svg class="filetree" role="tree" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <!-- Embedded icon sprite sheet -->
    <symbol id="ft-icon-folder" viewBox="0 0 16 16">...</symbol>
    <symbol id="ft-icon-javascript" viewBox="0 0 16 16">...</symbol>
    <!-- ... -->
  </defs>

  <!-- Node: my-project (root directory) -->
  <g class="ft-node ft-dir" role="treeitem" aria-level="1" aria-expanded="true">
    <use href="#ft-icon-folder" x="0" y="6" width="16" height="16" />
    <text x="20" y="20" class="ft-label ft-label-dir">my-project</text>
  </g>

  <!-- Node: src (child directory) -->
  <g class="ft-node ft-dir" role="treeitem" aria-level="2" aria-expanded="true">
    <line class="ft-connector" x1="10" y1="28" x2="10" y2="42" />
    <line class="ft-connector" x1="10" y1="42" x2="16" y2="42" />
    <use href="#ft-icon-folder" x="20" y="34" width="16" height="16" />
    <text x="40" y="48" class="ft-label ft-label-dir">src</text>
  </g>

  <!-- Node: index.js (leaf file with inline description) -->
  <g class="ft-node ft-file" role="treeitem" aria-level="3">
    <line class="ft-connector" x1="30" y1="56" x2="30" y2="70" />
    <line class="ft-connector" x1="30" y1="70" x2="36" y2="70" />
    <use href="#ft-icon-javascript" x="40" y="62" width="16" height="16" />
    <text x="60" y="76" class="ft-label ft-label-file">index.js</text>
    <text x="120" y="76" class="ft-description">app entry point</text>
  </g>

  <!-- ... -->
</svg>
```

---

## 4. Theming

The renderer should integrate with Mermaid's existing theme system. Theme variables for `filetree`:

```javascript
// Added to each theme definition
filetree: {
  connectorColor: '#999',      // tree line colour
  connectorWidth: 1,           // tree line stroke width
  dirLabelColor: '#333',       // folder label colour
  fileLabelColor: '#555',      // file label colour
  dirFontWeight: 'bold',       // folder labels are bold
  fileFontWeight: 'normal',    // file labels are normal
  iconColor: '#666',           // default icon fill
  highlightBg: '#fff3cd',      // background for :::highlight nodes
  highlightBorder: '#ffc107',  // border for :::highlight nodes
  descriptionColor: '#888',    // inline ## description colour
  descriptionStyle: 'italic',  // inline ## description font style
  fontSize: '14px',
  fontFamily: 'inherit',
}
```

Dark theme overrides:

```javascript
filetree: {
  connectorColor: '#555',
  dirLabelColor: '#e0e0e0',
  fileLabelColor: '#aaa',
  iconColor: '#888',
  highlightBg: '#3a3000',
  highlightBorder: '#ffc107',
  descriptionColor: '#666',
}
```

---

## 5. File structure in the Mermaid codebase

Following the existing pattern of other diagram types:

```
packages/mermaid/src/diagrams/filetree/
├── filetreeDetector.ts          # diagram detector (registers the diagram)
├── filetreeDiagram.ts           # diagram definition entry point
├── filetreeParser.ts            # indentation-based parser → AST
├── filetreeDb.ts                # diagram database (stores parsed AST)
├── filetreeRenderer.ts          # SVG rendering logic
├── filetreeStyles.ts            # theme-variable-driven CSS generation
├── filetreeIcons.ts             # icon maps + embedded SVG sprite data
├── filetreeTypes.ts             # TypeScript type definitions
└── __tests__/
    ├── filetreeParser.spec.ts   # parser unit tests
    ├── filetreeRenderer.spec.ts # renderer snapshot tests
    └── filetreeIcons.spec.ts    # icon resolution tests
```

### 5.1 Detector

```typescript
// filetreeDetector.ts
import type { DiagramDetector } from '../../diagram-api/types.js';

export const filetreeDetector: DiagramDetector = (txt: string): boolean => {
  return /^\s*filetree/.test(txt);
};
```

### 5.2 Diagram registration

```typescript
// filetreeDiagram.ts
import type { DiagramDefinition } from '../../diagram-api/types.js';
import { filetreeParser } from './filetreeParser.js';
import { filetreeDb } from './filetreeDb.js';
import { filetreeRenderer } from './filetreeRenderer.js';
import { filetreeStyles } from './filetreeStyles.js';

export const diagram: DiagramDefinition = {
  db: filetreeDb,
  renderer: filetreeRenderer,
  parser: filetreeParser,
  styles: filetreeStyles,
};
```

---

## 6. Parser detail

The parser is a hand-written indentation-aware line parser (no generated grammar needed — the syntax is too simple to warrant JISON/PEG tooling).

### 6.1 Format dispatcher

```
function parse(text: string): FileTreeAST {
    lines = text.split('\n')
    remove line 0 ("filetree" keyword)
    remove blank lines and comment lines (%%)

    format = detectFormat(lines)   // scan for │├└─ characters

    if format == 'ascii':
        return parseAscii(lines)
    else:
        return parseIndent(lines)
}
```

### 6.2 Indent-format algorithm

```
function parseIndent(dataLines): FileTreeAST {
    detect indentUnit from first indented line
    stack = []       // stack of { node, depth }
    roots = []

    for each line:
        rawIndent = count leading whitespace
        depth = rawIndent / indentUnit
        label, classes, iconOverride, description = parseLine(line.trim())
        nodeType = label.endsWith('/') ? 'directory' : 'file'
        if nodeType == 'directory': label = label.slice(0, -1)
        node = { label, nodeType, depth, classes, iconOverride, description, children: [] }

        // Pop stack until we find the parent
        while stack.length > 0 AND stack.top().depth >= depth:
            stack.pop()

        if stack.length == 0:
            roots.push(node)
        else:
            stack.top().node.children.push(node)

        stack.push({ node, depth })

    return { type: 'filetree', nodes: roots }
}
```

### 6.3 ASCII-format algorithm

```
function parseAscii(dataLines): FileTreeAST {
    stack = []
    roots = []

    for each line:
        // Strip box-drawing prefixes to determine depth
        // Each prefix segment is 4 chars: "├── ", "└── ", "│   ", "    "
        depth = 0
        pos = 0
        while pos + 4 <= line.length:
            segment = line[pos..pos+4]
            if segment matches /^(├── |└── |│   |    )$/:
                depth++
                pos += 4
            else:
                break

        content = line[pos..].trim()
        if content is empty: continue

        label, classes, iconOverride, description = parseLine(content)
        nodeType = label.endsWith('/') ? 'directory' : 'file'
        if nodeType == 'directory': label = label.slice(0, -1)
        node = { label, nodeType, depth, classes, iconOverride, description, children: [] }

        while stack.length > 0 AND stack.top().depth >= depth:
            stack.pop()

        if stack.length == 0:
            roots.push(node)
        else:
            stack.top().node.children.push(node)

        stack.push({ node, depth })

    return { type: 'filetree', nodes: roots }
}
```

The key insight is that `├──`, `└──`, `│`, and `    ` are all exactly 4 characters wide. Counting them gives the depth. The rest of the line is the label (plus any annotations).

### 6.4 Error handling

| Condition | Error |
|---|---|
| Mixed tabs and spaces | `Parse error: mixed indentation (tabs and spaces) on line N` |
| Indent jumps more than 1 level | `Parse error: unexpected indent on line N (expected depth ≤ X, got Y)` |
| Empty diagram (no nodes) | `Parse error: filetree diagram has no nodes` |
| Invalid :::class name | `Parse error: invalid class name "..." on line N` |
| Invalid icon() name | `Parse error: invalid icon name "..." on line N` |

---

## 7. Testing plan

### 7.1 Unit tests — parser

| Test case | Input | Expected |
|---|---|---|
| Single root directory | `filetree\n    mydir/` | 1 dir node, depth 0 |
| Single file | `filetree\n    file.txt` | 1 file node, depth 0 |
| Nested structure | 3-level example from §1.4 | Correct parent-child relationships |
| Multiple roots | Two top-level entries | 2 root nodes |
| Trailing `/` stripped from label | `src/` | `label: "src"`, `nodeType: "directory"` |
| `:::class` annotation | `file.js :::highlight` | `classes: ["highlight"]` |
| `icon()` override | `data.bin icon(database)` | `iconOverride: "database"` |
| `##` description | `file.js  ## entry point` | `description: "entry point"` |
| `##` with class + icon | `f.bin :::hl icon(db) ## note` | All three parsed correctly |
| Mixed tabs/spaces | Mix in one diagram | Parse error |
| Indent jump | Depth 0 → depth 2 | Parse error |
| Comments stripped | `%% comment` between nodes | Comment not in AST |
| Blank lines ignored | Blank lines between nodes | No effect on tree |
| **ASCII format** | `├── src/\n│   └── index.js\n└── package.json` | Correct tree with depths 0→1→0 |
| ASCII root without prefix | `my-project/\n├── src/` | Root at depth 0, src at depth 1 |
| ASCII with annotations | `├── App.tsx :::highlight ## main` | All parsed correctly |
| ASCII auto-detect | Mix of `├──` in input | Selects ASCII parser |

### 7.2 Unit tests — icon resolution

| Test case | Input | Expected icon ID |
|---|---|---|
| `.js` extension | `index.js` | `javascript` |
| `.ts` extension | `app.ts` | `typescript` |
| `.tsx` extension | `App.tsx` | `react` |
| Known filename | `.gitignore` | `git` |
| Known filename | `Dockerfile` | `docker` |
| Explicit override wins | `data.csv icon(database)` | `database` |
| Directory | `src/` | `folder` |
| Unknown extension | `thing.xyz` | `file-generic` |
| Multi-dot extension | `test.spec.ts` | `typescript` (uses final `.ts`) |

### 7.3 Snapshot tests — renderer

- Basic tree (3 levels, mixed files and dirs) renders correct SVG structure.
- Dark theme applies correct colour overrides.
- `:::highlight` class adds background rect to the node group.
- `showIcons: false` config omits `<use>` elements.
- Large tree (50+ nodes) renders without overlap.
- Accessibility attributes (`role`, `aria-level`) present on all nodes.

### 7.4 Integration / E2E tests

- Render in Mermaid live editor — visual regression screenshots via Playwright/Cypress.
- Verify diagram is detected by `filetreeDetector` and not by other detectors.

---

## 8. Accessibility

| Requirement | Implementation |
|---|---|
| Tree semantics | Root `<g>` has `role="tree"`; each node `<g>` has `role="treeitem"` |
| Depth announcement | Each node has `aria-level="{depth + 1}"` (1-based) |
| Directory state | Directories have `aria-expanded="true"` (always true in v1; becomes dynamic if collapse/expand is added) |
| Decorative lines hidden | All `<line>` connectors have `aria-hidden="true"` |
| Icons hidden | All `<use>` icon elements have `aria-hidden="true"` (the label text is sufficient) |
| Text is real text | Labels are `<text>` elements, not paths — selectable and searchable |

---

## 9. Performance considerations

- **Linear rendering**: No graph layout algorithms. Rendering is O(n) in the number of nodes.
- **No external network requests**: Icons are inlined at build time.
- **Large trees**: For diagrams exceeding ~200 nodes, consider a warning in the console. The SVG will still render correctly but may be visually unwieldy. The `defaultExpandDepth` config (future) would mitigate this.

---

## 10. Rollout plan

### Phase 1 — MVP (this PR)

- Dual-format parser (indent + ASCII, auto-detected)
- Renderer (connectors, labels, icons)
- Icon sprite sheet (~25 icons)
- Theme integration (default + dark)
- `:::class` support
- `icon()` override
- Accessibility attributes
- Unit + snapshot tests

### Phase 2 — Follow-up

- Collapse/expand interactivity (click on directory nodes)
- Link annotations (`url(https://...)` on nodes)
- Tooltip annotations
- Additional icon packs
- `direction TB/LR` for horizontal layout variant
- Mermaid Live Editor sidebar support (syntax highlighting, auto-complete)

---

## 11. Codebase integration analysis

This section addresses how `filetree` fits into the existing Mermaid architecture and what the integration effort looks like.

### 11.1 Architectural fit

`filetree` follows the exact same patterns as existing diagrams:

| Concern | filetree approach | Precedent |
|---|---|---|
| Detection | Regex `/^\s*filetree/` | Same as all 30+ diagrams |
| Loading | Lazy `import()` in loader | Standard since v10 |
| Parser | Hand-written line parser | Simpler than JISON/Langium; similar to mindmap's internal logic |
| Layout | Linear top-to-bottom (O(n)) | No graph library needed (unlike flowchart/ELK) |
| Rendering | Direct SVG construction | Similar to gantt, timeline |
| Theming | `filetree` block in theme defs | Standard pattern |
| Styles | CSS from theme variables | Standard pattern |

### 11.2 Bundle size impact

| Component | Est. minified | Gzipped |
|---|---|---|
| Parser + DB + Renderer | ~6 KB | ~2 KB |
| Icon maps (TS objects) | ~2 KB | ~0.5 KB |
| Icon SVG sprites (~30 icons) | ~8 KB | ~2.5 KB |
| **Total** | **~16 KB** | **~5 KB** |

For context, the full Mermaid bundle is ~200–250 KB gzipped. This adds ~2% — **and only when the diagram is used** (lazy loading means zero cost for non-users).

No new runtime dependencies. No graph layout library. No date parser. No D3 module.

### 11.3 Parser technology recommendation

Mermaid currently uses two parser technologies:

- **JISON** — used by most existing diagrams (flowchart, sequence, mindmap, kanban, etc.)
- **Langium** — used by newer diagrams (pie, packet, architecture, gitGraph, treeView, etc.)

For `filetree`, a **hand-written parser** is recommended because:

1. The syntax is pure "read lines, count indent/prefixes, extract label" — no operator precedence, no expression nesting, no ambiguity.
2. The ASCII format detection and prefix-counting logic don't map naturally to grammar rules.
3. The `mindmap` JISON grammar essentially just calls `yy.addNode(spaceCount, ...)` — the grammar is a thin wrapper around a hand-written algorithm anyway.
4. A hand-written parser is easier to maintain, debug, and test than a generated one for this level of complexity.

If maintainers prefer Langium for consistency with the project direction, the indent-format grammar would be ~60–80 lines. The ASCII format would still need a preprocessing step.

### 11.4 Integration touchpoints

Files outside the `diagrams/filetree/` directory that need changes:

| File | Change | Lines |
|---|---|---|
| `diagram-orchestration.ts` | Import detector, add to `registerLazyLoadedDiagrams()` | ~2 |
| `themes/theme-default.ts` | Add `filetree` theme variables | ~15 |
| `themes/theme-dark.ts` | Add `filetree` dark overrides | ~10 |
| `themes/theme-forest.ts` | Add `filetree` forest overrides | ~10 |
| `themes/theme-neutral.ts` | Add `filetree` neutral overrides | ~10 |
| `defaultConfig.ts` | Add `filetree` config defaults | ~10 |
| `docs/syntax/filetree.md` | Documentation page | ~200 |
| `cypress/integration/rendering/filetree.spec.ts` | E2E visual regression tests | ~80 |

This is a **low-risk, low-coupling** addition. No existing code is modified beyond registration and theming.

### 11.5 Comparison to similar PRs

| Diagram | LoC | Dependencies | Layout complexity |
|---|---|---|---|
| `pie` | ~400–500 | None | Trivial (circle + slices) |
| `mindmap` | ~800–1,000 | cytoscape (layout) | Medium (force-directed) |
| `kanban` | ~800–1,000 | None | Low (columns + cards) |
| **`filetree`** | **~1,200** | **None** | **Low (linear list)** |
| `architecture` | ~1,500+ | cytoscape | High (icon positioning) |
| `flowchart` | ~3,000+ | dagre or ELK | Very high (graph routing) |

`filetree` is in the sweet spot: useful enough to justify the code, simple enough to maintain long-term.

---

## 12. Open questions

1. **Should the root label be rendered or implicit?** — Proposal: render it. If the author wants no visible root, they just list top-level children directly.
2. **Should `node_modules/` be auto-collapsed by default?** — Proposal: no special-casing. The author controls what appears in the diagram.
3. **Icon licensing** — All bundled icons must be MIT/Apache-2.0 compatible. Recommended source: [Lucide](https://lucide.dev/) (ISC license) or custom-drawn.

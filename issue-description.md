# GitHub Issue — Copy/Paste Template

> **Title:** `New diagram type: filetree — directory trees with icons and connector lines`

> Paste everything below the line into the issue body.

---

## Proposal

Mermaid.js should add a **`filetree` diagram type** that renders directory/file hierarchies as styled SVG trees — with proper connector lines, file-type icons, and theme support — because directory trees are among the most common diagrams in software documentation, and the current approach (pasting ASCII `tree` output) is fragile, inaccessible, and impossible to style.

**Key points:**

- **Two input formats** (auto-detected): simple indentation or the familiar `├── / └──` ASCII tree syntax. The ASCII format means existing Markdown files can be upgraded to rendered diagrams with a one-line change (add ` ```mermaid ` + `filetree`).
- **Automatic file-type icons** based on extension and filename, with `icon()` override support.
- **`:::class` annotations** for highlighting specific nodes (consistent with existing Mermaid syntax).
- **Inline descriptions** via `##` — rendered next to labels in a lighter, italic font.
- **Fully themed** — works with default, dark, forest, and neutral themes out of the box.
- **Zero new dependencies** — no graph layout library needed (linear O(n) layout), no external icon fonts.
- **~1,200 lines of code**, lazy-loaded — zero cost for users who don't use it.

### Full write-up & interactive demo

I've prepared a detailed proposal, technical spec, and working proof-of-concept demo:

- **[Interactive Demo](https://notionparallax.github.io/mermaid_files/)** — live editor with real-time rendering, 30+ icons, light/dark toggle
- **[Feature Proposal](https://notionparallax.github.io/mermaid_files/feature-proposal.html)** — full motivation, syntax design, config options, accessibility, codebase integration analysis
- **[Technical Spec](https://notionparallax.github.io/mermaid_files/technical-spec.html)** — PEG grammar, AST structure, icon resolution, renderer algorithm, testing plan, PR file listing

## Example

**Indent format** (recommended for new content):

```
filetree
    my-project/
        src/
            components/
                Button.tsx :::highlight  ## new component
                Header.tsx
            App.tsx :::highlight  ## app root
            index.js  ## entry point
        .gitignore
        package.json  ## project manifest
        README.md
```

**ASCII format** (convert existing `tree` output by adding `filetree`):

```
filetree
my-project/
├── src/
│   ├── components/
│   │   ├── Button.tsx :::highlight  ## new component
│   │   └── Header.tsx
│   ├── App.tsx :::highlight  ## app root
│   └── index.js  ## entry point
├── .gitignore
├── package.json  ## project manifest
└── README.md
```

Both formats produce the same rendered output — an SVG tree with:

- Folder and file-type icons (auto-detected from extension)
- Proper connector lines (vertical bars + L-shaped terminators)
- Highlighted nodes (yellow background via `:::highlight`)
- Inline descriptions (green italic text via `##`)

## Screenshots

**Dark theme:**

<!-- TODO: Add a screenshot of the demo in dark mode -->
<!-- Take a screenshot of http://localhost:8080/demo.html and paste it here -->

**Light theme:**

<!-- TODO: Add a screenshot of the demo in light mode -->
<!-- Toggle to light mode, take a screenshot, paste it here -->

> Screenshots are from the [interactive demo](https://notionparallax.github.io/mermaid-filetree-proposal/demo.html). Try it yourself — the editor updates in real time.

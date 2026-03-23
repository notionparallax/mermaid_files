# GitHub Issue — Copy/Paste Template

> **Title:** `New diagram type: filetree — directory trees with icons and connector lines`

> Paste everything below the line into the issue body.

---

# Proposal

Mermaid.js should add a **`filetree` diagram type** that renders directory/file hierarchies as styled SVG trees — with proper connector lines, file-type icons, and theme support — because directory trees are among the most common diagrams in software documentation, and the current approach (pasting ASCII `tree` output) is fragile, inaccessible, and impossible to style.

Today, authors paste ASCII tree output into code blocks. This is fragile (breaks with proportional fonts or reflowed text), inaccessible (screen readers see a wall of box-drawing characters), visually flat (no icons, no colour, no highlights), and painful to maintain (adding a file means re-drawing every `├`, `│`, and `└` connector).

A native `filetree` diagram solves all of these — rendered with real SVG lines that always connect properly, with automatic file-type icons, CSS class annotations, and full theme support. It follows existing Mermaid conventions (`:::class`, `%%` comments, lazy-loaded module) and ships at ~1,200 lines with zero new dependencies.

- **[Interactive Demo](https://notionparallax.github.io/mermaid_files/)** — live editor with real-time rendering, 30+ icons, light/dark toggle
- **[Feature Proposal](https://notionparallax.github.io/mermaid_files/feature-proposal.html)** — full motivation, syntax design, config, accessibility, codebase integration analysis
- **[Technical Spec](https://notionparallax.github.io/mermaid_files/technical-spec.html)** — PEG grammar, AST, icon resolution, renderer algorithm, testing plan, PR file listing

# Use Cases

- **README files** — show project structure in onboarding docs, contributing guides, and tutorials
- **Architecture documentation** — illustrate module layout, package boundaries, deployment artefacts
- **Blog posts and articles** — explain framework conventions (e.g. Next.js `app/` directory, Django project layout)
- **Code reviews** — highlight new or changed files with `:::highlight` annotations
- **Teaching** — walk students through project scaffolding step by step
- **Upgrading existing docs** — millions of Markdown files contain ASCII `tree` output; supporting the `├──` format means they can be converted to rendered diagrams with a one-line change (add ` ```mermaid ` + `filetree`)

# Screenshots

**Dark theme:**

<!-- TODO: Add a screenshot of the demo in dark mode -->
<!-- Take a screenshot of http://localhost:8080/demo.html and paste it here -->

**Light theme:**

<!-- TODO: Add a screenshot of the demo in light mode -->
<!-- Toggle to light mode, take a screenshot, paste it here -->

> Screenshots are from the [interactive demo](https://notionparallax.github.io/mermaid_files/). Try it yourself — the editor updates in real time.

# Syntax

The diagram keyword is `filetree`. The parser accepts **two input formats**, auto-detected:

### Format 1: Indentation-based (recommended for new content)

Hierarchy is expressed through indentation. Directories end with `/`. Everything else is a file.

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

### Format 2: ASCII tree (for existing content)

The parser also accepts `tree` command output with box-drawing characters (`├──`, `└──`, `│`). Existing ASCII tree blocks can be converted by just wrapping them in ` ```mermaid ` and adding `filetree`:

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

Both formats produce identical rendered output — an SVG tree with:

- **File-type icons** — auto-detected from extension/filename, overridable with `icon(name)`
- **Connector lines** — proper vertical bars + L-shaped terminators
- **`:::class` annotations** — e.g. `:::highlight` for a yellow background (consistent with existing Mermaid class syntax)
- **`##` inline descriptions** — rendered next to labels in lighter italic text (like code comments)
- **`%%` comments** — invisible, stripped during parsing (standard Mermaid convention)
- **Theming** — works with default, dark, forest, and neutral themes

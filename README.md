# Mermaid `filetree` Diagram — Proposal

A proposal to add a new `filetree` diagram type to [Mermaid.js](https://github.com/mermaid-js/mermaid) that renders directory/file hierarchies as styled SVG trees with connecting lines and file-type icons.

## What's here

| Resource | Description |
|---|---|
| [**Interactive Demo**](https://notionparallax.github.io/mermaid_files/) | Live proof-of-concept with dual-format parser, 30+ file-type icons, light/dark themes |
| [**Feature Proposal**](https://notionparallax.github.io/mermaid_files/feature-proposal.html) | GitHub issue–style write-up: motivation, syntax, icons, config, accessibility |
| [**Technical Spec**](https://notionparallax.github.io/mermaid_files/technical-spec.html) | Spec: grammar, AST, icon resolution, renderer algorithm, testing plan, codebase integration |

## Quick preview

Two equivalent input formats — auto-detected:

**Indent format** (recommended for new content):

```
filetree
    my-project/
        src/
            index.js  ## app entry point
        package.json
        README.md
```

**ASCII format** (for converting existing `tree` output):

```
filetree
my-project/
├── src/
│   └── index.js  ## app entry point
├── package.json
└── README.md
```

Both render the same styled SVG tree with icons, connector lines, and theme support.

## GitHub Issue

The condensed issue description for [mermaid-js/mermaid](https://github.com/mermaid-js/mermaid/issues) is in [issue-description.md](issue-description.md).

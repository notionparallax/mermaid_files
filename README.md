# Mermaid `filetree` Diagram — Proposal

A proposal to add a new `filetree` diagram type to [Mermaid.js](https://github.com/mermaid-js/mermaid) that renders directory/file hierarchies as styled SVG trees with connecting lines and file-type icons.

## What's here

| Resource | Description |
|---|---|
| [**Interactive Demo**](https://USERNAME.github.io/mermaid-filetree-proposal/demo.html) | Live proof-of-concept with dual-format parser, 30+ file-type icons, light/dark themes |
| [**Feature Proposal**](https://USERNAME.github.io/mermaid-filetree-proposal/feature-proposal.html) | GitHub issue–style write-up: motivation, syntax, icons, config, accessibility |
| [**Technical Spec**](https://USERNAME.github.io/mermaid-filetree-proposal/technical-spec.html) | PR-ready spec: grammar, AST, icon resolution, renderer algorithm, testing plan, codebase integration |

> **Replace `USERNAME` with your GitHub username** after creating the repo.

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

## Setup

This repo is designed to be served via GitHub Pages:

1. Create a new GitHub repo (e.g. `mermaid-filetree-proposal`)
2. Push this folder to `main`
3. Go to **Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**
4. Update the URLs in this README with your GitHub username

## License

This proposal and demo are released into the public domain. The Mermaid.js project is MIT-licensed.

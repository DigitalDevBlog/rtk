# Understanding RTK

## What This Is

A versioned MkDocs Material site (`learn/`) that explains exactly how RTK (Rust Token Killer) works internally — hook rewriting, command registry, filter pipeline, tracking/analytics — with explanatory diagrams. Lives in a public fork (DigitalDevBlog/rtk) and is published statically to GitHub Pages. Modelled on the "Understanding PAUL" and "Understanding GSD Core" sites.

## Core Value

Someone can read the site and understand precisely how RTK cuts LLM token usage — from the Claude Code hook to the filtered output and savings tracking — well enough to explain or re-implement it.

## Current State

| Attribute | Value |
|-----------|-------|
| Type | Application (documentation site) |
| Version | 0.0.0 |
| Status | Shipped |
| Last Updated | 2026-09-29 |

## Requirements

### Core Deliverables

- MkDocs Material site under `learn/` with nav skeleton, search, dark mode
- Section-by-section explanation of RTK internals, verified against source
- Diagrams-as-code (Mermaid/PlantUML via Kroki) in every section
- Public fork DigitalDevBlog/rtk with GitHub Actions deploy to GitHub Pages
- PAUL-managed workflow (PLAN → APPLY → UNIFY per phase)

### Validated (Shipped)
None yet.

### Active (In Progress)
None yet.

### Planned (Next)
- Foundation, Big picture, Hook & rewrite, Filters, Tracking, Init/install, Build-your-own, Publishing

### Out of Scope
- Changing RTK's Rust code (docs only; upstream stays untouched)
- Translating the site

## Constraints

### Technical Constraints
- Static site only (GitHub Pages); diagrams rendered at build time
- Docs live in `learn/` to avoid clashing with RTK's existing `docs/`
- Claims must be verified against RTK source (src/hooks, src/discover, src/core, src/cmds)

### Business Constraints
- Fork must be public; work on branch `learn-site`, upstream remote kept as rtk-ai/rtk

## Key Decisions

| Decision | Rationale | Date | Status |
|----------|-----------|------|--------|
| Docs dir `learn/` + root mkdocs.yml | Matches gsd-core; RTK already owns docs/ | 2026-09-29 | Active |
| mkdocs-material + mike + kroki plugin | Same stack as paul/gsd-core reference sites | 2026-09-29 | Active |
| Fork to DigitalDevBlog/rtk | Authenticated gh account; user requested public fork | 2026-09-29 | Active |

## Success Metrics

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| mkdocs build --strict | passes | - | ✅ Achieved |
| Sections with diagrams | all | 0 | ✅ Achieved |
| Site live on GitHub Pages | HTTP 200 | - | ✅ Achieved |

## Tech Stack / Tools

| Layer | Technology | Notes |
|-------|------------|-------|
| Site generator | MkDocs Material | as in paul / gsd-core |
| Diagrams | Kroki (Mermaid, PlantUML) | build-time SVG |
| Versioning | mike | gh-pages branch |
| CI/Hosting | GitHub Actions + Pages | |

## Links

| Resource | URL |
|----------|-----|
| Repository | https://github.com/DigitalDevBlog/rtk |
| Upstream | https://github.com/rtk-ai/rtk |

---
*PROJECT.md — Updated when requirements or context change*
*Last updated: 2026-09-29*

# Plan: Site Foundation

**Created:** 2026-09-29
**Status:** Approved (goal-driven, autonomous)

## Objective
mkdocs.yml, requirements.txt, learn/ skeleton, Kroki diagrams verified, strict local build passes

## Acceptance Criteria
- AC-1: Pages exist and every technical claim is traceable to RTK source (src/hooks, src/discover, src/core, src/cmds, src/filters) or observed CLI behaviour
- AC-2: Each page carries at least one diagram
- AC-3: `mkdocs build --strict` passes

## Boundaries
- Do not modify RTK Rust sources; docs live in learn/ only

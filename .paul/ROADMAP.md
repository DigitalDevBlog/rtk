# Roadmap: Understanding RTK

## Overview
A static MkDocs Material site explaining how RTK works: from the Claude Code hook that rewrites commands, through the registry and per-command filters, to tracking and analytics — with diagrams — published on GitHub Pages from a public fork.

## Current Milestone
**v0.1 Initial Release** (v0.1.0)
Status: ✅ Complete (2026-09-29)
Phases: 8 of 8 complete

## Phases

| Phase | Name | Plans | Status | Completed |
|-------|------|-------|--------|-----------|
| 1 | Site Foundation | 1 | ✅ Complete | 2026-09-29 |
| 2 | Big Picture | 1 | ✅ Complete | 2026-09-29 |
| 3 | Hook & Rewrite | 1 | ✅ Complete | 2026-09-29 |
| 4 | Filter Pipeline | 1 | ✅ Complete | 2026-09-29 |
| 5 | Tracking & Analytics | 1 | ✅ Complete | 2026-09-29 |
| 6 | Init & Install | 1 | ✅ Complete | 2026-09-29 |
| 7 | Build Your Own | 1 | ✅ Complete | 2026-09-29 |
| 8 | Publishing | 1 | ✅ Complete | 2026-09-29 |

## Phase Details

### Phase 1: Site Foundation
**Goal:** Building MkDocs site in learn/ with nav skeleton, Kroki diagrams, local build verified.
**Depends on:** Nothing

### Phase 2: Big Picture
**Goal:** End-to-end flow: agent -> hook -> rtk -> command -> filter -> agent, with token-savings rationale.

### Phase 3: Hook & Rewrite
**Goal:** How hooks intercept Bash calls and src/discover (lexer, registry, rules) rewrite them.

### Phase 4: Filter Pipeline
**Goal:** src/cmds per-ecosystem filters, TOML filters, core runner/stream/truncate/tee.

### Phase 5: Tracking & Analytics
**Goal:** SQLite tracking, rtk gain, discover, telemetry.

### Phase 6: Init & Install
**Goal:** rtk init, per-agent integrations, config.

### Phase 7: Build Your Own
**Goal:** Minimal blueprint for a token-saving CLI proxy.

### Phase 8: Publishing
**Goal:** deploy workflow, GitHub Pages live, README link.

---
*Roadmap created: 2026-09-29*

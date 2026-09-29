# Understanding RTK

A guided tour of how **RTK (Rust Token Killer)** works internally. RTK is a single Rust
binary that sits between an AI coding agent and your shell, and shrinks the output of
commands like `git status`, `cargo test` or `ls` *before* the model has to read it.

!!! info "Source of truth"
    Everything here is traced to the RTK source (v0.49.0: `src/hooks`, `src/discover`,
    `src/core`, `src/cmds`, `src/filters`) and its in-repo docs. Where behaviour was
    observed on a live install it says so.

## The whole system in one picture

```kroki-mermaid
flowchart LR
    A["LLM agent<br/>(Claude Code, Cursor, ...)"] -->|"runs: git status"| H["PreToolUse hook"]
    H -->|"rtk rewrite"| R["Rewrite registry<br/>src/discover"]
    R -->|"rtk git status"| A
    A -->|"executes rewritten command"| C["rtk CLI<br/>src/main.rs"]
    C --> F{"Filter?"}
    F -->|"Rust module"| RF["src/cmds/**"]
    F -->|"TOML rule"| TF["src/filters/*.toml"]
    F -->|"none"| PT["Passthrough"]
    RF --> O["Compact output"]
    TF --> O
    PT --> O
    O --> A
    C --> T[("history.db<br/>token tracking")]
```

## What you'll find here

| Section | What it covers |
|---------|----------------|
| [Big Picture](big-picture/index.md) | Why RTK exists and the six-phase life of a command |
| [Hook & Rewrite](hook/index.md) | How a `git status` typed by the agent silently becomes `rtk git status` |
| [Filters](filters/index.md) | The Rust filter modules, the TOML DSL, and how elided output is recovered |
| [Tracking](tracking/index.md) | How savings are measured, stored and reported (`rtk gain`) |
| [Install & Config](install/index.md) | What `rtk init` writes, integrity checks, configuration |
| [Build Your Own](build-your-own/index.md) | A minimal blueprint for a token-saving CLI proxy |

## The five design rules

1. **Fail safe.** If a filter or the hook breaks, the raw command runs unchanged.
2. **Preserve exit codes.** RTK never turns a failing command into a passing one.
3. **Stay fast.** Single-threaded, no async, target under 10 ms of overhead.
4. **Stay transparent.** Unknown commands pass through untouched.
5. **Keep the bytes recoverable.** Anything elided can be fetched back with `rtk recall`.

!!! tip "Built with PAUL"
    This site was planned and written phase by phase with the PAUL framework
    (see the `.paul/` directory in the repository).

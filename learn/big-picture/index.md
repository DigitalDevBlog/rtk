# Big Picture

## The problem

Coding agents pay tokens for every byte a command prints. Most of that output is noise to
a model: ANSI colour codes, progress bars, "Compiling ..." lines, hundreds of passing tests,
hint text like *"(use git add to track)"*.

RTK's answer is to be a **proxy**: run the real command, keep the signal, drop the noise,
and tell the agent nothing changed.

```kroki-mermaid
flowchart LR
    subgraph before["Without RTK"]
        direction LR
        B1["git status"] --> B2["real git"] --> B3["~200 tokens of hints and formatting"] --> B4["Model context"]
    end
    subgraph after["With RTK"]
        direction LR
        A1["git status"] --> A2["rtk git status"] --> A3["~20 tokens: branch and changed files"] --> A4["Model context"]
    end
```

!!! note "What the percentages mean"
    RTK's headline savings (60-90 %) measure **bash output bytes**, not your bill. There is
    no tokenizer in the binary: tokens are estimated as `ceil(chars / 4)`
    (`src/core/tracking.rs`). Ratios are trustworthy; absolute token counts are approximate.

## Two cooperating halves

| Half | Job | Lives in |
|------|-----|----------|
| **Interception** | Notice a command the agent is about to run and rewrite it to its `rtk` equivalent | `hooks/`, `src/hooks/`, `src/discover/` |
| **Filtering** | Execute the command, compress the output, record the savings | `src/cmds/`, `src/filters/`, `src/core/`, `src/analytics/` |

The halves only meet at one point: the *string* `rtk git status`. The hook never filters,
and the filters never care how they were invoked, so each half can be tested on its own.

## Module map

```kroki-mermaid
flowchart TB
    main["main.rs<br/>Clap Commands enum + routing"]
    main --> cmds["cmds/<br/>9 ecosystems of Rust filters"]
    main --> hooks["hooks/<br/>init, rewrite, permissions, integrity"]
    main --> analytics["analytics/<br/>gain, cc-economics, session"]
    main --> discover["discover/<br/>lexer, registry, rules"]
    main --> fallback["run_fallback<br/>TOML filters or passthrough"]
    hooks --> discover
    cmds --> core["core/<br/>tracking, config, toml_filter,<br/>retriever, stream, utils"]
    fallback --> core
    analytics --> core
    hooks --> core
```

`core/` is a leaf: it knows nothing about any specific command, hook or agent, and is
imported by everyone. That is what prevents circular dependencies.

## Ecosystem coverage

`src/cmds/` groups the Rust filters by ecosystem: **git** (git, gh, gt, diff), **rust**
(cargo), **js** (npm, pnpm, vitest, lint, tsc, next, prettier, playwright, prisma),
**python** (ruff, pytest, mypy, pip), **go**, **dotnet**, **cloud** (aws, docker/kubectl,
curl, wget, psql), **system** (ls, tree, read, grep, find, json, log, env, deps), **ruby**,
**jvm** and **php**. On top of those, more than 60 declarative TOML filters live in
`src/filters/`.

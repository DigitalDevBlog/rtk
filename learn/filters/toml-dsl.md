# TOML Filter DSL

For line-oriented tools (brew, terraform, df, ps, shellcheck, ...) RTK ships **declarative
filters**: one `.toml` file per tool in `src/filters/` (63 files in v0.49.0). At build time
`build.rs` concatenates them alphabetically into a single blob embedded in the binary.

## A complete filter

`src/filters/jq.toml`:

```toml
[filters.jq]
description = "Compact jq output — truncate large JSON results"
match_command = "^jq\\b"
strip_ansi = true
strip_lines_matching = ["^\\s*$"]
max_lines = 40
truncate_lines_at = 120

[[tests.jq]]
name = "short output passes through"
input = """
{ "name": "test" }
"""
expected = "{ \"name\": \"test\" }"
```

Inline `[[tests.<name>]]` blocks are validated when you run `cargo test`.

## The 8-stage pipeline

```kroki-mermaid
flowchart TD
    RAW["raw stdout"] --> A["1 strip_ansi"]
    A --> B["2 replace<br/>line regex substitutions"]
    B --> C{"3 match_output<br/>pattern hit?"}
    C -->|"yes (and no 'unless' match)"| SC["return short message"]
    C -->|no| D["4 strip_lines_matching<br/>or keep_lines_matching"]
    D --> E["5 truncate_lines_at N chars"]
    E --> F["6 head_lines / tail_lines"]
    F --> G["7 max_lines cap"]
    G --> H{"8 empty?"}
    H -->|yes| OE["on_empty message"]
    H -->|no| OUT["filtered output"]
```

| Stage | Field | Notes |
|-------|-------|-------|
| 1 | `strip_ansi` | Remove colour and control codes |
| 2 | `replace` | Chainable, supports backreferences |
| 3 | `match_output` | Short-circuit: if output matches, emit a message. An `unless` field prevents swallowing errors |
| 4 | `strip_lines_matching` / `keep_lines_matching` | Mutually exclusive |
| 5 | `truncate_lines_at` | Unicode-safe |
| 6 | `head_lines` / `tail_lines` | With an "omitted" message |
| 7 | `max_lines` | Absolute cap after head/tail |
| 8 | `on_empty` | Message when nothing survives, e.g. `"my-tool: ok"` |

## Three-tier lookup (first match wins)

```kroki-mermaid
flowchart LR
    Q["command string"] --> P[".rtk/filters.toml<br/>project-local, needs rtk trust"]
    P -->|no match| U["user-global filters.toml<br/>in RTK config dir"]
    U -->|no match| B["built-in filters<br/>embedded via build.rs"]
    B -->|no match| PT["passthrough"]
```

## Why project filters need trust

A project-local `.rtk/filters.toml` is executable *policy* that changes what an agent sees, so a
cloned repo could hide output from the model. `rtk trust` explicitly approves it, and trust state is
tracked in `src/hooks/trust.rs`. Unreviewed project filters do not run.

## Where TOML filters run

TOML filters apply in `run_fallback()`: only when Clap fails to match a dedicated Rust command.
That keeps the two systems from fighting: **Rust first, TOML second, passthrough last**.
Adding a new one also needs a rewrite rule in `discover/rules.rs` so the hook knows to send the
command to `rtk` in the first place.

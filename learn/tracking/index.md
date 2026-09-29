# Tracking & Analytics

Every command RTK handles is measured. That is how `rtk gain` can tell you *"81.5M tokens saved
(50.4 %) across 47,235 commands"*.

## What is recorded

```kroki-mermaid
flowchart LR
    RAW["raw output<br/>(stdout + stderr)"] --> EST1["estimate_tokens<br/>ceil(chars / 4)"]
    FIL["filtered output"] --> EST2["estimate_tokens<br/>ceil(chars / 4)"]
    EST1 --> ROW["INSERT into commands"]
    EST2 --> ROW
    T["timer: exec_time_ms"] --> ROW
    CWD["project_path (cwd)"] --> ROW
    ROW --> DB[("history.db<br/>SQLite")]
```

The estimator is intentionally crude: **no tokenizer is shipped**. `estimate_tokens("abcde")` is 2
because `ceil(5 / 4)`. Ratios stay reliable, absolute numbers are approximate.

## Schema

```kroki-mermaid
erDiagram
    commands {
        INTEGER id PK
        TEXT timestamp "UTC ISO8601"
        TEXT original_cmd "ls -la"
        TEXT rtk_cmd "rtk ls"
        TEXT project_path "cwd"
        INTEGER input_tokens "raw, bytes/4"
        INTEGER output_tokens "filtered, bytes/4"
        INTEGER saved_tokens "input - output"
        REAL savings_pct "saved/input*100"
        INTEGER exec_time_ms
    }
    parse_failures {
        INTEGER id PK
        TEXT timestamp
        TEXT raw_command
        TEXT error_message
        INTEGER fallback_succeeded "1 yes, 0 no"
    }
```

- File: `history.db` in the platform data directory (`~/Library/Application Support/rtk/` on macOS,
  `~/.local/share/rtk/` on Linux). Override with `RTK_DB_PATH` or `tracking.database_path`.
  (Some source comments still say `tracking.db`; the constant is `HISTORY_DB = "history.db"`.)
- Retention: 90 days by default (`tracking.history_days`).
- Project-scoped queries use `GLOB`, not `LIKE`, so `_` and `%` in paths are not treated as wildcards.
- Every code path must call `timer.track()` - success, failure and fallback - *before*
  `process::exit`, otherwise metrics are lost. The raw string should include stdout **and** stderr.

## Reading it back

| Command | Shows |
|---------|-------|
| `rtk gain` | Totals, efficiency meter, top commands by tokens saved |
| `rtk gain --graph` / `--daily` / `--weekly` / `--monthly` / `-a` | Time series |
| `rtk gain --history` | Recent commands with their savings |
| `rtk gain -p` | Scoped to the current project |
| `rtk gain --failures` | Parse failures that fell back to raw execution |
| `rtk gain --recalls` | Recall efficiency per filter |
| `rtk gain -f json` / `csv` | Machine-readable |
| `rtk cc-economics` | Claude Code spend versus savings |
| `rtk session` | Adoption per session |
| `rtk discover` | Commands in past sessions that *could* have been rewritten |
| `rtk learn` | Detect corrected commands in error history and suggest rules |

## `rtk discover`: the rewrite brain, replayed

```kroki-mermaid
flowchart LR
    S["Claude Code session JSONL files"] --> P["SessionProvider trait"]
    P --> X["extract Bash commands"]
    X --> SP["split compound commands (lexer)"]
    SP --> CL["classify_command (same rules as the hook)"]
    CL --> AG["aggregate: missed rewrites,<br/>estimated savings, adoption rate"]
```

Because `discover` shares `classify_command` with the live hook, its estimate of what you
missed matches what the hook would really do. Currently only Claude Code sessions are parsed.

## Reading the numbers

The example install's `rtk gain` table shows why the top row is what it is:
`rtk grep` had the most *volume* (13,113 calls, 24.7M tokens saved) at a modest 25.6 %, while
`rtk ps aux` saved 98.9 % on only 120 calls. High-percentage filters and high-impact filters are not
the same list - impact is `calls x tokens saved`.

## Telemetry

RTK also sends a non-blocking, at-most-daily usage ping (`telemetry::maybe_ping`); it can be turned
off with `[telemetry] enabled = false`. Details are in `docs/TELEMETRY.md` of the repository.

# Command Lifecycle

Follow `git log --oneline -5` from the agent's keyboard to the tracking database.

## End to end

```kroki-plantuml
@startuml
skinparam shadowing false
actor "LLM agent" as A
participant "PreToolUse hook" as H
participant "rtk rewrite\n(registry)" as R
participant "rtk CLI" as C
participant "real git" as G
database "history.db" as D

A -> H : Bash tool call: git log --oneline -5
H -> R : rtk rewrite "git log --oneline -5"
R --> H : rtk git log --oneline -5 (exit 0 or 3)
H --> A : updatedInput.command = rewritten
A -> C : executes rtk git log --oneline -5
C -> G : spawn git log --oneline -5
G --> C : stdout, stderr, exit code
C -> C : filter output
C -> D : track(input_tokens, output_tokens)
C --> A : compact output, same exit code
@enduml
```

## The six phases inside `rtk`

`docs/contributing/ARCHITECTURE.md` names six phases once the rewritten command reaches the binary:

| # | Phase | What happens |
|---|-------|--------------|
| 1 | **Parse** | Clap turns `rtk git log --oneline -5 -v` into `Commands::Git` plus args and a verbosity level |
| 2 | **Route** | `main.rs` matches the enum variant and calls the module, e.g. `git::run(...)` |
| 3 | **Execute** | The module spawns the real tool with `std::process::Command` and captures stdout, stderr and the exit code |
| 4 | **Filter** | A strategy shrinks the text (stats extraction, grouping, failure focus, ...) |
| 5 | **Print** | Filtered text goes to stdout; debug detail to stderr with `-v` / `-vv` / `-vvv` |
| 6 | **Track** | Raw and filtered sizes are written to SQLite; the child's exit code is propagated |

```kroki-mermaid
flowchart TD
    P1["1 Parse (Clap)"] --> P2["2 Route (match Commands)"]
    P2 --> P3["3 Execute child process"]
    P3 --> P4{"4 Filter succeeds?"}
    P4 -->|yes| P5["5 Print filtered output"]
    P4 -->|no| RAW["Use raw output"] --> P5
    P5 --> P6["6 Track savings in SQLite"]
    P6 --> EX["Exit with child's exit code"]
```

## Preambles before routing

Before the `match`, `main.rs` does a few cheap things (`docs/contributing/TECHNICAL.md` §3.3):

1. `telemetry::maybe_ping()` - non-blocking, at most daily
2. `Cli::try_parse()` - parse against the `Commands` enum
3. `hook_check::maybe_warn()` - warns if the installed hook is outdated (rate limited to once a day)
4. `integrity::runtime_check()` - verifies the hook's SHA-256 for operational commands

If Clap cannot parse the command, control goes to the **fallback path**
(see [Filters](../filters/index.md)) rather than an error, unless the word is an RTK
meta-command such as `gain` or `init`.

## Verbosity

| Flag | Extra output |
|------|--------------|
| `-v` | Debug messages (e.g. "Git log summary") |
| `-vv` | The exact command being executed |
| `-vvv` | The raw output before filtering |

To bypass filtering entirely while still recording the call, use `rtk proxy <cmd>`:
it shows up in `rtk gain --history` with 0 % reduction.

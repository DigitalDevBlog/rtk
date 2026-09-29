# Install & Config

## What `rtk init` does

`rtk init` wires RTK into an agent. For Claude Code the default global install (`rtk init -g`):

```kroki-mermaid
flowchart TD
    I["rtk init -g"] --> H["1 write hook<br/>(or register native rtk hook claude)"]
    H --> S["2 store SHA-256 hash, read-only 0o444"]
    S --> P["3 patch settings.json<br/>register PreToolUse hook"]
    P --> M["4 write RTK.md awareness file<br/>and patch CLAUDE.md"]
    P --> B["settings backed up to .bak first"]
```

Safety properties (`src/hooks/README.md`): every file write is **atomic** (tempfile + rename), settings
are backed up to `.bak` before modification, and everything is **idempotent** - run it twice, get the same result.

### Patch modes

| Mode | Flag | Behaviour |
|------|------|-----------|
| Ask (default) | - | Prompt before settings changes; defaults to *No* when stdin is not a terminal |
| Auto | `--auto-patch` | Patch without prompting (CI / scripted installs) |
| Skip | `--no-patch` | Print manual instructions instead of editing |

### Install modes

| Mode | Command | Creates |
|------|---------|---------|
| Default (global) | `rtk init -g` | Hook, hash, `RTK.md`; patches `settings.json` and `CLAUDE.md` |
| Hook only | `rtk init -g --hook-only` | Hook and hash |
| Windsurf / Cline | `rtk init -g --agent windsurf` / `rtk init --agent cline` | Rules file (prompt-level) |
| Codex | `rtk init --codex` | `RTK.md`, `hooks.json`, `AGENTS.md` block |
| Cursor | `rtk init -g --agent cursor` | Cursor hook in `hooks.json` |
| Others | `--agent trae / antigravity / pi / omp / hermes` | Per-agent plugin or extension |

## Integrity verification

A hook runs on *every* command the agent issues, so a tampered hook would be a powerful attack.
RTK checks a SHA-256 hash.

```kroki-mermaid
stateDiagram-v2
    [*] --> NotInstalled
    NotInstalled --> Verified: rtk init stores hash
    Verified --> Tampered: hook file modified
    Verified --> NoBaseline: hash file deleted
    NoBaseline --> Verified: rtk init again
    Tampered --> Verified: reinstall
    Verified --> OrphanedHash: hook deleted
```

- **Verified** - hash matches.
- **Tampered** - mismatch; `integrity::runtime_check()` blocks execution.
- **NoBaseline** - hook exists but no hash (an older install).
- **NotInstalled** - neither exists.
- **OrphanedHash** - hash file without a hook.

`rtk verify` prints PASS/FAIL/WARN/SKIP for each check and also runs the TOML filters' inline tests.
`rtk hook audit` summarises what the hook decided (rewrite, or `skip:deny_rule`, `skip:defer`, ...).

## Configuration

Config is TOML at `config.toml` in the platform config directory (`~/Library/Application Support/rtk/`
on macOS, `~/.config/rtk/` on Linux). It is loaded **on demand**, so there is no config I/O on startup.

```toml
[tracking]
enabled = true
history_days = 90

[display]
colors = true
emoji = true
max_width = 120

[retriever]                # output recovery, see Filters
mode = "sqlite"            # sqlite | tee | disabled

[hooks]
exclude_commands = ["curl", "playwright"]   # never auto-rewrite these

[limits]
grep_max_results = 200
grep_max_per_file = 25
status_max_files = 15
```

### Environment overrides

| Variable | Effect |
|----------|--------|
| `RTK_DISABLED=1` (as a command prefix) | Skip rewriting that command |
| `RTK_DB_PATH` | Alternate tracking database |
| `RTK_RECALL=0` | Never write to `recall.db` from the hook path |
| `RTK_REWRITE_HOST=<agent>` | Tell `rtk rewrite` the caller owns approval |

## Uninstalling and inspecting

`rtk init --uninstall` reverses the install; shared config is only removed when definitively unshared.
Use `rtk proxy <cmd>` to run any command raw (still tracked) when debugging a filter.

# Filters

Once a rewritten command reaches the `rtk` binary, it has to be **run and compressed**. RTK has
two filter engines plus a passthrough.

```kroki-mermaid
flowchart TD
    IN["rtk &lt;command&gt; args"] --> CL{"Clap parses it into<br/>the Commands enum?"}
    CL -->|yes| RUST["Rust filter module<br/>src/cmds/&lt;ecosystem&gt;/"]
    CL -->|no| META{"RTK meta-command<br/>(gain, init, ...)?"}
    META -->|yes| ERR["show Clap error"]
    META -->|no| TOML{"TOML filter matches<br/>match_command regex?"}
    TOML -->|yes| TF["capture stdout,<br/>run 8-stage pipeline"]
    TOML -->|no| PT["passthrough:<br/>inherit stdio, track 0% reduction"]
    RUST --> DONE["print + track + exit code"]
    TF --> DONE
    PT --> DONE
```

## The standard module shape

Every Rust filter follows the same six steps (`docs/contributing/TECHNICAL.md` §3.4):

1. Start a timer (`TimedExecution::start()`)
2. Run the real command (`std::process::Command`)
3. Filter: strip boilerplate, group errors, truncate
4. **On filter error, fall back to raw output** (`eprintln!("rtk: filter warning: ...")`, never a silent `Err(_) => {}`)
5. Track token savings (must happen on *all* paths, before `process::exit`)
6. Propagate the child's exit code

```rust
pub fn run(args: MyArgs) -> Result<()> {
    let output = execute_command("mycmd", &args.to_cmd_args())
        .context("Failed to execute mycmd")?;
    let filtered = filter_output(&output.stdout)
        .unwrap_or_else(|e| { eprintln!("rtk: filter warning: {}", e); output.stdout.clone() });
    tracking::record("mycmd", &output.stdout, &filtered)?;
    print!("{}", filtered);
    if !output.status.success() { std::process::exit(output.status.code().unwrap_or(1)); }
    Ok(())
}
```

## Rust vs. TOML

| | Rust filter (`src/cmds/`) | TOML filter (`src/filters/*.toml`) |
|---|---|---|
| Good for | Structured output: JSON, NDJSON, state machines, multi-format | Predictable line-oriented text |
| Power | Arbitrary code, regexes, parsing | Strip / keep / truncate / cap lines |
| Cost to add | Code + tests + registry rule | One `.toml` file with inline tests |
| Rule of thumb | "reformat or summarise" | "strip noise lines; output still looks like the real thing" |

## Performance rules baked in

- **No async**: `tokio` would add 5-10 ms of startup, so everything is blocking, single-threaded.
- **`LazyLock<Regex>`** for every fixed, reused pattern: compiled once at first use.
- **No `unwrap()` in production** paths: use `.context("...")?`.
- Targets: startup < 10 ms, resident memory < 5 MB, stripped binary < 5 MB, output reduction
  of at least 20 % per filter as a floor (release blocker guidance in the repo rules is 60 %).

Continue with [Rust filter strategies](rust-filters.md), the [TOML DSL](toml-dsl.md), and how
elided output is [recovered](recovery.md).

# Rust Filter Strategies

`docs/contributing/ARCHITECTURE.md` catalogues **12 filtering strategies**. Each Rust module picks
the one (or a mix) that fits its tool's output.

```kroki-mermaid
flowchart LR
    subgraph reduce["Reduce volume"]
        S1["1 Stats extraction"]
        S2["2 Error only"]
        S7["7 Failure focus"]
        S9["9 Progress filtering"]
    end
    subgraph reshape["Reshape"]
        S3["3 Group by pattern"]
        S4["4 Deduplicate"]
        S8["8 Tree compression"]
        S5["5 Structure only"]
    end
    subgraph parse["Parse smartly"]
        S10["10 JSON / text dual mode"]
        S11["11 State machine"]
        S12["12 NDJSON streaming"]
        S6["6 Code filtering"]
    end
```

| # | Strategy | Technique | Typical reduction | Used by |
|---|----------|-----------|-------------------|---------|
| 1 | Stats extraction | Count and aggregate, drop details | 90-99 % | git status/log/diff, pnpm list |
| 2 | Error only | Keep stderr, drop stdout | 60-80 % | runner (err mode) |
| 3 | Grouping by pattern | Group by rule / file / code and count | 80-90 % | lint, tsc, grep |
| 4 | Deduplication | Unique lines with counts `(x5)` | 70-85 % | log |
| 5 | Structure only | JSON keys and types, values stripped | 80-95 % | json |
| 6 | Code filtering | Strip comments (`minimal`) or bodies (`aggressive`) | 20-90 % | read, smart |
| 7 | Failure focus | Hide passing tests, show failures | 94-99 % | vitest, playwright, runner (test) |
| 8 | Tree compression | Directory tree with counts | 50-70 % | ls |
| 9 | Progress filtering | Strip ANSI bars, keep final result | 85-95 % | wget, pnpm install |
| 10 | JSON/text dual mode | Prefer machine format, fall back to text | 80 %+ | ruff, pip |
| 11 | State machine | Track test state, extract failures | 90 %+ | pytest |
| 12 | NDJSON streaming | Parse events line by line, aggregate | 90 %+ | go test |

Ecosystem ranges quoted in the repo: git 85-99 %, JS/TS 70-99 %, Python 70-90 %, Go 75-90 %,
Ruby 60-90 %, .NET 70-85 %, cloud 60-80 %, system 50-90 %, Rust/cargo 60-99 %.

## Code filtering levels (`rtk read`)

`src/core/filter.rs` is a separate engine that filters *source files* rather than command output:

| Level | Effect | Reduction |
|-------|--------|-----------|
| `none` | Keep everything | 0 % |
| `minimal` | Strip comments (per-language delimiters) | 20-40 % |
| `aggressive` | Strip comments and function bodies, keep signatures | 60-90 % |

Supported languages: Rust, Python, JavaScript, TypeScript, Go, C, C++, Java, detected by
extension. Python is special: it has no block comments, and `"""` opens a *string* (docstring
or value), so it gets a string-aware path that only removes `#` comments.

## Example: measured on a live install

```text
$ rtk proxy git --no-pager log -5 | wc -c     # raw
1603
$ rtk git log -5 | wc -c                       # filtered
879
```

`git log` keeps hash, subject and a trimmed author/date header per commit but drops full bodies -
a modest 45 % here because the log was already terse. `git status` and `git diff` compress far
more because their raw form is mostly hint text and full hunks.

## Argument parsing is a first-class problem

Filters often need to add their own flags (`--porcelain`, `--format=json`) or detect the user's.
Naive `arg.starts_with('-')` checks fail on: a flag's *value* (`git log --grep -p` searches for `-p`),
attached values (`--flag=v`), short clusters (`-rn`), and everything after `--`. So all filters go through
`core/arg_tokenizer.rs` with a per-tool grammar. Rules: one grammar per tool and subcommand, scope
lookups to before `--`, inject RTK flags *before* the boundary, and detect and act using the same token.

## Truncation caps

`core/truncate.rs` defines four global caps - `CAP_ERRORS`, `CAP_WARNINGS`, `CAP_LIST`,
`CAP_INVENTORY` - that filters bind to local constants. A config value of `0` means "summary
only": the count and recovery hint are still printed. Caps are never refused, keeping the
"never block the user" philosophy.

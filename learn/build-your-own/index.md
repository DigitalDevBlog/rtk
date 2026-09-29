# Build Your Own

What is the *smallest* thing that reproduces RTK's core idea? Five parts.

```kroki-mermaid
flowchart LR
    A["1 Hook<br/>read tool-call JSON,<br/>emit updatedInput"] --> B["2 Rewriter<br/>tokenize, classify,<br/>prefix with proxy"]
    B --> C["3 Proxy binary<br/>run child, capture,<br/>preserve exit code"]
    C --> D["4 Filters<br/>per-tool compressors<br/>+ raw fallback"]
    D --> E["5 Ledger<br/>record raw vs filtered size"]
    D -.-> R["Recovery store<br/>hash -> raw output"]
```

## 1. The hook

Register a `PreToolUse` hook for the agent's Bash tool. Read JSON on stdin, pull out
`tool_input.command`, and respond with `updatedInput.command` set to the rewritten string. **Always
exit 0** - never let a bug in your hook block the agent.

## 2. The rewriter

Do not use string splitting. Write a small **shell lexer** (quotes, escapes, redirects, `&&`, `;`,
`|`). Rewrite each segment independently, keep pipeline intermediates raw, and re-attach env prefixes and
redirects. Keep a **deny-by-default rule table**: only commands with a known-safe filter get rewritten.

## 3. The proxy binary

```text
spawn child → capture stdout/stderr/exit → filter → print → record → exit(child_code)
```

If your filter throws, print the raw output. If the child fails, keep its exit code.

## 4. Filters

Start with three: **status-like** (counts instead of listings), **test runner** (failures only),
**log** (deduplicate with counts). Add a declarative layer (regex strip + head/tail + max lines) for
long-tail tools so new commands cost a config file, not code.

## 5. Ledger

Store `(cmd, raw_len/4, filtered_len/4, ms, cwd, ts)` in SQLite. A `gain` command that sums it is
what makes the tool feel worthwhile and shows which filters deserve work.

## Traps RTK already paid for

| Trap | RTK's answer |
|------|--------------|
| Rewriting `grep x \| wc -l` corrupts counts | Keep pipeline producers raw unless all consumers are display-only |
| `gh --json` output being summarised | Explicit veto for structured-output flags |
| Filter hides the one line the model needed | Recall store with hash hints |
| A hook that can be silently swapped | SHA-256 baseline checked at runtime |
| Cloned repo installs hostile filters | Project filters require explicit `rtk trust` |
| Per-tool flag parsing bugs | One argument grammar per tool, shared tokenizer |
| Sudden slowness | No async, lazy regex, startup budget under 10 ms |

## Exercises

1. Write the lexer and test it on `git commit -m "a && b" && git status`.
2. Implement a `git status` compressor and measure `bytes/4` before and after.
3. Add a recall store keyed by a content hash and print `[full output: recall <hash>]` on failure.

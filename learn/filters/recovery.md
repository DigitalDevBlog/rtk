# Output Recovery

Compression is lossy - so what if the model needed the line RTK dropped? Instead of telling the
agent to re-run the command (burning more tokens), RTK stores the original and prints a
**retrieval hint**.

```kroki-mermaid
sequenceDiagram
    participant AG as Agent
    participant RT as rtk cargo test
    participant DB as recall.db
    AG->>RT: run tests
    RT->>RT: raw output captured
    RT->>DB: store raw output (gzip, content-addressed)
    RT-->>AG: filtered summary + "[full output: rtk recall 3f9c2a81d4e7]"
    Note over AG: needs more detail
    AG->>RT: rtk recall 3f9c2a81d4e7
    RT->>DB: lookup by hash
    DB-->>AG: byte-faithful original
```

## Two triggers

| Trigger | Hint printed |
|---------|--------------|
| Command **failed** (non-zero exit) | `[full output: rtk recall <hash>]` |
| A list was **truncated** on success | `[+N hidden: rtk recall <hash>]` |

For truncated lists, the stored entry keeps an offset (the 1-based first hidden line) so the default
recall returns only the hidden tail.

## Storage modes (`[retriever]` config)

| Mode | Behaviour |
|------|-----------|
| `sqlite` (default) | `recall.db`, content-addressed, gzip blobs, lossless. Defaults: 10 MiB per entry, 200 entries FIFO, 30-day retention |
| `tee` | Legacy: one file per run, `{epoch}_{slug}.log` under the `tee/` directory, rotation by `tee_max_files` (20) and size cap (1 MiB) |
| `disabled` | No recovery |

Switch with `rtk config recall <sqlite|tee|disabled>`. `RTK_RECALL=0` prevents any `recall.db`
write from the hook path. Recovery **never changes** command output or exit code, and
`rtk gain --recalls` reports how often elided output is actually consulted, per filter - a direct
measure of whether a filter is cutting too deep.

## The consumer contract

Filters that parse structured output call `tee::tee_and_hint()` before `std::process::exit()`.
Truncating filters on success call `tee::force_tee_hint()` (multi-line blocks) or
`tee::force_tee_tail_hint(content, slug, offset)` (flat lists). Skipping this is a bug: the agent
would be told something is hidden with no way to get it.

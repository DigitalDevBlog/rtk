# Permissions

Rewriting a command must not accidentally bypass the user's security settings. RTK reads the
agent's permission rules and applies a least-privilege precedence:

```text
Deny > Ask > Allow (explicit) > Default (ask)
```

Rules come from every Claude Code `settings.json` (project and global, including `.local`
variants). Only `Bash(...)` rules are used.

```kroki-mermaid
flowchart LR
    CMD["command"] --> CHK["decision.rs<br/>single decision point"]
    CHK -->|Deny| DEN["defer: host handles denial"]
    CHK -->|Ask| ASK["rewrite + host prompts user"]
    CHK -->|Allow| ALW["rewrite + auto-allow"]
    CHK -->|"Default (no rule)"| DEF["rewrite + host prompts user"]
```

| Verdict | Trigger | `rtk rewrite` exit |
|---------|---------|--------------------|
| Deny | `permissions.deny` matched | 2 |
| Ask | `permissions.ask` matched | 3 |
| Allow | `permissions.allow` matched | 0 |
| Default | nothing matched | 3 |

## One decision point

`decision.rs` is the single place that decides deny / defer / rewrite-and-allow / rewrite-and-ask.
All three entry points route through it: the in-process `rtk hook <agent>` handlers, the
`rtk rewrite` subprocess, and the `rtk hook check` diagnostic. New gates go there, not in callers.

## Compound commands and the gate

`split_for_permissions` is intentionally the most conservative segmenter. Command or process
substitution and file-target redirects (`contains_unattestable_construct`) cannot be decomposed, so
the gate never auto-allows them. That is why the permission gate does not reuse the looser rewrite
segmenter.

## Hosts that own approval

Some agents evaluate permissions themselves *after* the hook runs (Codex, Trae, Antigravity) - for
them RTK just returns `updatedInput`. OpenClaw sets `RTK_REWRITE_HOST=openclaw` so that a
*Default* verdict exits 0 (no second prompt), while an explicit *Ask* still exits 3 and *Deny* still
exits 2. Naming a host can never relax an explicit deny or discard an explicit ask.

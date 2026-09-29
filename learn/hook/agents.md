# Supported Agents

The rewrite brain is shared; only the **JSON dialect** and the **install location** differ per agent.

```kroki-mermaid
flowchart TB
    REG["discover/registry<br/>one rewrite brain"]
    REG --> CL["Claude Code<br/>rtk hook claude"]
    REG --> CP["Copilot (VS Code + CLI)<br/>rtk hook copilot"]
    REG --> CU["Cursor<br/>rtk hook cursor"]
    REG --> GE["Gemini CLI<br/>rtk hook gemini"]
    REG --> CX["Codex<br/>rtk hook codex"]
    REG --> OC["OpenCode<br/>TypeScript plugin"]
    REG -.->|"prompt only"| RU["Cline / Roo, Windsurf<br/>rules files"]
```

| Agent | Mechanism | Can rewrite the command? |
|-------|-----------|--------------------------|
| Claude Code | `PreToolUse` in `settings.json` (`rtk hook claude`, or legacy shell script) | Yes (`updatedInput`) |
| GitHub Copilot (VS Code / CLI) | `rtk hook copilot` | Yes (`updatedInput`); CLI variant denies with a suggestion |
| Cursor | `rtk hook cursor` | Yes (`updated_input`) |
| Gemini CLI | `rtk hook gemini` | Yes (`hookSpecificOutput`), allow/deny only |
| Codex CLI | `rtk hook codex` + `hooks.json` | Yes (`updatedInput`) |
| OpenCode | TypeScript plugin, `tool.execute.before` | Yes (in-place mutation) |
| Cline / Roo Code, Windsurf | Rules file | No - prompt-level guidance only |
| Also in source | Trae, Mistral Vibe, Google Antigravity, Pi / Oh My Pi, Hermes, OpenClaw, Droid | Per-agent plugin or hook |

Agents without a programmatic hook fall on the **suggest** side: an instruction file
(`.clinerules`, `.windsurfrules`, `RTK.md`) asks the model to prefix commands itself, so adoption
depends on the model following instructions.

!!! note "Exact schemas"
    Each agent's JSON schema is documented in the RTK repo under `hooks/<agent>/README.md`.

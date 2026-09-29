# Diagrams

Diagrams are authored as code in fenced blocks (`kroki-mermaid`, `kroki-plantuml`) and rendered to
SVG at build time by the Kroki plugin. A network-reachable Kroki server is needed at build time
(`KROKI_SERVER_URL` overrides the default `https://kroki.io`).

```kroki-mermaid
flowchart LR
    hook --> rewrite --> filter --> track
```

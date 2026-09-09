# Orchestrator configuration (discarded approach)

> Kept as a record. This orchestrator was removed on 2026-08-21 — see the
> repository README for why. Nothing here is in use.

A versioned copy of the config file. **No secrets** — API keys lived in a
separate `.env` that was never committed.

## Decisions reflected here

| Key | Value | Why |
|---|---|---|
| `model.default` | `claude-sonnet-5` | Good cost/latency balance for conversation and capture. Heavy work was delegated to Claude Code. |
| `model.provider` | `anthropic` | Anthropic directly. **The installer ships an OpenRouter `base_url` by default** — it had to be removed, or an Anthropic key wouldn't work. |
| `approvals.mode` | `smart` | An auxiliary model assesses risk; dangerous operations are denied automatically. |
| `approvals.cron_mode` | `deny` | Scheduled tasks never auto-approve dangerous commands. |

In `.env` (outside git): a write-safe root pointing at the vault, which scoped
`write_file` and `patch` to that folder **at tool level**.

## Auxiliary models

The orchestrator used a separate cheap model for side tasks (vision, web
summarisation, context compression, session titles, session search). These were
left on `auto`: the config itself warns that pointing them at providers other
than the defaults is experimental.

There was no difficulty-based router for the main conversation. For a session
known to be heavy, you picked the model explicitly at launch.

## A trap when editing

`config set` **rewrites the file normalised and strips every comment** — 1924
lines down to 135 on the first run. The commented original was kept as a
`.bak` alongside it.

# personal-ai-os

Design notes for my personal assistant: **Claude Code as the agent, an Obsidian
vault as memory, and n8n for whatever has to run unattended.**

This repository is mostly an architecture document and a decision record. The
working parts live elsewhere — the vault has its own repository, and the
automations live in n8n.

## The main decision: removing the orchestration layer

The original design had a custom orchestrator, **Hermes**, at the centre: it
would take messages over Telegram, decide what to do, delegate to Claude Code
through a subprocess, and write to the vault.

It was installed, configured and tested. **It was discarded on 2026-08-21.**

The reason: the layer it was going to add already existed. Hermes was justified
as "the conversational layer", but Claude Code already is one, and it comes with
a terminal, a desktop app and a web client. Going capability by capability, the
only thing Hermes genuinely added was Telegram and WhatsApp. Everything else —
holding a conversation, recalling context, delegating work, capturing meetings,
running automations, acting proactively — was already covered by Claude Code, an
Obsidian plugin, or the n8n MCP server.

Building an orchestrator to close a messaging gap wasn't worth another moving
piece on the critical path.

```
        YOU  (terminal · desktop · Obsidian)
                     │
            ┌────────▼────────┐
            │   CLAUDE CODE   │  agent · scheduled tasks · MCP
            └────┬───────┬────┘
                 │       └──────────► n8n  (what runs with nobody watching)
                 ▼
          OBSIDIAN VAULT  ◄── source of truth
                 ▲
          only after human review
```

## The finding that saved the work

Part 1 of the architecture document solves a concrete problem: **running several
agent sessions in parallel against the same repository without them stepping on
each other.**

Two sessions editing the same checkout overwrite each other silently. The
solution I was about to build — scripts to create, list and close worktrees —
turned out to be unnecessary. Claude Code ships native worktrees with isolation
enforced at the tool level.

```bash
claude --worktree hermes
```

This isn't a convention the agent can ignore. It actively blocks edits outside
the worktree, commands whose working directory resolves back to the main
checkout, git redirection (`git -C`, `--git-dir`, `GIT_DIR`), and shell
constructs it can't trace without executing them. An agent can't escape its
worktree even if you ask it to.

Three planned scripts were never written.

## What's in here

| Path | Contents |
|---|---|
| `docs/architecture.md` | The full document. Part 1: parallel work, worktrees, the vault's write policy, git conventions. Part 2: the Hermes architecture that was discarded, with its declared risks |
| `CLAUDE.md` | Context and instructions for Claude Code sessions working on this repo |
| `hermes/` | Configuration for the discarded approach. Kept as a record, not in use |

The vault lives separately, in `~/Obsidian Vault`, with its own repository.

## Why this is public

Less for the code — there isn't much — and more for the reasoning: what was
evaluated, what was built, and why it was removed. The decision to delete a
piece that already worked tends to be documented far worse than the decision to
build it.



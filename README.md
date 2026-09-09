# personal-ai-os

A personal AI system I run daily: **Claude Code as the agent, an Obsidian vault
as memory, and n8n for whatever has to run unattended.**

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

This repository holds the architecture and the decisions. The parts that do the
work live where they belong: the vault has its own repository, and the
automations live in n8n.

## How it works in practice

**The vault is the memory, not the model's context.** Projects, decisions,
people, runbooks and meeting notes are plain Markdown with a small frontmatter
schema. An agent reads it with `grep` and frontmatter filters — no vector
database, because a knowledge base of a few hundred notes resolves in one step
and a vector DB adds a second source of truth to keep in sync.

**Writes are quarantined.** Anything an agent produces on its own lands in
`00 Inbox` marked unverified. Only a human promotes it to permanent knowledge.
An agent that writes something and later reads it back will believe itself, and
that's how a memory system starts drifting from reality.

**n8n handles what has to happen without anyone in front of the screen** —
scheduled jobs, webhooks, reconciliation. Claude Code reaches it through MCP
rather than reimplementing it.

**Nothing gets deleted.** Archived, never removed, in both the vault and the
automations. Same rule in two places is one rule to remember.

## Running several agent sessions at once

The problem this repo solved first: multiple agent sessions against the same
repository overwrite each other silently, because nothing locks files.

The answer turned out to be built in. `claude --worktree <name>` creates an
isolated checkout with **isolation enforced at tool level** — not a convention
the agent can talk itself out of:

| Check | What it blocks |
|---|---|
| Edits | `Edit`/`Write` pointing at the main checkout |
| Directory | Bash commands whose cwd resolves to the main checkout |
| Git redirection | `git -C`, `--git-dir`, `GIT_DIR`, `GIT_WORK_TREE` |
| Command shape | Constructs it can't trace without executing them |

Three scripts planned to manage worktrees were never written, because
`claude --worktree` and `git worktree list` already do it.

**The vault is the deliberate exception: it uses no worktrees at all.** Obsidian
indexes a single directory, so a worktree would be invisible to the app that
gives the vault its value. And with ~119 internal links across the notes, git
will merge two branches cleanly while leaving every `[[link]]` to a renamed note
pointing at nothing. Git compares text; a knowledge graph isn't text. The
protection is a link checker after every write, not a merge strategy.

## What's in here

| Path | Contents |
|---|---|
| `docs/architecture.md` | The full design. Parallel sessions, worktree isolation, the vault write policy, git conventions, and the evaluation of an orchestration layer that was ultimately not needed |
| `CLAUDE.md` | Operating instructions for agent sessions working on this repo |
| `hermes/` | Configuration from an evaluated orchestrator, kept as a record |

The vault lives separately, in its own repository.

## On the orchestration layer

The first design put a custom orchestrator in the middle: messaging gateway,
cron, delegation by subprocess. It was installed, configured and tested against
the real workload.

It didn't survive the evaluation. Going capability by capability — conversation,
context recall, delegation, meeting capture, automation, proactivity — every one
was already covered by the agent, an Obsidian plugin, or the n8n MCP server. The
only genuine gap was Telegram and WhatsApp messaging, and that didn't justify
another moving piece on the critical path.

The system described above is what shipped instead, and it's what I use every
day.

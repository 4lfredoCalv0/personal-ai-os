# CLAUDE.md — personal-ai-os

Repository for the personal assistant setup: architecture documents, skills and
scripts.

**This repo holds the system. The knowledge lives in the Obsidian vault.**

---

## 1. The boundary with the vault

`~/Obsidian Vault` is the **source of truth for knowledge** and has its own
`CLAUDE.md` with the conventions. This repo doesn't duplicate them — it
references them.

| Here | In the vault |
|---|---|
| Configuration | Projects, decisions, people |
| Skills (code and prompts) | Runbooks and procedures |
| Audio capture scripts | Distilled meeting notes |
| Architecture documents | The *why* behind decisions |

**Never** write credentials here. They go in `.env` (gitignored) or in the
relevant secret manager.

**Never** store meeting transcripts or audio in git. `meetings/` is ignored on
purpose: it's raw material that can be wrong and shouldn't outlive its
distillation.

---

## 2. Working in parallel

Several Claude Code sessions work on this repo at once through native
worktrees:

```bash
claude --worktree feature-x     # one isolated session
claude --worktree meetings      # another, in another terminal
```

Each lives in `.claude/worktrees/<name>/` on branch `worktree-<name>`. Claude
Code **blocks at tool level** any attempt to write to the main checkout from a
worktree — it isn't a convention, it's real isolation.

For read-only work (research, reading documentation) **no worktree is needed**:
plain `claude` is enough and creates no conflicts.

`git worktree list` shows the active ones. Worktrees with no changes clean
themselves up on exit.

---

## 3. Git rules

Scoped commits, so that four converging branches stay readable:

```
feat(agent): connect the vault over the filesystem
fix(meetings): correct system audio capture
docs(arch): record the delegation model
research(mcp): document findings
```

**Forbidden:** `push --force`, `reset --hard` on shared branches, deleting
someone else's branches, merging to `main` without human review.

---

## 4. The vault does NOT use worktrees

If a task touches `~/Obsidian Vault`, work on its **single directory**, never on
a copy. Reasons:

- Obsidian indexes one directory; a worktree would be invisible to the app.
- There are ~119 internal links across the notes. Git merges text without seeing
  that a `[[link]]` now points at a note renamed on another branch.
- Knowledge doesn't branch: there aren't two versions of a decision to merge.

**One writer at a time** for the vault's `CLAUDE.md`, index notes, templates,
scripts and `.base` views. New notes in different folders can go in parallel.

After any write to the vault:

```bash
cd ~/Obsidian\ Vault
git status --porcelain               # what changed
"99 System/Scripts/check-links.sh"   # links still resolve
```

The second one is what matters: **it catches what git can't see.**

---

## 5. Historical: the orchestrator

An earlier design put a custom orchestrator at the centre of this system. It was
built, tested and **discarded on 2026-08-21** — Claude Code already provided the
layer it was meant to add. See the README for the reasoning, and
`docs/architecture.md` Part 2 for the design as it stood.

`hermes/` keeps that configuration as a record. It is not in use, and its
security notes no longer apply to anything running.

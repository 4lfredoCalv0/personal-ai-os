# Personal AI OS — how to build it, and what to build

Two parts. The first defines **how the work happens** (parallel sessions, git,
the vault); the second **what gets built**. The first comes first because it
constrains the second.

---

# PART 1 — Working in parallel

## Context

The goal was three to five simultaneous Claude Code sessions that don't collide.
Opening several terminals and `cd`-ing into the same folder **is genuinely
dangerous**: Claude Code doesn't lock files, two sessions editing the same repo
overwrite each other silently, and each session writes
`.claude/settings.local.json`.

The solution already exists and didn't need building.

## The finding that avoided the work

**Claude Code ships native worktrees with isolation enforced at tool level.**

```bash
claude --worktree hermes
```

This creates `.claude/worktrees/hermes/` on branch `worktree-hermes` and starts
there. Repeat with another name in another terminal for another isolated
session.

It isn't a convention — Claude Code **actively blocks** four things while a
session is isolated:

| Check | What it blocks |
|---|---|
| Edits | `Edit`/`Write`/`NotebookEdit` pointing at the main checkout |
| Directory | Bash commands whose cwd resolves to the main checkout, or that it can't verify |
| Git redirection | `git -C`, `--git-dir`, `GIT_DIR`, `GIT_WORK_TREE`, or a `cd` back to the main checkout |
| Command shape | Constructs it can't trace without executing them (heredocs without a quoted delimiter, brace expansion) |

The last one can't be disabled. An agent can't escape its worktree even if you
ask it to.

**Direct consequence:** the three scripts originally planned to create, list and
close agent worktrees were never written. `claude --worktree <name>` and
`git worktree list` already do it.

### Two details that solve real problems

- **Permission approvals are stored in the main checkout** and apply to every
  worktree. That removes the `settings.local.json` conflict found during the
  audit.
- **`.worktreeinclude`** copies gitignored files (`.env`) into each new
  worktree. Without it, a worktree starts with no local credentials.

## An honest assessment: do you need five sessions?

**For code, yes. For the work actually queued up, probably not yet.**

The planned work was largely **sequential**: the meetings phase depended on the
orchestrator working; the skills depended on the vault being connected.
Parallelising dependent tasks doesn't speed anything up and multiplies merges.

What does parallelise well from day one:

- **Research** — reading documentation touches no files and conflicts with
  nothing.
- **Work in separate repos** — unrelated codebases share nothing.
- **Independent mechanical tasks** — two separate skills, two separate scripts.

The recommendation: **start with two sessions, not five.** One that builds, one
that researches. Move to three or four when the work is genuinely independent.

## Obsidian: no worktrees

This is the important decision, and it goes against applying one strategy
everywhere.

**The vault uses a single working directory and one writer at a time.**

Four reasons, in order of weight:

**1. Obsidian points at a single directory.** A worktree is another folder
Obsidian doesn't index. You'd be editing notes the app can't see: no backlinks,
no Tasks, no `.base` views. The tool that gives the vault its value would be
blind to half the work.

**2. Git can't see semantic coupling.** The vault has **119 internal links
across 40 notes**, with the busiest note linked 14 times. If one branch renames
a note while another creates links to the old name, **git merges cleanly and the
result is broken**. Git compares text; a knowledge graph isn't text.

**3. Knowledge doesn't branch.** A code branch is a hypothesis you test and
merge. What would a branch of a decision be? There aren't two versions of *why
we decided something* that merge — there's one correct one.

**4. The 21:00 auto-commit** runs `git add -A` on the vault's current branch.
With parallel branches, it commits whichever one happens to be checked out.

### Vault write policy

| Level | What | How |
|---|---|---|
| **Parallel, no risk** | Reading. Any session, always. | No coordination |
| **Parallel, with care** | Creating new notes in different folders | Dated or distinctive names avoid collisions |
| **One writer at a time** | `CLAUDE.md`, the index notes, templates, scripts, `.base` views, folder structure | **A single session. Never in parallel.** |

**No lock files, no queues.** For a single user with two or three terminals, a
lock server is infrastructure for a problem you don't have: writes take seconds
and you control who does what. The real protection is cheaper:

```bash
git -C "$VAULT" status --porcelain      # what changed
"99 System/Scripts/check-links.sh"      # links still resolve
```

`check-links.sh` is the one that matters: **it catches exactly what git can't
see.** If a note is renamed, git merges cleanly and every `[[link]]` to the old
name silently points at nothing.

As a backstop: commit after each writing session, not just the 21:00 automatic
one. That way the restore point is per session.

## Delegation and worktrees

**One worktree per delegated task is a mistake.** Runs launched with `-p`
**don't clean up their worktree on exit** and leave a lock in place until a
later sweep releases it. A worktree per task accumulates locked directories.

Beyond that, most delegations (*research this*, *analyse this code*, *why does X
fail*) are **read-only** — they need no isolation at all.

The policy:

| Kind of delegation | Isolation |
|---|---|
| Read, analyse, research | None. Run in the repo directory. |
| Write code | A **persistent** worktree, reused |
| Write to the vault | None — the vault doesn't use worktrees. Writes land in `00 Inbox` of the single checkout. |

**Rule kept regardless:** an agent never commits to `main` of a code repo. It
works on its branch, you review, you merge.

## Git conventions

Scoped commits, useful when four branches converge:

```
feat(agent): connect the vault over the filesystem
fix(meetings): correct system audio capture
docs(arch): record the delegation model
research(mcp): document the MCP server findings
```

**Forbidden for any agent:** `push --force`, `reset --hard` on shared branches,
deleting someone else's branches. No session merges to `main` without you
seeing it.

## Coordination between sessions

Git plus Markdown is enough. **No `/tasks` directory, no ticket system.**

- **What was done** → `git log` and the commit messages
- **What was decided** → the vault's decisions folder, which exists for exactly this
- **What's pending** → inline tasks in the project note, which the pending-items view already collects
- **In flight** → the branch name

Building a separate handoff mechanism would duplicate what the vault already
does.

## A real day

```bash
# Terminal 1 — building
cd ~/personal-ai-os
claude --worktree feature-x

# Terminal 2 — researching (no worktree: read only)
cd ~/personal-ai-os
claude

# Terminal 3 — an unrelated repo
cd ~/other-project
claude --worktree some-decision
```

When finished, in each worktree: commit, then `git worktree list` to see what's
left. Claude cleans up worktrees with no changes on exit.

## Verification test

1. Two sessions with `--worktree`, each creating a different file.
2. From one, explicitly ask it to edit a file in the main checkout → **it must be
   blocked**. If it succeeds, isolation isn't active.
3. `git worktree list` shows both.
4. Merge both branches to main without conflict.
5. In the vault, after any write: `check-links.sh` reports no broken links.

---

# PART 2 — The orchestrator architecture (discarded)

*Kept as a record. This design was built, tested and removed on 2026-08-21 —
see the README for why.*

The plan was a custom orchestrator sitting between the user and everything else:
messaging gateway, cron, and delegation to Claude Code by subprocess, with the
vault as the source of truth.

```
        YOU  (terminal · Telegram · Obsidian)
                     │
            ┌────────▼────────┐
            │   ORCHESTRATOR  │  cron · gateway
            └───┬────┬────┬───┘
   filesystem   │    │    │  MCP
   (write-safe) │    │    └──────► n8n
                │    └─ subprocess ──► CLAUDE CODE
                ▼
          OBSIDIAN VAULT  ◄── source of truth · git = undo
                ▲
         only after human review
                │
          local Whisper ← ScreenCaptureKit
```

**Why not everything over MCP:** the orchestrator's MCP server exposed **only
messaging tools** — no memory, no files. Claude Code would go by subprocess.
Obsidian would go through a scoped filesystem root; an Obsidian MCP server would
add a process and a failure mode in exchange for nothing.

**Memory — no conflict:** the orchestrator's own memory was a few thousand
characters about how to talk to you, not about your business. External memory
backends were ruled out: they run *alongside* the internal one, and that's where
a second source of truth appears. **No vector DB** — `grep` plus frontmatter
resolves in one step.

**Meetings:** the orchestrator's voice mode is for talking to it, not for
transcribing meetings. Separate layer, local. **Raw transcripts never enter the
vault** — only the distilled version you reviewed.

**Security:** approvals in smart mode, cron denied, a write-safe root pointing
at the vault. Never `--yolo`. Never delete: `status: archived` plus an archive
folder.

**Skills: three, not fourteen.** The vault's `CLAUDE.md` stays the definition of
conventions; a skill references it rather than copying it.

## Declared risks

**The orchestrator was six months old.** Very active development, but young. The
vault didn't depend on it: they're text files that survive if the project
disappears.

**The installer was `curl | bash`.** The official route, but it's the pattern
its own scanner flags as dangerous. Read the script first, and only from the
official domain.

**There were content farms impersonating the project** — several lookalike
domains, at least one publishing a version dated before the project launched.
Only the official GitHub organisation and domain count as sources.

**An agent that writes and then reads what it wrote believes itself.** That's the
reason for quarantining writes in `00 Inbox` and for human review of meeting
notes.

---

## What was actually done

1. Create the repo with its `CLAUDE.md`, `.gitignore` (including
   `.claude/worktrees/`) and `.worktreeinclude`.
2. Test worktree isolation with the verification above.
3. Evaluate the orchestrator — and, having done so, remove it.

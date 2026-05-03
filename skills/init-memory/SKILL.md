---
name: init-memory
description: Scaffold a tool-agnostic, markdown-only persistent agent memory layer (`.memory/` committed + `.scratch/` gitignored) in a target git repo so AI agents working across context windows, terminal sessions, and tool boundaries (Claude / Cursor / Codex / OpenAI / etc.) stop losing progress, re-deriving goals, and re-learning lessons. Use whenever a user says `/init-memory`, "init memory", "set up agent memory", "scaffold memory tracking", "add `.memory/`", "add persistent memory", or describes pain like "the AI keeps forgetting context across sessions", "I have to re-explain the project to every fresh chat", "I want a way for the agent to remember decisions across runs", "agent state keeps getting lost", "how do I make the AI remember things between chats". Especially trigger when a user mentions cross-context state loss, agent amnesia, or wants a markdown convention for goals / decisions / lessons / postmortems / tasks that survives across chat sessions and AI tools. Trigger even if the phrasing is indirect (e.g. "I keep starting from scratch every chat", "the AI doesn't remember our past discussions", "set up that memory folder thing"). Don't trigger for general doc/wiki requests, code-search RAG indexes, or project handbooks aimed at humans.
---

# Init Memory

Scaffold a tool-agnostic, markdown-only **persistent agent memory layer** in a target repository. The deliverable is two directories at the repo root plus three small wiring touches to existing files. Once installed, any AI agent (or human) following the convention picks up state across context windows, terminal sessions, and tool boundaries -- no plugins, no scripts, no caches, no out-of-band setup.

## Mental model

- **`.memory/`** (committed) -- long-term knowledge that should outlive any single session: goals, decisions, lessons, postmortems, plus shared templates and per-category INDEX.md.
- **`.scratch/`** (gitignored) -- live, ephemeral state: in-flight tasks, daily journal, inbox. Never committed; lazily created by agents at first use.

The split is the point: committed material is what *future* agents (and humans) need to know; gitignored material is the working space for the *current* session. If you find yourself wanting to commit something in `.scratch/`, it should probably be promoted to `.memory/` first.

## When to invoke this skill

Trigger on any of:

- User says `/init-memory`, "init memory", "set up agent memory", "scaffold memory", "add `.memory/`", "add persistent memory", "add memory tracking" (any phrasing).
- User describes pain about agents losing context: "the AI keeps forgetting", "fresh chats lose state", "I have to re-explain", "I want the agent to remember decisions".
- User wants a multi-repo (monorepo or sibling-repo) workflow where cross-session / cross-repo state must survive.
- User is starting a new repo and asks "what's a good convention for tracking AI agent context".

Don't invoke for:

- Pure docs / wiki requests aimed at humans (this is for AI agent state).
- Code-search / RAG / index systems (this is markdown convention, not a search index).
- One-off README updates.

## What gets created

```
<repo>/
|-- .memory/                          (committed, this skill writes it)
|   |-- README.md                     (the agent-facing protocol -- the most important file)
|   |-- templates/
|   |   |-- task.md
|   |   |-- goal.md
|   |   |-- decision.md
|   |   |-- lesson.md
|   |   |-- postmortem.md
|   |   `-- journal-entry.md
|   |-- goals/INDEX.md
|   |-- decisions/INDEX.md
|   |-- lessons/INDEX.md
|   `-- postmortems/INDEX.md
`-- .scratch/                         (gitignored, NOT pre-created -- agent makes it on first task)
```

`.scratch/` is *deliberately* not pre-created. An empty `.scratch/` would either need a `.gitkeep` (noise) or commit-skipping (more complexity). The first task an agent does will create `.scratch/tasks/T-0001-<slug>.md` from `.memory/templates/task.md`; the directory springs into being then.

## Scaffold steps

### 1. Confirm target repo

Ask the user (or infer from `pwd` / open editor) which repo to scaffold. Refuse non-git directories -- `.memory/` belongs in version control. Run `git -C <path> rev-parse --show-toplevel` to confirm and pin the target path.

### 2. Inspect for collisions

Before writing anything:

- If `<repo>/.memory/` exists -> STOP. Ask the user: (a) abort, (b) merge missing files only [default], (c) overwrite. Default to (b) -- a fresh scaffold should never silently overwrite curated state.
- If `<repo>/.scratch/` exists -> leave its contents alone; only ensure it's gitignored.
- If `<repo>/AGENTS.md` exists -> don't overwrite; insert the new section in a sensible place. Prefer just before the closing "Notes" / "References" section if present, after a "Tech Stack" / "Project Information" section, or simply append. Skip insertion if a heading literally titled `## Persistent Agent Memory` already exists.
- If `<repo>/CLAUDE.md` exists with an "Important Constraints" section -> append the bullet there. Otherwise skip CLAUDE.md and surface the bullet to the user with a note that they can place it manually.
- If `<repo>/.agents/` exists (e.g. for Claude Code Skills, OpenCode plugins, or any other agent-capability convention) -> it's a SEPARATE concept from `.memory/`. They coexist. Add a "See also" callout to the `<repo>/.memory/README.md` after copying it from the asset (the asset has the wording prepared as a commented hint).

### 3. Copy assets

Copy the entire `assets/memory/` tree from this skill's directory into `<repo>/.memory/`. The source tree mirrors the target tree exactly:

| Source (relative to this skill) | Target (in repo) |
|---|---|
| `assets/memory/README.md` | `<repo>/.memory/README.md` |
| `assets/memory/templates/*.md` | `<repo>/.memory/templates/` |
| `assets/memory/goals/INDEX.md` | `<repo>/.memory/goals/INDEX.md` |
| `assets/memory/decisions/INDEX.md` | `<repo>/.memory/decisions/INDEX.md` |
| `assets/memory/lessons/INDEX.md` | `<repo>/.memory/lessons/INDEX.md` |
| `assets/memory/postmortems/INDEX.md` | `<repo>/.memory/postmortems/INDEX.md` |

Copy verbatim. The README contains placeholder example names (`<repo-a>`, `<repo-b>`, `<repo-c>`, `<topic-keyword>`, `<service-name>`, etc.) -- leave them as-is unless the user explicitly asks to substitute their real names.

### 4. Wire `.gitignore`

Append to `<repo>/.gitignore` (or create if missing). Skip if the marker `.scratch/` already appears anywhere in the file:

```
# Live agent scratch state (committed long-term knowledge lives in .memory/)
.scratch/
```

### 5. Wire `AGENTS.md`

Insert this section (skip if any heading literally titled `## Persistent Agent Memory` already exists). Use the wording verbatim:

```markdown
## Persistent Agent Memory

This repo uses a `.memory/` (committed) + `.scratch/` (gitignored) convention so agents working across context windows, terminal sessions, and tool boundaries (Claude / Cursor / Codex / etc.) don't lose progress. **Read [`.memory/README.md`](./.memory/README.md) before doing real work.** Condensed protocol:

- **Session start** -- scan `.scratch/tasks/` (in-flight), the latest `.scratch/journal/` entry, and the Active section of `.memory/{goals,decisions,lessons,postmortems}/INDEX.md`. Pick up the highest-priority task; if none fits, copy `.memory/templates/task.md` to `.scratch/tasks/T-NNNN-<slug>.md`.
- **During work** -- keep the active task file's "Plan / Status log / Decisions / Blockers / Next session pickup" sections current. Anything reusable (a sharp edge, a non-obvious fix) gets a quick capture in `.scratch/inbox.md` for later promotion.
- **Session end** -- write the **Next session pickup** paragraph in every in-flight task. Promote anything settled to `.memory/decisions/`, `.memory/lessons/`, or `.memory/postmortems/` using the templates in `.memory/templates/`. Update the corresponding `INDEX.md` Active table in the same commit.
- **Eviction** -- knowledge that's wrong, outdated, or replaced gets `Status: superseded | stale | corrected` and a `Superseded by:` link, NOT a delete. Update the entry and the INDEX in the same commit.

If a planned change conflicts with an Active Decision in `.memory/decisions/`, halt and reconcile (supersede the old decision in the same commit) before writing code.
```

If `AGENTS.md` does NOT exist, create a minimal scaffold:

```markdown
# Project Development Guide

> This document is for AI Agents and developers. Read [`CLAUDE.md`](./CLAUDE.md) (when present) for full project context; this file is the agent-protocol entry point.

## Tech Stack

(fill in)

## Persistent Agent Memory

(insert the section defined above, verbatim)

## Notes

(fill in)
```

### 6. Wire `CLAUDE.md` (if present)

If `<repo>/CLAUDE.md` exists AND has a constraints / conventions section near the bottom, append:

```markdown
- **Persist agent memory** -- read [`.memory/README.md`](./.memory/README.md); on session start scan `.scratch/tasks/` + latest journal + INDEX Active sections, on session end promote insights to `.memory/{decisions,lessons,postmortems}/` (tombstone-and-link any superseded entries in the same commit).
```

If `CLAUDE.md` doesn't exist, skip silently. If it exists without an obvious constraints list, surface the bullet to the user and let them place it manually -- don't guess at where to inject in an unknown structure.

### 7. Report

Print a tree of what was created vs. skipped. Flag any wiring step that was skipped (existing `.memory/`, missing `AGENTS.md`, no `CLAUDE.md` constraints section) so the user knows what's left to do manually. Do NOT `git add` or commit on the user's behalf unless they explicitly asked.

Suggested commit message for the user (when they're ready):

```
docs: add persistent agent memory layer

Add a `.memory/` (committed) + `.scratch/` (gitignored) convention so
agents working across context windows, terminal sessions, and tool
boundaries (Claude / Cursor / Codex etc.) don't lose progress. `.memory/`
carries goals / decisions / lessons / postmortems with status-based
eviction (Status / Superseded by / Last validated / Recurrence guard) and
per-category INDEX.md split into Active vs Superseded; `.scratch/` holds
in-flight tasks and daily journal locally.
```

## Status semantics -- quick reference

The full table is in `.memory/README.md`, but the gist:

| Status | Meaning | Lives in INDEX |
|---|---|---|
| `active` | Current truth -- agents act on this | Active table |
| `superseded` | Replaced by a newer entry. Required: `Superseded by: <relative path>` | Superseded table |
| `stale` | No longer applies (referenced code got deleted, etc.); no replacement exists | Superseded table |
| `corrected` | Was factually wrong; kept as anti-pattern. Required: a "Why this was wrong" body section | Superseded table |
| `done` / `resolved` | Terminal state | Done table (goals) / Resolved table (postmortems) |
| `abandoned` | Goal dropped | Done table |
| `recurrence-detected` | Postmortem's prevention guard failed; similar incident happened again | Active table (with note) |

**Eviction rule**: never `rm` a committed memory file; flip Status. The history of why something was decided is what saves the next agent from re-discovering the same trap. A hard delete destroys that signal.

**Anti-amnesia primitives** (specific fields the templates require so future-you isn't lost):

- Task `Next session pickup` -- one paragraph at session end: what to read first, what to do first, current head state. Single most important field in the whole system.
- Decision `Context at decision time` -- what was true when this was decided. Read before second-guessing.
- Lesson `When to apply` -- the trigger pattern, specific enough to pattern-match without reading every file.
- Postmortem `Recurrence guard` -- whether the prevention is still in place. If `removed` or `partially-in-place`, similar incidents can recur.

## Cross-repo coordination

For a single repo, ignore this section. For sibling repos sharing a workspace (or multi-service monorepos):

- `T-NNNN` task IDs are **per-repo, not shared** -- avoids manual cross-repo number sync overhead.
- Use relative paths in `Related:` fields: `../<sibling-repo>/.scratch/tasks/T-0042-<slug>.md`.
- A decision affecting multiple repos: pick a single source of truth in the lowest-layer repo; other repos' `decisions/INDEX.md` carry a cross-ref row pointing at the relative path. (Per-repo copies are also OK when consequences materially differ -- the README.md asset has a worked example.)
- A lesson applicable to multiple repos: copy into each repo. Lessons describe a class of mistake; copies don't drift the way decision bodies do.
- Cross-repo supersession works via `Supersedes:` / `Superseded by:` with relative paths into the sibling repo. Update both repos' INDEX in coordinated MRs (or strictly sequential merges so the link doesn't dangle).

## Idempotency

The skill is safe to re-run. Each step checks for existing markers / files first. Re-running on a fully-scaffolded repo is a no-op that prints "already scaffolded".

To force re-scaffold, the user can `rm -rf <repo>/.memory/` (history is preserved in git, so the previous version is recoverable). Don't do this on the user's behalf.

## Anti-patterns (don't do these, and refuse if asked)

- Hard-deleting a committed memory file. Status flip + INDEX move is the eviction path.
- Writing a new entry without scanning the same category for conflicts (silent overwrites).
- Letting `INDEX.md` disagree with the actual files' `Status:` values.
- Pre-creating empty `<category>/archive/` (only created lazily when there's >20 non-active entries in a category).
- Storing live in-flight TODOs in `.memory/` (those go in `.scratch/`).
- Promoting trivia to `lessons/` (only promote things specific enough that a future agent can pattern-match the trigger; generic "be careful with X" notes are noise).
- Auto-committing on the user's behalf as part of scaffolding (the commit is a separate explicit step).

## References

- `assets/memory/README.md` -- the agent-facing protocol that lands at `<repo>/.memory/README.md`. THE document agents read at session start.
- `references/design-rationale.md` -- the "why" behind every design choice (committed-vs-gitignored split, status-based eviction over `rm`, no-scripts-by-design, anti-amnesia field selection). Read if the user asks "why this design" or wants to adapt it for unusual constraints.

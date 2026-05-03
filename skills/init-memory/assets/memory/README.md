# Persistent Agent Memory

This directory is the long-lived knowledge base for AI agents working on this
repo. It survives across context windows, terminal sessions, and AI tool
boundaries (Claude / Cursor / Codex / OpenAI / etc.).

The companion directory `.scratch/` (gitignored) holds active in-flight state.
The two together form the agent-memory protocol; the entry point is
[`AGENTS.md`](../AGENTS.md), with project-level context in
[`CLAUDE.md`](../CLAUDE.md) (if present).

> **See also** (delete this block if your repo has no `.agents/` directory):
> Some repos also carry an `.agents/` directory (e.g. for Claude Code Skills,
> OpenCode plugins, or other agent-capability conventions). That is a
> **capability-injection** layer -- code / configuration that an agent runtime
> loads to gain new tools. This `.memory/` directory is **knowledge / state**
> -- markdown read by any agent following AGENTS.md. The two coexist; don't
> conflate them. Skills add capabilities; memory persists context.

## Layout

```
.memory/                       (committed, this directory)
|-- README.md                  (this file)
|-- templates/                 (copy when creating new entries; never edit in place)
|   |-- task.md
|   |-- goal.md
|   |-- decision.md
|   |-- lesson.md
|   |-- postmortem.md
|   `-- journal-entry.md
|-- goals/
|   |-- INDEX.md               (Active + Done/Abandoned tables)
|   `-- G-NNNN-<slug>.md       (one per goal; e.g. G-0001-<feature-slug>.md)
|-- decisions/
|   |-- INDEX.md               (Active + Superseded tables)
|   `-- NNNN-<slug>.md         (one per ADR-lite; e.g. 0001-<topic-slug>.md)
|-- lessons/
|   |-- INDEX.md
|   `-- <topic>-<slug>.md      (e.g. <subsystem>-<gotcha>.md)
`-- postmortems/
    |-- INDEX.md
    `-- YYYY-MM-DD-<slug>.md   (e.g. 2026-04-30-<incident-slug>.md)

.scratch/                      (gitignored, agent creates on first use)
|-- tasks/
|   `-- T-NNNN-<slug>.md       (in-flight task; e.g. T-0042-<feature-slug>.md)
|-- journal/
|   `-- YYYY-MM/
|       `-- YYYY-MM-DD.md      (daily session log, append-only)
`-- inbox.md                   (quick capture for unsorted notes)
```

## Quick start (first session ever)

1. Pick the first task. Copy `templates/task.md` to
   `.scratch/tasks/T-0001-<slug>.md` and fill in `Definition of Done` + `Plan`.
2. Start today's journal: copy `templates/journal-entry.md` to
   `.scratch/journal/YYYY-MM/YYYY-MM-DD.md`.
3. As you work, append facts / commands / output to today's journal entry.
4. Before ending the session, write the **Next session pickup** paragraph in
   your task -- this is the single most important anti-amnesia primitive.

## Per-session protocol

### Session start

1. Read `goals/INDEX.md` Active section to refresh on current high-level intent.
2. Read `.scratch/tasks/` if present (any in-flight tasks from prior sessions).
3. Read latest `.scratch/journal/YYYY-MM/*.md` (your prior context: last
   terminal command, last decision).
4. Skim `decisions/INDEX.md` and `lessons/INDEX.md` Active sections for items
   relevant to today's work.
5. If you spot >=2 active conflicting entries, mark them as sweep candidates --
   don't pause work for the sweep.

### During session

- Update task `Status log` after each meaningful sub-goal.
- Append facts / commands / output to today's journal entry.
- For long-running terminals (dev server, watcher, build, queue consumer,
  background daemon, etc.), record terminal id / PID / log path in the task's
  `Running terminals` field. Future you (in another context) needs this to
  find what's still running.

### Session end

- Write **Next session pickup** in every in-flight task. One paragraph: what to
  read first, what to do first, current head state.
- If this session produced an architectural decision, a reusable pattern, or a
  production-impact error, **promote** it to `decisions/`, `lessons/`, or
  `postmortems/` (see promotion protocol below).
- If a task reached `done` or `abandoned`, delete the
  `.scratch/tasks/T-NNNN-*.md` file. Its essence is already promoted; keeping
  the scratch is noise.

## Promotion protocol (forced scan before write)

Before writing any new `decisions/`, `lessons/`, or `postmortems/` entry, the
agent **must**:

1. Grep the target category for related entries:

   ```bash
   grep -l "<topic-keyword>" .memory/<category>/*.md
   ```

2. If a conflict or supersession exists, the **same commit** must include all
   three actions:
   - **Old entry**: flip `Status:` to `superseded` / `corrected` / `stale`;
     add a `Superseded by: <new file>` line.
   - **New entry**: add a `Supersedes: <old file>` line.
   - **INDEX.md**: move the old row from the Active table to the
     Superseded / Stale / Corrected table.

3. If new and old coexist (different angles, not replacement), the new entry's
   `When to apply` (lessons) or `Context at decision time` (decisions) must
   explicitly state the boundary so future agents can pick the right one.

This protocol prevents silent overwrites, where an agent writes a new entry
without acknowledging that an old contradictory entry still claims authority in
the INDEX.

## Status field semantics

Every committed memory file carries `Status:` in its top metadata block.

| Status | Meaning | Used by |
|--------|---------|---------|
| `active` | Current; should influence agent behavior | all categories |
| `superseded` | Replaced by a newer entry; kept for traceability | decisions, lessons |
| `stale` | Underlying code / context no longer exists or has shifted; lesson's `When to apply` no longer triggers | lessons |
| `corrected` | Was wrong from the start; kept as anti-pattern reference (with a `Why this was wrong` note appended) | decisions, lessons |
| `done` / `resolved` | Reached terminal state | goals (`done`), postmortems (`resolved`) |
| `abandoned` | Explicitly dropped | goals |
| `deferred` | Punted to a later window | goals |
| `recurrence-detected` | A similar incident happened again after the original fix | postmortems |

**Agents reading the memory base only act on `Status: active` entries** unless
explicitly searching history.

`lessons/` additionally carry `Last validated: YYYY-MM-DD`. If today's task
touches code referenced by a lesson, the agent should re-validate the lesson
and bump the date -- or flip `Status: stale` if the code has moved on.

`postmortems/` additionally carry
`Recurrence guard: in-place | partially-in-place | removed | not-applicable`.
When the prevention measure is rolled back or no longer enforced, flip the
guard status to surface the gap.

## Cross-repo conventions

For a single-repo project, skip this section. For sibling repos sharing a
workspace, or for multi-service monorepos that need cross-service tracking:

```
<repo-a>          (e.g. frontend / SPA)
   |
   v  (network / IPC / RPC / etc.)
<repo-b>          (e.g. middleware / BFF / API gateway)
   |
   v
<repo-c>          (e.g. core service mesh / multi-service backend)
```

(Substitute `<repo-a>` / `<repo-b>` / `<repo-c>` with your actual sibling
names. The example uses placeholders.)

- Don't share `T-NNNN` numbering across repos -- avoid manual cross-git sync
  cost.
- Use relative paths in `Related:` fields:

  ```
  Related: ../<repo-b>/.scratch/tasks/T-0042-<slug>.md
  ```

- For business features that span multiple repos, `Related:` may carry up to
  two or three entries; one per affected repo.
- **Downstream first**: cross-repo work generally starts at the lowest layer
  (e.g. `<repo-c>` in the diagram), propagates up through middleware
  (`<repo-b>`), then to the surface (`<repo-a>`). Create tasks in the same
  order code changes happen, so the upper layer doesn't wait on a missing
  lower-layer endpoint.
- **Decisions affecting multiple repos** -- two valid patterns; pick one
  per decision and stick with it:
  - *Single source of truth* (preferred when one repo is clearly the
    primary owner): write the decision in that repo; each other affected
    repo's `decisions/INDEX.md` gets a cross-ref row pointing at the
    relative path.
  - *Per-repo copy* (when each repo has materially different consequences):
    write a sibling decision file in each affected repo and link them via
    `See also:` lines in the bodies, plus `Affects:` listing every repo on
    each copy.
- **Lessons applicable to multiple repos** -- copy the lesson into each
  applicable repo's `lessons/`. Make `Applies to:` and `When to apply:`
  explicit about which repo the lesson triggers in -- a multi-repo lesson
  with vague triggers is worse than no lesson at all.
- **Cross-repo supersession** -- the `Supersedes:` / `Superseded by:` fields
  accept relative paths into a sibling repo. The repo whose decision is being
  superseded is responsible for flipping its own INDEX from Active to
  Superseded. Worked example -- a `<repo-a>` decision superseded by a new
  `<repo-b>` decision (e.g. `<repo-a>` originally exposed a paginated
  endpoint, then a new `<repo-b>` design folded pagination into a single
  response):
  - Old (`<repo-a>`): flip `Status: superseded` and add
    `Superseded by: ../<repo-b>/.memory/decisions/0007-<new-slug>.md`.
  - New (`<repo-b>`): add
    `Supersedes: ../<repo-a>/.memory/decisions/0003-<old-slug>.md`
    and `Affects: <repo-a>, <repo-b>`.
  - Both `INDEX.md` files reflect the new state in coordinated MRs (or
    strictly sequential merges, with the new-decision repo merging first so
    the `Superseded by:` link doesn't dangle on the old side).

For multi-service monorepos (one repo, several services), the same pattern
applies *intra-repo*: tasks may have `Related: ../tasks/T-0017-<slug>.md` and
cross-service decisions go in `.memory/decisions/` once with
`Affects: <service-a>, <service-b>` rather than duplicating into per-service
folders.

## Eviction (memory hygiene)

- **Never `rm`** a committed memory file. Always flip `Status:` instead. The
  history of why a thing was decided is what saves the next agent from
  re-discovering the same trap; hard-deleting destroys that signal.
- When `<category>/` accumulates more than 20 non-active entries, batch-move
  them into `<category>/archive/` (still committed) for visual decluttering.
  INDEX keeps a one-line cross-ref:

  ```
  | 0007 | foo | 2026-01-15 | -> archive/0007-foo.md |
  ```

- Stale sweeps are **event-driven**, never time-driven:
  - Major refactor lands -> re-validate lessons / decisions touching the
    affected modules.
  - On-call rotation hand-off / quarterly review -> walk the INDEX,
    spot-check entries with `Last validated:` > 90 days.
  - Conflict detected during session-start scan -> sweep that topic
    immediately.

## Anti-patterns (forbidden)

- Hard-deleting a committed memory file.
- Writing a new entry without scanning the same category for conflicts.
- Letting `INDEX.md` disagree with the actual file `Status:` values. INDEX is
  the authoritative table of contents -- a "lying inventory" breaks the trust
  the session-start scan depends on.
- Pre-creating empty `<category>/archive/` (only create when needed, lazily).
- Storing live in-flight TODOs in `.memory/` (those go in `.scratch/`, which
  is gitignored).
- Promoting trivia to `lessons/`. Only promote things specific enough that a
  future agent can pattern-match the trigger; avoid generic "be careful
  with X" notes.
- Putting state / journal content into a capability-injection directory like
  `.agents/skills/` (those are SKILL.md / capability files, not memory) or
  putting a SKILL.md into `.memory/` (this is memory, not capabilities).

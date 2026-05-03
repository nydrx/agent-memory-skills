# Design Rationale -- Persistent Agent Memory Layer

This document explains *why* the layer is shaped the way it is. Read it when
you need to adapt the convention to unusual constraints, or when reviewing the
design with a skeptical eye.

## Constraints we tried to satisfy

1. **Tool-agnostic**. Cursor users, Claude Code users, Codex users, OpenAI API
   scripts, and humans all need to read and write the layer with no extra
   tooling. The substrate must be plain text that any process can parse with
   `cat` / `grep` / a text editor.

2. **Survives context flips**. The single most common failure mode for AI
   agents in 2025-2026 is the fresh chat with no priors -- the agent
   re-derives goals, contradicts last week's decisions, and re-asks questions
   the user already answered. The layer must compress past context into
   discoverable artifacts that the next session reads at start-up.

3. **Survives terminal-window flips**. A user juggling 4 terminals + 2 chat
   sessions + a background dev server has no central state. The layer must
   make "what's currently running" and "what task am I on" easy to recover
   even when none of those processes share a parent.

4. **Survives tool boundaries**. When the user moves from Cursor to Claude
   Code to a manual `claude -p` invocation, the same memory should follow.
   Tool-private caches / databases break this; markdown in git survives it.

5. **Doesn't pollute git history with noise**. Live work-in-progress (today's
   journal, half-finished task notes, "TODO check this thing") is not history
   the team wants to read forever. Committed material must earn its place.

6. **Doesn't lose hard-won lessons to evolution**. When a decision changes,
   the team needs to know *why* the old decision was made AND why it changed.
   Hard-deleting the old file destroys the trail; future agents re-discover
   the same trap.

## Why two directories instead of one

`.memory/` (committed) vs `.scratch/` (gitignored) is the answer to #5 and #6
together. Live state and historical knowledge are different artifacts with
opposite lifecycles:

| Aspect | `.memory/` | `.scratch/` |
|---|---|---|
| Audience | future agents (and humans) | current agent / current week |
| Lifecycle | sediments forever (status flip on change) | volatile, deletable |
| Authority | settled | unsettled |
| Git policy | committed | ignored |

Putting them in one directory creates a recurring debate: "should this
half-formed thought be committed?" Splitting them resolves the debate
structurally -- if it's done, it goes to `.memory/`; if it's in flight, it
stays in `.scratch/`.

## Why dot-prefixed names (`.memory/`, `.scratch/`)

Three reasons:

1. **Visual signal**. Dot-prefixed directories are universally understood as
   meta / tooling space (`.git/`, `.github/`, `.devcontainer/`, `.cache/`).
   Putting agent memory into the same family makes the boundary obvious;
   nobody mistakes `.memory/README.md` for product documentation.

2. **Hides from default `ls`**. The user's `ls` doesn't show these unless they
   ask. The layer is invisible during normal repo browsing, which is what we
   want -- it shows up when an agent (or a curious dev) explicitly looks for
   it.

3. **Avoids name conflicts**. `memory/` (no dot) could collide with a real
   product subsystem named "memory" (e.g. RAM allocation logic, chat history,
   etc.). The dot puts it firmly in tooling space.

We considered `agent/` (no dot), `agents/` (no dot), `.agents/`,
`.agent-state/`, `.ai/`, `.brain/`, and a few more. `.memory/` won because:

- The verb that matters is *remember*, not *agentify*. The substrate is
  knowledge, not agent personalities.
- Some repos already use `.agents/` for Claude Code Skills (capability
  injection). Re-using the name conflates capabilities with knowledge.
- `.scratch/` as the live-state companion has clear semantics ("scratchpad")
  that anyone gets at first read.

## Why markdown-only / no scripts / no CLI

The skill ships zero scripts on purpose. Reasons:

1. **Tool-agnostic** (#1 above). The minute the layer requires a Python /
   Node / Rust binary to read or write, it stops being usable from a stock
   `vim` or a fresh chat session.

2. **No version skew**. Scripts have versions; templates and conventions
   don't. A 2026-04 .memory/ scaffold is byte-identical to a 2027-04 one
   (modulo template tweaks), even though the agent runtimes have churned
   through 50 releases.

3. **Lower surface area for bugs**. A buggy script can corrupt memory; a
   buggy template just produces a bad-looking entry that the next agent can
   hand-fix.

4. **Easier to mirror across repos**. Copying templates and INDEX scaffolds
   into a sibling repo is a 10-second `cp -r`. Copying a script library is a
   maintenance burden.

We accept that this means agents must follow the protocol manually each
session -- read the README, scan the INDEX, write the templates. The
trade-off is intentional: the cost of not having scripts is paid once per
session in agent attention; the cost of having scripts is paid forever in
maintenance.

## Why status-based eviction instead of `rm`

Concrete scenario: the team chose Postgres over MongoDB in a decision
(`0003-postgres-over-mongo.md`), then 6 months later switched to MongoDB
after a real-world load test (`0017-mongo-after-load-test.md`).

If we `rm` the Postgres decision, the next agent walks into the codebase
seeing MongoDB everywhere and wonders "should we be using Postgres? It's a
mature SQL DB, why aren't we?" -- and the answer ("we tried, it didn't pan
out under load") is gone. That agent might propose switching back, the team
re-runs the load test, re-discovers the same answer, burns a sprint.

Status-flipping the old file (`Status: superseded`, `Superseded by:
0017-mongo-after-load-test.md`) keeps the history. The next agent's grep for
"postgres" finds the old file, sees it's superseded, reads the
`Superseded by` link, and absorbs the trail in 2 minutes instead of 2 weeks.

The status taxonomy (`superseded` / `stale` / `corrected`) lets us
distinguish:

- `superseded` -- there's a successor; we just decided differently.
- `stale` -- the code that motivated this is gone; no successor is needed.
- `corrected` -- it was factually wrong from the start; kept as an
  anti-pattern reference with a `Why this was wrong` body.

`corrected` is the spiciest: it lets a confidently-written-and-wrong entry
survive as a teaching tool rather than getting silently buried.

## Why "forced scan before promotion"

The most expensive failure mode in a memory base is an *active inventory of
contradictions*: two entries both say `Status: active`, but they disagree.
The next agent doesn't know which to follow.

The fix is procedural: before writing a new entry, the protocol forces a grep
for related entries. If a conflict exists, the new entry's commit MUST include
the supersession of the old (`Status` flip + `Superseded by` link + INDEX
move). This makes silent overwrites mechanically impossible -- the agent
can't ship a contradiction without first acknowledging it.

The two-table INDEX (Active / Superseded) is the secondary defense: even if a
file's `Status` field gets out of sync with the INDEX, the INDEX itself is
the authoritative roll-call of "what's currently active". Agents trusting the
INDEX get a consistent view; the rare drift between INDEX and file is caught
by the next sweep.

## Why event-driven sweeps instead of scheduled

Time-based "every-90-days review the lessons" rituals fail in two predictable
ways:

1. **Boil the ocean.** Reviewing every entry every quarter is too much work;
   reviewers skim and rubber-stamp.
2. **Hit nothing relevant.** The lesson the team needs to re-validate is the
   one near the code being changed *this week*; a calendar review is just as
   likely to look at lessons that haven't moved in years.

Event-driven sweeps work better:

- "Major refactor lands" -> re-validate lessons that touched the refactored
  module. (Refactors are when assumptions change.)
- "Conflict detected during session-start scan" -> sweep that topic now.
  (You're already there; the cost is low.)
- "Postmortem recurrence-detected" -> the prevention guard failed; revisit
  what we missed.

Calendar reviews still have a role (a hand-off / quarterly walk through the
INDEX is fine), but they're a backstop, not the primary mechanism.

## Why `Next session pickup` is the single most important field

When an agent loses context, the recovery sequence is:

1. Find the right task file.
2. Figure out what was done.
3. Figure out what to do next.
4. Figure out what was running.
5. Get back to the point of progress.

Steps 2-5 are expensive without a hint. The `Next session pickup` paragraph
collapses 2-5 into one read. It's written by the agent *while still in
context*, which is the only time the agent has the information cheap.

A task without a `Next session pickup` is salvageable but expensive. A task
with a stale `Next session pickup` (didn't update at session end) is worse
than no field, because the next agent trusts it. Hence the protocol's
insistence on updating it at session end.

## Why we kept it boring

Throughout the design we resisted features that "would be nice but aren't
strictly needed". Examples:

- **No structured frontmatter** (YAML / TOML at the top of each file).
  Markdown headers + `Status:` lines are enough; YAML adds parser fragility.
- **No automated indexing**. The INDEX.md is curated by humans / agents in
  the same commit as the entry. No "regenerate the INDEX from scanning the
  files" tooling. The INDEX *is* the source of truth; entries are checked
  against it.
- **No GUID / UUID identifiers**. `T-NNNN` / `G-NNNN` / `0001-` numbering is
  human-readable and per-category-monotonic. The trade-off: cross-repo sync
  is impossible, which is fine because we deliberately scoped IDs per repo.
- **No automated promotion**. The agent decides when something is settled
  enough to leave the scratch. Tooling can't make that judgment well; humans
  / agents do it case by case.

The discipline is: **add a feature only when its absence has caused a
concrete failure**. Until then, the simpler shape is the better shape.

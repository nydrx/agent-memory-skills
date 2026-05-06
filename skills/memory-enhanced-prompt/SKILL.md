---
name: memory-enhanced-prompt
description: "Force-trigger the agent-memory recall protocol before answering. When the user types `/memory-enhanced-prompt`, says 'load memory', 'snapshot memory', 'what was I working on', 'remember context', or otherwise asks the agent to ground its next reply in the persistent `.memory/` knowledge base, this skill runs three phases: (1) Scan -- unconditionally read `.memory/{goals,decisions,lessons,postmortems}/INDEX.md` Active sections + `.scratch/tasks/T-*.md` + the latest `.scratch/journal/YYYY-MM/<date>.md`; (2) Wrap -- emit a compact 'Memory Snapshot' block (<= 80 lines) that prepends to the next prompt's context; (3) Audit -- verify all four INDEX Active tables were read before producing any substantive answer, and refuse to act on tasks that contradict an Active Decision without an explicit supersession step. Companion to `init-native-memory` (which scaffolds) -- this is the runtime-side activation. Triggers on the literal slash command, on natural-language phrasings about loading/recalling memory, on explicit asks to summarize what was decided / learned / postmortemed, and at session start when the user wants the agent to deliberately read memory rather than guess at context. Do NOT trigger on routine code questions, do NOT trigger when `.memory/` doesn't exist (defer to `init-native-memory` instead)."
---

# Memory-Enhanced Prompt

This skill is the runtime companion to [`init-native-memory`](../init-native-memory/SKILL.md). `init-native-memory` *scaffolds* the persistent agent-memory protocol into a repo (`.memory/` committed + `.scratch/` gitignored, plus templates / INDEX files / cross-repo conventions). This skill *activates* it at runtime: it forces the agent to read what the protocol says before answering. Without a hard force-trigger, agents skip the L0 scan to save tokens, leak L3 reads when they shouldn't, and silently violate Active Decisions because they didn't bother to read the INDEX. All three failure modes are silent -- the agent looks confident, the user assumes context was loaded, and the protocol's anti-amnesia value evaporates.

This skill is a hard gate: before answering any substantive question after invocation, the agent MUST have run all three phases below. Phases are not optional and not skippable for token economy. The whole point is that the gate exists.

## Mental model

Three phases, run in order, no skipping:

1. **Scan** — force-read the L0 + L1 surfaces of the `.memory/` protocol.
2. **Wrap** — emit a "Memory Snapshot" block (the agent's working summary of what it just read) that the user can verify and the agent can refer back to.
3. **Audit** — before producing any substantive answer, verify the scan was complete and the planned response doesn't silently contradict an Active Decision.

The skill is the simplest possible runtime activation of `init-native-memory`'s read protocol; it just says "do the protocol, now, here, before you answer".

## When to invoke

Trigger on any of:

- Literal slash command: `/memory-enhanced-prompt`.
- Natural-language phrasings: "load memory", "snapshot memory", "load `.memory/`", "what was I working on", "remember context", "before we start, read everything", "ground in memory first", "pick up where we left off", "load context from memory", "refresh on prior decisions".
- Session-start when the user signals they want deliberate recall rather than guesswork: e.g. "let's continue", "what's the state", "where did we leave off", "catch me up", combined with a project that already has `.memory/`.
- A planned change that could plausibly contradict an Active Decision and the agent isn't confident the relevant decisions have been read this session.
- The user references a prior task / decision / lesson / postmortem the agent should know about but hasn't read in this session yet.

Anti-triggers (do NOT invoke):

- Routine code questions where `.memory/` content wouldn't be informative ("rename this variable", "add a console.log here", "what does this regex match"). Loading memory for these wastes tokens.
- The repo doesn't have a `.memory/` directory. Halt and defer to `init-native-memory` to scaffold it first; running this skill against a missing protocol is a no-op and confuses the user.
- Mid-tool-loop polling (e.g. waiting on a long build, polling test output). The snapshot block costs tokens; running it during a hot loop helps nothing.
- The same session has already produced a Memory Snapshot in the last few turns and nothing has changed since. Re-running blindly bloats the transcript -- check whether `.memory/INDEX.md` files have been edited or new tasks created before re-snapshotting.

## Phase 1 -- Scan (force-trigger, unconditional)

Read these paths, in order, regardless of how the agent feels about whether they're relevant. The scan is fixed-cost and bounded -- INDEX files are summary tables, not full bodies.

1. `.memory/goals/INDEX.md` -- read the **Active** section (table). If a `## Done / Abandoned` table exists below, skip it.
2. `.memory/decisions/INDEX.md` -- read the **Active** section. Skip the Superseded / Stale / Corrected tables.
3. `.memory/lessons/INDEX.md` -- read the **Active** section. Skip the Superseded / Stale tables.
4. `.memory/postmortems/INDEX.md` -- read the **Active** section. Skip the Resolved table unless `Recurrence guard:` is `removed` or `partially-in-place` for any of those entries.
5. `.scratch/tasks/` -- list filenames first. For each `T-NNNN-*.md` present, open it and read ONLY the `## Status log` and `## Next session pickup` sections (these two are the anti-amnesia primitives; the rest of the body is L3 detail).
6. Latest `.scratch/journal/YYYY-MM/<date>.md` -- sort entries by filename (date), pick the last one, read the head (top ~20 lines covers `## Active tasks touched today` + `## Commands / facts worth pasting`).

Scan rules (hard):

- Do NOT promote to L2/L3 unless an INDEX row's `TL;DR` text matches the topic of the upcoming task. The protocol's session-start cost is supposed to stay O(1) in corpus size; jumping to full bodies from the snapshot defeats it.
- Do NOT skip a category because "it probably isn't relevant". Missing the active counts means missing a soft-cap breach (Compaction trigger).
- Do NOT pre-load all `.scratch/tasks/T-*.md` files in full. The two-section read (`Status log` + `Next session pickup`) is by design.

Failure modes (soft):

- A category INDEX is missing or empty -> log it as `[INFO] no <category> entries yet`, continue. A fresh repo legitimately has empty INDEX files.
- `.scratch/tasks/` is empty or missing -> log `[INFO] no in-flight tasks`, continue.
- `.scratch/journal/` has no entries this month -> walk back one or two months to pick up the last entry; if there's none at all, log `[INFO] no journal yet`, continue.

Failure modes (hard, halt):

- `.memory/` directory does not exist -> halt the skill, surface the gap to the user, and defer to `init-native-memory`. Do NOT silently no-op; the user invoked the skill expecting a snapshot.
- `.memory/README.md` exists but `.memory/{goals,decisions,lessons,postmortems}/INDEX.md` are all missing -> halt and ask the user whether scaffolding is incomplete or the INDEX files were committed elsewhere.

## Phase 2 -- Wrap (Memory Snapshot block)

Emit a single block, in the chat, with this exact structure. The block prepends to the agent's working context for the next substantive answer; it's also the artifact the user can sanity-check ("yes, you understood the active state").

```markdown
## Memory Snapshot

**Active goals** (top 3):
- G-NNNN: <TL;DR from goals/INDEX.md>
- ...

**In-flight tasks** (`.scratch/tasks/`):
- T-NNNN <slug>: <Next session pickup, <= 1 line>
- ...

**Recent decisions** (top 5 by date):
- NNNN <slug>: <TL;DR from decisions/INDEX.md>
- ...

**Recent lessons** (top 5 by Last validated):
- <slug>: <TL;DR from lessons/INDEX.md>
- ...

**Open postmortems** (`Recurrence guard:` not `not-applicable`):
- YYYY-MM-DD <slug>: <TL;DR>
- ...

**Latest journal head** (`.scratch/journal/YYYY-MM/<date>.md`, top 5 lines):
> <verbatim>

---
```

Snapshot rules (hard):

- **Total length <= 80 lines**, including blank lines and the closing `---`. The point of a snapshot is that it's cheap to re-read; a 300-line "snapshot" is just a second copy of the corpus.
- **TL;DR text comes verbatim from the INDEX `TL;DR` column** (or, if the INDEX is missing the column, from the entry's `## TL;DR` block). Do NOT paraphrase. The whole protocol relies on the TL;DR being byte-identical between the entry and the INDEX; the snapshot inherits that contract -- if the agent paraphrases, future searches for the TL;DR text won't find the snapshot.
- **No L3 reads in this phase.** The snapshot is L0 (filenames + headers) plus L1 (INDEX TL;DR column) only. If a row's TL;DR isn't enough to summarise it, that's a signal the entry's TL;DR is stale -- log it and surface it as a sweep candidate, but don't open the entry body to "fix" the snapshot.
- **Truncation priority** when categories overflow the 80-line budget:
  1. Active goals (3 max)
  2. In-flight tasks (5 max; truncate `Next session pickup` to one line each)
  3. Recent lessons (5 max)
  4. Recent decisions (5 max)
  5. Open postmortems (only `Recurrence guard != not-applicable`)
  6. Latest journal head (5 lines max, verbatim)

  Drop categories from the bottom up if the budget is tight; never silently drop from the top. If goals + tasks alone exceed 80 lines (rare, only happens in genuinely bloated repos), surface the bloat to the user and recommend a Compaction sweep.

- **Empty categories surface as a one-line note**, not absence: e.g. `**Active goals**: (none)`. Absence in a snapshot is information; the user can tell whether the protocol is empty vs. whether the agent forgot to read.

## Phase 3 -- Audit (completeness check)

Before producing any substantive answer (anything beyond restating the snapshot), the agent MUST verify three conditions. This is the hardest part of the skill -- it's what stops the agent from snapshotting and then immediately ignoring the snapshot.

1. **All four INDEX files were read this session.** Provable by checking the agent's tool-call log for the four `Read` operations on `goals/INDEX.md`, `decisions/INDEX.md`, `lessons/INDEX.md`, `postmortems/INDEX.md`. If any is missing, halt: re-run Phase 1 for the missing category before answering.

2. **No silent contradiction with an Active Decision.** Walk the in-scope task vs. the Active decisions just read. If any Active Decision is in conflict with what the agent is about to do, three valid outcomes (and one forbidden):
   - **(a)** The user authors a supersession in the same response: flip the old decision's `Status: superseded`, add `Superseded by: <new file>`, write the new decision, move the row in `decisions/INDEX.md`.
   - **(b)** The user clarifies that the new task complements (not replaces) the old decision: the agent records the boundary in the new entry's `## Context at decision time` block.
   - **(c)** The agent halts and asks the user which path applies.
   - **Forbidden**: silently writing code that contradicts the Active Decision. The protocol's whole value is that the next agent (or the user reading commits later) can trust `Status: active` to mean current truth.

3. **Relevant Active Lessons are acknowledged.** If a lesson's `When to apply` clause matches the upcoming task, the agent's plan must cite the lesson by filename and either follow its `What works` recommendation or explain why it's deviating (e.g. the lesson is itself stale -- in which case the same response should bump `Last validated:` or flip `Status: stale`). Silent lesson-violation is the same anti-pattern as silent decision-violation, just lower-stakes.

Audit failure handling: do NOT proceed to substantive work. Surface the gap to the user explicitly:

> "Memory audit gap: <which check failed, with the specific Active Decision / Lesson / unread INDEX>. I can either (a) <explicit option>, (b) <explicit option>, or (c) halt for your call. Which?"

The user picks; the agent then proceeds with the chosen path.

## Output format

After the three phases, the agent's response shape is:

```
## Memory Snapshot
<the 80-line block from Phase 2>

---

<the substantive answer to the user's actual question, now grounded in the snapshot>
```

A concrete sample (truncated for the example; real snapshots are populated from the user's repo):

```
## Memory Snapshot

**Active goals** (top 3):
- G-0007: corpus upload MVP -- BFF accepts uploaded text + queues for analysis; success = end-to-end smoke passes in PPE.

**In-flight tasks** (`.scratch/tasks/`):
- T-0042 corpus-upload-ui: pickup -> wire SPA dropzone to /api/v2/service/corpus/upload; backend returns 202 + task_id.

**Recent decisions** (top 5 by date):
- 0011 bff-base-path: SPA always calls `/api/v2/service/*`; never bypass to srm directly.
- 0007 oidc-bff-flow: BFF owns OIDC; frontend reads `/auth/me`, never the provider directly.

**Recent lessons** (top 5):
- oidc-byted-claim-quirks: `byted_*` userinfo fields are JSON strings sometimes literally `"null"`; parse via `_parse_byted_label`.

**Open postmortems**:
- (none)

**Latest journal head** (`.scratch/journal/2026-05/2026-05-06.md`, top 5 lines):
> # 2026-05-06
> ## Active tasks touched today
> - T-0042: wired Goofy preview to dev backend; 401 still bouncing on `/auth/me`.
> ## Commands / facts worth pasting
> - `pnpm dev -- --host 0.0.0.0 --port 5173` works inside devbox.

---

<the agent's actual answer to the user's question follows here>
```

## Trigger phrases (frontmatter description excerpted)

For grep convenience -- the `description` field's natural-language phrases that route invocations to this skill:

- `/memory-enhanced-prompt`
- "load memory"
- "snapshot memory"
- "load `.memory/`"
- "what was I working on"
- "remember context"
- "before we start, read everything"
- "ground in memory first"
- "pick up where we left off"
- "load context from memory"
- "refresh on prior decisions"
- "let's continue" / "where did we leave off" / "catch me up" (only when `.memory/` exists)

## Anti-patterns

- **Silently dropping a phase to save tokens.** The point of the skill is force-trigger; if it's optional, it's not enforced, and the silent failures the skill exists to prevent come back.
- **Deepening to L3 in Phase 1 or Phase 2.** The snapshot must stay L0 + L1 only. Loading full entry bodies to "make the snapshot accurate" defeats progressive disclosure -- the INDEX `TL;DR` column is the contract; if it's stale, fix the INDEX, don't paper over it in the snapshot.
- **Auto-editing `.memory/` content as part of recall.** Phase 1 and Phase 2 are read-only. Promotions, supersessions, and `Last validated:` bumps are explicit Phase-3-audit follow-up steps -- the user must agree before any write happens.
- **Snapshotting once and ignoring the snapshot for the rest of the session.** The audit is per substantive answer, not per session. Each new task surfaces fresh decision / lesson conflicts; one snapshot at session start is necessary but not sufficient.
- **Confusing this skill with `init-native-memory`.** That one *creates* the protocol; this one *uses* it. If `.memory/` doesn't exist, halt and defer; do not try to scaffold from this skill.
- **Putting in-flight TODOs into the Memory Snapshot block.** The snapshot summarises committed knowledge (`.memory/INDEX` files) plus the explicit anti-amnesia handoff fields (`.scratch/tasks/T-*.md` `Next session pickup`). Working TODOs that haven't been written to a task file yet stay in the chat; they belong in `.scratch/inbox.md` or a new `T-NNNN-*.md`, not in the snapshot.
- **Paraphrasing TL;DRs in the snapshot.** Byte-identical TL;DR is the read contract. A paraphrased TL;DR breaks string search, breaks future agents' ability to cross-reference, and quietly degrades the corpus.
- **Re-running the skill on every turn.** A snapshot is per-task-context, not per-message. If the user asked one follow-up question and nothing in `.memory/` has changed, the previous snapshot is still authoritative -- don't re-snapshot reflexively.
- **Skipping the audit when the answer is "obvious".** The audit catches silent decision violations precisely when the answer feels obvious -- that's when an agent skips checking. If the audit reveals no conflict, the audit cost was a one-line note ("[OK] no Active Decision conflicts"); cheap.

## See also

- [`init-native-memory/SKILL.md`](../init-native-memory/SKILL.md) -- companion (scaffolding side). Run this first to install the protocol; this skill assumes the install is already done.
- [`init-native-memory/assets/memory/README.md`](../init-native-memory/assets/memory/README.md) -- the agent-facing protocol that this skill activates. The Progressive disclosure tier table, the Compaction soft caps, the Status field semantics, and the Cross-repo conventions all live there; this skill just enforces the read order and the audit.

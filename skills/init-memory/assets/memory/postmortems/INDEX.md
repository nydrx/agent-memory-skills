# Postmortems Index

Production-impact incidents with root cause + fix + recurrence guard status.

If `Recurrence guard != in-place`, the prevention measure is missing and a
similar incident can recur -- investigate before assuming the original fix is
still active.

The `TL;DR` column is the L1 read surface (see `../README.md` → Progressive
disclosure); keep it byte-identical to the entry's `## TL;DR` block. Soft cap:
20 active entries — over that, run a cluster sweep (see Compaction).

## Active

| Date | Title | TL;DR | Severity | Recurrence guard |
|------|-------|-------|----------|------------------|
| _no entries yet -- copy `../templates/postmortem.md` to `YYYY-MM-DD-<slug>.md` to add one_ | | | | |

## Resolved with guard removed / Recurrence-detected

| Date | Title | Severity | Notes |
|------|-------|----------|-------|
| _none_ | | | |

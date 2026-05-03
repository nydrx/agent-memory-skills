# YYYY-MM-DD: <incident title>

- Status: active | resolved | recurrence-detected
- Severity: <low | medium | high | prod-impact>
- Owners: <names / teams>
- Recurrence guard: in-place | partially-in-place | removed | not-applicable
- Subsumes: <older entries this aggregate replaces, if compaction-aggregate>

## TL;DR

<<= 2 lines: symptom + root cause + whether guard is in place. Copy verbatim
into the INDEX TL;DR column in the same commit. This is what L1/L2 readers
see; if missing or stale, progressive disclosure breaks.>>

## Symptom

<what users / monitors saw>

## Timeline

- HH:MM: ...

## Root cause

<one paragraph; cite specific commits / configs>

## Fix

<what was actually changed; link MR / commit>

## Prevention

<concrete guard rails; if `Recurrence guard != in-place`, list which guard is
missing and why>

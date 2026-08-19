---
name: memory-triage
description: Audit memory files for staleness, redundancy, and readiness to graduate into a rule, doc, or code comment. Use periodically, when the memory index has grown noisy, or when several memories look ready to promote. In a repo with its own memory-triage skill, defer to that one.
user-invocable: true
---

# Memory Triage

Memory is a staging area. Triage decides what stays, what goes, and what has earned a permanent home. Every entry is paid for on every session that loads the index, so bloat is the real cost of keeping things "just in case."

## Dispositions

**KEEP** · **CULL** · **CONSOLIDATE** (several memories circling one concept) · **GRADUATE** (earned a permanent home).

## Reading the signals

Age is a signal, not a verdict — a memory can be old and load-bearing, or fresh and redundant.

- Content already in a doc, rule, or skill → usually CULL as redundant.
- **Unless the behavior was violated anyway** → then GRADUATE. The memory isn't redundant, it's evidence that the existing rule has a trigger gap. This is the distinction most worth getting right.
- Referenced file, function, or flag no longer exists, or the memory contradicts current code → CULL. Verify by reading the thing, not the memory.
- Cited repeatedly across sessions → GRADUATE.
- Resolved incident or concluded experiment → CULL.

By type: **feedback** graduates most often and goes stale least. **project** decays fastest — check whether the work actually closed. **reference** breaks when the external system moves; verify the pointer. **user** rarely needs culling, sometimes needs updating.

## Graduating

**A graduation is an admission, not a move.** Memory's bar is deliberately low, so clearing it says nothing about clearing the bar of a rule or doc. Run it through `/knowledge-intake` — which can legitimately answer *remain*: a recall-triggered or person-scoped fact is already in its correct tier, and declining a graduation is a verdict, not a failure.

**Scope the target to the most specific place where it universally applies.** A correction relevant only to one workspace goes in that workspace's `CLAUDE.md`; one that applies to all coding work goes at that level; only genuinely universal guidance goes global. Don't default upward.

**If the memory is anchored to a specific code site — a gotcha someone trips over while editing that exact spot — the target is a comment there, not a doc.** The test: would a future editor of that line make the mistake without ever opening the doc you'd otherwise write? If yes, the doc is the wrong shelf.

## Before deleting anything

**Re-verify the citation yourself.** If a search was delegated, treat "already covered by X" as a claim, not a finding — fabricated coverage citations are common, and the cull is irreversible. Grep the cited location before removing the memory it justifies.

Don't consolidate memories that are only superficially similar; a gotcha and a workflow serve different purposes.

---
name: reconcile
description: Decide what to do with candidate work — findings, ideas, a single "we should file this" — by checking it against what already exists, before creating any tracker item. Use whenever you're about to say "should I file this?", and for whole finding-sets from an audit or walkthrough. In a repo with its own reconcile skill, defer to that one.
user-invocable: true
---

# Reconcile

Owns the decision, not the execution. Stops at an approved disposition.

The most expensive issue is the one duplicating an epic nobody knew existed; the second is the build issue filed for work whose shape is still an open question. Both come from creating before reconciling.

## Survey three axes — always, scaled to the set

**Issue graph.** Search across at least three vocabularies: the underlying problem, the affected system, the user-facing symptom. The original author used different words than you. Read neighbor *bodies*, not titles — titles miss half the overlap. Read every result touching a file your candidates would change.

**Code.** Name the closest existing mechanism. Extend before building new. Cast past application code — build config, hooks, CI, and env files are mechanisms too, and "the app doesn't do this" isn't the same finding as "this repo doesn't do this."

**File history — required, not a flourish.**

```bash
git log --follow -- <file>          # why this file looks the way it does
git log -L <start>,<end>:<file>     # why one function does
```

Topic search finds issues; file search finds decisions. A "gap" is often a deliberate prior scope-down, and the reasoning lives in the commit message — reachable only from the file it touched. Current code reads fluently and gives no hint it was a decision rather than an accident. That reframing flips dispositions outright: what looked like New becomes Drop.

## Dispositions

Exactly one per candidate. An item without one isn't reconciled.

**Maps #N** (exists — link) · **Groom #N** (exists but thin or stale — fold the evidence in) · **New** · **Subsume** (several collapse into one) · **Design-gated** (an open design question blocks it — route to the design question, don't file build work) · **Drop** (+ reason: out of scope, not real, or correct-but-pointless) · **Discuss**.

**Discuss is the easiest row to talk yourself out of**, because every other row is available and picking one feels like finishing. The test isn't how uncertain you are — it's *where the answer lives*. If settling it requires knowing why a mechanism was built, what it's for, or whether it should exist at all, the answer is in someone's head and no amount of surveying reaches it. Design-gated is the near miss: it also defers, but it presumes the thing should exist and only the shape is open.

## One candidate

Same disposition, no ceremony: **one line, not a table.** "Drop — mooted by the pending decision." "Groom #1793 — same gap, add the measurement."

**Drop is a complete answer, not a failure to find work.** Most single observations should drop, and saying so beats filing to look thorough.

**Never surface an intention to file for someone else to adjudicate.** "Should I file this?" is the question this answers — asking it out loud means it didn't run. Report the disposition instead. Discuss isn't an exception, it's a different answer: state the question itself, phrased so it can be answered without first asking you what you found.

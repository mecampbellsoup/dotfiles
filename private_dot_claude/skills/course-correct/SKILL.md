---
name: course-correct
description: Diagnose a live gap between what the user expected and what Claude did, then fix it durably. Use when the user's phrasing names that gap — "why didn't you...", "why aren't you...", "this keeps happening" — or on explicit /course-correct. Not for ordinary redirects ("hold on", "let's reconsider") that name no gap. In a repo with its own course-correct skill, defer to that one.
user-invocable: true
---

# Course Correct

A correction that recurs after it's been documented means the rule exists but isn't firing. Turn the live catch into a durable fix now, rather than an acknowledgment that evaporates by next session.

## The one thing that makes this work

**Assume the docs conflict or are incomplete. Hold that until it's disproven.**

"I wasn't paying attention" is the explanation that requires no investigation, which is why it gets reached for first — and adopting it ends the diagnosis before it starts, because the remedy becomes "try harder." That fixes nothing.

The falsifiable version is cheap. Read every doc governing the thing you got wrong *against each other*, not just the one you already followed:

- two docs giving opposite guidance for the same situation
- one doc pulling both ways, with no ordering between them
- an instruction whose plain reading differs from its intent
- a rule stated in a parent doc and missing from the specialization actually read — check this last, because it flatters you most

Start with the always-loaded context (`~/.claude/CLAUDE.md`, the project's `CLAUDE.md`, rules files), not the doc you happened to be following. Those were in context regardless of which skill ran. A check that skips them can only find defects — it has no way to *exonerate* the docs, so "documentation defect" becomes the guaranteed output rather than a finding.

Expect an incomplete check to produce the more flattering answer. What's being diagnosed is your own miss.

Coherent, complete docs are a real finding that relocates the cause — not a failed check, and not a cue to keep digging until something turns up. Say which docs you actually read.

## Then

State the gap in prose — expected, actual, rule violated (or none) — before touching a tool.

A rule that existed and was violated anyway is the highest-value case: it's under-triggered, not missing, so the fix is wording and placement, not new content.

Before proposing anything, check whether some mechanism already covers it — a hook, a script, an existing rule — and grep for the *pattern* rather than the symptom, so the fix isn't narrower than the problem.

Propose the change and its net line delta, and **get approval before executing**. Appending always looks available; accumulated instructions compete with each other. If nothing is being replaced or deleted, look again.

A first incident goes to memory, not to an always-loaded file — one occurrence isn't evidence for a rule every future turn pays for. It graduates on recurrence.

## Related

The live, single-incident path. `/conversation-audit` sweeps retrospectively for patterns across sessions; `/wind-down`'s skill lens does it at session end. Fixing it here pre-empts both.

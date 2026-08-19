---
name: crystallize
description: Turn an amorphous feature request into a clear picture of the whole thing, then cut it into atomic, dependency-ordered units that each ship independently. Use when work spans multiple PRs or subsystems, or when an epic needs decomposing. In a repo with its own crystallize skill, defer to that one.
user-invocable: true
---

# Crystallize

Form the **gem** — the complete feature as a user would experience it — then cut it into **crystals** that each ship on their own.

## Imagine the ideal before re-grounding against code

Sketch the gem before opening a single file. Product decisions get made for what's best for the product, not for what the current implementation makes easy.

Only then re-ground against the code — and re-grounding's job is to find what has to change to get there, not to talk the gem down to what's already convenient. A constraint discovered while re-grounding is real information: it changes the plan, the ordering, even which crystal comes first. It does not get to shrink the gem. If a found constraint makes you want to write "well, we could just keep doing X," that's the implementation talking — name it as something an early crystal must prove out or route around, and keep the gem describing the experience you'd want if the constraint didn't exist.

## Decompose from the gem, not from existing issues

Existing child issues are **hypotheses**, not starting points — reading them first anchors your seams to theirs. One exception: prior PRs carrying review feedback are **evidence**, because review comments record what actually didn't fit.

If someone who only read the source issue could have written your gem, it's an echo. The depth should come from seeing further into the problem.

Verify the epic's own claims before anchoring to them. "There's no way to do X today" may have been true when it was written. A thirty-second grep is cheap; decomposing around stale framing is expensive.

Define two or three **tenets** alongside the gem — "when X and Y conflict, prefer X" — so crystal-level calls don't each escalate.

## The first crystal proves the foundational assumption

Every epic rests on one load-bearing belief. Prove it with the cheapest experiment that could falsify it, and let the shape follow the question rather than convention: existing data → a query, often no crystal at all; unstructured history → an analysis pass; live events only → instrumentation; "does this capability even work" → a spike, one call, eyeball the output. A dashboard measures trends in live events; it can't tell you whether the signal exists in data you already have.

Measure the **unmet need**, not adoption of the current workaround.

## Depth scales with distance

The next one or two crystals get full specs. Further-out ones get a name, a dependency, and rough scope — detailed specs for distant work are false precision, since the code and the requirements will both move. Where scope depends on what an earlier crystal finds, say so.

Resolve conditionals before filing. "If X exists…" is a finding deferred to the reader; if a grep answers it in under a minute, answer it.

**Crystals describe what and why. How belongs in the plan**, when the crystal is picked up.

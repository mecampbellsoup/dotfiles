---
name: plan-implementation
description: Produce a reviewable implementation plan before writing code, scaled to the work — a few sentences for a small fix, a written doc for a decision-dense change. Use when starting a feature or multi-file change, or when the user says "plan this" / "how should we implement this". In a repo with its own plan-implementation skill, defer to that one.
user-invocable: true
---

# Plan Implementation

**The question: do we agree on what's being built, how, and why — before starting?**

The plan is proportional, never optional. A single-file fix gets a few sentences in conversation; a decision-dense change gets a written doc.

**Which form it takes is yours to decide and state — not a question to hand over.** "I'd skip the doc, say the word if you'd rather have one" reframes a required phase as an opt-in extra. Name the form and the reason in one line, then keep going.

## Written doc when it's decision-dense, not when the diff is big

Line count is the wrong axis — a two-file change can carry three independent ways to be silently wrong. Any one of these is enough:

- **One contract wired at more than one call site.** Wiring one and missing another is the characteristic failure, and a written enumeration is what catches it.
- **A decision whose wrong branch is silently plausible** — two similar-looking objects, where picking wrong still runs clean and records false data.
- **A correctness trap a naive test passes anyway** — logic at the wrong point in a sequence, where a single-item assertion succeeds and only the multi-item case exposes it.

## Before planning

**Sketch the shape first** (`/design-flow`) unless it already happened this conversation. The plan operates on the mental model the sketch builds.

Then gather what you need to design to the project's own standards — you can't extend existing patterns you haven't looked at. Related issues read in full, prior PRs' actual diffs, architecture docs, the closest analogous feature already implemented, and the test patterns used for this kind of work.

State the alternatives you rejected and why. A plan that presents one option hides the decision it made.

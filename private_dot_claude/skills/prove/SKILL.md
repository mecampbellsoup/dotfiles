---
name: prove
description: Run an experiment to learn what "correct" looks like, before writing code that assumes it. Use when facing a genuine unknown (unfamiliar API, external service, performance feasibility) or an unverified assumption (a framework default, a constraint, "this looks wrong"). Not for investigating something already broken — that's /evidence. In a repo with its own prove skill, defer to that one.
user-invocable: true
---

# Prove

Test-first assumes you know what correct looks like. Some work has genuine unknowns — you can't write a test for behavior you haven't observed. Run it, look at what comes back, *then* lock it in.

    measure (/evidence) → experiment (/prove) → lock in (a test)

## Two families, one discipline

**Unknowns — "what does correct even look like?"** First use of an unfamiliar API. Structured or binary output that needs inspecting rather than asserting. Feasibility questions where the timing data drives the design. Anything owned by an external service.

**Assumptions — "I think I know, but haven't checked."** Framework and library defaults, before writing code that duplicates what you already get for free. Something that *looks* wrong, before fixing it — prove it causes a real problem first. Each constraint an approach depends on, proven independently before building on it.

Both ask the same thing: observe the actual behavior before committing code to it.

## Not this

If the expected behavior is already well-defined, skip straight to writing the test. If something is broken and you're diagnosing why, that's `/evidence` — this skill is for learning what correct *is*, not for finding out what went wrong.

The experiment is scaffolding. Once it has told you what correct looks like, that knowledge belongs in a test that outlives it.

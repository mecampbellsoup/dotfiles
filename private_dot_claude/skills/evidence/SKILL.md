---
name: evidence
description: Instrument and measure before changing anything. Use when investigating a bug, evaluating an optimization, or checking a comparative claim — and especially when you catch yourself about to say "probably", "should", or "I think I see the problem". In a repo with its own evidence skill, defer to that one.
user-invocable: true
---

# Evidence

## Observe-only, until told otherwise

While this is active: **no fixes.** No refactors, no cleaning up along the way. Instrumentation stays in until the user says to move to implementation. "I think I see the problem" is not permission to start fixing — state the hypothesis, say what would confirm or falsify it, then go get that.

Outputs are limited to instrumentation, observations, hypotheses, and predictions.

**A confirmed diagnosis is not a proven fix.** Evidence explaining *why* something happens says nothing about whether the specific remediation works — that's a new claim the diagnosis never tested. When a fix is identified, the next step is verifying that exact remediation live (rerun the reproduction against it, hit the real endpoint, reload the real page), not "want me to implement this?" Assume the mental model is still missing something until the fix has been *observed* working, not reasoned into working.

## The loop

**Assess first.** What's known — confirmed by logs, output, screenshots. What's unknown. Which assumptions would be dangerous if wrong. And challenge the premise: given what's known, is this actually worth solving? Don't accept the framing uncritically.

**Instrument before changing.** Add the diagnostics that would distinguish between candidate explanations — including one that would *falsify* your favored theory, not only confirm it.

**Make hypotheses observable.** "If X is true, then Y should be observable" — specifically enough that Y is something you can actually go look at. Vague hypotheses don't discriminate.

**Predict before changing code.** What will change, what should stay unchanged, what observable behavior confirms success. This is what makes a regression obvious instead of mysterious.

**Instrument the fix too, not just the investigation.** Add diagnostics to the new path so its first run confirms it executed with the values expected. Don't ship a change and then run it blind.

**One logical change at a time.** One selector, one property, one branch, one signature. Two changes at once and the evidence stops attributing.

**Re-anchor when confused.** If the observations stopped making sense, stop changing things and go back to what's actually confirmed. Confusion is a signal that a premise is wrong, not a cue to try more variations.

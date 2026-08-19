---
name: content-standards
description: Standards for any prose a person or agent will read later — a doc section, an issue or bug report, a PR body, a findings write-up, a README, a Slack summary, a living-doc entry. Load before drafting, not after. "It's just a quick paragraph" is the case that ships weak content. In a repo with its own content-standards skill, defer to that one.
---

# Content Standards

**The goal: a reader picks this up cold and walks away fully contextualized.** No hunting, no hallway conversation, no "you had to be there."

Write for an agent reading it with no institutional memory. That's the forcing function — an agent has no shortcuts, so if one can start doing useful work from your content, a human certainly can. The reverse doesn't hold: content that "works for humans" fails agents silently, because humans paper over the gaps without noticing.

## Standards

**Summarize and link.** Key insight in a sentence or two, then point to where the depth lives. Punchline first, depth optional.

**Inline what won't survive.** If the context lives in a scratchpad, a working file, or this conversation, extract it into the content. Never reference a session-local file by path — it won't exist tomorrow.

**Narrate a high-signal reaction as a story, not a summary.** When someone walks through a failure or a rationale in detail — what they did, what they saw, in what order — that sequence *is* the evidence. Compressing it to one clause and leading with your own analysis throws away exactly what a reader would need to verify or reproduce it. Let the story lead and the mechanism follow, tied back to specific moments. Break the sequence into one beat per line; a story is still a wall of text as a dense paragraph.

**Link to specifics, not neighborhoods.** The exact comment where a decision was made, with what was decided — not "see the thread."

**Show the thing.** Code, output, before/after. Not prose describing them.

**Punts need trigger conditions.** Not "we deferred X" but "when Y happens, revisit." Each punt is a trail marker.

## The defaults to fight

- **The context-free drop** — states what it wants, never why. The reader has no mental model to attach it to.
- **The solution masquerading as a problem** — an implementation stated as the requirement, foreclosing better options. A borrowed solution launders just as easily as an invented one: citing how someone else did it is evidence, not a requirement.
- **The orphan** — no links backward to what caused it or forward to what follows.
- **The stale description** — contradicts current state. Worse than nothing; it actively misleads.
- **The prose wall** — three paragraphs where five lines of output would do.
- **The summarized reproduction** — the walkthrough becomes a parenthetical and the analysis leads. Now the analysis can't be checked against what actually happened.

## Before publishing

Could someone start work from this alone? Does it link to the work around it? Am I showing or describing? Will every reference still resolve tomorrow? If someone handed me a detailed reproduction, is it still in here in their order and close to their words?

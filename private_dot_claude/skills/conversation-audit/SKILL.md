---
name: conversation-audit
description: Mine past conversation history for recurring patterns worth turning into rules, skills, or memories — repeated corrections, re-explained context, friction. Use periodically, or when the same friction keeps recurring and isn't captured anywhere. In a repo with its own conversation-audit skill, defer to that one.
user-invocable: true
---

# Conversation Audit

The retrospective counterpart to `/course-correct`, which handles one incident live. This looks across many sessions for what recurs.

## What to look for, in priority order

| | |
|---|---|
| **Repeated corrections** | The user says no, pushes back, redirects. Highest value — a rule needs writing or sharpening. |
| **Repeated context** | The same thing re-explained across sessions. Should be durable somewhere. |
| **Friction** | Thrashing, high interruption rate, visible frustration. Often a missing skill. |
| **Wrong assumptions** | Stated confidently, corrected by the user. |
| **Recurring workflows** | The same multi-step sequence, repeatedly. Candidate for a skill. |
| **Domain knowledge** | The user teaching how something works. Usually docs. |

**Only patterns appearing more than once count.** A single correction is noise; that's what `/course-correct` is for.

## Then

Run `/knowledge-intake`'s survey before proposing anything — it owns the sweep for prior art and the check for whether a mechanism already enforces the thing. A pattern a hook already catches is the system working, not a finding.

Map what you found onto its dispositions: genuinely new → **Admit** (or **Displace**, if it supersedes something — name what goes). Exists but was violated anyway → **Amend** the wording or placement, don't add a neighbor.

**Audit for pruning too, not just addition.** Adding without removing makes the instruction set worse — every rule competes with every other for attention. A finding that some rule should be *deleted* is as valuable as a new one.

Propose; don't batch-implement. The judgment calls are the point.

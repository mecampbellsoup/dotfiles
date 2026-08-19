---
name: knowledge-intake
description: The gate before knowledge is written anywhere durable — a CLAUDE.md rule, a doc section, a skill, a memory, a living doc. Decides whether it belongs at all and which tier it goes in, and waits for approval before anything is written. Use whenever you're about to add a rule or append a section, and when /course-correct, /wind-down, or /conversation-audit surfaces something worth keeping. In a repo with its own knowledge-intake skill, defer to that one.
---

# Knowledge Intake

Owns the decision, not the writing. The tell that this should have fired and didn't: content landed somewhere durable and nobody asked what it replaced.

## Two questions nothing else asks

**Is a mechanism already enforcing this?** A hook, script, linter, or test that already makes the failure loud or impossible needs no prose. Check by reading what actually fires, not by name — prose describing a mechanism reads exactly like a rule, which is why this is the highest-yield check and the most skipped.

**What event should make this content appear?** That answer picks the tier. Routing by what content is *about* is topic-matching — "it's about testing, so it goes in the testing doc" ignores that the testing doc only loads when someone opens it. If you can't name a trigger, the candidate isn't ready.

Then search for the rule, not its wording. Two independent statements of one rule drift apart and the reader can't tell which is current.

## Dispositions

Assign exactly one:

| | |
|---|---|
| **Covered** | Already stated. Link or point; name the existing statement. |
| **Amend** | Something existing is wrong, thin, or stale — edit it in place. Preferred over adding a neighbor. |
| **Admit** | Genuinely new. Name its tier; a tier is mandatory, not a follow-up. |
| **Displace** | Admit it, and name what it removes. |
| **Remain** | Already in the right tier — leave it. Declining a graduation is a verdict, not a failure. |
| **Drop** | Don't write it. Not durable, or a mechanism covers it — say which. |

Admit and Displace are the same verdict with different honesty. Prefer Displace whenever something is genuinely superseded; where an Admit truly adds without removing, say so out loud. A run of net-additions is invisible one change at a time and obvious only in aggregate.

## The always-loaded tier needs recurrence

`~/.claude/CLAUDE.md` and unscoped rules have no router — every session and every subagent pays for them on every turn. A first incident routes to memory and graduates on a second occurrence. Conviction fresh from an incident writes prose that doesn't hold; nothing is lost by staging it.

## The gate

**Present the table and wait. Write nothing before approval.**

A session that just spent an hour learning something is the worst available judge of whether it deserves permanence. The conviction is real and is not evidence. Approval may change dispositions — the table is an argument, not a verdict.

Writing the rule first and dispositioning after makes every verdict except Admit read as wasted work, and turns the gate into a formality.

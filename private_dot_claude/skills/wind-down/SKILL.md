---
name: wind-down
description: End-of-session hygiene check plus structured reflection — replay what happened, look for skill gaps and missed invocations, decide what (if anything) is worth making durable. Use when the user signals the session is ending ("wrapping up", "anything left?", "good for today") or invokes it explicitly. In a repo with its own wind-down skill, defer to that one.
user-invocable: true
---

# Wind Down

## Never start this on a hunch

A finished task, a long context, or no obvious next step are pauses, not endings. Reading them as endings is the failure this gate exists to prevent. Start only when the user says so, or when a routing skill determined it against a condition it could actually check.

## Hygiene

Report what needs attention; skip clean categories silently.

- Uncommitted or untracked changes that should be committed or reverted
- Temp files left at the project root — plans, review scratch, screenshots
- Whatever else this project's own state requires (open PRs, branch, containers, test config)

**Before any destructive cleanup, check whether another session is live in this workspace** (`ListAgents`, plus the OS if it comes back empty — registration lags). A sibling's in-flight work looks identical to your own residue. Report shared state; don't act on it.

## Replay

Reconstruct the arc in writing — 3–6 bullets. Skip this and you will miss everything else.

Task and outcome · skills invoked · where the session branched · where it stuck, retried, or needed correction · what unblocked it.

**Name the pivots explicitly.** A user correction, a failed approach, a tool that revealed something unexpected, a question that reframed the problem — that's where the learnings are.

Glancing at the session, thinking "nothing jumps out," and writing "routine session" is the most common failure here. The replay is what prevents it.

## Look for gaps

Against everything loaded, not just this project's workflow:

- **Gaps** — should a skill have fired and didn't? Was there repeated manual work a step could encode? Something figured out mid-session that a skill should already say?
- **Missed invocations** — would each skill's author expect it to fire here? Walk the session's *side outputs*, not the headline work: prose written, docs touched, things shipped ad hoc. The main workflow usually invokes its skills correctly; the quiet acts around it skip their owners.
- **Thrash** — repeated failed attempts, tool calls that clearly weren't working but continued, investigative spirals a single check would have cut short. Name what triggered it and what would have prevented it. These are the highest-signal findings.

A gap has to be real and recurring to warrant changing anything. Zero candidates is a common, correct result.

## Route what's left

**The deliverable is the analysis, not a set of changes.** Whether anything needs to change is downstream and conditional. Most sessions should end with few or zero edits — "analyzed it, nothing here warrants a change" is a complete outcome. Optimizing for edits shipped manufactures churn that makes future sessions worse.

Analyze deeply; change conservatively. Under-analyzing and over-acting are different failures, and the second is the one that feels productive.

When something does warrant action, compose `/knowledge-intake` for the disposition and tier, and prefer the skill that owns the work over an improvised edit. When in doubt, write the learning into this output and change nothing — it will resurface and earn its change if it's real.

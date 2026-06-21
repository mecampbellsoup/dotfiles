---
name: review-documentation
description: Review the documentation web for any project — CLAUDE.md, memory files, living docs, skills, or docs/ — checking for redundancy, gaps, coherence, and cross-references. Invoke proactively before adding new docs or after editing CLAUDE.md or memory files. Invoke reactively when the user asks "is this documented?", "should we document X?", or "review the docs". For Swarf repos (has .claude/skills/), defer to swarf-toolkit:review-documentation which adds doc-drift-detector, skill chaining audit, and deeper tooling.
user_invocable: true
---

# Review Documentation

A doc web review for any project context. The goal: each concept lives in exactly one place, each doc stands alone as a complete mental model, and the web is navigable.

## Detect the landscape first

Before reviewing, identify what doc layers exist in this project:

```bash
ls .claude/skills/ 2>/dev/null && echo "swarf"   # → defer to plugin
ls ~/personal/*.md 2>/dev/null                    # personal workspace
ls docs/ CLAUDE.md README.md 2>/dev/null          # generic project
```

**If this is a Swarf repo** (`.claude/skills/` exists): stop and invoke `swarf-toolkit:review-documentation` instead — it adds doc-drift-detector, skill-chaining audit, and drift detection that this skill doesn't replicate.

**Personal workspace** (`~/personal/`): the web is CLAUDE.md + `~/.claude/CLAUDE.md` + memory files + living docs (`~/personal/*.md`).

**Generic project**: the web is CLAUDE.md (global + project) + `docs/` + any `*.md` at root.

## The process

### 1. Survey

List every doc layer present. Name the hierarchy — what's the entry point, what routes to what, what's load-bearing vs. supplementary. Don't react yet; just map.

### 2. Coherence

Two lenses, both required:

**Node quality** — for each doc in scope, ask: could a reader land here cold and walk away with a complete mental model? Does it carry its own reasoning, current state, and constraints — or does it assume context the reader doesn't have?

**Web quality** — across docs, ask: is the same concept explained in more than one place? Does each concept have exactly one canonical home? Are there gaps where something is referenced but never defined?

### 3. Connections

Are related docs linked to each other? Can a reader traverse from any doc to related work? Flag missing cross-references and orphaned docs (exist but nothing points to them).

### 4. Propose

One concrete change at a time — a diff, not a description. Common actions:

- **Consolidate**: merge two docs that cover the same concept
- **Redirect**: replace duplicated content with a pointer to the canonical source
- **Add cross-reference**: link two docs that should point to each other
- **Split**: a doc that's trying to be two things
- **Remove**: content that's stale, temporal, or covered elsewhere

Show the exact lines to add, remove, or modify. "This section could be clearer" is not a proposal.

## When to invoke

**Proactively** — before adding a new doc or memory file (would this create redundancy?), after editing CLAUDE.md (did this create drift?), after a session that produced multiple new living docs.

**Reactively** — user asks "is this documented?", "should we document X?", "clean up the docs", or invokes `/review-documentation`.

## Anti-patterns

- Proposing additions without checking for existing coverage first — grep the web before writing anything new
- Vague proposals ("this could be clearer") — always produce a diff
- Reviewing without reading — check the actual file content, not just the filenames
- Adding to CLAUDE.md when a memory file or living doc is the right home — CLAUDE.md is for universal, durable rules only

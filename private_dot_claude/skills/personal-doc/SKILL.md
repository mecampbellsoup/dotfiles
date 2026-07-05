---
name: personal-doc
description: Create or update a living doc in ~/personal/ for any substantive personal topic — health insurance, GC search, trip planning, investments, home projects, vendor negotiations, purchases, decisions, etc. PROACTIVELY invoke this after any action that changes real-world state — email sent, site visit done, appointment confirmed, item ordered/returned, decision made — without waiting to be asked. Also invoke when Matt says "update X", "log that", "track this", "mark X done", or "note that." This applies just as much to patching a few fields into an already-existing doc as to creating a new one — "log these findings," "add this to the doc," or "update the Now table" are all triggers, not just first-time creation. Default to a living doc for anything involving a real-world entity or evolving state. Memory is for trivial atomic facts only; behavioral corrections go to ~/.claude/CLAUDE.md. The goal is that completing an action and updating the doc feel like one step, not two. Do not substitute a direct Edit/Write tool call on a ~/personal/<topic>.md file for this skill, even for a small, well-understood patch — invoke the skill first.
user_invocable: true
---

# Personal Doc

Structured living docs in `~/personal/` for personal topics that are too multi-dimensional for a flat memory blob. The file is always source of truth; memory is just the pointer.

**This skill is the only path to editing a `~/personal/<topic>.md` file — not a suggestion to consider alongside a direct Edit/Write call, but the required path.** Reaching for Edit/Write directly, even on a change that seems small or fully understood, skips the memory-hygiene step and the Now/Coming Up/Decide/Log conventions this skill enforces, and the doc's structure drifts over time as a result. If the task is "patch these findings into an existing doc," that's still a full invocation of this skill's Update flow, not a shortcut around it.

## The gate

Before doing anything else:

1. Does `~/personal/<topic>.md` already exist?
   - **Yes** → go to **Update**
   - **No** → go to step 2

2. Is this substantive? A topic is substantive if it involves a real-world entity (vendor, person, trip, purchase, project, decision) or facts that might evolve or need context later.
   - **Yes** → go to **Create**
   - **No** (truly trivial — a phone number, a login alias, a fact so stable it will never need context) → memory is fine, stop here

**Default to a living doc.** The bar for "trivial" is higher than it sounds — if uncertain, create the doc.

**Routing quick-reference:**

| What | Where |
|------|-------|
| Substantive topic (vendor, project, trip, purchase, person, decision) | Living doc in `~/personal/` |
| Behavioral correction (how Claude should act) | `~/.claude/CLAUDE.md` rule |
| Trivial atomic fact (phone number, login alias, one-liner that won't evolve) | Memory |

## Template

When creating, use this literally — populate only what you actually know, no placeholders:

```markdown
# <Topic Title>

_Last updated: YYYY-MM-DD_

## Now

| Who / Track | State | Since |
|-------------|-------|-------|

## Coming Up

| Date | What | Who |
|------|------|-----|

## Decide

- [ ] open question or decision

## How to Reach

Non-obvious contacts, portals, accounts only — skip anything findable in 5 seconds.

## Log

### YYYY-MM-DD — what happened
One tight paragraph. Newest first.
```

## Create

1. Write the file from the template above
2. Add a row to the **Active living docs** table in `~/personal/CLAUDE.md`
3. Replace the memory body with: `See ~/personal/<topic>.md` (keep frontmatter intact)
4. Commit: stage CLAUDE.md + all new topic files together — batch commits are fine when creating multiple docs in one triage pass

## Update

Read the file first, then patch exactly what the action warrants:

- **Memory hygiene:** If a memory file exists for this topic and has content below the `See ~/personal/<topic>.md` pointer line, trim it to pointer-only. The living doc is the source of truth; the memory file should never accumulate content.
- **Now** — update any track whose state just changed
- **Coming Up** — drop rows that passed, add new ones
- **Decide** — check off resolved items, add new ones
- **How to Reach** — add only if something changed (new contact, new portal)
- **Log** — prepend `### YYYY-MM-DD — <what happened>`, one paragraph

Commit.

## Close

When a topic fully resolves:

1. Prepend `## CLOSED YYYY-MM-DD — <outcome>` above **Now**
2. Remove the row from the Active living docs table in `~/personal/CLAUDE.md`
3. Move the file to `~/Documents/<domain>/` per File Lifecycle rules in CLAUDE.md
4. Drop the memory entry
5. Commit

## Anti-patterns

- Don't pad the template with placeholder rows — only what you know
- Don't let Log become a wall; summarize resolved threads before appending
- Don't leave the old memory blob intact after converting — pointer only
- Don't forget to remove the CLAUDE.md table row on close
- **`~/Documents/` files are out of scope** — `~/personal/` is for everything Claude writes and maintains; `~/Documents/` is read-only archive (PDFs, statements, received docs). If asked to update a file in `~/Documents/`, it probably belongs in `~/personal/` instead — move it there, then update it via this skill

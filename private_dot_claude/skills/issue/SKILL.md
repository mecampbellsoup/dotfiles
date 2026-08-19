---
name: issue
description: Create or groom GitHub issues. Use when filing a bug or feature, grooming an existing issue against current state, or sweeping an epic's children. In a repo with its own issue skill, defer to that one.
user-invocable: true
---

# Issue

The executor. `/reconcile` decides *whether* and *what*; this writes it.

## Reconcile first

Run `/reconcile` before writing anything — it owns the survey and the disposition, and it gates creation on approval. This skill then acts on the approved result: create the New and Subsume rows, groom the Groom rows in place, link the Maps rows. Skipping it is how duplicate epics and design-blocked build issues get filed.

```bash
gh issue list --search "keywords" --state open --json number,title --limit 10
gh issue view NUMBER --json body,title,state,labels
gh issue list --search "child of #EPIC" --state open --json number,title
```

## Writing one

Compose `/content-standards` — an issue body is read cold by someone with no memory of the conversation that produced it.

**State the problem, not a solution.** "Create a shared module at X" is an implementation; the problem is the duplication, and naming the fix as the requirement forecloses better ones. Citing how another system solved it is evidence for the Context section, not a requirement to inherit.

**Describe what and why; leave how to the plan.** An issue defines an observable capability and its success criteria — not function signatures, file paths, or architecture.

**Ground claims in specifics.** "Creation happens through `resource_create()` at `views.py:1890` — no other path exists" is grounded. "Resources are created via the modal" is echoed framing.

When grooming, fold new evidence in additively and preserve the original author's voice rather than rewriting around it.

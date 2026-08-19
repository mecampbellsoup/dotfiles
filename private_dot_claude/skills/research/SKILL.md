---
name: research
description: Answer a question by going to the source instead of guessing. Use when you catch yourself speculating about how something external behaves ("it probably works like...", "should work"), when the only answer found so far is a blog or forum post, or before modifying an unfamiliar system. In a repo with its own research skill, defer to that one.
user-invocable: true
---

# Research

Research before acting is cheaper than debugging after. The finish line is being able to say "I know X because I verified it in Y" — not "I think X because I read somewhere that Y."

## Source hierarchy

Authoritative: **source code** (how it actually works) → **official docs** (intended behavior) → **changelogs** (what changed, and when) → **upstream issues/PRs** (known bugs, edge cases, design arguments).

Hints needing verification: forums, Stack Overflow, blog posts, and any model-written summary of the above.

## Check upstream issues early

When something external looks broken, misconfigured, or limited, search the project's issues *before* instrumenting or reading source. Someone has usually filed it, and the thread often carries the workaround, the root cause, or confirmation that it's a known limitation. Thirty seconds here regularly saves hours.

```bash
gh issue list --repo owner/repo --state open --search "keyword"
```

Applies to anything with a public repo — libraries, CLIs, plugins, MCP servers.

## Traps

- Stopping at the first plausible answer
- Reading one example and generalizing
- Assuming current behavior matches documented behavior — check the version
- **Querying a data store without reading its storage model.** External libraries write data their own way, not the way you'd assume, and a wrong query returns confidently wrong results rather than an error. Read how it writes before writing anything that reads.

Writing a minimal repro to confirm understanding is research, not a detour.

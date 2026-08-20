---
name: codex-sync
description: 'Mirroring CLAUDE.md files to AGENTS.md for Codex, including which repos need a direct-copy sync, the webapp migrate-to-codex script and its required --mcp --subagents flags, why the bare command is destructive (it writes drifting skill copies into ~/.agents/skills), and the rule that the copy must be the LAST step before committing or a rebase silently reverts content. Load whenever a CLAUDE.md is edited in an owned repo, before committing a CLAUDE.md change, and whenever Codex, AGENTS.md, or ~/.codex comes up. Triggers: AGENTS.md, codex, sync agents file, migrate-to-codex, edited CLAUDE.md.'
---

## Codex Sync

**Rule:** Every CLAUDE.md that is NOT inside a `.claude/` directory gets a mirrored AGENTS.md — direct copy, no substitutions — except third-party repos you can't commit to. Run this sync whenever CLAUDE.md changes in any owned repo. **Exception (explicit inclusion):** the global user file `~/.claude/CLAUDE.md` IS mirrored to **`~/.codex/AGENTS.md`** — that is where Codex reads global instructions (CODEX_HOME; verified against the Codex manual 7/1/2026). Do NOT mirror to `~/.claude/AGENTS.md` — Codex never reads that path (an old copy there was removed 7/1/2026). Per-repo `.claude/CLAUDE.md` files (e.g. skill-adjacent) are NOT mirrored.

**Repos needing direct-copy sync** (pattern: `cp CLAUDE.md AGENTS.md && git add AGENTS.md && git commit -m "sync AGENTS.md: ..."`):
- `~/personal/`
- `~/personal/finance/`
- `~/code/swarf/business-vault/`
- `~/code/swarf/webapp/apps/chatbot/` — use `chore:` prefix (commitizen required)
- Webapp worktrees (webapp-dev, webapp-dev-2, webapp-dev-3) — sync the chatbot subdir; picks up at merge

**The copy is a point-in-time snapshot, so it has to be the LAST thing before the commit — re-run it after any rebase that touched the source, and confirm with `diff` rather than assuming.** A `cp` taken earlier and committed later doesn't just go stale: `AGENTS.md` then carries pre-rebase text for lines the rebase changed in `CLAUDE.md`, so committing it silently *reverts* incoming content. This is the snapshot-is-not-a-merge rule under § Workflow arriving through a different mechanism than `git stash` — same failure, no stash involved, and nothing warns. Near-miss 8/14/26 (`apps/chatbot`, PR #1464): a mirror copied before a rebase would have reverted a sibling PR's just-landed wording; caught only by re-running `diff CLAUDE.md AGENTS.md` afterward, which is the cheap check that makes this visible.

**`~/code/swarf/webapp` AGENTS.md + subagents + MCP:** Run the migrate script instead of `cp` (handles "Claude Code" → Codex substitutions). **Always pass `--mcp --subagents`. Never run it bare:**
```bash
python3 ~/.codex/skills/migrate-to-codex/scripts/migrate-to-codex.py \
  --source ~/code/swarf/webapp --target ~/.codex --mcp --subagents
```
This syncs AGENTS.md, 3 MCP servers, and 5 subagents. Run after significant CLAUDE.md, subagent, or MCP changes.

**Its `AGENTS.md` output lands on `~/.codex/AGENTS.md` — the global slot — carrying webapp content.** That is a layer violation, not just a file conflict: `~/.codex/AGENTS.md` holds the *global* mirror of `~/.claude/CLAUDE.md`, while the webapp's own project instructions belong at `webapp/AGENTS.md` (already synced separately). No flag skips the instructions stage, so re-copy the global mirror after every script run: `cp ~/.claude/CLAUDE.md ~/.codex/AGENTS.md`.

**Omitting the flags runs all three stages including skills, which is the one stage you never want.** The skills stage writes *copies* of every `.claude/skills/*/SKILL.md` into `~/.agents/skills/` — and Codex loads that directory, so each copy then shadows the live skill under the same name while silently drifting from it, outliving deletions, and mangling frontmatter (it drops `user-invocable` and leaves a `## MANUAL MIGRATION REQUIRED` stub in the body). Fifteen such copies accumulated between March and July 2026 before being deleted; the bare command recreates all of them. Confirm with `--plan` before any run — a plan naming `stage: skills` means the flags were forgotten.

Codex reaches the webapp's project skills through the repo's committed `.agents/skills/` symlinks instead — no copies, no drift. See `docs/AI_TOOLING.md` § Codex for how that bridge works and when to rebuild it.


## Who I Am

Matt Campbell — software engineer, co-founder of Swarf AI. Lives in Brooklyn, NY with wife Yury. Previously at CoreWeave for 5 years.

## Email Accounts

| Account | Use for |
|---------|---------|
| `mecampbell25@gmail.com` | Personal — taxes, personal finance, personal correspondence |
| `matt@swarf.app` | Work — Swarf business, investors, professional contacts |

**Email ops:** Always use `gog` for all email — both accounts. Never use Gmail MCP; `gog` handles everything and is authenticated to both accounts.

**gog gotchas (recurring):**
- There is no `gog gmail message get` — single message = `gog gmail get <id> --json` (or `gog gmail raw <id>` for lossless API JSON)
- Plain output truncates bodies and drops CC — use `--json` + base64 decode
- Account alias: use full email (`mecampbell25@gmail.com`), not short alias
- **HTML order confirmation emails (SharkNinja, Amazon, retailers):** product name is buried past a CSS preamble (~2,500 chars of style tags + `&zwnj;` spam). Read `text[2000:6000]`, not `text[:3000]` — or strip `<style>` blocks first before stripping all tags.

**Drafting rules:**
- **Pre-draft gate (run before writing the first word):**
  1. Extract counterparty's own vocabulary from the thread — use their terms, not generic ones
  2. All cost/fee references: institutional framing ("Focus fees," "what I pay") — never "your fee"
  3. For each question you plan to ask: can the recipient actually answer it? Flag rhetorical or presupposing questions before drafting
- **Verifying recipient addresses**: Never guess email format (e.g. `firstname.lastname@domain.com`). Before sending to a named individual, confirm via `gog -a <account> gmail search "from:domain.com in:anywhere"`. Name-keyword searches miss people whose email uses initials (`lpelley@` not `lance.pelley@`). Domain search first, always.
- Reply-all by default: CC everyone already on the thread, plus anyone else who is a relevant party to the subject matter. If you're not confident someone should be included, verify rather than omit. **Before creating each reply draft, extract the CC field from the thread JSON — never assume it's empty, never invent recipients.** (See § gog Gmail Drafts below for the extraction snippet and full draft creation template.)
- Reply on the most recent thread for the topic — check thread dates first
- For reply drafts: `--reply-to-message-id <last-msg-id>` + `--quote` + `--cc <extracted>`. To revise the body, delete and recreate — `draft update --body` silently drops the quote. Dollar signs in `--body` get eaten by bash (`$8M` → `M`) — write body to `/tmp/body.txt` and pass `--body "$(cat /tmp/body.txt)"`.
- **`--quote` CID failure:** Before using `--quote`, scan whether the email has CID attachments: `gog gmail raw <msg-id> | grep -c 'Content-ID:'`. If count > 0, skip `--quote` entirely — go straight to manual plain-text extraction and prepend `> ` to each line. Never skip the quote. Use `--body-file /tmp/reply.txt` (body + quote combined) instead of `--quote`.
- **After sending:** `gog draft send` sometimes leaves a stale draft artifact. Verify the send via thread message count (`thread get --json` → `len(messages)` increased) and delete any leftover draft.
- **Terminology:** before drafting a reply, extract the counterparty's own terms from the thread — use their vocabulary, not inferred labels.
- Never send without explicit approval ("send it"). Draft → show → wait; expect 1–3 revision rounds
- Only include facts the user stated or directly evidenced in the thread — no inferred context
- Never attribute a statement to a person unless confirmed
- Plain, direct prose — no dramatic framing, no gushing
- Before sending: verify every question in the draft is one the recipient can actually answer — not rhetorical, not presupposing a framing they haven't seen
- Match the audience: deferential + warm greeting for older/old-school vendors; casual for friends/family
- Before asking a vendor/org a question, sweep email + texts for an answer they already gave

- **"Did X get [thing]?"** — parse who X is before searching. Check whether X was on the CC of the relevant thread; don't report from the asker's inbox perspective.

**Search strategy:**
- Search literal words first; simplest query before compound ("andrew", not "andrew socceria")
- Zero results → broaden, never narrow. Split compound words: "metrogroup" → try "metro" and "group"
- Add `in:anywhere` when results seem incomplete (catches trash/spam)
- Find all threads on a vendor topic before drafting — helpdesk replies come from subdomains, not `vendor.com`

## iMessage (imsg)

- Find any chat (incl. groups) by participants: `imsg chats --limit 500 --json`, filter on `contact_name` — **never ask for a phone number**
- All `--json` output is JSONL (one object per line) — parse line-by-line, never `json.load`
- Read messages: `imsg history --chat-id <id> --limit N` (there is no `messages` subcommand; plain output is a readable transcript)
- Send: `imsg send --chat-id <id> --text "..."`
- `imsg search` is broken — search content by extracting `attributedBody` from the sqlite DB via Python. **Don't filter `WHERE handle_id IS NOT NULL`** — sent messages have NULL `handle_id`, so that filter silently drops everything *you* sent (real miss: a `LIKE '%electric%'` returned zero hits because the match was in Matt's own sent message; the thread was only found by dumping a date window). Filter on `attributedBody IS NOT NULL`, and add a `date BETWEEN` window when you know roughly when.
- Before drafting, read recent sent messages in that chat and match style (short bursts, casual)
- Same draft → approval → send gate as email — never send without "send it"

## 1Password CLI

**`gh` commands:** Never use `op run --` with `gh` — it authenticates via macOS keychain independently and works without any wrapper.

**1Password Environments (MCP):** The `1password` MCP server is configured. Use `mcp__1password__*` tools for Environments (`.env` file management). Requires one Touch ID prompt per CC session via `mcp__1password__authenticate`.

**General `op` vault queries** (`op item get`, `op read`, `op item list`): These prompt for Touch ID on every call from within Claude Code (no-TTY by design). Use them when needed and expect a prompt. Do not wrap with `op run --` — it doesn't help.

If you see `missing required scopes`, check your GitHub CLI token scopes.

## gog Gmail Drafts

Before drafting any reply, extract CC from JSON — plain text silently drops them:

```bash
gog -a <account> gmail thread get <thread-id> --json 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
msgs = data['thread']['messages']
for m in msgs:
    h = {h['name'].lower(): h['value'] for h in m.get('payload', {}).get('headers', [])}
    if h.get('cc'): print('CC:', h['cc'])
"
```

Then draft with all extracted CC addresses:

```bash
gog -a <account> gmail draft create \
  --to "recipient" \
  --cc "cc1@example.com, cc2@example.com" \
  --reply-to-message-id <msg-id> \
  --quote \
  --body "..."
```

## Workflow

- **Orient before acting.** When a personal topic surfaces (vendor, project, person, decision), state your working model before making any claim or taking any action: "Here's what I think I know: [X] — is that current?" Treat memory entries and living docs as last-known-state hypotheses, not ground truth. Two questions to answer before proceeding: (1) is my knowledge accurate? (2) is this topic still open? If the answer to #2 is no — decision made, relationship ended, task closed — the right move is cleanup, not an update. Don't surface closed topics as open tasks.

- **State theory, then prove.** When you form a theory about why something behaves as it does, state it explicitly (falsifiable) and design + run an experiment/check to prove or refute it — reach for `/prove` or `/evidence` — *before* acting on it (editing code, reporting it as fact). Evidence merely consistent with a theory is not proof; the instrument may need a small code change, but that is building the test, not the fix.
- Before designing anything new, name the existing mechanism closest to what you need — extend it if possible, build something new only if it genuinely can't stretch.
- Propose changes before implementing — wait for approval
- When a directive is ambiguous, stop and ask rather than guessing — especially for irreversible actions like sending messages or submitting forms
- On command failures, check in with me before retrying
- After 2 failed attempts at the same approach, STOP — tell me what you tried, what failed, and what you'd try next
- When a search/lookup returns nothing, broaden it before changing approach — and verify each step incrementally on multi-step operations
- When waiting for remote/external results (CI, deploys, GitHub checks): poll in 10-15s intervals, don't sleep for the full expected duration. For container log reads (test results, build output), use the project's hardened monitoring snippet — one read, no sleep prefix, no polling loop.
- Capture decisions as they happen — don't leave them only in conversation. Write to the appropriate place: code comment (WHY), PLAN doc update, memory, or issue.
- **Doc routing — where things go:** universal working rules → `~/.claude/CLAUDE.md` (global); repo-specific facts/commands → that repo's `CLAUDE.md` (never generic best-practices — a project file is project-specific only); team-shared vs personal/machine-local within a repo → `CLAUDE.md` vs `.claude.local.md`/`settings.local.json`; evolving real-world state → living doc (`~/personal/` or a repo knowledge layer); atomic facts & behavioral corrections → memory (`feedback`/`project`/`reference`/`user`); harness-enforced automation & permissions → `settings.json` (user vs project) via `/update-config`.
- Before committing in any repo, verify staged files with `git diff --staged --stat` — especially when other work is in progress in the same repo. Staged files from unrelated work get swept into commits silently.
- Always run `/plan-implementation` fully after `/design-flow` — never shortcut from design to execution. If gaps emerge between what `/design-flow` produced and what `/plan-implementation` generates, treat that as signal worth investigating, not noise to skip past.
- For financial or legal findings that will be communicated to professionals (accountants, lawyers), verify from the primary source document before drafting — never rely on an agent or intermediate summary alone.

## Branch Naming

Use initials `mc` for branches: `mc/323-fix-auth-flow`

## Codex Sync

**Rule:** Every CLAUDE.md that is NOT inside a `.claude/` directory gets a mirrored AGENTS.md — direct copy, no substitutions — except third-party repos you can't commit to. Run this sync whenever CLAUDE.md changes in any owned repo. **Exception (explicit inclusion):** the global user file `~/.claude/CLAUDE.md` IS mirrored to `~/.claude/AGENTS.md` so Codex reads the same global rules, despite living in `.claude/`. Per-repo `.claude/CLAUDE.md` files (e.g. skill-adjacent) are NOT mirrored.

**Repos needing direct-copy sync** (pattern: `cp CLAUDE.md AGENTS.md && git add AGENTS.md && git commit -m "sync AGENTS.md: ..."`):
- `~/personal/`
- `~/code/finance/`
- `~/code/swarf/business-vault/`
- `~/code/swarf/webapp/apps/chatbot/` — use `chore:` prefix (commitizen required)
- Webapp worktrees (webapp-dev, webapp-dev-2, webapp-dev-3) — sync the chatbot subdir; picks up at merge

**`~/code/swarf/webapp` AGENTS.md + skills:** Run the migrate script instead (handles "Claude Code" → Codex substitutions):
```bash
python3 ~/.codex/skills/migrate-to-codex/scripts/migrate-to-codex.py \
  --source ~/code/swarf/webapp --target ~/.codex
```
This syncs 11 project skills + AGENTS.md with Codex substitutions. Run after significant CLAUDE.md or skill changes.


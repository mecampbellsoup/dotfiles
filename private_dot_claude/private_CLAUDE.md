## Who I Am

Matt Campbell — software engineer, co-founder of Swarf AI. Lives in Brooklyn, NY with wife Yury. Previously at CoreWeave for 5 years.

## Email Accounts

| Account | Use for |
|---------|---------|
| `mecampbell25@gmail.com` | Personal — taxes, personal finance, personal correspondence |
| `matt@swarf.app` | Work — Swarf business, investors, professional contacts |

**Email ops:** Always use `gog` for all email — both accounts. Never use Gmail MCP; `gog` handles everything and is authenticated to both accounts.

**Calendar ops:** Same rule — use `gog calendar`, not the claude.ai Calendar MCP (its session expires constantly; failed twice before `gog` worked first try, 7/2/26). Create: `gog -a <acct> calendar create primary --summary "..." --from "2026-07-30T09:00:00-04:00" --to "..." --timezone "America/New_York" --description "..." --json`.

**gog gotchas (recurring):**
- There is no `gog gmail message get` — single message = `gog gmail get <id> --json` (or `gog gmail raw <id>` for lossless API JSON)
- Plain output truncates bodies and drops CC — use `--json` + base64 decode
- **`--json` shape:** `gmail search` returns `{threads:[{id,date,from,subject,labels,messageCount}]}` — flat fields, NO payload (don't look for `messages`/headers there). Full content via `gmail thread get <id> --json` → `thread.messages[].payload.headers` (read each header's `.get('value')` — bare `['value']` KeyErrors on some headers) and base64-decode `body.data` walking `parts`.
- Account alias: use full email (`mecampbell25@gmail.com`), not short alias
- **Attachment flag (drafting):** `--attach` not `--attachment` — wrong flag silently fails draft creation. Also: **`--attach` splits its value on commas** — a filename containing a comma is parsed as multiple paths and fails with "no such file or directory." Rename the file comma-free before attaching.
- **Deleting a draft non-interactively needs `--force`:** `gog gmail draft delete <id>` refuses without it ("not recoverable... without --force") — and if stderr is suppressed (`2>/dev/null`), the refusal is invisible and the draft silently survives as an orphaned DRAFT message on the thread. Always pass `--force`, and verify deletion via thread message labels, not the command's silence. (Bit on 7/16/26: "deleted" draft resurfaced in a thread count check.)
- **Downloading an attachment:** `gog -a <acct> gmail attachment <messageId> <attachmentId> --out <path>` — both IDs are *positional* (not `--attachment-id`), and the save flag is `--out` (not `-o`). Get the attachmentId from the message payload parts (`body.attachmentId`).
- **A re-attached file with a familiar filename is a theory about its contents, not a verified fact** (see § Workflow "State theory, then prove") — a reply that re-sends a file matching one already seen (e.g. a shared scope doc) could be an annotated/updated version, not the same content. Download and read it, or `diff`/checksum it against the prior version, before characterizing it in a log, summary, or draft.
- **HTML order confirmation emails (SharkNinja, Amazon, retailers):** product name is buried past a CSS preamble (~2,500 chars of style tags + `&zwnj;` spam). Read `text[2000:6000]`, not `text[:3000]` — or strip `<style>` blocks first before stripping all tags.
- **Formatted (HTML) drafts:** `draft create` supports `--body-html-file <path>` (and `--body-html`) alongside plain `--body` — pass both for a proper multipart/alternative. Plain `--body` alone renders as unformatted text; if Matt says a draft "looks unformatted," recreate it with an HTML body (inline styles, no `<style>` blocks).
- **Token expired (`invalid_grant: "Token has been expired or revoked"`):** this is an interactive re-auth, not something to retry around. Stop and ask Matt to run `gog auth add <account> --services drive,docs,gmail,calendar` himself (opens a browser) — don't attempt the OAuth flow or guess a fix.

**Drafting rules:**
- **Summarizing doc changes in an email:** extract the actual language from the source doc — don't paraphrase from memory. The doc and the email body are separate files; a fix in one doesn't propagate to the other.
- **Pre-draft gate (run before writing the first word):**
  1. Extract counterparty's own vocabulary from the thread — use their terms, not generic ones
  2. All cost/fee references: institutional framing ("Focus fees," "what I pay") — never "your fee"
  3. For each question you plan to ask: can the recipient actually answer it? Flag rhetorical or presupposing questions before drafting
  4. If the topic has a living doc (`~/personal/<topic>.md`), read it first — don't draft questions already answered there
  5. If a question is about a vendor's public pricing or documented scope, verify via web search before listing it as open
  6. Never assume two vendors/advisors are aware of each other or of a parallel engagement — confirm before referencing one in correspondence with another
  7. Match specificity to what the user actually knows — don't invent implementation details, and don't assert something is unaffected by a change without checking
- **Pre-send gate (run before presenting or sending any draft):**
  1. CC list extracted from thread JSON — never assumed empty
  2. Required attachments (e.g. scope PDF) actually attached, not just referenced in text
  3. The draft-create tool call actually ran — confirm before saying "ready to review, say send it"
  4. Explicit "send it" received — past-tense or ambiguous mentions ("sent it") are not a send command
  5. If the body was staged in a scratch file that went through multiple revision rounds, confirm the draft-create call read the file's *current* content, not an earlier write — re-fetch the created draft and check its actual body text before sending, don't trust that the create call used the latest version
  6. Confirm the draft is grammatically addressed *to* the recipient, not narrated *about* them — a draft composed while explaining context to the user (third person: "she's probably got...") can accidentally get sent as-is instead of being rewritten in second person ("you probably have..."). Read the final text once as if you were the recipient before sending. Case: a text meant for Valerie was drafted/sent in the register of talking to Matt about Valerie ("she's probably got my old... info on file") — caught only because Matt read it after send and un-sent it.
- **Verifying recipient addresses**: Never guess email format (e.g. `firstname.lastname@domain.com`). Before sending to a named individual, confirm via `gog -a <account> gmail search "from:domain.com in:anywhere"`. Name-keyword searches miss people whose email uses initials (`lpelley@` not `lance.pelley@`). Domain search first, always.
- **Verifying addresses for cold outreach to a new external contact (no prior email history to search):** third-party contact-lookup aggregators (RocketReach, Wiza, ZoomInfo, etc.) are unreliable — they've produced both a wrong address (one character off, e.g. `jrwaxman@` vs. the real `jwaxman@`, which bounced) and a literal placeholder that looked like a real address at a glance. Treat any aggregator result as a lead, never as the address to send to. Always verify against the organization's own site — an attorney/staff bio page or the org's contact page — before sending. If the org's own site doesn't publish the individual's email, don't guess a pattern; use the org's general contact address, a web contact form, or ask the user how to proceed.
- Reply-all by default: CC everyone already on the thread, plus anyone else who is a relevant party to the subject matter. If you're not confident someone should be included, verify rather than omit. **Before creating each reply draft, extract the CC field from the thread JSON — never assume it's empty, never invent recipients.** (See § gog Gmail Drafts below for the extraction snippet and full draft creation template.)
- Reply on the most recent thread for the topic — check thread dates first
- For reply drafts: `--reply-to-message-id <last-msg-id>` + `--quote` + `--cc <extracted>`. To revise the body, delete and recreate — `draft update --body` silently drops the quote. Dollar signs get eaten by bash in any inline flag value (`--body`, `-m`, etc.) — see general rule under § Workflow.
- **`--quote` CID failure:** Before using `--quote`, scan whether the email has CID attachments: `gog gmail raw <msg-id> | grep -c 'Content-ID:'`. **Count 0 is NOT safe** — the HTML can *reference* a `cid:` with no matching MIME part (e.g. a quoted signature image from an earlier message), which also breaks `--quote` ("preserve quoted inline images... no matching MIME part"); check `grep -c 'cid:'` on the raw HTML too. If either count > 0, skip `--quote` entirely — go straight to manual plain-text extraction and prepend `> ` to each line. Never skip the quote. Use `--body-file /tmp/reply.txt` (body + quote combined) instead of `--quote`.
- **Sent vs draft:** a message appearing in `gog gmail thread get` output does NOT mean it was sent — drafts show up in threads too. Before reporting a message as sent, check its `labelIds` (`SENT` vs `DRAFT`).
- **After sending:** `gog draft send` sometimes leaves a stale draft artifact. Verify the send via thread message count (`thread get --json` → `len(messages)` increased) and delete any leftover draft.
- **`draft send` needs the draft ID, not the message ID:** `draft create`'s JSON output returns both `draftId` (e.g. `r-...`) and `message.id` — passing the message ID to `draft send` 404s even though the draft is not stale. This is a different root cause from the Mail.app-consumed-the-draft 404 below (no Mail.app involved); the fix is simply to pass `draftId`, not `message.id`.
- **Mail.app + API drafts:** `gog draft create` drafts behave identically to native Mail.app drafts when `--quote` is included: they sync to the Drafts mailbox (~30s IMAP), thread correctly with the original, and show Charlie's email collapsed below ("See More from X") which expands inline. Without `--quote`, the draft is bare — no quoted original — which looks orphaned even though threading is technically correct. Always use `--quote` for reply drafts (after the CID check). They do NOT appear when viewing the original from Inbox — but neither do native drafts. **If `draft send` 404s:** Mail.app consumed the draft ID — don't investigate, recreate immediately and send in one compound step (`gog ... draft create ... && gog ... draft send <new_id>`). **If Matt says he edited a draft in Mail.app:** the old message ID is stale — run `gog -a <account> gmail draft list --json` first to get the current draft ID before reading or acting on it. **Any draft-op 404 (delete/get/send), even unannounced:** same cause and fix — `draft list --json`, re-resolve (edited drafts come back with an `s:`-prefixed ID). Before recreating, extract the current draft's body (text/html part) and diff against yours — Matt's Mail.app edits must survive the recreate.
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
- **Prior session found an email via thread ID?** Extract the thread ID from that session's JSONL first (`grep -o '"thread_id":"[^"]*"' ~/.claude/projects/.../session.jsonl` or look at prior Bash tool calls), then `gog gmail thread get <id>` directly. Keyword searches for retailer/vendor emails frequently return 0 results even when the email exists.

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

**Live web-session logins (need to actually authenticate into a site, not just reference a field):** Don't attempt to print 1Password item fields (username/password) directly to stdout first — the permission classifier blocks this as credential materialization, regardless of intent. Go straight to Playwright: open the login page, then hand off authentication to the user (they type credentials/2FA themselves) rather than trying to retrieve-then-type the credential yourself.

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

- **Never use a real send/write/create call to "test" that a command or flag is valid.** Verify syntax via `--help`, a dry-run flag, or reading tool docs instead — a probe call on a send-type tool (`imsg send`, `gog draft send`, calendar create, etc.) is a real, irreversible side effect, not a no-op, regardless of the placeholder text used. Case: sent a real iMessage to Dad reading "PREVIEW-ONLY-NOT-SENT" while trying to confirm the `imsg send` command existed, directly violating the never-send-without-"send it" rule.
- **Never pipe a secret's raw value into tool output, even transiently.** `grep 'API_KEY=' .env`, `cat` on a credentials file, or printing a fetched secret to confirm it worked all put the real value in the conversation transcript — the permission classifier blocks hardcoding a raw secret into a command (e.g. a bash `export KEY="sk-..."`), but does not catch this softer leak vector, so it's on you to avoid it in the first place. When a script needs a secret already sitting in `.env`/settings, read it via the app's own config/settings object *inside* the process that needs it (e.g. `settings.OPENAI_API_KEY` in a Django shell) rather than shelling out to read the file, or inject it via `op run`/an `op://` reference. Verify a value's *presence* with a length check or a masked echo, never by printing it whole. Case: grepped `.env` for `OPENAI_API_KEY` and the full key landed in tool output while wiring up a comparison script — no tool stopped it, unlike the hardcoding attempt seconds later which the classifier correctly blocked.
- **Dollar signs get eaten by bash in any inline flag value.** `$8M` → `M`, `$120` disappears entirely — bash interprets `$X` as variable interpolation inside unquoted or double-quoted inline strings (`--body "..."`, `git commit -m "..."`, etc.), regardless of tool. Whenever the text contains a `$` amount, write it to a file first and pass it via `"$(cat file)"` or a heredoc instead of inlining it directly in the flag. Applies everywhere, not just `gog` — hit this in a `git commit -m` in addition to `gog draft create --body`.
- **Orient before acting.** When a personal topic surfaces (vendor, project, person, decision), state your working model before making any claim or taking any action: "Here's what I think I know: [X] — is that current?" Treat memory entries and living docs as last-known-state hypotheses, not ground truth. Two questions to answer before proceeding: (1) is my knowledge accurate? (2) is this topic still open? If the answer to #2 is no — decision made, relationship ended, task closed — the right move is cleanup, not an update. Don't surface closed topics as open tasks.
- **Run `date` before translating any relative date/day-name into a concrete date — every time, no exceptions.** This covers scheduling (CronCreate), interpreting "tomorrow"/"this Thursday"/"next week" in a message, day-name math mid-session, AND reading email header timestamps across time zones (a UTC offset can put a message's *local* send time on a different calendar day than its header date). Anchor every relative-date translation to a fresh `date` call, never to conversational assumption or a prior day-name used earlier in the session. Recurred 4x with real consequences: an 8 AM reminder set for the wrong day, a vendor callback slot miscalculated as 3.5 hours early, a mid-session delivery schedule drifting a day off, and a calendar invite sent for July 10 instead of July 9 because "tomorrow" in a late-night email was computed from today's date instead of the email's own timestamp.

- **State theory, then prove.** When you form a theory about why something behaves as it does, state it explicitly (falsifiable) and design + run an experiment/check to prove or refute it — reach for `/prove` or `/evidence` — *before* acting on it (editing code, reporting it as fact). Evidence merely consistent with a theory is not proof; the instrument may need a small code change, but that is building the test, not the fix. **This includes document/OCR extraction output** (scanned PDFs, garbled multi-column tables, any text pulled from a hard-to-parse source): a first-pass extraction is a theory about the document's content, not a verified fact. A mismatch between two extracted numbers is not evidence of a real discrepancy — before reporting "X doesn't reconcile with Y," verify both sides were read correctly (e.g. render to image and visually confirm), especially before sharing the finding with someone else. (Case: a scanned landscape table's garbled first-read produced a false "advisor report doesn't match custodian statements" discrepancy that was actually a self-inflicted misread — re-rendering at 300dpi and reading visually resolved it in minutes.)
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
- For financial or legal findings that will be communicated to anyone (family or professionals) or used to draft correspondence, verify from the primary source document before drafting — never rely on an agent, an intermediate summary, or your own recollection of the law/rules alone. This applies to domain claims stated confidently to a layperson audience too, not just formal advisor correspondence.
- **This extends to ordinary facts, not just finance/legal:** delivery/tracking status, dates, sizes, who-said-what — check the actual source (email, thread, tracker, doc) before stating it, even in casual conversation with Matt, not only in outbound drafts.
- Before asking Matt to relay a fact he'd have to go look up (order status, a code, a date, a personal detail), check gog/imsg/local docs yourself first — this applies broadly, not just OTP/verification codes.
- Before reasoning about a change to a trip/schedule/booking, restate what's confirmed vs. still-flexible, and confirm structure (round-trip vs. one-way, exact dates) back to the user before submitting — don't let a hypothetical alter confirmed-booking reasoning.
- **Chezmoi tracks `~/.claude/*` but has no idea when it's edited directly on disk.** Editing `~/.claude/CLAUDE.md`, `settings.json`, or a skill file changes the target, not the chezmoi source (`~/.local/share/chezmoi/private_dot_claude/`) — `chezmoi diff`/`apply` will read those edits as drift and silently propose *deleting* them until synced back. After editing any file under `~/.claude/`, run `chezmoi re-add <path>` (autoCommit/autoPush then pushes it automatically). Before ever running `chezmoi apply`, run `chezmoi diff` first and check which side — source or target — actually has the newer content; don't assume apply is safe just because it's the "sync" command. **`chezmoi diff` convention (read this every time, don't re-derive it):** in the diff, `a` = current target (the live file on disk), `b` = source-as-it-would-be-after-`apply`. So `+` lines are content that exists in **source only** (apply would add them to target; `re-add` would delete them from source if run instead), and `-` lines are content that exists in **target only** (apply would remove them from target; `re-add` would add them to source). Misreading this direction and running the wrong command (`re-add` when you meant `apply`, or vice versa) silently deletes real content from one side — happened once already (7/11/26, `tmuxinator/swarf.yml`'s `webapp-5` entry deleted from source by a backwards read, caught and reverted same session). Verify the read against a case you're sure of (e.g. a field you know is only live on one side) before acting, not after.

## Branch Naming

Use initials `mc` for branches: `mc/323-fix-auth-flow`

## Codex Sync

**Rule:** Every CLAUDE.md that is NOT inside a `.claude/` directory gets a mirrored AGENTS.md — direct copy, no substitutions — except third-party repos you can't commit to. Run this sync whenever CLAUDE.md changes in any owned repo. **Exception (explicit inclusion):** the global user file `~/.claude/CLAUDE.md` IS mirrored to **`~/.codex/AGENTS.md`** — that is where Codex reads global instructions (CODEX_HOME; verified against the Codex manual 7/1/2026). Do NOT mirror to `~/.claude/AGENTS.md` — Codex never reads that path (an old copy there was removed 7/1/2026). Per-repo `.claude/CLAUDE.md` files (e.g. skill-adjacent) are NOT mirrored.

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
This syncs 11 project skills + AGENTS.md with Codex substitutions. Run after significant CLAUDE.md or skill changes. **Scope is `.claude/skills/` only** — the script globs `.claude/skills/*/SKILL.md` and does not touch `plugins/swarf-toolkit/skills/`. Edits to swarf-toolkit *plugin* skills (`/tdd`, `/commit-changes`, etc.) are distributed via the marketplace, not this script, so there's nothing to run for them.


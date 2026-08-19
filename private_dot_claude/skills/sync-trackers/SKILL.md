---
name: sync-trackers
description: Sweep the "awaiting/pending" rows of all ~/personal living docs against email (gog, both accounts), iMessage, and financial ground-truth (Simplifi, card portals) to catch replies and refund/payment postings the trackers missed — and separately, cull docs/rows that have gone stale for a different reason (no activity in months, topic abandoned or superseded, life moved on) by proposing closes for Matt's confirmation. Invoke when Matt asks to sync the trackers, asks "any updates?" / "where do things stand?" across topics, before producing any cross-tracker status digest, and at session wind-down in the personal workspace when the session touched tracked topics. Docs go stale two ways: counterparties reply same-day on NEW threads while a doc claims "awaiting reply" for days, and also trackers just quietly outlive their relevance. Never report waiting-state or financial status from doc contents alone, and never auto-archive a culling candidate without confirmation.
---

# Sync Trackers

Living docs record when something was sent, but nothing re-checks for the reply — or, for money, whether it actually posted. This skill is the re-check. Motivating incidents: Armen proposed a meeting date on a fresh thread while the doc claimed 5 days of silence (7/20/26); Vestwell + CoreWeave HR both replied within hours on 7/9 while the tracker sat at "awaiting reply" for 11 days; a Swanson $49.96 refund was logged as "Processing" on Amex on 7/2 and carried that label for weeks until an unrelated pass happened to catch that it had actually landed (7/23/26) — nothing had gone back to check the account itself.

## 0. Cull first

Before sweeping for replies, scan for docs/rows that are stale for a different reason: not "no reply yet" but "life moved on." Signals:

- No `## Log` entry in 60+ days
- An "awaiting" row where the realistic next step never happened and nothing suggests it still will (vendor ghosted with no recourse, Matt decided against pursuing it, topic superseded by a newer doc/decision)
- The doc's own content shows the decision was effectively made or the need evaporated, even if never formally marked CLOSED

**Silence is not automatically staleness** — a slow-moving legal or bureaucratic matter (lawsuit, government filing, HSS-style application) can be genuinely quiet and still open; don't flag those just because they're old. Flag when the *thread itself* looks dead, not just aged.

Culling is a *proposal*, not an action — never auto-archive. Surface candidates in the digest (see Step 3) and only apply the Close flow (CLOSED header, archive per CLAUDE.md's File Lifecycle & Git routing, CLAUDE.md active-docs row removal) after Matt confirms.

## 1. Build the target list

```bash
rg -n -i "awaiting|pending|no reply|nudged|verify refund|verify.*posted|refund processing|awaiting refund|awaiting payment" ~/personal/*.md
```

- Collect Now-table and Coming Up rows where the wait is on a **counterparty** (person/vendor/org) *or* on a refund/charge/payment that's supposed to post somewhere.
- Also collect **Matt-owed** rows — "awaiting Matt's reply", "Matt to send X", "Matt to nudge Y" — don't skip these, they get a different check (Step 2b).
- Skip fixed future dates for *sweeping* — deadlines have no counterparty to poll — but **collect them for the digest** (Step 3). A date nobody replies to is still a date that can pass unnoticed.
  When collecting, **do not scan for ISO dates only.** Living docs write deadlines however they were written down: `Aug 31 2026`, `Fri 8/21`, `13:00 Pacific Aug 29`, `Sep 21 20:00 JST`. A `20\d\d-\d\d-\d\d` regex silently returns nothing for all of those and reads as "no upcoming deadlines" — the failure is indistinguishable from a clean pass (see global CLAUDE.md § Workflow on checks that can't revoke their own conclusion). Grep the `## Coming Up` sections and any row containing `expires`/`due`/`deadline` and read the dates yourself.
  Incident (8/19/26): an ISO-only scan reported the next 21 days as nearly empty while a surprise dinner sat 2 days out and three ANA award waitlists were expiring within 12.
- For each target, extract the channel: email address/domain (from How to Reach or Log entries), iMessage chat-id/phone, or — for a refund/payment row — the card/account it should post to. If the doc doesn't record the address, find it in the doc's logged thread/message IDs.

## 2. Sweep each counterparty

- **Email:** `gog -a <acct> gmail search "from:<address-or-domain> in:anywhere" --max 10 --json` — use the account the topic lives on (personal = mecampbell25, Swarf = matt@swarf.app). **Never just re-poll known thread IDs** — new replies often arrive as NEW threads with new subjects (see global CLAUDE.md § Search strategy). Search the domain, not the exact address, when helpdesk subdomains are possible.
- **iMessage:** `imsg history --chat-id <id> --limit 20` for known chats; content search requires the sqlite `attributedBody` method (see global CLAUDE.md § iMessage).
- **Resolve shortened links before concluding a message says nothing.** A message that reads as a bare `maps.app.goo.gl` / `bit.ly` / share-link with no text is not contentless — the content *is* the link target, and a sweep that reads only message text will report the thread as noise. Follow each redirect and read the destination:
  `curl -sIL "<url>" -o /dev/null -w '%{url_effective}\n'` (then URL-decode). This applies to any channel, but iMessage is where it bites: people send a venue, a listing, or a booking as a naked pin.
  Incident (8/19/26): Yury sent five Google Maps pins proposing a full Tokyo day — two museums, Ueno Zoo, a dinner she had already *requested* a reservation for, and a Kyoto site. Read as raw text it looked like "Yury sent some links"; every venue name, and the pending reservation, only appeared after resolving the redirects.

## 2b. Sweep Matt-owed rows

For rows where the doc says Matt owes a reply/send/nudge, don't rely on incidentally finding it inside a counterparty thread — that only works if the action landed on an existing thread. Run an independent sent-mail check: `gog -a <acct> gmail search "from:<matt's account> to:<counterparty-address-or-domain> in:anywhere" --max 10 --json`, scoped to after the row's logged date. This catches replies Matt sent on a brand-new thread of his own, not just ones threaded onto a prior message.
- **Financial ground-truth (refund/charge/payment rows):** a merchant's "refund issued" or "processing" email is a claim, not a receipt — the money actually posting is what closes the loop. Check Simplifi first (search the transaction by merchant/amount/date); if it's not showing there yet, fall back to the card issuer's own portal (Amex, Chase, Schwab). Simplifi syncs typically lag a few days behind real posting, so "not showing yet" isn't proof it didn't post — treat that as inconclusive and note the lag, not as a negative finding.
- Compare the newest **inbound** item's date against the doc row's Since/last-log date. Anything newer → read the actual message/thread (check `labelIds` — a thread hit is not proof of direction or send).
- **Always fetch the full thread (every message), never just one.** Grabbing only the latest or first message can miss a reply that landed right after it — e.g. pulling `msgs[0]` to "quickly check" a thread misses Matt's own reply if it's `msgs[1]`. Print the whole message list (from/date/labelIds for each) before concluding a thread is unanswered or silent. (Incident 7/22/26: a thread was reported as "unanswered" from Matt because only its first message was fetched; the very next message was his same-day reply.)

## 3. Patch and report

- Update each stale doc per `/personal-doc` conventions (Now / Coming Up / Decide / Log; first edit of a given doc in a session goes through that skill's gate). Fully-resolved topics get the Close flow (CLOSED header, archive, CLAUDE.md row removal).
- Commit each change; keep CLAUDE.md active-docs rows and AGENTS.md in sync when descriptions change.
- Digest for Matt, in this order: (1) answered/resolved items found, (2) still genuinely awaiting — with days-silent counts, (3) new Matt-owned actions the sweep surfaced, (4) culling candidates from Step 0 — name the doc/row and why it looks dead, and wait for Matt's confirmation before closing/archiving anything.

## 4. Before drafting off a sweep finding

If a sweep finding turns into a reply/nudge draft (not just a doc patch), re-read the doc's own just-updated log entry for the correct thread/ticket/message ID before drafting — don't reuse a thread ID recalled from earlier in the same turn or from memory of the topic. Two incidents (7/20/26): a SharkNinja nudge was first drafted assuming a refund was still pending when the sweep itself had just found it processed, and a second draft replied on a stale ticket thread when the doc's own most recent log entry already named the correct, newer case thread. The sweep finds the truth; drafting must re-anchor to it, not to a thread ID already in context.

## Anti-patterns

- Reporting "no reply yet" from doc state or a thread-ID poll without a fresh `from:` sweep — the exact failure this skill exists to prevent.
- **Narrowing the sweep window because a sync ran recently.** "The last pass was yesterday, so only the last day matters" assumes the previous pass was complete — the same assumption this skill exists to distrust. Sweep a wide inbound window (~10 days) regardless of when the last run was; it is one query and it catches what the prior pass missed. Incident (8/19/26): a 2-day window anchored to the previous day's sync missed four of five real findings, including an e-file authorization awaiting signature and a delivered order with a same-day deadline.
- Treating a naked link as an empty message — resolve it (Step 2).
- Fetching only one message (first or last) from a thread and concluding it's unanswered — always pull the full message list first.
- Sweeping all ~40 docs indiscriminately — only rows with a sweepable counterparty.
- Updating the digest but not the docs — the doc patch is the deliverable; the digest is a byproduct.
- Drafting a reply off a stale thread ID held in context instead of the doc's freshly-patched log entry — see step 4.
- Treating a merchant's refund/shipping confirmation email as ground truth without checking the account or portal it should actually post to — the email is a claim, not a receipt.
- Auto-closing/archiving a doc off a culling signal without Matt's confirmation — Step 0 finds candidates, it doesn't decide.
- Flagging a slow-but-still-open matter (lawsuit, application, filing) as stale just because it's quiet — staleness is about a dead thread, not elapsed time alone.
- Only catching Matt's sent replies incidentally inside a counterparty-found thread — Matt-owed rows need their own `from:me` sweep (Step 2b), since a reply on a brand-new thread won't surface otherwise.

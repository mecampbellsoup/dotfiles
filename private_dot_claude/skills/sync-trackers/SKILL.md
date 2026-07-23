---
name: sync-trackers
description: Sweep the "awaiting/pending" rows of all ~/personal living docs against email (gog, both accounts), iMessage, and financial ground-truth (Simplifi, card portals) to catch replies and refund/payment postings the trackers missed. Invoke when Matt asks to sync the trackers, asks "any updates?" / "where do things stand?" across topics, before producing any cross-tracker status digest, and at session wind-down in the personal workspace when the session touched tracked topics. Docs go stale silently — counterparties reply same-day on NEW threads while a doc claims "awaiting reply" for days, and refund/payment rows carry a merchant's "processing" email as if it were the actual posting. Never report waiting-state or financial status from doc contents alone.
---

# Sync Trackers

Living docs record when something was sent, but nothing re-checks for the reply — or, for money, whether it actually posted. This skill is the re-check. Motivating incidents: Armen proposed a meeting date on a fresh thread while the doc claimed 5 days of silence (7/20/26); Vestwell + CoreWeave HR both replied within hours on 7/9 while the tracker sat at "awaiting reply" for 11 days; a Swanson $49.96 refund was logged as "Processing" on Amex on 7/2 and carried that label for weeks until an unrelated pass happened to catch that it had actually landed (7/23/26) — nothing had gone back to check the account itself.

## 1. Build the target list

```bash
rg -n -i "awaiting|pending|no reply|nudged|verify refund|verify.*posted|refund processing|awaiting refund|awaiting payment" ~/personal/*.md
```

- Collect Now-table and Coming Up rows where the wait is on a **counterparty** (person/vendor/org) *or* on a refund/charge/payment that's supposed to post somewhere.
- Skip rows waiting on Matt, and fixed future dates (deadlines aren't sweepable).
- For each target, extract the channel: email address/domain (from How to Reach or Log entries), iMessage chat-id/phone, or — for a refund/payment row — the card/account it should post to. If the doc doesn't record the address, find it in the doc's logged thread/message IDs.

## 2. Sweep each counterparty

- **Email:** `gog -a <acct> gmail search "from:<address-or-domain> in:anywhere" --max 10 --json` — use the account the topic lives on (personal = mecampbell25, Swarf = matt@swarf.app). **Never just re-poll known thread IDs** — new replies often arrive as NEW threads with new subjects (see global CLAUDE.md § Search strategy). Search the domain, not the exact address, when helpdesk subdomains are possible.
- **iMessage:** `imsg history --chat-id <id> --limit 20` for known chats; content search requires the sqlite `attributedBody` method (see global CLAUDE.md § iMessage).
- **Financial ground-truth (refund/charge/payment rows):** a merchant's "refund issued" or "processing" email is a claim, not a receipt — the money actually posting is what closes the loop. Check Simplifi first (search the transaction by merchant/amount/date); if it's not showing there yet, fall back to the card issuer's own portal (Amex, Chase, Schwab). Simplifi syncs typically lag a few days behind real posting, so "not showing yet" isn't proof it didn't post — treat that as inconclusive and note the lag, not as a negative finding.
- Compare the newest **inbound** item's date against the doc row's Since/last-log date. Anything newer → read the actual message/thread (check `labelIds` — a thread hit is not proof of direction or send).
- **Always fetch the full thread (every message), never just one.** Grabbing only the latest or first message can miss a reply that landed right after it — e.g. pulling `msgs[0]` to "quickly check" a thread misses Matt's own reply if it's `msgs[1]`. Print the whole message list (from/date/labelIds for each) before concluding a thread is unanswered or silent. (Incident 7/22/26: a thread was reported as "unanswered" from Matt because only its first message was fetched; the very next message was his same-day reply.)

## 3. Patch and report

- Update each stale doc per `/personal-doc` conventions (Now / Coming Up / Decide / Log; first edit of a given doc in a session goes through that skill's gate). Fully-resolved topics get the Close flow (CLOSED header, archive, CLAUDE.md row removal).
- Commit each change; keep CLAUDE.md active-docs rows and AGENTS.md in sync when descriptions change.
- Digest for Matt, in this order: (1) answered/resolved items found, (2) still genuinely awaiting — with days-silent counts, (3) new Matt-owned actions the sweep surfaced.

## 4. Before drafting off a sweep finding

If a sweep finding turns into a reply/nudge draft (not just a doc patch), re-read the doc's own just-updated log entry for the correct thread/ticket/message ID before drafting — don't reuse a thread ID recalled from earlier in the same turn or from memory of the topic. Two incidents (7/20/26): a SharkNinja nudge was first drafted assuming a refund was still pending when the sweep itself had just found it processed, and a second draft replied on a stale ticket thread when the doc's own most recent log entry already named the correct, newer case thread. The sweep finds the truth; drafting must re-anchor to it, not to a thread ID already in context.

## Anti-patterns

- Reporting "no reply yet" from doc state or a thread-ID poll without a fresh `from:` sweep — the exact failure this skill exists to prevent.
- Fetching only one message (first or last) from a thread and concluding it's unanswered — always pull the full message list first.
- Sweeping all ~40 docs indiscriminately — only rows with a sweepable counterparty.
- Updating the digest but not the docs — the doc patch is the deliverable; the digest is a byproduct.
- Drafting a reply off a stale thread ID held in context instead of the doc's freshly-patched log entry — see step 4.
- Treating a merchant's refund/shipping confirmation email as ground truth without checking the account or portal it should actually post to — the email is a claim, not a receipt.

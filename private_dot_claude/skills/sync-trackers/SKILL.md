---
name: sync-trackers
description: Sweep the "awaiting/pending" rows of all ~/personal living docs against email (gog, both accounts) and iMessage to catch replies the trackers missed. Invoke when Matt asks to sync the trackers, asks "any updates?" / "where do things stand?" across topics, before producing any cross-tracker status digest, and at session wind-down in the personal workspace when the session touched tracked topics. Docs go stale silently — counterparties reply same-day on NEW threads while a doc claims "awaiting reply" for days. Never report waiting-state from doc contents alone.
---

# Sync Trackers

Living docs record when something was sent, but nothing re-checks for the reply. This skill is the re-check. Two real incidents motivated it (both 7/20/26): Armen proposed a meeting date on a fresh thread while the doc claimed 5 days of silence, and Vestwell + CoreWeave HR both replied within hours on 7/9 while the tracker sat at "awaiting reply" for 11 days.

## 1. Build the target list

```bash
rg -n -i "awaiting|pending|no reply|nudged" ~/personal/*.md
```

- Collect Now-table and Coming Up rows where the wait is on a **counterparty** (person/vendor/org).
- Skip rows waiting on Matt, and fixed future dates (deadlines aren't sweepable).
- For each target, extract the channel: email address/domain (from How to Reach or Log entries) or iMessage chat-id/phone. If the doc doesn't record the address, find it in the doc's logged thread/message IDs.

## 2. Sweep each counterparty

- **Email:** `gog -a <acct> gmail search "from:<address-or-domain> in:anywhere" --max 10 --json` — use the account the topic lives on (personal = mecampbell25, Swarf = matt@swarf.app). **Never just re-poll known thread IDs** — new replies often arrive as NEW threads with new subjects (see global CLAUDE.md § Search strategy). Search the domain, not the exact address, when helpdesk subdomains are possible.
- **iMessage:** `imsg history --chat-id <id> --limit 20` for known chats; content search requires the sqlite `attributedBody` method (see global CLAUDE.md § iMessage).
- Compare the newest **inbound** item's date against the doc row's Since/last-log date. Anything newer → read the actual message/thread (check `labelIds` — a thread hit is not proof of direction or send).

## 3. Patch and report

- Update each stale doc per `/personal-doc` conventions (Now / Coming Up / Decide / Log; first edit of a given doc in a session goes through that skill's gate). Fully-resolved topics get the Close flow (CLOSED header, archive, CLAUDE.md row removal).
- Commit each change; keep CLAUDE.md active-docs rows and AGENTS.md in sync when descriptions change.
- Digest for Matt, in this order: (1) answered/resolved items found, (2) still genuinely awaiting — with days-silent counts, (3) new Matt-owned actions the sweep surfaced.

## 4. Before drafting off a sweep finding

If a sweep finding turns into a reply/nudge draft (not just a doc patch), re-read the doc's own just-updated log entry for the correct thread/ticket/message ID before drafting — don't reuse a thread ID recalled from earlier in the same turn or from memory of the topic. Two incidents (7/20/26): a SharkNinja nudge was first drafted assuming a refund was still pending when the sweep itself had just found it processed, and a second draft replied on a stale ticket thread when the doc's own most recent log entry already named the correct, newer case thread. The sweep finds the truth; drafting must re-anchor to it, not to a thread ID already in context.

## Anti-patterns

- Reporting "no reply yet" from doc state or a thread-ID poll without a fresh `from:` sweep — the exact failure this skill exists to prevent.
- Sweeping all ~40 docs indiscriminately — only rows with a sweepable counterparty.
- Updating the digest but not the docs — the doc patch is the deliverable; the digest is a byproduct.
- Drafting a reply off a stale thread ID held in context instead of the doc's freshly-patched log entry — see step 4.

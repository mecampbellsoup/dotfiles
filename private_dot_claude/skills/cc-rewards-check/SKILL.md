---
name: cc-rewards-check
description: Check the status of Matt's credit card recurring credits/bonuses (Amex Platinum for Schwab, Amex Gold, Chase Sapphire Reserve, Delta SkyMiles Reserve) across AwardWallet and Simplifi, then produce one digest of what's unclaimed and what's expiring soon. Use this whenever Matt asks to check his credit card rewards/credits/bonuses/offers, wants to know what statement credits he hasn't used yet, asks something like "did I use my Resy credit this quarter" or "what card credits am I about to lose", mentions AwardWallet/CardPointers/Simplifi in the context of card benefits, or wants a rewards/credits digest. This skill already knows which of those three tools to trust for what — don't re-derive that from scratch.
user_invocable: true
---

# CC Rewards Check

Matt has recurring statement credits scattered across 4 cards (Amex Platinum for Schwab, Amex Gold, Chase Sapphire Reserve, Delta SkyMiles Reserve), each on its own cadence (monthly/quarterly/semi-annual/annual). No single tool tracks all of them reliably — the point of this skill is to know which tool to trust for which piece of data, so Matt doesn't have to check three places and cross-reference himself every time.

**Bilt Mastercard has no dollar credits** (pure points-multiplier card, confirmed 2026-07-20) — skip it entirely. Apple Card, Wells Fargo Cash Wise Visa, and Home Depot Credit Card are simple cash-back cards with nothing to reconcile — also out of scope.

The living doc `~/personal/credit-card-rewards-tracking.md` has the full research behind every claim below (per-card benefit tables, why each source is trusted or not, dated log entries). Read it once at the start of a run — it's the source of truth for *what benefits exist*; this skill's job is checking *current status*, not re-deriving the list.

**This is not a backgroundable/scheduled skill as written — it needs Matt present.** Every step drives a live, authenticated Playwright browser against AwardWallet and Simplifi (and occasionally the issuer portals directly). That means real-time login handoffs when a session's expired, and — this happened during the skill's own dogfood run on 2026-07-20 — real browser-contention with whatever else Matt is doing in Playwright locally, since only one session can hold the shared browser profile at a time. An unattended run has no one to hand a login page to, no one to say "the browser's free now" after a collision, and no one to notice a hung session and recover it. Don't invoke this on a cron/schedule or any other unattended trigger until that's solved with something more robust (stored sessions, an isolated browser profile so it can't collide with Matt's manual use, real retry/backoff) — treat it as interactive-only for now.

## Why three tools, not one

- **AwardWallet** (awardwallet.com) is the best source for real dollar status — it syncs balances directly from the issuers. But its sync can go stale silently (both the Amex and Chase connections were fully broken until Matt reset the stored passwords on 2026-07-20) and 3 specific Chase benefits never populate at all, resync or not.
- **Simplifi** (app.simplifimoney.com) has ground-truth transaction data and is the fallback for the fields AwardWallet can't fill.
- **CardPointers** has no web dashboard (native app only, can't be automated) and its dollar values are self-reported, not synced — confirmed via direct comparison that it shows "$0 spent" on credits Amex's own portal confirms were already used. Only use it as an occasional manual cross-check for the benefit *list* (it caught a Global Entry credit the manual issuer research missed) — never for dollar status, and never as part of the automated run.

## Step 1 — Check AwardWallet is actually current before trusting it

This is the step that would have caught today's problem automatically instead of needing Matt to notice CLEAR+ looked wrong.

Navigate to `https://awardwallet.com/account/list/` (account is `mecampbell25+awardwallet@gmail.com` — see memory `reference-awardwallet-account.md`). If not authenticated in the browser session, hand off to Matt the same way as any live login: tell him you're on the login page and wait for him to say he's in.

For the **Amex (Membership Rewards)** and **Chase (Ultimate Rewards)** institution rows, read the "Last updated" timestamp next to each. If either is stale beyond ~30 days, don't silently report numbers from it as current — say explicitly in the digest that "AwardWallet's [issuer] connection looks stale (last synced [X] ago) and may need Matt to reset the stored password," and treat that institution's benefits as unknown for this run rather than reporting a number that might be wrong.

If a connection needs a password reset, that's Matt's to do (it's his bank credential) — flag it and move on, don't try to fix it yourself.

## Step 2 — Pull AwardWallet's sub-account data

For each institution that passed the freshness check in Step 1, expand its sub-accounts and read, for every benefit listed in the living doc's per-card tables: the `$X / $Y remaining` pair and the `expires in …` text.

While you're in there, if a sub-account shows a benefit that isn't in the living doc's tables yet (this happened once already — a $100 Global Entry credit on the Amex Platinum), note it so it can get added later. Don't silently drop new information just because it wasn't in the plan.

## Step 3 — Fill the confirmed AwardWallet gaps via Simplifi

Three Chase Sapphire Reserve benefits never populate in AwardWallet regardless of sync freshness (confirmed 2026-07-20, not a staleness issue): the Chase Travel hotels credit, the Global Entry/TSA/NEXUS credit, and the StubHub credit.

For these, check `https://app.simplifimoney.com` instead (1Password item "Simplifi"). Search the Chase Sapphire Reserve account's transactions for the current year. Annual-reset credits tend to post under a payee named `<Benefit> Year` (e.g. "Hotel Year", "Travel Year") — sum the matching transactions for the year and compare against the benefit's cap from the living doc.

If Simplifi doesn't resolve a benefit either (e.g. the credit hasn't posted at all yet, or the payee naming doesn't match what you expect), it's fine to report it as "status unknown, check the Chase portal directly" rather than guessing — a wrong number is worse than an honest gap.

## Step 4 — Compile the digest

One digest, sorted by urgency:

1. **Expiring within 7 days** — flagged first, regardless of dollar amount, since these are the ones Matt could actually lose
2. **Everything else with money still available** — sorted by days-to-expiration ascending
3. **Data quality notes** — anything from Step 1 or Step 3 that came back unknown/stale, so Matt knows what this digest does and doesn't cover this run

Skip anything already fully claimed for its current period — the digest is for what's actionable, not a full status dump of every benefit.

## Step 5 — Update the living doc

Use the `/personal-doc` skill to patch `~/personal/credit-card-rewards-tracking.md`'s `Now` table and `Log` section with this run's findings — don't `Edit` the doc directly (per `feedback_personal-doc-skill-invocation.md`, this keeps the living-doc conventions consistent). Include any new benefits discovered in Step 2 and any stale-connection flags from Step 1, since those are exactly the kind of thing that should persist across sessions instead of being rediscovered each time.

## Step 6 — Present the digest

Give Matt the digest directly in the conversation — that's the actual deliverable, the living-doc update in Step 5 is bookkeeping, not the answer to what he asked.

---
name: hermes-handoff
description: Hand off past conversation context to Hermes Agent (@Hermes), searching across Slack, Gmail, and iMessage to find it. Use when the user wants Hermes caught up on something — Slack permalinks, "share this thread with hermes", "get hermes up to speed on X", or any reference to a past conversation in any channel.
user_invocable: true
---

# Hermes Handoff

Find a past conversation — via Slack permalink, or via /recall for natural-language descriptions — synthesize a clean handoff summary framed for Hermes, get the user's sign-off, and send it to their Hermes DM.

This is the historical-context complement to Hermes's own `session_search`. Hermes's session DB only contains conversations Hermes itself participated in, starting from its install date. For anything older — or for threads in channels Hermes wasn't part of — Hermes can't recall them. This skill bridges that gap on demand.

## Why ephemeral

The handoff is delivered as a **regular Slack message in the user's DM with Hermes**, not written to Hermes's persistent memory. Three reasons:

- Hermes's `MEMORY.md` has a hard char limit (~2200) that should be reserved for ambient user facts, not history dumps
- Slack messages flow into Hermes's session DB automatically, so the handoff becomes searchable for future sessions without touching Hermes's storage
- Sending as a user message lets Hermes incorporate the context just like any other prompt — no special markers, no schema risk

## Sending as the user

The Slack MCP plugin (`mcp__plugin_slack_slack__*`) authenticates as the logged-in human, not as a bot. So `slack_send_message` posts the handoff under the user's own identity — functionally identical to copy-pasting it themselves. This is why auto-sending after approval is safe: nothing happens that the user couldn't have done by hand in the same Slack client.

## Workflow

### 1. Resolve the source

Two paths depending on what the user gives you:

**Slack permalink** (URL containing `/archives/<channel_id>/p<ts>`): parse it directly — don't invoke /recall. The `p<ts>` portion is the timestamp with the dot removed: `p1234567890123456` → `1234567890.123456`. Extract channel_id and ts, then go to step 2.

**Natural-language description** ("the convo where we figured out X", "what I talked to Hermes about re: Y"): invoke `/recall` with the description as the query. Recall searches across Slack, Gmail, and iMessage and returns a synthesis. Skip steps 2–3 and go straight to step 4, using the recall synthesis as your source material.

If the user provides BOTH a permalink and a description, the permalink wins — they're being precise on purpose.

### 2. Fetch the Slack conversation (permalink path only)

For threaded messages, use `mcp__claude_ai_Slack__slack_read_thread` with the channel_id and parent message ts to get the entire thread.

For non-threaded conversations or DM ranges, use `mcp__claude_ai_Slack__slack_read_channel` with `oldest`/`latest` timestamps to fetch a window of messages around the target. Start with a limit of 50; expand if the conversation clearly extends past the boundaries you fetched.

You're trying to capture the **complete substantive arc** of the conversation, not just the user-flagged message.

### 3. Filter the noise (permalink path only)

Bot-driven Slack threads contain process artifacts that bloat the summary. Drop these:

- **Tool-call status lines** — lines matching `:<word>: <word>: "..."` (bot announcing what tool it's calling)
- **Ephemeral status messages** — "Searching...", "I'm working on it...", "Let me check that for you..."
- **Single-emoji reactions** added as separate messages (rare but happens)

Keep all user messages, all substantive bot prose responses, and any artifacts produced inline (drafts, code, summaries). When in doubt, keep it.

### 4. Synthesize the handoff summary

Produce a structured message in roughly this shape, framed for Hermes as the reader:

```
Hey Hermes — bringing you up to speed on a past conversation you weren't part of.

*What we discussed:* [1-2 sentences framing the topic]

*Key facts established:*
• [bullet of decisions, numbers, names, dates that matter]
• [more bullets as needed]

*Where we left off:* [the state of things at the end — what was decided, what was done, what wasn't]

*Open threads / follow-ups:* [anything that needed to happen but didn't, OR "none" if it concluded cleanly]

[Optional one-line context about WHY I'm sharing this now, if obvious from the user's invocation — e.g., "I'm about to ask you to draft a similar listing" or "this connects to the X work we're doing today"]
```

The framing matters: lead with "Hey Hermes" so it reads naturally as a user message rather than a wall of metadata. Use Slack mrkdwn (`*bold*`, `•` bullets), not markdown headers (`##` won't render).

Length should match the source. A 5-message thread becomes a paragraph; a 30-message debug session becomes the structured shape above. Don't pad.

### 5. Show the draft for approval

Present the synthesized handoff to the user with:

- The full draft as a code block (so they can read it as it'll appear)
- The destination channel (e.g., "Send to your Hermes DM (D0ACL8H2D4G)?")
- A brief note on what was filtered out, if non-trivial (e.g., "Dropped 8 tool-call status lines")

Wait for explicit approval before sending. Acceptable approvals: "send", "ship it", "go", "yes", "lgtm". If the user wants edits, apply them and show the revised draft. If they want to cancel, do nothing.

### 6. Send via Slack MCP

After approval, send the message using `mcp__plugin_slack_slack__slack_send_message`. The default destination is the user's Hermes DM. For Matt specifically that's channel `D0ACL8H2D4G`, but the skill should look up the right channel each time it runs (the user might have a different DM with their Hermes-equivalent bot, or might want to send to a different channel entirely).

To find the right channel: search for the user's DM with the Hermes bot via `slack_search_channels` (looking for the bot username, e.g., "clawdbot") if you don't already know it. Cache the channel ID for the duration of the conversation.

If the user explicitly specifies a different channel ("send it to #project-x instead"), use that channel.

After sending, confirm to the user that the message went through (the MCP should return a ts on success). Don't wait for a Hermes response — that's a separate task.

## Edge cases and gotchas

- **Permalink with `?thread_ts=`**: This is a thread reply, not the parent. Always fetch the full thread starting from `thread_ts`, not from the linked reply alone. The user almost always wants the whole thread context.
- **Permalink to a single message in a long channel**: Fetch a window around it (e.g., 10 messages before and 20 after) and let the synthesis step decide what's substantive. Don't try to be clever about session boundaries.
- **No matches from /recall**: Relay what recall reported — don't synthesize from nothing. Suggest a permalink if the user knows where the thread lives.
- **Conversation in a private channel the user isn't in**: The Slack MCP can only see what the authenticated user can see. If you can't read it, say so plainly — don't fabricate.
- **Very long threads (50+ messages)**: Synthesize aggressively. The user can ask Hermes to drill into specifics later if they need more detail.

## Notes

- This skill is **strictly ephemeral**. Don't write to `~/.hermes/memories/MEMORY.md`, don't insert into `~/.hermes/state.db`, don't touch any Hermes persistent storage. The handoff lives only as a Slack message in the user's DM with their bot. That's by design — see "Why ephemeral" above.
- The default Hermes bot channel for Matt's setup is `D0ACL8H2D4G` (DM with Hermes). For other users, look it up via `slack_search_channels` looking for the bot.
- Filtering rules are defaults, not gospel. If a future test shows the skill is dropping something useful, the rule in step 3 should be revised — that's the skill's main quality dimension, and it's worth iterating on.

---
name: find-conversation
description: Find past Claude Code conversations across worktrees. Use when the user wants to find, recall, or resume a previous session.
user-invocable: true
---

# Find Conversation

Locate past Claude Code conversations by topic, timeframe, or keyword.

## JSONL Format

Conversation files live in `~/.claude/projects/<project-dir>/*.jsonl`. Each line is a JSON object with a `type` field:

| type | Contains |
|------|----------|
| `user` | User messages — the primary search target |
| `assistant` | Claude's responses |
| `attachment` | File attachments, skill loads |
| `permission-mode` | Session metadata |
| `file-history-snapshot` | File state snapshots |
| `queue-operation` | Internal queue ops |
| `last-prompt` | Session end marker |

**Extracting user text** from a `type: "user"` line:

```python
obj = json.loads(line)
if obj.get("type") == "user":
    msg = obj.get("message", {})
    content = msg.get("content", [])
    if isinstance(content, str):
        text = content
    elif isinstance(content, list):
        text = " ".join(
            block["text"] for block in content
            if isinstance(block, dict) and block.get("type") == "text"
        )
```

User messages often start with skill/command content (`<command-name>`, `<system-reminder>`, doctoc markers). Filter these when looking for the user's actual topic — the real intent is usually after the boilerplate or in a later message.

## Protocol: Narrow Fast

Search in three stages. Stop as soon as you have a confident match.

### Stage 1: Time Window

Use the user's timeframe hint to set `find -mmin`. Map natural language to minutes:

| Hint | `-mmin` |
|------|---------|
| "just now", "a few minutes ago" | -15 |
| "earlier", "30 minutes ago" | -60 |
| "an hour ago" | -120 |
| "today", "this morning" | -720 |
| "yesterday" | -2880 |
| "this week" | -10080 |
| "recently" (no hint) | -1440 (24h) |

```bash
find ~/.claude/projects/ -name "*.jsonl" -not -path "*/subagents/*" -mmin -MINUTES 2>/dev/null
```

### Stage 2: First User Message (Topic ID)

For each candidate file, extract the first 3 real user messages. This is the fastest way to identify what a conversation was about — much faster than keyword grep, which matches memory/env context noise.

```python
import json, os, sys

def extract_user_messages(filepath, max_msgs=3):
    """Extract first N real user messages from a conversation file."""
    msgs = []
    with open(filepath) as f:
        for line in f:
            try:
                obj = json.loads(line)
            except (json.JSONDecodeError, ValueError):
                continue
            if obj.get("type") != "user":
                continue
            msg = obj.get("message", {})
            if not isinstance(msg, dict):
                continue
            content = msg.get("content", [])
            if isinstance(content, str):
                text = content
            elif isinstance(content, list):
                text = " ".join(
                    block["text"]
                    for block in content
                    if isinstance(block, dict) and block.get("type") == "text"
                )
            else:
                continue
            # Skip noise: skill loads, system reminders, short acks
            if any(marker in text[:200] for marker in [
                "<system-reminder>", "START doctoc", "command-name>/hermes",
                "command-name>/wind-down", "command-name>/exit",
                "Request interrupted", "Tool loaded",
                "UserPromptSubmit", "You are an expert",
                "You are an AI assistant", "local-command-caveat",
            ]):
                # But check for command-args which contain the real user intent
                if "<command-args>" in text:
                    import re
                    m = re.search(r"<command-args>(.*?)</command-args>", text, re.DOTALL)
                    if m and len(m.group(1).strip()) > 10:
                        msgs.append(m.group(1).strip()[:300])
                continue
            if len(text.strip()) < 5:
                continue
            msgs.append(text.strip()[:300])
            if len(msgs) >= max_msgs:
                break
    return msgs
```

Present candidates to yourself as a table:

```
ID: <basename without .jsonl>
Project dir: <which project>
Modified: <timestamp>
First messages:
  1. <first user message>
  2. <second user message>
```

Match the user's description against the first messages. The right conversation is usually obvious from the opening message.

**Before Stage 3, check the retention floor.** Session transcripts are not retained indefinitely, so ask what the oldest surviving **file** is before spending keyword-grep rounds on a window that no longer exists:

```bash
find ~/.claude/projects -name '*.jsonl' -not -path '*subagents*' \
  -exec stat -f '%Sm %N' -t '%Y-%m-%d' {} + 2>/dev/null | sort | head -1
```

**Read file mtimes, not the timestamps inside the files.** Those answer different questions, and the difference is large enough to send a search after a transcript that was deleted weeks ago. A long-running or resumed session keeps writing to one file, so its *earliest* embedded timestamp can predate its own file's retention window by a wide margin — the minimum across all files is therefore the start of the oldest surviving **conversation**, never the boundary of what still exists. Measured 2026-08-27: earliest embedded timestamp 2026-07-02 (56 days), oldest surviving file 2026-07-29 (29 days). Reading the first as the floor said a mid-July transcript should still be there; it had been gone for weeks.

The window is rolling, not a fixed date — four files aged out within one hour of each other during that same session, so re-run the check rather than reusing an earlier answer.

**What the floor bounds is last activity, not session start.** Retention tracks when a file was last written, so a conversation that began well before the floor is still present if it was resumed after it. A stated timeframe that predates the floor makes the transcript unlikely rather than impossible — say so in those terms, and let a long-running session be the exception that occasionally survives.

If the user's stated timeframe predates the floor, say so directly and stop. This check takes one command and should run as soon as the user gives a rough timeframe, before Stage 2 or Stage 3 burn rounds on a search that can't succeed.

Two cases, and the second is why this now reads file mtimes:

- **2026-07-31** — five rounds of GitHub-issue search, git log search, and keyword grep across every project directory, for a conversation the user placed "about 2 months ago," all failed before anyone checked the floor. The floor check would have shortcut the whole search.
- **2026-08-27** — the floor check *ran*, returned 2026-07-02 from embedded timestamps, and cleared a search for a 2026-07-14 session that no longer existed. Two more rounds followed. The instrument answered a question nobody asked, and answered it confidently.

### Stage 3: Keyword Grep (Fallback Only)

Only use keyword grep if Stage 2 didn't find a match. When you do:

- Use **specific, unusual terms** the user mentions — not common words like "openai" or "hermes" that appear in env/memory context of every conversation
- Grep for the keyword, then still extract first user messages to confirm topic
- Combine `find -mmin` with `grep -l` to stay within the time window

### Escalation — when to stop searching and ask

Two rounds of broadened search with no confident hit → stop and ask the user for a disambiguating detail (a rough time, which app/window they were in, a distinctive phrase they remember) rather than continuing to broaden indefinitely. A "round" is a genuinely different search angle — a new keyword combination, a wider time window, a different project directory sweep — not a re-run of the same grep. This mirrors the same principle behind the user's gog email-search rule ("two rounds of broadened keyword search with no confident hit → ask the user to search their own Mail.app"): the topic likely used different wording than you're guessing, and the user already knows it. Asking costs one turn; a third or fourth broadened round rarely succeeds where the first two didn't (2026-07-30: five search rounds — keyword grep, combined-term grep, a full first-message scan across every session in a 48h window, a date-restricted grep, and a home-directory sweep — all failed before asking the user directly, which resolved instantly).

## Project Directories

All conversation history is under `~/.claude/projects/`. Directory names encode the filesystem path with dashes replacing slashes. Scan all directories, not just webapp worktrees:

```bash
ls -d ~/.claude/projects/*/
```

Common ones for this user:
- `-Users-mcampbell-code-swarf-webapp{,-dev,-dev-2,-dev-3}` — webapp worktrees
- `-Users-mcampbell-code-swarf-business-vault` — business docs
- `-Users-mcampbell-code-swarf` — parent repo
- `-Users-mcampbell` — home directory sessions

## Output

Once found, present:

1. **Conversation ID** (the JSONL filename without extension)
2. **Worktree/project** it belongs to
3. **When** (modification timestamp)
4. **Topic summary** from the first few user messages
5. **How to resume**: `claude --resume <id>` from the corresponding worktree directory

To answer "where did we leave off?", extract the LAST ~10 user+assistant
turns instead of the first: same parsing as Stage 2, but collect (type, text)
tuples for both "user" and "assistant" lines into a list and print the tail
(`turns[-12:]`, truncating each to ~1500 chars). The final assistant messages
usually contain the open-loops summary.

## Common Pitfalls

- **Don't grep for broad keywords first.** Terms like "openai", "hermes", "test" appear in memory/env context loaded into every conversation. You'll get dozens of false positives. Always prefer Stage 2 (first user message) over Stage 3 (keyword grep).
- **Don't guess the JSONL schema.** The message format uses `type: "user"` at the top level (not `role: "user"`). Content is in `message.content`, which can be a string or a list of typed blocks.
- **This conversation matches too.** The current conversation file also lives in the projects directory. Filter it out if it shows up (it will contain the user's search query as a false match).

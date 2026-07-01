---
name: recall
description: Search across all communication channels — Slack, Gmail (both personal and work accounts), iMessage, and the local Photos library (OCR of photographed documents) — to answer "what happened with X" questions. Use this skill whenever the user wants to find a past conversation, recall what was decided, check what someone said, dig up context from any medium, or locate a photo/scan of a document they took. Triggers on phrases like "what happened with", "what did we decide about", "did I talk to X about", "what was the deal with", "remind me about", "check my messages/emails/texts about", "find the photo/scan of", or any time the user references something they half-remember and wants to track down. Also use when another skill (like hermes-handoff) needs to resolve a natural-language reference to a past conversation.
---

# Recall

Search across Slack, Gmail, and iMessage to surface what was said about a topic, synthesized into a single coherent answer.

## Step 0: Check living docs first

Before searching any channel, check if a living doc exists for this topic — in `~/personal/` (personal living docs) **and, when working inside a project repo, that repo's own doc web** (e.g. business-vault's `business/`, `projects/`, `orgs/`; a code repo's `docs/`). These are maintained summaries that are often more complete and faster than raw message search.

```bash
ls ~/personal/ | grep -i <keyword>
# if inside a project repo, also grep its content (not just filenames):
grep -ril <keyword> <repo-root> --include="*.md"
```

If a relevant doc exists, read it. It may fully answer the question — skip channel search entirely, or use it to confirm/extend what you find in messages. Grep **content**, not just filenames — a topic can live inside a broader doc (e.g. a "Tax" section inside `operations.md`) without the keyword appearing in the filename.

Living docs live at paths like `~/personal/transformer-table.md`, `~/personal/condo-gc.md`, `~/personal/health-insurance.md`. When in doubt, `ls ~/personal/` and scan the list for anything topically related.

## Source routing

Read the query for medium signals before searching. Route accordingly:

- **"text", "texted", "iMessage", "imsg", person's name with personal context** → iMessage first; fan out to others if thin
- **"email", "emailed", "gmail", formal/vendor context** → Gmail first; fan out to others if thin
- **"Slack", "DM", "channel", "Hermes", bot names** → Slack first; fan out to others if thin
- **"photo", "picture", "scan", "screenshot", "I took a photo of", a sought-after letter/form/document** → Photos library (see below); also fan out to text channels
- **No clear signal** → fan out to all three message channels in parallel

When in doubt, fan out. The cost of one extra empty search is lower than missing the answer.

**When text channels come up empty for a document/letter the user is sure exists, the Photos library is the channel that's usually been missed** — a photo taken but never sent is invisible to iMessage, email, *and* Spotlight. Always run the Photos OCR pass before concluding "it's nowhere."

## Searching each source

### Slack

```
mcp__claude_ai_Slack__slack_search_public_and_private(
  query=<keywords>,
  channel_types="public_channel,private_channel,mpim,im",
  include_bots=true
)
```

Result files can be very large. Use `python3` to slice by character range rather than reading the whole file. For hits with replies, fetch the full thread via `slack_read_thread`.

**Noise filter** before synthesizing: drop lines matching `:<word>: <word>: "..."` (bot tool-call announcements) and ephemeral filler ("Searching...", "Let me check..."). Keep all user messages and substantive bot prose.

### Gmail

Search both accounts in parallel unless the query is clearly personal or work:

```bash
gog -a mecampbell25@gmail.com gmail search "<keywords> in:anywhere" --json
gog -a matt@swarf.app gmail search "<keywords> in:anywhere" --json
```

For hits, fetch the body:
```bash
gog gmail get <id> --json
```
Then base64-decode the body field. Plain output silently truncates and drops CC — always use `--json`.

HTML-heavy emails (order confirmations, retailer receipts): the product name is buried past ~2,500 chars of CSS. Read `text[2000:6000]`, not `text[:3000]`.

### iMessage

`imsg search` works but only covers ~0.3% of messages — those with `text` populated (some SMS/MMS and a small fraction of iMessages). The other 99.7% store content only in `attributedBody` and are invisible to it. Scan `attributedBody` for comprehensive search:

```python
python3 << 'PYEOF'
import sqlite3, re
from datetime import datetime, timedelta

db = sqlite3.connect("/Users/mcampbell/Library/Messages/chat.db")
cursor = db.cursor()
cursor.execute("""
    SELECT m.ROWID, m.date, c.chat_identifier, c.display_name, m.is_from_me, m.attributedBody
    FROM message m
    JOIN chat_message_join cmj ON m.ROWID = cmj.message_id
    JOIN chat c ON cmj.chat_id = c.ROWID
    WHERE m.attributedBody IS NOT NULL
    ORDER BY m.date DESC
""")

keywords = [b'KEYWORD1', b'KEYWORD2']  # lowercase bytes
for row in cursor:
    rowid, date, chat_id, display_name, is_from_me, body = row
    if not body or not any(kw in body.lower() for kw in keywords):
        continue
    try:
        text = body.split(b'NSString')[1] if b'NSString' in body else body
        readable = re.findall(rb'[\x20-\x7e\xc0-\xff]{4,}', text)
        readable_str = b' '.join(readable[:5]).decode('utf-8', errors='replace')
    except:
        readable_str = "<could not decode>"
    dt = datetime(2001, 1, 1) + timedelta(seconds=date / 1e9)
    print(f"[{dt:%Y-%m-%d %H:%M}] chat={chat_id} name={display_name} from={'me' if is_from_me else 'them'}")
    print(f"  {readable_str[:200]}\n")
db.close()
PYEOF
```

For context around hits: `imsg chats --limit 500 --json` → filter on `contact_name` → `imsg history --chat-id <id> --limit 30`.

### Photos library (photographed documents)

Use when the target is a **photo or scan of a document** (a letter, form, statement, receipt) the user took — and especially when text channels come up empty. A photo taken but never sent lives **only** here, invisible to iMessage and email.

**Spotlight does NOT cover this.** `mdfind "Internal Revenue Service"` finds text inside PDFs and *some* images, but it does **not** index the Photos library's Live Text OCR. You must OCR the originals yourself with macOS Vision.

**Two traps that cause silent total failure — get these right:**
1. The library path contains a **space**: `~/Pictures/Photos Library.photoslibrary`. Write the file list to a file and have the OCR script read that list directly. **Never** pass `$(cat list.txt)` unquoted to a command — the shell word-splits every path on the space and every file silently fails to load.
2. **Verify the pipeline on a known-positive before trusting any negative.** Run the OCR on one document you *know* contains the target text; if it reads correctly, a subsequent "0 matches" is real. If you skip this, a broken pipeline looks identical to a genuine miss.

Filter by date first (`-newermt`) to keep the set tractable, then OCR via a small Swift/Vision script that reads the list file. OCR over hundreds of HEICs is slow and its stdout is block-buffered — **run it in the background** and expect output only when it completes.

```bash
# 1. build a newline-delimited list of candidate originals (note the quoted path)
lib="$HOME/Pictures/Photos Library.photoslibrary/originals"
find "$lib" -type f \( -iname "*.heic" -o -iname "*.jpg" -o -iname "*.png" \) \
  -newermt "2026-01-01" > /tmp/ocr_list.txt   # narrow the date window to the topic

# 2. OCR script reads the LIST FILE (arg1), never an arg-list of space-bearing paths
cat > /tmp/ocr.swift << 'SWIFT'
import Foundation; import Vision; import AppKit
let paths = (try! String(contentsOfFile: CommandLine.arguments[1], encoding: .utf8))
    .split(separator: "\n").map(String.init)
let kws = ["keyword one","keyword two"]  // lowercase, topic-specific & low-false-positive
for p in paths { autoreleasepool {
    guard let i = NSImage(contentsOfFile: p), let t = i.tiffRepresentation,
          let b = NSBitmapImageRep(data: t), let cg = b.cgImage else { return }
    let r = VNRecognizeTextRequest(); r.recognitionLevel = .accurate
    try? VNImageRequestHandler(cgImage: cg, options: [:]).perform([r])
    var s = ""; if let o = r.results { for x in o { s += (x.topCandidates(1).first?.string ?? "") + " " } }
    if kws.contains(where: { s.lowercased().contains($0) }) {
        print("MATCH\t\(p)"); print("TEXT\t" + s.prefix(550).replacingOccurrences(of:"\n",with:" ")) }
} }
print("DONE")
SWIFT
swift /tmp/ocr.swift /tmp/ocr_list.txt > /tmp/ocr_hits.txt 2>/dev/null   # run in background
```

For a confirmed hit, convert the HEIC to JPG (`sips -s format jpeg -Z 2000 <orig> --out /tmp/x.jpg`) and **Read it** to capture details OCR truncated. Archive the document to its domain folder if it's worth keeping.

## Synthesis

Produce one coherent narrative that answers the query, with inline attribution:

- "Amy confirmed via email (Feb 27) that..."
- "In a Slack thread with Hermes (Mar 3)..."
- "Krishna texted on Feb 15..."

Weave hits from multiple sources into a single story — don't group by source. The user wants to know what happened, not what each channel said.

If hits are thin or zero: say so plainly and suggest broadening — simpler keywords, different spelling, check if the conversation might be in a different medium. If the target is a document the user is sure exists and the text channels are empty, **run the Photos-library OCR pass before concluding it's nowhere** — that's the channel most often missed.

## Composition

When another skill invokes recall (e.g. hermes-handoff), the synthesis is the return value. The calling skill reframes it for its own purpose — no special output format needed, just the same narrative you'd give the user directly.

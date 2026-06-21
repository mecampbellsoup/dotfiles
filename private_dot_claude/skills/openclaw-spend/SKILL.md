---
name: openclaw-spend
description: OpenClaw spend monitor — estimates monthly Anthropic API cost from local session data and compares against the $100/month agent SDK credit cap. Use when asked about OpenClaw costs, whether usage fits under the credit cap, or Anthropic billing for agent usage.
---

# OpenClaw Spend Monitor

Answers: **"Will my OpenClaw usage fit under the $100/month Anthropic agent SDK credit cap?"**

## Background

Starting June 15, 2026, Anthropic separates agent SDK usage (`claude -p`, third-party tools like OpenClaw) into a $100/month credit pool, separate from the Max subscription's interactive usage. This skill estimates OpenClaw's monthly spend from local session data to check against that cap.

## Data Source

One input: OpenClaw's raw session store file.

| Source | What it tells you | How to get it |
|--------|-------------------|---------------|
| Session store | Per-session token counts (input, output, cache read, cache write) by model | `~/.openclaw/agents/main/sessions/sessions.json` |

**Why the raw file, not `openclaw sessions --json`:** The CLI output omits `cacheRead` and `cacheWrite` fields. Cache tokens dominate cost (~87% in measured usage). Without them, the estimate would be off by an order of magnitude.

## Steps

### 1. Read and aggregate session data

```bash
python3 -c "
import json, time
from datetime import datetime, timezone
from collections import defaultdict

with open('$HOME/.openclaw/agents/main/sessions/sessions.json') as f:
    data = json.load(f)

now = datetime.now(timezone.utc)
current_month = now.strftime('%Y-%m')
day_of_month = now.day

# Anthropic pricing ($/MTok) — update when models change
PRICING = {
    'claude-sonnet-4-6':  {'input': 3.00, 'output': 15.00, 'cache_read': 0.30, 'cache_write': 3.75},
    'claude-sonnet-4-5':  {'input': 3.00, 'output': 15.00, 'cache_read': 0.30, 'cache_write': 3.75},
    'claude-opus-4-6':    {'input': 15.00, 'output': 75.00, 'cache_read': 1.50, 'cache_write': 18.75},
    'claude-haiku-4-5':   {'input': 0.80, 'output': 4.00, 'cache_read': 0.08, 'cache_write': 1.00},
}

totals = defaultdict(lambda: {'input': 0, 'output': 0, 'cache_read': 0, 'cache_write': 0, 'sessions': 0})
models_seen = set()

for key, session in data.items():
    updated = session.get('updatedAt', 0)
    if not updated:
        continue
    dt = datetime.fromtimestamp(updated / 1000, tz=timezone.utc)
    month = dt.strftime('%Y-%m')
    model = session.get('model', 'unknown')
    models_seen.add(model)

    totals[month]['input'] += session.get('inputTokens', 0) or 0
    totals[month]['output'] += session.get('outputTokens', 0) or 0
    totals[month]['cache_read'] += session.get('cacheRead', 0) or 0
    totals[month]['cache_write'] += session.get('cacheWrite', 0) or 0
    totals[month]['sessions'] += 1

# Calculate cost for current month
t = totals.get(current_month, {'input': 0, 'output': 0, 'cache_read': 0, 'cache_write': 0, 'sessions': 0})

# Use first model's pricing (usually all the same); fall back to sonnet
model = list(models_seen)[0] if models_seen else 'claude-sonnet-4-6'
p = PRICING.get(model, PRICING['claude-sonnet-4-6'])

cost_input = (t['input'] / 1_000_000) * p['input']
cost_output = (t['output'] / 1_000_000) * p['output']
cost_cache_read = (t['cache_read'] / 1_000_000) * p['cache_read']
cost_cache_write = (t['cache_write'] / 1_000_000) * p['cache_write']
total_cost = cost_input + cost_output + cost_cache_read + cost_cache_write

daily_rate = total_cost / day_of_month if day_of_month > 0 else 0
projected = daily_rate * 30
cap = 100.0
headroom = cap - projected
pct = (projected / cap) * 100

if pct < 60:
    verdict = 'SAFE -- projected spend well under cap'
elif pct < 90:
    verdict = 'WARNING -- approaching cap, monitor closely'
else:
    verdict = 'OVER -- projected to exceed cap'

print(f'OpenClaw Spend Report -- {current_month} ({day_of_month} days elapsed)')
print()
print(f'  Sessions:        {t[\"sessions\"]}')
print(f'  Models:          {\", \".join(sorted(models_seen)) or \"none\"}')
print()
print(f'  Token breakdown:')
print(f'    Input:         {t[\"input\"]:>12,} tokens    \${cost_input:.2f}')
print(f'    Output:        {t[\"output\"]:>12,} tokens    \${cost_output:.2f}')
print(f'    Cache read:    {t[\"cache_read\"]:>12,} tokens    \${cost_cache_read:.2f}')
print(f'    Cache write:   {t[\"cache_write\"]:>12,} tokens    \${cost_cache_write:.2f}')
print(f'                                          ------')
print(f'    Month-to-date:                        \${total_cost:.2f}')
print()
print(f'  Daily burn rate: \${daily_rate:.2f}/day')
print(f'  Projected month: \${projected:.2f}')
print(f'  Credit cap:      \${cap:.2f}')
print(f'  Headroom:        \${headroom:.2f} ({100-pct:.0f}%)')
print()
print(f'  Verdict: {verdict}')

# Show previous months if any
prev_months = sorted([m for m in totals if m != current_month])
if prev_months:
    print()
    print('  Previous months:')
    for m in prev_months:
        pt = totals[m]
        pc = sum([
            (pt['input'] / 1e6) * p['input'],
            (pt['output'] / 1e6) * p['output'],
            (pt['cache_read'] / 1e6) * p['cache_read'],
            (pt['cache_write'] / 1e6) * p['cache_write'],
        ])
        print(f'    {m}: \${pc:.2f} ({pt[\"sessions\"]} sessions)')
"
```

### 2. Interpret the verdict

| Verdict | Meaning | Action |
|---------|---------|--------|
| SAFE (<60%) | Projected spend well under $100 cap | No action needed |
| WARNING (60-90%) | Approaching the cap | Watch daily burn; consider reducing usage or budgeting for overage |
| OVER (>90%) | Projected to exceed cap | Enable "extra usage" billing or reduce OpenClaw usage |

### 3. If approaching the cap

Options to reduce spend:
- Switch OpenClaw to a cheaper model (`openclaw config set agents.defaults.model.primary anthropic/claude-haiku-4-5`)
- Reduce conversation frequency or length
- Enable Anthropic's "extra usage" billing (billed at standard API rates beyond the credit)

## Limitations

- **Session store is ephemeral.** OpenClaw's `sessions.json` may be pruned over time. Historical data accuracy depends on retention.
- **Token counts may be incremental.** The `inputTokens` field appears to track only non-cached input per turn, not cumulative. `cacheRead` carries the bulk of actual input volume.
- **Pricing is hardcoded.** Update the `PRICING` dict when Anthropic changes rates or when OpenClaw switches models.
- **No server-side verification.** Anthropic's Admin Usage API requires a full organization (not individual accounts). This skill estimates from local data only.

## Context

- OpenClaw authenticates via Claude Code CLI OAuth (`sk-ant-oat01-*`), billing against the Anthropic Max subscription
- The $100/month agent SDK credit pool is separate from interactive Claude Code/chat usage
- See memory `project-openclaw-eval.md` for OpenClaw's current config and auth method
- See `/openclaw` skill for troubleshooting
- Analogous to `/hermes-spend` which monitors OpenAI costs for Hermes Agent

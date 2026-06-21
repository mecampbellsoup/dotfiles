---
name: hermes-spend
description: Agent overflow monitor — checks if the ChatGPT plan tier is right for the current workload by comparing Codex OAuth quota exhaustion frequency against API fallback spend and plan upgrade cost. Use when asked about Hermes costs, plan tier decisions, or whether to upgrade/downgrade.
---

# Agent Overflow Monitor

Answers: **"Is my ChatGPT plan tier right for my agent workload, or should I upgrade/downgrade?"**

## Data Sources

Three inputs, combined into a decision:

| Source | What it tells you | How to get it |
|--------|-------------------|---------------|
| Codex OAuth exhaustion frequency | How often the subscription quota runs out | `~/.hermes/auth.json` credential pool |
| API fallback spend | Dollar cost of overflow traffic | OpenAI costs API (Swarf org, Hermes project) |
| Plan tier cost | What you're paying now vs alternatives | Known constants (see tier table) |

## Steps

### 1. Read exhaustion history from auth.json

```bash
python3 -c "
import json, time
d = json.load(open('$HOME/.hermes/auth.json'))
cred = d['credential_pool']['openai-codex'][0]
print(f\"Status: {cred['last_status']}\")
print(f\"Last exhausted: {cred.get('last_status_at', 'never')}\")
print(f\"Reset at: {cred.get('last_error_reset_at', 'n/a')}\")
print(f\"Error: {cred.get('last_error_message', 'none')}\")
"
```

Also check `hermes auth list` for a formatted summary with time-remaining.

**No proactive quota API exists.** OpenAI does not expose an endpoint to check remaining Codex OAuth subscription quota. The only signals are: (1) the 429 response when exhausted, and (2) the `x-ratelimit-remaining-tokens_usage_based` header on successful responses (unreliable for subscription quota — may show -1). Track exhaustion events over time to build the frequency picture.

### 2. Query API fallback spend (Hermes project)

```bash
source .env 2>/dev/null
START=$(date -v-30d +%s)
END=$(date +%s)
curl -s "https://api.openai.com/v1/organization/costs?start_time=$START&end_time=$END&group_by=project_id&limit=100" \
  -H "Authorization: Bearer $OPENAI_ADMIN_KEY"
```

Filter results for the Hermes project. This is ONLY fallback spend — the primary path (Codex OAuth) bills against the ChatGPT subscription, not the API.

For per-model breakdown, use `&group_by=line_item` instead of `project_id`.

**Setup:** The admin key is in the worktree `.env` file as `OPENAI_ADMIN_KEY` (read-only scope on the Swarf org). Data lags 1-2 days behind platform.openai.com/usage.

### 3. Produce the decision table

Compare current combined cost against plan alternatives:

| Plan | Subscription | Expected overflow | Combined | vs Current |
|------|-------------|-------------------|----------|------------|
| Plus ($20) | $20/mo | ${fallback_spend}/mo | ${total} | — |
| Pro $100 | $100/mo | ~${reduced}/mo | ${total} | +$X/−$Y |
| Pro $200 | $200/mo | ~$0/mo | $200 | +$X/−$Y |

**Tier reference:**

| Plan | Monthly cost | Codex quota multiplier | Notes |
|------|-------------|----------------------|-------|
| Plus | $20 | 1x (30-150 local msgs/5hrs) | Current plan |
| Pro $100 | $100 | 5x | |
| Pro $200 | $200 | 20x | Effectively unlimited for current usage |

**Decision logic:**
- If overflow spend > plan upgrade delta → recommend upgrade
- If overflow spend < $10/mo → stay on Plus (current sweet spot)
- If overflow is zero for 30+ days → consider downgrade if on Pro

### 4. Format output

Present as:
1. Current exhaustion frequency (times/week or times/month)
2. Current monthly overflow cost (API fallback)
3. Decision table (see above)
4. Verdict: **Stay** / **Upgrade to X** / **Downgrade to X**

## Context

- Hermes fallback chain: Codex OAuth → `OPENAI_API_KEY` (Swarf org) → OpenRouter (Claude Sonnet)
- Swarf org has auto-recharge ON ($5 threshold → tops up to $10)
- See `reference-hermes-fallback-architecture.md` for the full billing flow
- See `reference-openai-orgs-and-tiers.md` for org structure and plan history
- As of May 2026: Plus ($20) + ~$8/mo overflow = ~$28/mo total. Cheaper than Business was ($40/mo for 2 seats).

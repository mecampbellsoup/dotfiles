---
name: reconcile-portfolio
description: Options portfolio reconciliation — pull trades from Schwab, update the ledger, generate position and P&L summaries. Use when Matt asks about his portfolio, trades, positions, P&L, or wants to reconcile against Schwab. Also use when he's preparing for advisor meetings (JPM, Clay/Schwab) and needs a current position summary.
user_invocable: true
---

# Portfolio Reconciliation

Maintain a persistent ledger of options/equity trades across Schwab accounts. The ledger (`trades.jsonl`) is the source of truth; positions and P&L are derived from it.

**The code does the work — this skill routes intent to it.** Extraction, normalization, `activity_id` dedup, position-netting, reconciliation, and P&L are all implemented and tested in the repo (`normalizer.py`, `positions.py`, `reconcile.py`, `portfolio.py`, `unwind_quotes.py`). Don't reimplement them here. The architecture and ledger schema are documented in **`CLAUDE.md`** — read that, don't restate it.

- **Working dir:** `~/code/finance` (NOT `~/finance` — that path doesn't exist)
- **Run everything via** `uv run python …`

## Intent → command

Most questions don't need a fresh extraction — pick the narrowest tool for what's actually being asked.

| User is asking… | Run | Notes |
|---|---|---|
| "Are my positions right?" / "what do I hold?" | `portfolio.py check` | Live Schwab positions vs ledger; flags mismatches |
| "Did X fill?" / "pull new trades" | `portfolio.py sync` | Incremental 14-day window, `activity_id` dedup, appends new only |
| "Reconcile / is the ledger accurate?" | `portfolio.py reconcile` | 4-layer: completeness, qty cross-check, cash flows, closed round-trips |
| "Rebuild history from scratch" | `portfolio.py build [--since YYYY-MM-DD]` | Full re-baseline (1-yr default); backs up the old ledger first |
| "How are my collars doing?" | `portfolio.py collars` | Long-put/short-call MTM across accounts |
| "What orders are working?" | `portfolio.py orders` | Live working orders |
| "Cancel orders" | `portfolio.py cancel [--symbol X] [--all]` | Interactive cancel of working orders |
| "Price my positions / P&L" | `unwind_quotes.py [SYMBOL …]` | Live quotes, unwind cost, P&L vs entry; filter by symbol |
| "IV / vol surface" | `vol_surface.py SYMBOL puts\|calls [--save]` | ATM term structure + smile |
| "Scan a collar / roll / spread" | `combo_scan.py …` | N-leg combo scanner (see CLAUDE.md for syntax) |

Scope to a symbol whenever one is named (e.g. `unwind_quotes.py CRWV`). If the ask is just positions/P&L and they haven't asked to pull, skip extraction — derive from the existing ledger and live API.

**Verifying fills:** confirm via `check` (live positions diff) or `sync`, **not** the working-orders list — that endpoint lags intraday and goes blank after ~4pm ET, so absence from it doesn't mean filled (could be filled, rejected, or canceled). For a single order's true disposition, query it by ID. See memory `project_schwab_orders_afterhours`.

## Auth

- `token.json` and `config.json` live in the repo root. Only accounts with a `hash` field in `config.json` are queried.
- Access token auto-refreshes (30-min TTL). The **refresh token expires after 7 days** and needs a manual browser re-login.
- Re-auth **opens a browser even with `interactive=False`** on an expired refresh token — don't kill the callback-server process mid-flow, and don't loop on auth failures. If a script dies on auth, the refresh token has likely expired; ask Matt to re-run an interactive auth. See memory `project_schwab_reauth_browser`.

## Account context

Don't hardcode account roles here — they change (and have, twice). Current roles, margin treatment, and which account to route trading to live in memory **`project_account_roles_margin`** — read that memory's latest dated update, not its history (stable identities only: 043 = primary trading; 299 = Focus OIO/Premium Income; 906 = former Mariner). Margin capacity is volatile (release rates have swung 20%↔60% without notice): before sizing any new short-option order, verify via `preview_order`'s `projectedAvailableFund` rather than trusting remembered state. Historical inter-account journaling path: 906 → 299 → 043 (Mariner exit); journal entries carry their origin account in the ledger.

## Meeting prep

When Matt's prepping for an advisor meeting (JPM, Clay/Schwab, Focus), produce a presentation-ready snapshot rather than raw CLI dumps: run `check` + `collars` + `unwind_quotes.py`, then synthesize current positions, mark-to-market, and realized/open P&L into a clean summary.

## Wash-sale / RGL enrichment (the only workflow not in code)

Wash-sale data isn't in the API — it's on Schwab's Realized Gain/Loss page. When it's needed, use Playwright:

1. Navigate to `https://client.schwab.com/app/accounts/RGL`
2. Extract per-lot: symbol, description, qty, proceeds, cost basis, adjusted cost basis, gain/loss, wash-sale disallowed, settlement date
3. Match to existing ledger close/sell records by `(symbol, expiry, strike, qty, settlement date)`
4. Add `cost_basis`, `adjusted_cost_basis`, `wash_sale_disallowed` fields — additive only; never create records

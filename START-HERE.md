# START HERE — AI Day Trading Bot

**Read this first. If you are Claude Code: this file is your orientation and troubleshooting brief for this project.**

---

## What this is

An autonomous day-trading system. Claude Code reads a skill called `robinhood-day-trader` that contains all the trading rules, and places real orders through Robinhood's official Agentic MCP into a separate $100 account. It logs every trade and learns from the log.

Owner: Carlos. Built August 2026.

---

## Current state (updated 2026-08-27)

| | |
|---|---|
| Robinhood connector | ✅ Connected and verified against the live account |
| Agentic account | ✅ Exists — nickname "Agentic", ends 1910, type `limited_margin` |
| Account balance | ✅ **$100.00 funded** (deposit pending as of 2026-08-26 evening; buying power already $100.00) |
| Skill installed | ✅ Synced to Claude's skill library — loads by name in any session |
| Files in place | ✅ `trade_ledger.md`, `learning.md`, `START-HERE.md` in this repo |
| Phase 1 scanner | ✅ Saved on the account — scan_id in `learning.md` platform notes. Reuse it, don't recreate it. |
| Market-hours verification | ⬜ Spread/L2 check and RVOL normalization check still need one pass during regular hours |
| First live session | ⬜ Ready once the deposit settles and the market-hours check passes |

---

## Where things live

| Thing | Location |
|---|---|
| The rulebook (skill) | Claude's synced skill library: `robinhood-day-trader` (SKILL.md + 3 reference files + 2 templates) |
| `trade_ledger.md` | Repo root. Permanent trade database. Do not rename. |
| `learning.md` | Repo root. Performance stats + lessons + platform notes. Do not rename. |
| `START-HERE.md` (this file) | Repo root. Orientation and troubleshooting. |
| Setup guide | `docs/SETUP-GUIDE.txt` — reference only, full 10-step walkthrough. |

**This repo (`carlosvibes/Robinhood`) is the system's permanent home.** If a session runs in a remote/cloud environment, that container is ephemeral — **commit and push the ledger and learning file at the end of every session or the day's records are lost.** On a desktop `~/trading` folder, push weekly at minimum.

---

## Copy-paste prompts for Claude Code

**Verification (do this before any trading):**

```
Load the robinhood-day-trader skill. Do NOT place any orders. Systems check only:
1. Run get_accounts, confirm you can see the agentic account, report last 4 digits and type
2. Report equity and buying power
3. Pull a live SPY quote with exact price and timestamp
4. Pull the 9-period EMA and VWAP on 5-minute bars for SPY
5. Run the saved "Phase 1 Day Trader Universe" scan (scan_id in learning.md).
   Report how many names come back and list up to 5
6. Pull the Level 2 book for one of them and report the spread
Confirm the platform notes table in learning.md matches what you see.
```

**A trading session:**

```
Load the robinhood-day-trader skill and run a full trading session for today.
Phase 1 rules. Stay active until the flatten time.
```

**If something is broken:**

```
Read START-HERE.md, then load the robinhood-day-trader skill and read SKILL.md
and all files in references/. Here is what went wrong: [describe it].
Diagnose it against the rules and the troubleshooting table, and tell me what
to fix. Do not place any orders.
```

**To read your own logs as a PDF:**

```
Export my trade_ledger.md and learning.md as a formatted PDF I can read.
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent says buying power is $0 and stops | Account not funded, or deposit reversed | Check the **agentic** account in the Robinhood app, not the main one |
| Scanner returns zero names | Missing `interval` on a filter, or a percentage passed as a whole number (`5` means 500%, use `0.05`). Or the relative-volume filter isn't normalizing for time of day. | Check the filter table in SKILL.md. **Report this one back to Carlos** — an empty scan looks exactly like "no setups today," which is the most misleading failure here. |
| OAuth flow dead-ends at a localhost error page | Normal | Copy the full URL from the browser address bar and paste it back into Claude Code |
| Agent won't trade, says HALTED | A loss limit fired | Check the `STATUS:` line at the top of `trade_ledger.md`. `HALTED_DAY` clears itself next morning. `HALTED_WEEK` only Carlos can clear, by editing that line. |
| Agent refuses a setup that looks good | Usually the 2:1 reward-to-risk gate, the ATR rule, or the VWAP filter | Working as designed. It should say which rule blocked it and log the conflict. |
| Stop order rejected | Probably a fractional quantity | Stops don't work on fractional shares. Whole shares only. Position should have been closed immediately. |
| Order response timed out | Transport failure | Retry **once with the same `ref_id`**. Never with a new one. Anything else: reconcile against the broker. |
| Ledger and broker disagree on positions | Partial fill, or a figure logged from intent instead of the actual fill | **Broker is always right.** Reconcile to it, then investigate how the ledger drifted. |
| No trades for several days | Filters are tight by design | Check the scanner is returning names at all. Zero trades is often correct; zero *candidates* is a bug. |

---

## Verified platform facts

Tested directly against the live account, 2026-08-26. Don't re-derive these.

- **Bars:** 15sec–4hour, OHLCV with volume. The 1-minute bar is named `minute`, not `1minute`.
- **Indicators:** server-side — ema, sma, rsi, macd, **vwap**, atr, bollinger, adx, williams_r, and more. FMP is not needed for these.
- **Order types:** market, limit, **stop_market**, stop_limit.
- **Stops do NOT work on fractional shares.** Fractional is market-orders-only. This is why the system trades whole shares in a $5–$10 price band.
- **No OCO/bracket orders.** A resting stop holds the shares, so cancel it and verify the cancellation *before* submitting any exit.
- **Level 2 depth is available** via `get_equity_price_book` (empty outside regular hours — that's normal, not an error).
- **`ref_id`** is an idempotency key on orders. Every order gets its own.
- **`review_equity_order`** simulates without placing and returns PDT and halt alerts. Call it before every real order.
- **Account is `limited_margin`** — no settlement delay, but it *can* short. A duplicate sell will open a short position rather than being rejected.
- **The saved scanner works** — created 2026-08-26 with the exact SKILL.md filter set, returned 7 names on first run. scan_id is in `learning.md`.

---

## The rules, in brief

Phase 1: $10 max position, $0.25 max risk per trade, whole shares, $5–$10 stocks, minimum 2:1 reward-to-risk, long only above VWAP, stop at least 1× ATR away.

Halts at: 3 consecutive losses (~$0.75), $1.00 daily, $2.50 weekly, or 6 opening orders.

Flat 10 minutes before the close, every day, no exceptions.

The agent may propose strategy changes but **may not change risk rules**. A proposal becomes active only when Carlos adds a dated row to the changelog in `learning.md`.

---

## Honest expectation

One-share system. Winners ≈ +$0.50, losers ≈ −$0.25, a good day ≈ +$0.60 net. A strong month is single-digit dollars.

The point is the system and the trade log, not the income. After 50 trades the log says whether there's an edge. If yes, adding capital is evidence-based. If no, that cost $20 to find out instead of $2,000.
